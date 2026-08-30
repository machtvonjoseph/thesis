# idea.md — NUMATyping: the argument

*What this project claims, why the claim is shaped the way it is, and what the
evidence currently supports. Companion to [`design.md`](design.md), which
describes how the system is built. Neither file is tracked by this repository.*

Written 2026-08-29 against commit `0d134040`.

---

## 1. The problem

A modern multi-socket server does not have one memory. It has several, each
attached to a different set of cores, and reaching the wrong one costs
**1.5x–3x** in latency and a comparable amount of bandwidth. On the reference
machine (`stormbreaker`, 2 CPU-bearing nodes) the remote penalty is ~2.0x. On
the target HPC machine (Perlmutter, AMD EPYC in NPS4, 8 nodes) the two nodes the
experiments use are cross-socket at distance 32 — **3.2x** — and each node
commands only a quarter of its socket's memory bandwidth, so a misplaced access
burns a scarce resource as well as paying latency.

Nothing in the C++ programming model expresses this. The language presents a
flat address space; placement is decided by whoever happens to touch a page
first, and then possibly re-decided by the kernel. The existing options all sit
at the wrong altitude:

- **First-touch** (the Linux default) gives good placement *by accident*, as
  long as the thread that allocates is the thread that reads and neither is ever
  migrated. It degrades silently: as a run progresses and cross-node operations
  mutate the structure, locality erodes with no signal to the program.
- **AutoNUMA** migrates pages and tasks based on sampled faults. It operates at
  **page granularity**, which is the wrong unit — application objects with
  genuinely different affinities get mixed onto the same page, and a 2 MB
  transparent huge page holding ~13,000 records is certain to be touched from
  both nodes. It cannot recover an affinity the program never expressed.
- **`libnuma` / `numactl` / `mbind`** are correct and precise, but they are
  administrator- or call-site-level. They do not compose, they do not survive
  refactoring, and — crucially — **they discard the programmer's intent**. The
  programmer usually *knows* that this tree and those threads belong together.
  That knowledge is thrown away at the moment it is most useful.

So the gap is not mechanism. Mechanisms exist. The gap is a **place to say it**.

## 2. The idea

Placement is a property *of the data*. The construct C++ already has for
attaching semantics to data is the type.

```cpp
numa<BinarySearchTree, 2>* t = new numa<BinarySearchTree, 2>();
thread_numa<2> worker(run, t);
```

`numa<T, k>` means "this object lives on node `k`". `thread_numa<k>` means "this
thread runs on node `k`'s CPUs". Matching `k`s is the whole programming model:
one annotation at the declaration, and the co-location is stated rather than
hoped for.

The inspiration is `std::atomic<T>` — a type qualifier that changes what the
compiler and runtime must guarantee about an object without changing how the
programmer thinks about it. `numa<T,k>` is the same move for physical locality.

### 2.1 The part that is actually hard

A wrapper that pins one object is trivial. It is also nearly worthless: a tree's
*root* on node 2 with a million *nodes* scattered anywhere buys nothing.

What is needed is that the qualifier **propagates through the type's structure**:

```
numa<BinarySearchTree, 2>
  └── numa<BinaryNode, 2>          (member type, retyped)
        └── numa<BinaryNode*, 2>   (its pointers, retyped)
```

`numa<BST,2>`'s fields must themselves be node-2 typed, recursively, so that
every allocation the structure performs at runtime is node-2 bound — not just
the first touch, but every node inserted at minute nine of a ten-minute run.

**C++ cannot express this.** Templates are powerful at compile-time computation
but cannot *enumerate and rewrite the member fields of an arbitrary user-defined
class* — C++20 has no general static reflection. Boost.Hana and Boost.Fusion
approximate it, but only for classes that opt in with extra annotations, which
defeats the point (the goal is to retype existing code). STL allocator
parameters propagate through *library* containers and stop dead at user types.

That gap is the technical contribution: **introspective typing**. A Clang
front-end pass reads the AST, finds the `numa<T,k>` declarations, walks each
type's members, and emits real C++ specializations for the whole transitive
closure. It is template expansion — but with the reflection step C++ lacks,
performed by a compiler pass, and emitting **pure C++** as output.

### 2.2 Why source-to-source, and why that matters

The output of the transform is ordinary C++20 that any compiler can build. There
is no modified backend, no runtime language support, no new dialect to maintain.
That is a deliberate adoption argument: the technique is a front-end pass and a
header library, so the cost of trying it on existing code is close to zero, and
every NUMA optimization expressible in C++ remains expressible in the output.

It also constrains the design in ways worth naming, because most of the system's
oddities descend from it:

| Constraint | Consequence |
|---|---|
| Clang can only parse **valid** C++ | The unexpanded program must compile, so `numa` is declared as `template<typename T, int N> class numa : T` before the pass, and replaced with an empty definition after |
| The output must be real C++ | `numa<T,k>` and `T` are *unrelated types* after expansion; the "is-a" relationship must be rebuilt manually |
| The relationship is rebuilt with `reinterpret_cast` | Legal only because both types share field layout and offsets, which the expansion guarantees by construction |
| Inheritance would have been the natural choice | Rejected: a child cannot *redeclare a parent field at a different type*, so it would either keep non-`numa` members or shadow them — and a cast to the parent would then reach the wrong fields |
| Casting to `T*` must still dispatch `numa`-aware methods | Achieved by aligning vtables — which requires methods to be `virtual` |

The last row is a real cost, and reviewers named it: mandatory virtual dispatch
inhibits inlining and adds an indirection on exactly the latency-sensitive paths
this work targets. It has not been isolated experimentally. It should be.

### 2.3 What is guaranteed, and what is only encouraged

Two safety invariants are enforced by construction:

1. **No `numa` object on the stack.** A thread stack is contiguous; a
   node-pinned object inside it is nonsensical. Caught at expansion time.
2. **A `numa` object's fields are on the same node.** Enforced by the type
   system itself: expanding `numa<C,k>` retypes field `T f` to `numa<T,k> f`, so
   a mismatched node is a type error. (A *pointer* to another node is fine: the
   pointer itself lives on `k`, giving `numa<numa<C',k'>*, k>`.)

What is **not** guaranteed is full referential integrity — "everything
transitively reachable from a `numa<Stack,0>` is on node 0". That would require
controlling allocations the object does not own (`push(Node* n)` receives a
pointer from somewhere else), which cannot be done soundly on pre-existing code.

The pragmatic answer is a **heuristic**: inside a `numa<C,k>` method body, `new`
expressions are rewritten to allocate on `k` and cast back. This is what makes
a tree's million nodes land correctly, and it is deliberately a heuristic, not a
theorem. The formal work proves the invariant that *is* guaranteed.

### 2.4 The formal claim, honestly scoped

`Fnuma` is a Featherweight-Java-style calculus with a store, mutable fields, and
addresses as `⟨node, location⟩` pairs. The store is well-formed when every
object of type `numa<C,q>` sits at an address whose node component is `q`, and
**Theorem 5.1** (proved in Coq) says every reduction step preserves that.

This is the right property, but it is a *narrow* one, and Review C is correct
that the abstract oversells it. It says: *given* the specialization has already
happened, the semantics never places a node-`q` object anywhere but `q`. It says
nothing about the Clang transformation being faithful, about `reinterpret_cast`
being sound in C++'s object model, or about the dynamic-allocation heuristic. It
is a specification sketch plus one invariant — valuable as a statement of intent,
not a verification of the implementation.

### 2.5 Logical nodes, not physical ones

`numa<T,k>` and `thread_numa<k>` take a **logical partition id**, resolved to a
physical node at runtime by a shared map (`numa_nodemap.hpp`). This is the direct
answer to "what happens if I compile for 2 nodes and run on 8?" — nothing
breaks; the same binary runs anywhere.

The map does three things:

- **Skips memory-only nodes.** `stormbreaker` has 4 NUMA nodes but only 2 with
  CPUs (2 and 3 are CXL tiers). Only CPU-bearing nodes are candidates.
- **Orders them outside-in**: `[0,1,...,n-1] → [0,n-1,1,n-2,...]`. On an 8-node
  machine logical 0,1 become physical 0,7 — the *farthest apart* pair, which is
  the worst-case locality this project is trying to study. On a 2-node machine
  it is the identity.
- **Wraps**: logical `k` → `order[k % |order|]`. Four logical partitions on a
  2-node box become 0,1,0,1 — two partitions per physical node, which is exactly
  what the `DataStructureTests_four` suite exercises.

Allocator and thread pinner consult the *same* map, so data and threads can
never drift apart. `NUMA_NODE_ORDER` overrides it per machine.

The honest caveat: this makes the *node ids* portable, not the *partitioning*.
Using more than two partitions is still a code change — `NUMA_NODE_NUM` and
`MAX_NODE` are compile-time, and the YCSB benchmark hard-codes two partitions
structurally (`ht_node0`/`ht_node1`, `threads/2`, a two-column CSV).

## 3. What the evidence says

### 3.1 The paper's numbers

| Benchmark | Machine | Claim |
|---|---|---|
| Transactional BST | 2-node Intel | `numa/numa` **+27%** over `regular/regular` (AutoNUMA off), **+15%** over `numa/regular` |
| Transactional BST | 8-node EPYC | **+31%** over `regular/regular` (AutoNUMA off) |
| YCSB workload C | 2-node Intel | **+21%** — read-only, so no write noise, pure interconnect latency |
| YCSB write workloads | 2-node Intel | **+7–10%** over `regular/regular` |
| Code expansion | both suites | **1.2–1.3x** LOC, all of it mechanical (e.g. `BinarySearchTree` 219 → 639 lines) |

Two secondary findings are as interesting as the headline:

- **AutoNUMA on top of explicit pinning is harmful**, especially on long runs and
  large machines. When both data and threads are already pinned, its sampling is
  pure overhead, and it actively ping-pongs pages for the `numa/regular` config.
- **`regular th. / numa ds.` is the configuration to avoid.** Pinned data with
  unpinned threads gets no first-touch benefit and pays the allocator cost. If
  you use `numa<T,k>`, use `thread_numa<k>` too.

### 3.2 What the allocator rework changed (2026-07, merged 2026-08)

This is the most consequential internal result and it is not in the paper.

The custom UMF/jemalloc pool had a bug that **specifically penalized the
configuration the paper is about**. Per-thread jemalloc tcaches were all created
up front *on the main thread*, and `tcache.create` binds permanently to the
calling thread's arena — so every worker's `free()` flushed into one arena's
bins. Free cost went from 26 ns (1 thread) to **377 ns (20 threads)**, invisible
in any single-threaded microbenchmark.

Fixing it (lazy per-thread tcache creation bound to that thread's own arena,
plus dropping arena rotation and caching allocation flags in TLS) took an
alloc+free pair at 20 threads from **408 ns to 43 ns**, flat across thread
counts and within 1.3x of plain jemalloc.

The effect, on the small configuration used to debug it
(`--num_DS=4 --duration=10 --keyspace=200000 --num_threads=32`, mean of 3 runs):

| | before | after |
|---|---|---|
| BST†, `numa/numa` vs `numa/regular` | **−11.3%** (losing) | **−2.5%** |
| YCSB write-heavy (A), steady state | — | **+4.5%** (`numa/numa` wins; the interval ranges do not overlap) |

† **Read this row narrowly.** Four trees under four locks for ten seconds is an
allocator harness, not the benchmark — it is lock-contention-bound, with deep
trees (keyspace 200000) over a tiny array, the opposite regime from the paper's
1M shallow trees. `archive/allocator.md` generalized this row into "on the BST it
reached parity but not a win"; **the campaign data at paper scale says
otherwise** (§3.3). What the row legitimately shows is that the allocator bug
was large enough to invert a result, which is the point.

**The allocator was a confound, and it was pointing against the headline
configuration.** Everything before the rework should be read with that in mind —
including the fact that campaigns 01–03 unknowingly ran the old allocator because
the UMF static libraries are build artifacts that a source merge does not rebuild.

### 3.3 The BST is the strongest result, not the weakest

At paper scale and post-allocator-fix, the transactional BST shows the **largest
`numa/numa` vs `numa/regular` gap of any benchmark on stormbreaker** —
`Campaigns/DS/campaign01-storm`, 1M indices × 80 keys, 80 threads, 600 s, four
configs, commit `3b2e54da`:

| config | M ops / 60 s interval | vs `numa/regular` | vs `regular/regular` |
|---|---|---|---|
| `numa` / `numa` | 3759.26 | **+11.12%** | **+21.51%** |
| `numa` / `regular` | 3383.21 | — | +9.35% |
| `regular` / `numa` | 3242.34 | −4.16% | +4.80% |
| `regular` / `regular` | 3093.85 | −8.55% | — |

(AutoNUMA off. With it on: +11.79% and +16.28%. `DS4`, the same total work split
four ways: **+12.86%** and +19.47%.)

That is **2.5x the gap YCSB shows on the same machine** (+4.5% on workload A).
So the ordering is: BST ~+11%, YCSB-stormbreaker ~+4.5%, YCSB-Perlmutter ~+17.9%
— and the machine matters more than the benchmark.

The per-interval series says the gap is **structural, not a slow decay**:

```
interval           1     2     3     4     5     6     7     8     9    10
numa/numa       3814  3862  3813  3767  3753  3751  3715  3700  3712  3707
numa/regular    3470  3469  3436  3393  3374  3357  3341  3334  3330  3329
lead            9.9% 11.3% 11.0% 11.0% 11.2% 11.7% 11.2% 11.0% 11.5% 11.3%
```

The lead is already ~10% in the first interval and then flat. Both configs decay
slightly over ten minutes, `numa/regular` a little faster (−4.1% vs −2.8%), which
widens the gap by about a point and no more.

**Why the transactional workload hurts `numa/regular` specifically.** The 20% of
operations that are cross-node transactions *mutate* trees belonging to the other
partition. Under `numa/regular`, an insert performed by a node-0 thread into a
node-1 tree allocates through the ordinary heap and first-touch puts the new node
on **node 0** — wrong, and permanently so. Under `numa/numa` the same insert goes
to node 1 because the *type* says so, regardless of which thread executes it.
First-touch is not merely eroding here; it is **wrong by construction from the
first transaction**. With only 80 keys per tree the structures saturate almost
immediately, so the misplaced fraction reaches steady state within the first
interval — which is exactly the flat ~11% the series shows.

This is the cleanest demonstration in the repo of what the type system buys,
because it is the case where the allocating thread is *not* the owning thread.

The general principle, which should probably be the paper's framing:

> **numa-typing wins exactly where placement differs from first-touch.** Where
> the allocating thread is the reading thread, first-touch already gives
> `numa/regular` the same locality for free, and there is little left to win.
> The value appears wherever the two come apart — cross-partition mutation
> (BST transactions), workloads with real locality left on the table (YCSB's
> mixed spec), long runs, and structures that keep growing.

The Perlmutter campaign is the strongest evidence for this, and it is much
larger than the stormbreaker numbers:

| | AN off | AN on |
|---|---|---|
| `numa/numa` vs `numa/regular`, geomean over 7 workloads | **+17.88%** | **+16.01%** |

with node balance 0.998–1.004 across all configs, 56 runs, zero failures. Two
structural reasons the gap is bigger there: remote costs 3.2x rather than 2.0x,
and each node has ~4x less bandwidth per thread. Notably `regular/numa` (1.03–
1.06x) **beats** `numa/regular` (1.00x) on that machine — *typing the data is the
more valuable half; typing only the threads buys almost nothing* — which is the
opposite of the ordering on stormbreaker and directly supports the framing above.

### 3.4 The AutoNUMA null result (open)

On Perlmutter, AutoNUMA's effect is within ±2.5% for every config, with **no
migration transient** — AN_on tracks AN_off from the first interval.

Working hypothesis: AutoNUMA has **no affinity signal to act on**. Uniform random
access over 300M keys and a 40 GiB working set means any given page is touched
rarely and from either node with roughly equal probability, while AutoNUMA's
two-stage filter needs consecutive faults from the same node. THP `always` makes
it worse — a 2 MB page holds ~13,000 records and is certain to be touched from
both sides.

**Unconfirmed.** The pilot that showed 685,828 migrations ran on a login node
under `THP=never` at 4 KB granularity, where pages *are* re-touched constantly.
Closing this needs one config run on an exclusive compute node under
`THP=always` with `/proc/vmstat` deltas captured around it (~10 min). Near-zero
counters would turn an absence into a mechanism — and it would be a
cross-machine result, since it also explains the weaker AutoNUMA effects on
stormbreaker.

## 4. What the reviewers said, and what it implies

Three OOPSLA 2026 reviews: **Weak Accept, Weak Reject, Weak Reject**. The
critiques converge on one thing, and it is not the idea.

### 4.1 The baseline problem (all three reviewers)

The headline number is against `regular/regular` — "do nothing". Every reviewer
independently asked for the **real** baseline: *pinned threads plus a hand-coded
NUMA allocation strategy* — and observed that the gap over that baseline is
visibly much smaller in the figures.

Review C goes further and asks for the contributions to be **separated**:

1. `thread_numa` (pinning alone)
2. first-touch initialization
3. the modified allocator
4. recursive field retyping
5. method/allocation rewriting

This is the single most valuable experiment the project could run, and the
allocator rework made it *possible* — before the fix, (3) was a confound large
enough to swamp (4) and (5). The `numa/regular` vs `numa/numa` contrast already
isolates roughly (4)+(5) given (1)+(2)+(3), and that is where the honest number
lives: **+17.9% on Perlmutter YCSB, +11.1% on stormbreaker BST (+12.9% at four
partitions), +4.5% on stormbreaker YCSB-A.** Those are real numbers against a
baseline that already has pinned threads, first-touch locality, and the same
allocator.

Nobody has yet written the libnuma hand-coded baseline, and Review A's version
of the ask is cheaper: quantify the *refactoring burden*. The LOC tables (1.2–
1.3x expansion, 219→639 lines for one class) already measure what the tool
writes; what is missing is the comparison against what a competent developer
would have to write by hand to get the same guarantee.

### 4.2 Portability and topology (Review B, Review C)

*"What if I write for 2 nodes and run on 8?"* — asked by two reviewers, and this
is now **answered in code**: `numa_nodemap.hpp` resolves logical ids at runtime,
outside-in, skipping memory-only nodes, with both the allocator and the pinner
sharing the map. Review B even guessed the design (*"allowing for a larger
number of virtual nodes ... mapped to real nodes at runtime"*).

What is still true is that the *number of partitions* is compile-time, and the
benchmarks structurally assume two. `DataStructureTests_four` is the first
four-partition suite and it took a real bug fix to work — its allocator header
defaulted `NUMA_NODE_NUM` to 2, so partitions 2 and 3 reached the pool binder
with a null pool and, under `-DNDEBUG`, allocated **unbound while still producing
plausible numbers**. That failure mode is worth remembering: silent, plausible,
and only visible if you check placement.

### 4.3 Workload realism (Review B, Review C)

Both benchmarks are synthetic and both have a locality split chosen by the
experimenter. Review B asks specifically about **skew** — every experiment
partitions data uniformly, which may only be true of synthetic benchmarks — and
about what happens when pinning prevents the migration that would have rebalanced
a skewed load.

The campaign infrastructure was built partly for this: campaigns 01 vs 02
isolate `mix=uniform` vs `mix=zipfian theta=0.9`, and 01 vs 01.1 isolate the hash
load factor (0.75 vs 1.67 — shallow vs deep chains). That is the beginning of a
skew answer, not the end of one. The real gap is a **non-synthetic** benchmark.

### 4.4 Unresolved correctness questions (Review A, Review C)

- Does `reinterpret_cast<T*>(numa<T,k>*)` yield a pointer through which field
  access and method calls are *defined behavior*, or is it layout-compatible in
  practice but not by the standard? There is currently no stated C++ subset or
  compilation model, and Review A adds a sharper version: once methods are
  `virtual`, both types carry vtable pointers and **the vtables differ** — the
  layout-compatibility argument needs to explicitly cover the vtable slot.
- Virtual dispatch overhead is unmeasured.
- The ~30% allocator penalty vs plain jemalloc is acknowledged and then set
  aside; an allocation-heavy microbenchmark would define the applicability
  envelope. (`allocator_test/` now has exactly this machinery.)
- No confidence intervals anywhere. Between-session drift on stormbreaker is
  ~0.4%; gaps in the few-percent range need error bars to mean anything.

## 5. The current position

**The idea holds up.** No reviewer disputed that placement belongs in the type
system, that recursive retyping is the right mechanism, or that C++'s lack of
reflection makes a front-end pass genuinely necessary. Review B called the
transformation "rather simple" — which it is, and which is arguably the point.

**The evaluation is what is contested**, and the project's own internal work
agrees. The allocator rework revealed that a chunk of the measured effect was an
allocator artifact pointing the wrong way. The spread across benchmarks and
machines (+4.5% to +17.9%) says the amount of locality left for the type system
to win is set by the workload and the machine, not by the mechanism. The
Perlmutter data says the effect grows where remote access actually hurts; the
BST data says it is largest where the allocating thread is not the owning
thread.

The through-line for the next iteration:

1. **Change the baseline.** Report against `numa/regular` (pinned threads +
   first-touch + the same allocator), not `regular/regular`. The number gets
   smaller and defensible, and it is still a real effect everywhere it was
   measured: ~+16–18% (Perlmutter YCSB), ~+11–13% (stormbreaker BST, two and
   four partitions), ~+4.5% (stormbreaker YCSB-A).
2. **Decompose the contribution** along Review C's five axes. Most of the
   machinery exists; the allocator fix is what makes the decomposition meaningful.
3. **Answer skew and topology with data**, not design. The nodemap answers
   topology in code; a zipfian-vs-uniform campaign pair answers skew partially;
   neither is written up.
4. **Say what the theorem covers.** It is a specification-level invariant, not
   an end-to-end guarantee, and claiming less would cost nothing.
5. **Lead with the mechanism the BST already demonstrates** — cross-partition
   mutation, where first-touch is wrong from the first operation rather than
   slowly eroding. That is the sharpest available statement of when this system
   is worth using, and the benchmark that shows it best is already in the repo.

## 6. Where it could go

The type qualifier is a *carrier for placement intent*, and NUMA is only the
first thing that fits in it. The same introspective expansion applies to any
property that (a) is a property of data, (b) must propagate to member fields, and
(c) can be realized by an allocator:

- **`persistent<T>`** — placement in NVM, with persistency semantics to define.
- **`secret<T>`** — placement in enclave/secure memory, with TCB reduction as the
  goal.
- **CXL device ids** — the `N` parameter maps naturally onto a pooled or
  disaggregated memory target. The paper motivates with CXL and never returns to
  it; the reference machine literally has two CXL memory-only nodes sitting
  unused, which makes this the cheapest unexplored direction in the repo.

Nearer-term, mechanically:

- **Transparent huge pages.** UMF's OS provider never calls `MADV_HUGEPAGE`, so
  the BST runs on 100% 4 KB pages. Extents are large and contiguous where the
  regular heap is fragmented, so THP should help `numa/numa` *more* than
  `numa/regular` — but mbind/THP interaction needs its own measurement pass.
- **`-flto`**, so the now-`inline` `umf_alloc` folds into the generated
  `operator new`.
- **Drop the per-suite header copies.** Ten directories carry duplicated copies
  of `umf_numa_allocator.hpp` / `numa_nodemap.hpp` / `pool_jemalloc.h`, they have
  drifted before, `-Iinclude/` shadows `-InumaLib/`, and that drift is exactly
  how the DS4 partitions ended up allocating unbound.
