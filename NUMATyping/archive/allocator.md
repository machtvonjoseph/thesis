# archive/allocator.md — the UMF/jemalloc allocator, across three versions

*Scope: the NUMA-bound allocator only — what changed between the three versions
of the jemalloc pool, why, and what each change measured. Benchmark-level
results and their interpretation are deliberately **not** here; they live in
`idea.md` §3 and in the campaign manifests under `Campaigns/`.*

Rework dated 2026-07-20, merged 2026-08 (commit `a45515bd`, from the
`NUMATyping-umfAlloc` fork). Measurements on `stormbreaker`: 4 NUMA nodes (0/1
have CPUs, 2/3 are memory-only CXL), 80 threads, jemalloc 5.2.1, THP `madvise`.
Not tracked by this repository.

---

## The three versions

| | what it is | tcache strategy | arenas |
|---|---|---|---|
| **v0** | upstream `oneapi-src/unified-memory-framework` v0.9.0 | `MALLOCX_TCACHE_NONE` — none | default |
| **v1** | the fork ("advisor's UMF") | explicit tcaches, **all created up front on the main thread** | 160/pool, rotated per allocation |
| **v2** | current | explicit tcaches, **created lazily in the owning thread** | one fixed arena per thread; count = online CPUs |

Everything below is the story of how v0 → v1 → v2 happened and what each step
cost or bought.

---

## v0 — upstream

Upstream's jemalloc pool passes `MALLOCX_TCACHE_NONE` on every allocation, so
each one takes an arena bin lock.

```
42,305 ns/alloc at 20 threads
```

That is not a typo. **Do not go back to it.** Allocation without a thread cache
does not scale past a couple of threads, and this is the single reason the fork
existed in the first place.

v0 also routes every call through the `umf_memory_pool_ops_t` vtable, so there is
an indirect call per allocation on top of the lock.

---

## v1 — the fork

Three changes against v0, all confined to the jemalloc pool:

1. **Per-thread tcaches** instead of `TCACHE_NONE`. The important one, and it
   survives into v2.
2. **160 arenas per pool**, with the arena rotated on every allocation.
3. **`umfFastJemallocMalloc` / `umfFastJemallocFree`** — inline fast paths in the
   public header that skip the vtable.

### The bug: tcaches bound to the wrong arena

`mallctl("tcache.create")` binds the new tcache to **whatever arena the calling
thread is currently using**, permanently. There is no way to rebind it later.

v1's `op_initialize()` created all `MAX_JEMALLOC_THREADS` tcaches up front, in a
loop, **on the main thread**. So every worker thread's tcache was bound to the
*main thread's* arena, and every `free()` from every worker flushed into that one
arena's bins.

Measured, 32 B objects, steady state, pinned threads:

| threads | alloc | free |
|---|---|---|
| 1 | 32.0 ns | 25.8 ns |
| 8 | 30.8 ns | 38.9 ns |
| 20 | 30.8 ns | **377.4 ns** |

Alloc is flat; free explodes. **This is invisible in any single-threaded
microbenchmark**, which is why it survived so long.

### Three more defects in v1

- **`op_finalize` leaked 159 of 160 arenas per pool.** It destroyed `arena_index`
  `num_arenas` times instead of `arena_index + i`.
- **`MAX_JEMALLOC_THREADS` was a hard 200-thread ceiling with no bounds check** —
  thread 200 walked off the end of `tcaches[]`.
- **It was 200 in the built library and 256 in the header copies** under
  `numaLib/` and `Array*/`, i.e. **two different `sizeof(jemalloc_memory_pool_t)`
  on either side of the ABI.**
- **160 arenas** bought nothing: `tid()` already seeded a distinct starting arena
  per thread. The rotation cost ~9 ns/alloc by scattering tcache refills across
  cold arenas, and 160 arenas per pool is pure RSS and page-table bloat.

---

## v2 — current

### `unified-memory-framework/src/pool/pool_jemalloc.c` + `include/umf/pools/pool_jemalloc.h`

- **Tcaches are created lazily, per thread** (`umfJemallocBindThread`), bound to
  that thread's own arena — so the tcache and the arena match and each thread
  flushes into its own bins. Replaces the `tcaches[MAX_JEMALLOC_THREADS]` array
  that lived in `jemalloc_memory_pool_t`.

  > **Trap:** `-ljemalloc` interposes the global `malloc`, so `thread.arena` also
  > governs ordinary non-numa-typed allocations in that thread. Set it and walk
  > away and plain `malloc()` silently starts returning node-bound memory. v2
  > saves it and puts it back once the tcache has taken its snapshot.

- **A `pthread_key` destructor** (`destroy_thread_tcaches`) reclaims them on
  thread exit. Explicit tcaches are not garbage collected by jemalloc, so without
  this a thread-churning process leaks one tcache per (thread, pool).
- **No arena rotation.** One fixed arena per thread,
  `arena_index + (tid % num_arenas)`. (~5 ns of the win.)
- **Flags cached in TLS.** `umf_je_alloc_flags[slot]` / `umf_je_free_flags[slot]`
  are computed once per (thread, pool). The fast path is a TLS bit test plus
  `mallocx()`. `umfFastJemallocMallocSlot(slot, size)` takes the slot directly;
  for `numa<T,k>` the slot is `k`, a compile-time constant, so the whole thing
  folds to `mallocx(size, <constant-indexed TLS word>)`.
- **`num_arenas` defaults to the online CPU count** (was hardcoded 160),
  overridable with `UMF_JE_ARENAS_PER_POOL`.
- **`op_finalize` fixed** — no more leaked arenas.
- **`MAX_JEMALLOC_THREADS` is gone.** No thread limit at all; `UMF_JE_MAX_POOLS`
  (16) caps NUMA nodes instead, with a `static_assert`.

### `numaLib/umf_numa_allocator.hpp` (the pool front-end)

- **`umf_alloc` / `umf_free` are now `inline`.** They were ordinary out-of-line
  definitions in a header — a call per allocation, and no constant folding.
- They call the `*Slot` fast path directly, skipping the
  `hPool->pool_priv->pool_slot` chase.
- **`jemalloc_pool` / `NUMA_HANDLES` are `inline` globals** and `umf_alloc_init()`
  is guarded, so multiple TUs including this header share one set of pools
  instead of racing to create duplicates. Pool slot `k` must mean logical node
  `k`; duplicate pools break that identity. The `__attribute__((constructor))`
  moved to an anonymous-namespace shim so pools are still up before `main()`.

---

## Measurements across the three versions

Allocator microbenchmark, 32 B objects, steady state, ns per operation:

| threads | version | alloc | free | pair |
|---|---|---|---|---|
| 1 | v1 | 32.0 | 25.8 | 57.8 |
| | **v2** | **22.2** | **20.4** | **42.6** |
| | plain jemalloc `malloc` | 16.7 | 16.7 | 33.4 |
| 20 | v1 | 30.8 | 377.4 | 408.2 |
| | **v2** | **22.7** | **20.4** | **43.1** |
| | plain jemalloc `malloc` | 16.4 | 16.6 | 33.1 |

(v0 does not appear in this table because at 42,305 ns/alloc it is off the scale;
it is a different order of magnitude, not a different column.)

v2 is flat across thread counts and within 1.3x of a plain `malloc`/`free` pair.

### Against `numa_alloc_onnode`

`numa_alloc_onnode` is the libnuma call in the `#else` (non-UMF) branch of the
generated `numa<T,k>::operator new` — the naive way to get node-local memory. It
`mmap`s + `mbind`s + `munmap`s every object, so it is slow, does not scale (the
per-process `mmap_lock` serialises threads), and wastes a whole 4 KB page per
small object.

Full alloc + first-touch + free cycle, 32 B objects, node 0, from
`allocator_test/bin/throughput_compare` (ns/op, then speedup):

| threads | `numa_alloc_onnode` | v1 | v2 | v1 vs numa | **v2 vs numa** |
|--:|--:|--:|--:|--:|--:|
| 1 | 5,394 | 42 | 26 | 127x | **205x** |
| 8 | 88,257 | 50 | 33 | 1,758x | **2,683x** |
| 20 | 366,142 | 345 | 45 | 1,061x | **8,191x** |

The v1-vs-v2 gap at 20 threads (1,061x vs 8,191x) is entirely the free-path fix:
v1's free stalls at ~318 ns/op there, v2 holds ~45 ns.

> This table answers "was building a real pooled allocator worth it versus the
> naive libnuma call" — emphatically yes. It is **not** the benchmark baseline:
> in the `UMF=1` benchmarks, `regular` data is interposed jemalloc `malloc`,
> which is fast. Do not quote these ratios as the system's speedup.

---

## Alternatives tried and rejected

- **jemalloc's automatic tcache** (`dallocx(ptr, 0)`) — faster still
  (20.5 / 18.2 ns) and **wrong**. jemalloc's tcache bins are per size class, not
  per arena, so a node-1 request gets handed whatever the bin happens to hold.
  Measured: **200 of 4096 node-1 allocations landed on node 0.** The entire
  approach depends on that not happening. There is a regression test for it.
- **A TLS magazine** (batch freelist) layered on top — **slower** (24.6 vs
  21.7 ns). jemalloc's tcache already is one.

---

## Regression testing

```shell
cd allocator_test && make UMF=1 ROOT_DIR=$HOME/NUMATyping
numactl --cpunodebind=0,1 --membind=0,1 ./bin/verify_allocator      # exits non-zero on failure
numactl --cpunodebind=0,1 --membind=0,1 ./bin/throughput_compare    # the speedup sweep above
```

- **`bin/verify_allocator`** — ns/op vs thread count (catches a free-path
  regression) plus three NUMA placement checks: single-thread interleaved,
  plain-`malloc`-still-unbound, 8-thread interleaved. **Run it after any change
  to the jemalloc pool.** The placement checks are the real regression test for
  the tcache isolation property — keep them green.
- **`bin/throughput_compare`** — the UMF-vs-`numa_alloc_onnode` sweep at 1/8/20
  threads.

Rebuild after any change here:

```shell
cd unified-memory-framework/build && make -j16
cd ../../Output/DataStructureTests && make UMF=1 ROOT_DIR=$HOME/NUMATyping
```

---

## Two hazards that outlive the rework

**The static libraries are build artifacts.** A source merge alone does *not*
update an existing `build/lib/*.a`. Campaigns 01–03 unknowingly ran the v1
allocator this way; see the corrected provenance notes in
`Campaigns/ycsb/campaign0*/manifest.md`. Verify the binary actually carries the
new code:

```shell
nm Output/ycsb/bin/ycsb | grep BindThread
```

**The header copies drift.** `umf_numa_allocator.hpp`, `numa_nodemap.hpp` and
`pool_jemalloc.h` are duplicated across ten suite directories, and `-Iinclude/`
comes before `-InumaLib/`, so a local copy silently shadows the canonical one.
They had already drifted once (`DataStructureTests` still had a pre-`numa_nodemap`
copy with a hardcoded identity node mapping; `MAX_JEMALLOC_THREADS` was 200 in
one place and 256 in another). All were synced during the rework — including
`ycsb/include/`, which the fork itself had missed. **This will drift again.** The
real fix is to drop the per-suite copies and rely on `-InumaLib`.

---

## Allocator work still open

1. **Huge pages.** Nothing gets THP today — the box is `madvise` and UMF's OS
   provider never calls `MADV_HUGEPAGE`, so allocations are 100% 4 KB pages. A
   `madvise(addr, size, MADV_HUGEPAGE)` after the `mbind` in `os_alloc()`
   (`src/provider/provider_os_memory.c`) should help pointer-chasing workloads.
   Needs its own measurement pass: the mbind/THP interaction is worth checking,
   and extents must be 2 MB aligned and sized for it to apply at all.
2. **`-flto`** on the benchmark builds, so the now-`inline` `umf_alloc` folds
   into the generated `operator new`.
3. **The remaining ~30% gap to plain jemalloc** (43 ns vs 33 ns per pair) is the
   cost of binding. Whether any of it is recoverable has not been investigated.
