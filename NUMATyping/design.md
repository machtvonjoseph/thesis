# design.md — NUMATyping: system design

*How the system is built: the layers, what each one guarantees, where they meet,
and which seams are load-bearing. Companion to [`idea.md`](idea.md), which
argues why. Neither file is tracked by this repository.*

Written 2026-08-29 against commit `0d134040`. Reference machine `stormbreaker`;
target machine Perlmutter (NERSC).

---

## 1. Overview

Five layers, each usable and testable on its own:

```
  user code:  numa<BST,0>* t = new numa<BST,0>();  thread_numa<0> w(...);
       │
       ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ 1. numaLib/        header library: numa<T,k>, thread_numa<k>,│
  │                    NumaAllocator, logical→physical node map  │
  ├─────────────────────────────────────────────────────────────┤
  │ 2. numa-clang-tool/  two LibTooling passes                   │
  │      recurse → generate numa<T,k> specializations, recursing │
  │                into every member type                        │
  │      cast    → rewrite in-method `new`, insert the casts     │
  │                                        ↓ emits pure C++20    │
  ├─────────────────────────────────────────────────────────────┤
  │ 3. unified-memory-framework/  UMF + jemalloc: one MPOL_BIND  │
  │                    pool per node, per-thread tcache/arena    │
  ├─────────────────────────────────────────────────────────────┤
  │ 4. benchmarks/     ycsb, DataStructureTests{,_four},         │
  │                    Histogram, Array*, allocator_test         │
  ├─────────────────────────────────────────────────────────────┤
  │ 5. scripts/        numafy → campaign → analysis, with        │
  │                    git-gated provenance                      │
  └─────────────────────────────────────────────────────────────┘
```

The compile pipeline for one suite:

```
<SUITE>/  ──copy──►  numa-clang-tool/input/<SUITE>/
                          │  pass "recurse"
                          ▼
                     numa-clang-tool/output/<SUITE>/
                          │  pass "cast"
                          ▼
                     numa-clang-tool/output2/<SUITE>/
                          │  copy
                          ▼
                     Output/<SUITE>/   ──make UMF=1──►  bin/<binary>
```

`Output/` is gitignored. It is a build artifact and must be regenerated per
machine and after any change to `numaLib/`, the tool, or the suite source.

---

## 2. Layer 1 — `numaLib/`

Four headers. Everything else in the system either generates code that uses
them, or is a backend they call.

### 2.1 `numatype.hpp` — the type

Declares the primary template and **two partial specializations**, chosen by
`enable_if` on the wrapped type:

```cpp
template<typename T, int NodeID,
         template<typename,int> class Alloc = NumaAllocator,
         typename E = void>
class numa;                                    // primary: declared, not defined
```

**Primitives and pointers** (`is_fundamental || is_pointer`) get a *by-value box*:
a `T contents` member, `load()`/`store()`, an implicit `operator T&` so it
behaves like a `T` in expressions, `operator->` guarded by a `static_assert` for
pointer types only, and node-bound `operator new`/`new[]`. This is the leaf of
the recursion — `numa<BinaryNode*, 0>` is a pointer stored on node 0.

**Class types** get `class numa<T,NodeID,...> : public T`. This is the
**pre-transform parse shim**, and it exists for one reason: Clang can only parse
valid C++, so the *unexpanded* program must compile. Deriving from `T` makes
`numa<T,k>` behave statically like a `T` (member access, method calls, implicit
conversion) so the original source type-checks before the tool has run. The
`recurse` pass replaces it with a real specialization; in the transformed output
the empty `template<typename T, int N> struct numa {};` form is what remains.

`NumaAllocator<T,NodeID>` is a standard-shaped allocator whose `allocate`
dispatches on `#ifdef UMF`:

```cpp
UMF   →  umf_alloc(NodeID, n*sizeof(T), alignof(T))    // pooled, fast
else  →  numa_alloc_onnode(n*sizeof(T), NodeID)        // libnuma, slow
```

The non-UMF branch is the naive path the project measures against: it
`mmap`+`mbind`+`munmap`s per object, serializes on the per-process `mmap_lock`,
and wastes a 4 KB page per small object. Measured at 20 threads it is
**8,191x slower** than the UMF path. It is kept as a correctness fallback and as
the "what you would otherwise do" comparison point — **not** as the benchmark
baseline (see §6.2: `regular` data uses interposed jemalloc `malloc`, which is
fast).

`get_numa_node_id(void*)` wraps `move_pages(2)` and is the placement oracle used
by the regression tests.

### 2.2 `numathreads.hpp` — the thread

`thread_numa<NodeID> : std::thread`. The constructor forwards to `std::thread`
and then pins:

1. Resolve `NodeID` (logical) → physical via `numa_node_map()`.
2. `numa_node_to_cpus(phys, …)` → libnuma bitmask → `cpu_set_t` sized from
   `sysconf(_SC_NPROCESSORS_CONF)` (not a compile-time `MAX_CPUS`).
3. **Intersect with the process's allowed set** (`sched_getaffinity`) — without
   this, a run inside a cgroup/cpuset gets `EPERM`. This matters on Perlmutter,
   where jobs are cpuset-confined.
4. Throw if the mask comes out empty; otherwise `pthread_setaffinity_np`.

Masks are cached per node in a `static inline std::map`.

Copy is deleted, move is supported. Note the pin happens *after* the thread is
already running, so a few instructions may execute unpinned — irrelevant at the
timescales measured, but real.

### 2.3 `numa_nodemap.hpp` — logical → physical

The single most important portability seam. **Both** the allocator and the
thread pinner call it, which is what makes "data and threads never drift apart"
structurally true rather than a convention.

```
1. NUMA_NODE_ORDER="0,7"     → use it verbatim          (set by machine.env)
2. else auto-detect CPU-bearing physical nodes, ascending
     (numa_node_to_cpus weight > 0 → skips memory-only / CXL tiers)
3. reorder outside-in: [a,b,…,y,z] → [a,z,b,y,…]
4. numa_node_map(k) = order[k % order.size()]
```

Consequences worth internalizing:

| machine | CPU nodes | order | logical 0,1 → | logical 0..3 → |
|---|---|---|---|---|
| stormbreaker | 0,1 (2,3 are CXL, no CPUs) | `0,1` | 0,1 | 0,1,0,1 |
| Perlmutter (NPS4) | 0..7 | `0,7` | 0,7 | 0,7,0,7 |

Outside-in is deliberate: consecutive logical partitions land on the
**farthest-apart** physical nodes, so the locality contrast means the same thing
on a 2-node box and an 8-node one. On Perlmutter that is distance 32,
cross-socket, 3.2x remote.

Built once via C++11 function-local static, so it is thread-safe and costs one
branch after the first call.

### 2.4 `umf_numa_allocator.hpp` — the pool front-end

Creates, before `main()`, one UMF memory provider + one jemalloc pool per
logical node:

```
for i in 0..NUM_NODES-1:
    phys = numa_node_map(i)
    memspace ← umfMemspaceCreateFromNumaArray(&phys, 1)
    policy   ← umfMempolicyCreate(UMF_MEMPOLICY_BIND)      // MPOL_BIND, hard
    provider ← umfMemoryProviderCreateFromMemspace(...)
    pool     ← umfPoolCreate(umfJemallocPoolOps(), provider, DISABLE_TRACKING)
```

Because pools are created in logical order, **pool slot `k` == logical node
`k`**, which is what lets the fast path take the slot directly:

```cpp
inline void* umf_alloc(unsigned NodeId, size_t size, size_t)
    { return umfFastJemallocMallocSlot(NodeId, size); }
```

For a generated `numa<T,k>::operator new`, `k` is a compile-time constant, so
this collapses to a TLS bit test plus `mallocx()` with a constant flag word.

Three details that were bugs and are now load-bearing invariants:

- `umf_alloc`/`umf_free` are **`inline`** — they were out-of-line definitions in
  a header, costing a call per allocation and blocking constant folding.
- `jemalloc_pool`/`NUMA_HANDLES` are **`inline` globals** and `umf_alloc_init()`
  is guarded, so multiple TUs including the header share one set of pools rather
  than racing to create duplicates (which would break the slot==node identity).
- `NUMA_NODE_NUM` is `static_assert`ed against `UMF_JE_MAX_POOLS` (16).

> **Hazard.** `NUMA_NODE_NUM` defaults to **2**. A suite with four logical
> partitions and a stale local copy of this header registers only pools 0 and 1;
> partitions 2 and 3 then reach `umfJemallocBindThread()` with a null pool —
> an assert in a debug build, and under `-DNDEBUG` **a silent `mallocx()` with
> uninitialised flags**, i.e. half the partitions allocating unbound while still
> producing plausible throughput numbers. This is exactly what `0d134040` fixed
> in `DataStructureTests_four`. Placement failures here are silent; check with
> `allocator_test/bin/verify_allocator`, never by eyeballing throughput.

---

## 3. Layer 2 — `numa-clang-tool/`

A Clang LibTooling binary, `build/bin/clang-tool`, selecting a pass with
`--pass=recurse|cast`. ~2,000 lines across `actions/` (FrontendActions),
`consumer/` (ASTConsumers), `transformer/` (the actual work), `casting/`,
`numafy/`, `utils/`.

Structure per pass: `FrontendAction` → `ASTConsumer` →
`Transformer::start()` → AST matchers → `Rewriter` edits →
`WriteOutput()` walks the SourceManager's file list and writes each rewritten
buffer, mapping the path `…/input/…` → `…/output/…`.

### 3.1 Pass 1: `recurse` — introspective expansion

`RecursiveNumaTyper.cc` (812 lines, the core of the system).

**Discovery.** Two AST matchers find the triggering sites:

```cpp
cxxNewExpr(hasType(pointsTo(qualType(hasDeclaration(
             namedDecl(matchesName("^::?numa")))))))
varDecl (hasType(pointsTo(qualType(hasDeclaration(
             namedDecl(matchesName("^::?numa")))))))
```

both excluding system headers, `numatype.hpp`, `numathreads.hpp`, and `*/umf/*`
— the library must not rewrite itself.

**Expansion.** For each matched `numa<T,k>`, extract `T` (a `CXXRecordDecl`) and
`k` (an integral template argument) from the `ClassTemplateSpecializationDecl`,
then emit `template<> class numa<T,k> { … }` into the header where `T` is
defined, with:

- each field `U f` retyped to `numa<U,k> f`;
- for each field type `U` that is itself a class, **recurse** — this is the
  introspection C++ cannot do, and it is what makes `numa<BST,k>` imply
  `numa<BinaryNode,k>`;
- the method bodies copied over;
- `utils::getNumaAllocatorCode()` appended: node-bound
  `operator new`/`new[]`/`delete`/`delete[]`, each `#ifdef UMF`-switched between
  `umf_alloc(k, sz, alignof(T))` and `numa_alloc_onnode(sz, k)`.

> `operator new[]` **must** use `sz`, not `sizeof(T)`. It previously allocated
> room for exactly one element regardless of the requested count — a heap
> overflow on any array-new of a numa-typed class. Fixed; already-generated
> headers under `Output/` were patched, but the correct action after any tool
> change is to re-run `numafy.py`.

The `allocator_funcs` string supplies the primitive/pointer box's conversion
operators, `operator->`, indexing, and a `constexpr operator int()` returning
`NodeID`.

### 3.2 Pass 2: `cast` — allocation rewriting and coercion

Three cooperating transformers:

- **`CastNumaAlloc.cc`** — the dynamic-allocation heuristic. It walks every
  generated `numa<T,k>` specialization's methods, finds `CXXNewExpr`s in the
  bodies, and rewrites them to allocate on `k` and `reinterpret_cast` back to the
  unqualified type. This is what puts a tree's *nodes* on the right node, not
  just its root:

  ```cpp
  Node* n = reinterpret_cast<Node*>(new numa<Node,0>());
  ```

- **`reinterprete_cast.cc`** — inserts the coercions between `numa<T,k>*` and
  `T*` at the sites the expansion broke. Sound only because expansion preserves
  field order and offsets, so a member has the same offset and name in both
  types. (The vtable-slot half of that argument is asserted, not proved — see
  `idea.md` §4.4.)

- **`NumaTargetNumaPointer.cc`** — handles assignments where a member expression
  on one side is `numa`-typed and the other is not, so pointer fields inside a
  specialization stay consistent.

`inclusiondirective.cc` hooks `InclusionDirective` to track which headers a TU
pulled in, so the writer knows which files it may rewrite.

### 3.3 Building the tool

```shell
cmake -S numa-clang-tool -B numa-clang-tool/build   # -DHELP=ON for guidance
cmake --build numa-clang-tool/build -j
```

Needs LLVM/Clang **≥ 20** dev libraries. `build/` is gitignored and
version-sensitive: rebuild per machine.

> **The `libLLVM` shape problem** (fixed in `adae5a5c` / `473ce150`). Two valid
> installation shapes exist and the CMake must handle both:
>
> - **(a)** `libclang-cpp.so` has a `DT_NEEDED` on `libLLVM.so` → link both.
> - **(b)** `libclang-cpp.so` embeds LLVM statically → link **only**
>   `libclang-cpp`; linking `libLLVM` too causes a duplicate-registration abort
>   at exit (`munmap_chunk`).
>
> Perlmutter (PrgEnv `llvm/21.1.4`) is shape **(b)** — and confusingly
> `libLLVM-21.so` *is* present in the libdir. Presence in the libdir and being
> depended upon are independent; conflating them is what hid the bug. Detect with
> `readelf -d libclang-cpp.so | grep -c 'NEEDED.*libLLVM'`. The `libLLVM` lookup
> must be **optional** and must come **after** detection.

---

## 4. Layer 3 — the allocator

`unified-memory-framework/` is a fork of Intel's UMF v0.9.0 with local changes
confined to the jemalloc pool. Full rationale and measurements:
[`archive/allocator.md`](archive/allocator.md).

### 4.1 Shape

```
numa<T,k>::operator new
   → umf_alloc(k, sz, align)            [inline, numaLib]
   → umfFastJemallocMallocSlot(k, sz)   [inline, pool_jemalloc.h]
   → mallocx(sz, umf_je_alloc_flags[k]) [TLS-cached flags]
   → jemalloc arena bound to a provider that MPOL_BINDs node phys(k)
```

### 4.2 The design rules, and why each exists

| Rule | Why |
|---|---|
| **Per-thread tcaches, never `MALLOCX_TCACHE_NONE`** | Upstream's setting costs **42,305 ns/alloc at 20 threads** — every allocation takes an arena bin lock |
| **Create each tcache lazily, in the owning thread** (`umfJemallocBindThread`) | `mallctl("tcache.create")` binds permanently to the *calling* thread's arena. Creating them up front on the main thread made every worker's `free()` flush into one arena: 26 ns → **377 ns** at 20 threads |
| **Save and restore `thread.arena`** around tcache creation | `-ljemalloc` interposes global `malloc`, so leaving it set makes ordinary non-numa allocations silently node-bound |
| **Never use jemalloc's automatic tcache** (`dallocx(ptr,0)`) | Faster (20.5 ns) and **wrong**: tcache bins are per size class, not per arena — measured **200 of 4096 node-1 allocations landed on node 0**. The entire result depends on this not happening |
| **One fixed arena per thread**, no rotation | `tid()` already seeds a distinct arena; rotation cost ~9 ns/alloc in cold-arena refills |
| **Flags cached in TLS**, `*Slot` entry point | For `numa<T,k>` the slot is a compile-time constant |
| **`pthread_key` destructor** reclaims tcaches | Explicit tcaches are not GC'd by jemalloc; a thread-churning process would leak one per (thread, pool) |
| **`num_arenas` = online CPU count** | Was hardcoded 160 — pure RSS and page-table bloat |
| **No `MAX_JEMALLOC_THREADS`** | Was a 200-thread ceiling with no bounds check, and *200* in the built library vs *256* in header copies — two different `sizeof(jemalloc_memory_pool_t)` across the ABI |

Also rejected after measurement: a TLS magazine layered on top (24.6 vs 21.7 ns —
jemalloc's tcache already is one).

### 4.3 Results

| metric | before | after |
|---|---|---|
| alloc+free pair, 20 threads | 408 ns | **43 ns** (flat 1→20) |
| free, 20 threads | 377 ns | **20 ns** |
| BST `numa/numa` vs `numa/regular` | −11.3% | −2.5% |
| node-1 allocations on the wrong node | — | **0 / 4096** |
| vs `numa_alloc_onnode`, 20 threads | 1,061x | **8,191x** |

### 4.4 Regression gate

```shell
cd allocator_test && make UMF=1 ROOT_DIR=$PWD/..
numactl --cpunodebind=0,1 --membind=0,1 ./bin/verify_allocator     # exits non-zero on failure
numactl --cpunodebind=0,1 --membind=0,1 ./bin/throughput_compare   # UMF vs numa_alloc_onnode
```

`verify_allocator` checks both properties that matter: **free-path flatness vs
thread count**, and **placement correctness** (single-thread interleaved,
plain-`malloc`-still-unbound, 8-thread interleaved). Run it after any change to
the jemalloc pool. The placement checks are the real regression test for tcache
isolation — keep them green.

> **The build-artifact trap.** `unified-memory-framework/build/lib/*.a` are build
> artifacts. A source merge does **not** update an existing archive. Campaigns
> 01–03 unknowingly ran a months-old allocator this way. Always rebuild, and
> verify the binary actually carries the new code:
> `nm Output/ycsb/bin/ycsb | grep BindThread`.

---

## 5. Layer 4 — benchmarks

All suites share the same CLI shape: `--th_config={numa,regular}`,
`--DS_config={numa,regular}`, and a `Makefile` taking `UMF=1` and `ROOT_DIR`.
Runs are wrapped in `numactl $NUMACTL_BIND`. The 2×2 configuration matrix is the
experiment:

| th / ds | meaning |
|---|---|
| `numa` / `numa` | the intended use: pinned threads, node-typed data |
| `numa` / `regular` | pinned threads, ordinary heap — **the honest baseline** (gets first-touch locality free) |
| `regular` / `numa` | typed data, unpinned threads — the anti-pattern; documented as "do not do this" |
| `regular` / `regular` | NUMA-unaware baseline |

### 5.1 `DataStructureTests/` — transactional BST (bench key `DS`)

A large array of `BinarySearchTree`s, one lock per index, split across two
partitions. Threads run **80% local lookups, 20% transactions**; transactions use
two-phase locking and transfer values between keys, so `txn%4` cases deliberately
touch both nodes' trees. Two-step hash: into the index, then into the tree.
Emits a per-interval CSV row.

Setup uses the type system directly:

```cpp
BSTs0 = reinterpret_cast<BinarySearchTree**>(new numa<BinarySearchTree*,NODE_ZERO>[num_DS]);
BSTs0[i] = reinterpret_cast<BinarySearchTree*>(new numa<BinarySearchTree,NODE_ZERO>());
```

Paper defaults: 1M indices × 80 keys (~2 GB), 10 minutes.

### 5.2 `DataStructureTests_four/` — four partitions (bench key `DS4`)

The same benchmark split four ways (`thread_numa<0..3>`). On a 2-node machine
`numa_node_map` sends them to 0,1,0,1 — two partitions per physical node.
`numDS` and `threads` are **totals**, divided by four inside the benchmark, so
`DS` and `DS4` do the same total work at the same parameter values. Emits four op
columns (`DS4_HEADER`), which is why it needs its own bench entry.

Fixed in `0d134040`: `NUMA_NODE_NUM` (§2.4 hazard) and `numIntervals()` — both
`global_init()` and the worker sized vectors with `duration/interval`, which
**truncates**, so any duration that is not a multiple of interval under-sized
them and workers wrote past the end (`-D 5 -i 20` dumped core).

### 5.3 `ycsb/` — NUMA-adapted YCSB

An array of `HashTable`s (chained `HashNode`s, TTAS locks), two partitions.
Deviations from stock YCSB, all deliberate:

- **Locality is part of the workload spec.** `--workload=A-50-50-50,A-100-0-50`
  is a comma-separated list of thread groups, each
  `<workload>-<local%>-<remote%>-<thread%>`: half the threads run A at a 50/50
  local/remote split, half run A fully local. This emulates several YCSB
  instances in parallel, some with locality — stock YCSB is uniform and offers
  nothing for a NUMA partitioning to exploit. `parse_mixed_workload()` divides
  `num_threads` by the percentages, so the count must divide cleanly.
- **Workload AD** — a synthetic mix (A at 50/50, D fully local) → 72.5% read,
  25% update, 2.5% insert.
- **`get()` touches every payload byte** on a hit, so a read genuinely pulls the
  record across the interconnect; **`update()` mutates in place**, so writes do
  not lean on the allocator.
- **Prefill is every *other* key** (`keys/2`), leaving the odd half to be filled
  by inserts during the run.
- `--mix uniform|zipfian` (+ `--theta`), `--hash djb2|mix` (placement hash),
  `--payload` bytes, `--warmup` seconds untimed before measurement.

CSV: `Date, Time, Num_Tables, Num_Threads, Thread_Config, DS_Config, Mix,
Buckets, Workload, Duration, Num_Keys, Interval, Ops_Node0, Ops_Node1,
Total_Ops`. Commas in the workload string become dashes so it stays one cell.

> **`Total_Ops` is cumulative**, not a per-interval rate. Difference consecutive
> rows to get throughput. Every analysis script depends on this.

Sizing:

```
record bytes  = payload + 32                       (32 B hash-node overhead)
prefill bytes = (keys/2)*(payload+32) + tables*buckets*8
load factor   = (keys/2) / (tables*buckets)
steady state  = 2x–4x prefill                      (measured)
```

The working set **grows mid-run** — updates and inserts fill the odd half, and
remote operations duplicate keys into the other node's tables. **Manifests record
prefill-only figures**, so a manifest number is not the peak. Workload C is
read-only and stays at prefill.

`tables`, `buckets` and `keys` must scale **together** to hold the load factor,
because LF (0.75 vs 1.67 — shallow vs deep chains) is the variable campaigns 01
vs 01.1 and 02 vs 02.1 exist to isolate.

### 5.4 The rest

`Histogram/`, `Array/`, `Array_lkfree/`, `Array_txn/`, `Exprs/` are
micro-benchmarks and variants. `allocator_test/` is the allocator's own test
suite (§4.4). `Data-Structures/` is the upstream suite the BST benchmark derives
from.

---

## 6. Layer 5 — the experiment harness

`scripts/README.md` is the operational reference; this section covers the design.

```
numafy.py  ──►  campaign.py  ──►  an_comparison.py
 transform      run the sweep      tables + AutoNUMA figures
 + compile          │
                    └──►  bar_plot_ycsb.py / campaign_comparison.py

run.py  — same machinery, no git gate, no manifest (scratch runs)
```

All commands run **from the repository root**.

### 6.1 `benchmarks.py` — the only file to edit to add a benchmark

Each entry declares binary, CSV header, `argv` builder, default workloads, cwd,
and a list of `param(name, type, default, help)` specs. `campaign.py` and
`run.py` are benchmark-agnostic: they build their CLI dynamically from these
(`add_bench_args`) and read them back (`extract_params`), so they never mention
`--threads`/`--buckets` themselves.

Note the bench key and suite directory differ (`DS` → `DataStructureTests`);
there is an explicit `"suite"` field for exactly this reason. Use
`BENCHES[bench]["suite"]` for any path construction.

### 6.2 `campaign.py` — provenance

The design principle: **a result is only as good as the record of what produced
it.**

- **Refuses to start on a dirty git tree.** You commit first, so the manifest
  commit is always meaningful.
- Writes `manifest.md` (commit, machine, NUMA binding, kernel tunables, every
  parameter, and one line per run) plus `git_diff.txt`.
- **AutoNUMA is not a flag.** The script *reads* `/proc/sys/kernel/numa_balancing`
  (or the `--an-mode` override where there is no root) and writes into `AN_off/`
  or `AN_on/`. To get both arms you run the *identical* command twice with the
  state flipped between; the second run appends to the same manifest and
  **refuses** unless the commit and every parameter match. That guard is what
  makes the AN_off vs AN_on comparison trustworthy.
- Records THP and `numa_balancing` **per run**, not per campaign — the two
  Perlmutter arms ran 4.5 hours apart on *different compute nodes*, so per-run
  capture is what proves they shared a configuration.
- Does **not** abort on a failed individual run.

```
Campaigns/ycsb/campaign05/
    manifest.md      git_diff.txt
    AN_off/  ycsb_<workload>.csv + .log
    AN_on/   ycsb_<workload>.csv + .log
```

Wall time for the default 7-workload × 4-config ycsb sweep: **~3 h per AutoNUMA
mode** at `--warmup 60 --duration 300`. Run under `screen`.

> **Always pass `--numafy`** — see §4.4's build-artifact trap.

`run.py` is the same machinery without the gate or the manifest, writing to
`Runs/<bench>/AN_<mode>/<name>.csv`. Use it to sanity-check a parameter before
committing three hours to it.

### 6.3 Analysis

| script | produces |
|---|---|
| `an_comparison.py` | per-config AutoNUMA effect, per-config-vs-baseline gap and how it shifts between modes, geomean rows; `autonuma_effect` and `gap_shift` figures |
| `campaign_comparison.py` | every pair of campaigns × every shared AN mode: throughput change and gap shift |
| `bar_plot_ycsb.py` | grouped normalized bars, solid = AN on, striped = AN off |
| `analyze_duration.py` | "how long must a config run" — Welch's t-test over growing windows, reporting the shortest window that is significant *and stays* significant |
| `line_plot_bst.py`, `bar_plot_bst.py`, `plot_perfs.py` | BST time-series, BST bars, `perf` plots |

Two interpretation rules that are properties of the *plots*, not the data:

- **Each AutoNUMA mode is normalized to its own baseline** in `bar_plot_ycsb.py`,
  so solid and striped bars are **not comparable to each other**. A taller solid
  bar means the gap widened, not that AutoNUMA was faster. Use
  `--baseline-from off` to put both on one scale, or `an_comparison.py`.
- Aggregates are **geometric** means of ratios — the correct aggregate for
  ratios.
- Between-session drift on stormbreaker is ~0.4%, so cross-campaign throughput
  changes under ~1% are inconclusive. The *gap* columns compare within-campaign
  ratios and are far more robust.

### 6.4 Machine configuration

`detect_machine.sh` probes the topology once per machine into `machine.env`
(gitignored, so it does **not** arrive with a clone — re-run it after cloning):

```
NUM_PHYS_NODES  CPU_NODES  NUM_CPU_NODES  TOTAL_THREADS
NUM_PARTITIONS  NUMA_NODE_ORDER      # read at runtime by numa_nodemap.hpp
PARTITION_THREADS                    # threads on the bound nodes ONLY
NUMACTL_BIND                         # e.g. --cpunodebind=0,1 --membind=0,1
```

It picks the `NUM_PARTITIONS` farthest-apart CPU-bearing nodes and binds to
exactly those — *not* to every node on the machine. That is what makes the
worst-case-locality comparison mean the same thing on a 2-node box and an 8-node
one. Hand-editable afterward.

`configure_machine.sh [machine] --stages=topo,clangtool,umf,suites --suites=…`
drives the whole from-scratch build, with each stage checking the previous
stage's artifacts.

> **`membind` restricts the run to the *bound nodes'* RAM.** Perlmutter's nodes
> 0+7 give 128 GB and 64 threads — **less** memory and **fewer** threads than
> stormbreaker's 0+1 (~257 GB, 80 threads), on a machine four times larger
> overall. Size the working set to the binding, not the machine.

---

## 7. Configuration surface

| knob | where | default | notes |
|---|---|---|---|
| `UMF` | make / numafy | on | off falls back to `numa_alloc_onnode` |
| `NUMA_NODE_NUM` | compile-time macro | **2** | number of UMF pools = number of logical partitions. §2.4 hazard |
| `MAX_NODE` | compile-time macro | **1** | the "other" logical node id in the benchmarks |
| `NUMA_NODE_ORDER` | env (`machine.env`) | auto-detected | logical→physical order |
| `NUM_PARTITIONS` | `detect_machine.sh` | 2 | how many nodes to bind |
| `UMF_JE_ARENAS_PER_POOL` | env | online CPU count | jemalloc arenas per pool |
| `UMF_JE_MAX_POOLS` | `pool_jemalloc.h` | 16 | ceiling on NUMA nodes, `static_assert`ed |
| `/proc/sys/kernel/numa_balancing` | sysctl (root) | machine-dependent | the independent variable; `--an-mode` where there is no root |
| THP `enabled` | sysfs (root) | machine-dependent | recorded per run; Perlmutter compute is `[always]` and unchangeable, so `always` on stormbreaker is the only comparable setting |

The `-DMAX_NODE=` / `-DNUMA_NODE_NUM=` injection in `numafy.py` is currently
**commented out**, which is why raising the partition count is a code change
rather than a config change.

---

## 8. Invariants

Things that must remain true; each has a way to check it.

1. **Slot identity.** UMF pool slot `k` == logical node `k`. Broken by creating
   pools out of order or twice (hence the `inline` globals and the init guard).
2. **Shared node map.** Allocator and thread pinner both call `numa_node_map()`.
   Never hardcode a physical node id.
3. **Placement.** A `numa<T,k>` allocation lands on `numa_node_map(k)`.
   → `allocator_test/bin/verify_allocator`.
4. **Free-path flatness.** ns/op must not grow with thread count.
   → same binary.
5. **Layout compatibility.** `numa<T,k>` and `T` have identical field order and
   offsets, or every `reinterpret_cast` in the output is wrong. Guaranteed by
   construction in the expansion; not otherwise checked.
6. **Cumulative CSV columns.** `Total_Ops` and per-node columns are cumulative;
   the op columns must sum exactly to `Total_Ops` and stay monotonic.
7. **Manifest match.** The second AutoNUMA arm must share commit and every
   parameter with the first. → enforced by `campaign.py`.
8. **Binary freshness.** After any allocator change,
   `nm Output/<suite>/bin/<bin> | grep BindThread`.

---

## 9. Known hazards

**Duplicated headers.** `umf_numa_allocator.hpp`, `numa_nodemap.hpp` and
`pool_jemalloc.h` are copied into ~10 suite directories, and **`-Iinclude/`
precedes `-InumaLib/`**, so the local copy silently shadows the canonical one.
They have drifted before (`DataStructureTests` once carried a pre-nodemap copy
with a hardcoded identity mapping; `MAX_JEMALLOC_THREADS` was 200 in one place
and 256 in another, giving two struct sizes across the ABI). Edit **all or
none**. The real fix is to delete the copies and rely on `-InumaLib`.

**Two-partition assumptions.** ycsb hard-codes `ht_node0`/`ht_node1`,
`threads_per_node = num_threads/2`, and a two-column CSV. `--threads` must be
even and must not exceed the CPUs of the *bound* nodes.

**Stale entry points.** All `*.slurm` files at the repo root still call
`python3 load.py` / `env.py` / `runYCSB.py` / `runExperiments.py` from the root;
commit `29f5e7db` moved all of those into `scripts/`. Their NERSC preamble is
still correct and worth keeping. `README.md`'s Build section has the same stale
paths. `line_plot_bst.py --AN both` raises `NameError` in legacy (non-campaign)
mode. `perfYCSB.py:106` hardcodes `0,1`.

**`runYCSB.py --perlmutter` defaults are the paper's numbers and are not
reproducible with today's code** — commit `f5dcf278` changed prefill from the
whole keyspace to every other key and added the payload option, so the same
`keys` value no longer means the same record count. Its `--duration` default is
1200 (the paper's 20 minutes); every current campaign uses 300.

**Result data is machine-local.** `Campaigns/`, `Runs/`, `Old_Results/`,
`FlameGraphs/` are gitignored, exist only on the machine that produced them, and
cannot be regenerated without multi-hour reruns. Do not delete, do not commit
(a previous attempt hit GitHub's file-size limit). `stormbreaker`'s root
filesystem is a single 879 GB volume that has hit 100% before — check `df -h /`
before starting a campaign.

**Ownership.** `stormbreaker.md` and `perlmutter.md` are per-machine profiles
owned by different agents; neither edits the other's. Shared-surface changes
(`numa-clang-tool/src/CMakeLists.txt`, `scripts/*`, `numaLib/*`) are flagged in
commit messages.

---

## 10. Extending it

**Add a benchmark** → one entry in `scripts/benchmarks.py` (binary, header,
`argv`, workloads, cwd, params) plus a `suite_map` entry in `scripts/numafy.py`.
Nothing else should need to change. Give the CSV the same column names
(`Thread_Config`, `DS_Config`, `Duration`, `Total_Ops`) and every analysis script
works unchanged.

**Add a pass** → a `FrontendAction` + `ASTConsumer` + `Transformer`, then one
line in `makeFactoryForPass()` in `main.cc`.

**More than two partitions** → a code change, in this order:
1. raise `NUMA_NODE_NUM` for the suite (≤ `UMF_JE_MAX_POOLS`) — otherwise
   partitions ≥2 allocate **silently unbound** (§2.4);
2. un-comment the `-DMAX_NODE=` / `-DNUMA_NODE_NUM=` injection in `numafy.py:119`
   and the `Output/*/Makefile` line;
3. generalize the benchmark's structural two-way split;
4. widen the CSV header and the bench entry (see `DS4`);
5. `NUM_PARTITIONS=4 bash scripts/detect_machine.sh` to rebind;
6. verify with `verify_allocator` before trusting a single number.

**A different memory technology** (`persistent<T>`, `secret<T>`, a CXL device
id) → the transform is agnostic. It needs a new UMF provider (the pool machinery
and the slot fast path carry over unchanged) and a new specialization template
in the equivalent of `getNumaAllocatorCode()`. The reference machine already has
two CXL memory-only nodes (2 and 3) sitting unused behind the node map, which
currently filters them out for lacking CPUs.
