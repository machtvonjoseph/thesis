# Idea: persistence as a type property

*The "why" document. For the "how", see [design.md](design.md). For the running
log of what was done when, see [HANDOFF.md](HANDOFF.md).*

---

## 1. The problem

Persistent memory (Intel Optane DC, or the kernel's `memmap=` emulation) is
byte-addressable and survives power loss. That combination is what makes it
interesting and what makes it hard: a store instruction can now *outlive the
process that issued it*, so an ordinary C++ program running on pmem produces
durable garbage. To write a correct persistent program you must simultaneously
get four things right:

1. **Placement** — the object has to live inside a pool that is mapped back on
   the next run, not on the DRAM heap.
2. **Pointer representation** — a raw `T*` is meaningless after restart unless
   the pool maps at the same virtual address. Pointers stored *inside* pmem must
   be position-independent (PMDK's `PMEMoid` = pool UUID + offset).
3. **Failure atomicity** — a multi-field update (`push` = allocate node, link it,
   bump size) must be all-or-nothing across a crash, which means every write is
   preceded by an undo-log entry.
4. **Reachability** — an object nobody can name after restart is a permanent
   leak. The pool needs a root, and everything durable must hang off it.

Miss any one and the program still compiles, still runs, and still prints the
right answer on the happy path. The failures are all *silent* and all show up
one process lifetime later. This project's whole reason to exist is that the
list above is a type-system problem wearing a runtime-library disguise.

## 2. Prior art, and where it stops

Three systems define the space, and all three are read as background here (PDFs
in the repo root, notes in [Docs/clobber-nvm.md](Docs/clobber-nvm.md)):

| System | User annotates | Logging | Cost |
|---|---|---|---|
| **Atlas** (OOPSLA '14) | nothing — outermost lock scopes become failure-atomic sections | undo | most transparent; infers atomicity from existing lock structure |
| **Mnemosyne** (ASPLOS '11) | `atomic { ... }` blocks | redo, via STM | clean semantics; *every* persistent store must sit inside a transaction the user wrote |
| **Clobber-NVM** (ASPLOS '21) | transaction boundaries | logs only writes that clobber transaction inputs | lowest logging overhead, heaviest compiler analysis |

They differ enormously in mechanism and barely at all in one respect: **the type
stays ordinary and the annotation goes on a region of control flow.** `Node` is
still `Node`. What you mark is *when* — the atomic block, the FASE, the
transaction. Persistence is a property of the code that runs, and the data type
is a passive participant.

That has a specific consequence. Because the type doesn't know, the compiler
can't check. A store to a persistent field outside any atomic region is a
well-typed program in all three systems; it is simply wrong at runtime, on a
machine that has crashed, on a run you weren't watching.

## 3. The insight this project borrows

The `numa<T, NodeID>` work (`~/NUMATyping/numaLib/`, and its recursive Clang
tool) makes the opposite move for a different resource. There, *placement* is a
property of the type: `numa<Stack, 0>` means "this object lives on NUMA node 0",
and a recursive source-to-source pass propagates that placement through the
type's pointer-typed fields, so `Node*` inside a node-0 `Stack` becomes a
node-0 `Node*` too. One declaration at the top, and the whole reachable object
graph inherits the property.

The claim here is that **persistence is the same shape of property, and a
strictly better fit for the technique** — because NUMA placement is a
*performance* property (get it wrong and you pay remote-access latency) whereas
persistence is a *correctness* property (get it wrong and you lose data). The
case for pushing a property into the type system is strongest exactly when the
failure is silent and unrecoverable.

## 4. The idea

**One type constructor, `persistent<T>`, that carries all four obligations from
§1, and a compiler pass that propagates it recursively through the type graph
reachable from a single user declaration.**

`persistent<T>` is not "T, but allocated somewhere else". It means:

- the object is allocated from the global pmem pool (`operator new` → `pmem_alloc`);
- every write to it snapshots into the active PMDK transaction's undo log before
  it lands (`store()` → `pmemobj_tx_add_range_direct`);
- its outgoing pointers are `pmem_ptr`, i.e. `PMEMoid`s, so they survive restart;
- it is reachable from a generated pool root, so it is findable next run.

And the user's total obligation is one line:

```cpp
persistent<Stack>* s = new persistent<Stack>();
```

Everything downstream — the root struct, the find-or-create, the full template
specialization of `persistent<Stack>` with `pmem_ptr<persistent<Node>> top`, the
recursively-generated `persistent<Node>`, the transaction wrapping every method
body, the `.get()` at every pointer-field read — is derived by the tool from
that declaration plus the ordinary, persistence-unaware `Stack` and `Node` the
user already wrote. See [`kvnode/user_kvnode.cpp`](kvnode/user_kvnode.cpp) next
to [`Output/kvnode/user_kvnode.cpp`](Output/kvnode/user_kvnode.cpp) for the
smallest honest illustration.

## 5. What is actually new

Two things, and it is worth being precise about which is which.

**(a) The transparency point on the curve is new.** Atlas is the most
transparent prior system and it still requires the programmer to have structured
their code into lock scopes that happen to coincide with the failure-atomic
units they want. Here the annotation is a *type*, applied once, at the point
where the programmer already had to say what they were allocating. The
transaction boundaries are generated from method boundaries rather than found in
the existing synchronisation structure.

**(b) Some errors become inexpressible rather than merely wrong.** This is the
sharper claim and the one worth defending in a paper. As of the 2026-08-11
library work, `persistent<T>` for scalar and pointer `T` exposes only
`operator const T&() const`; the mutable `operator T&()` was deliberately
deleted, and member `++`/`--`/`+=`/… were added to cover the idioms that
overload resolution used to route around it. The consequence:

```cpp
int& r = *counter;   // compile error
counter++;           // goes through store(), therefore through the undo log
```

An unlogged write to a persistent scalar is no longer a discouraged pattern; it
does not typecheck. In Atlas, Mnemosyne, and Clobber-NVM, the same mistake is a
valid program. Similarly, allocation and deallocation join the enclosing
transaction (Phase 2.5), so a crash mid-`push` rolls back the node allocation
rather than orphaning it — the object *lifecycle*, not just the field updates,
is inside the atomic unit.

## 6. The cost the idea imposes, honestly stated

The numa work gets a free escape hatch: `numa<Node*, 0>` is one machine word,
byte-identical to `Node*`, so `reinterpret_cast<Stack*>(new numa<Stack,0>())` is
legal and persistent-typed data can be handed to ordinary code.

**That trick is unavailable here, and this is the central semantic cost.**
`pmem_ptr<persistent<Node>>` is a 16-byte `PMEMoid`; `Node*` is 8 bytes. The
cast would compile and read the wrong bytes. So the persistent universe is
*closed*: once a type is persistified there is no cheap coercion back to the
plain type, and any code that wants to consume a `Stack*` must either be
persistified too or receive an explicit deep copy.

Both of the project's known structural gaps fall directly out of this:

- **Local reassignment.** `Node* p = new Node(); p = some_dram_func();` — the
  tool rewrites the declaration to `persistent<Node>*`, and the second
  assignment then has no legal type. There is no single LHS type that accepts
  both, because the two are unrelated.
- **Free-function signatures.** `void moveDisks(Stack*, ...)` cannot be
  destructively rewritten to take `persistent<Stack>*` without breaking every
  DRAM call site.

The current stance is deliberate and documented: **fail loudly at compile time
and make the programmer split the variable or type the parameter explicitly**
(see `hanoi/user_hanoi.cpp`, where `moveDisks` takes `persistent<Stack>*` by
hand). The principled fix — "Approach 3" — is whole-program monomorphization:
generate a `persistent<T>*` variant of each free function and select per call
site. That is a real piece of research, not a patch, and it is out of scope
until a benchmark demands it.

This is a good trade to have made *for a paper*: a compile error that the
programmer must resolve is a categorically better failure than a silent
unlogged write, which is what the alternative designs give you.

## 7. Why this is a research contribution and not a wrapper library

The library alone would be unremarkable — it is a thin, opinionated skin over
`libpmemobj`. The contribution is the pair:

> a type whose meaning is a set of durability obligations, plus a source-to-source
> pass that discharges those obligations across an entire reachable type graph
> from one declaration.

The pass is what makes the type usable. Without it, the user would hand-write a
full `template<> class persistent<Stack>` per type — which is exactly what
`concept/` contains, and exactly why `concept/` exists: it is the hand-written
reference for what the tool must produce, and the measure of how much work the
tool is actually saving.

## 8. What would falsify it

Stated as testable claims, in the order they should be attacked:

1. **Expressiveness.** Does the technique survive a workload nobody designed it
   for? The four pedagogical benchmarks (counter, stack, hanoi, DS) were written
   alongside the tool and prove very little on their own. YCSB is the honest
   test, and the gap analysis in HANDOFF.md is candid that it is not yet passed:
   arrays of pointers, multi-TU root emission, and thread-safe root creation are
   all still missing. **This is the live risk.**
2. **Overhead.** Per-method transaction wrapping is coarse. Every method body
   gets a `transaction::run`, flattened by PMDK when nested — correct, but it
   logs more than a Clobber-NVM-style analysis would. The comparison against a
   hand-written PMDK baseline (`concept/baseline_*.cpp`) is the measurement that
   decides whether transparency cost too much.
3. **The inexpressibility claim.** It should be attacked directly: is there a
   way to write an unlogged store to a `persistent<int>` through the current
   API? `memcpy` over the object, a `const_cast` off the const conversion, or a
   `reinterpret_cast` of the enclosing object all deserve to be tried. The claim
   is "not expressible in well-typed code", not "not expressible in C++".
4. **Recovery.** Recovery is entirely PMDK's — undo-log replay on
   `pmemobj_open` — which is a strength (no custom recovery code to be wrong)
   and a ceiling (no resumption semantics à la Clobber-NVM). Recovery *time* is
   therefore a property of PMDK plus how much we logged, which loops back to (2).

## 9. Direction

Near term, in dependency order: array support in the tool (the last structural
gap before a hash table works), then a cut-down single-TU YCSB, then the
multi-TU and thread-safety work for the real thing. In parallel, real Optane
access — CloudLab `r6525` or Chameleon — because emulated pmem is fine for
correctness and worthless for numbers.

Then evaluation: throughput against a DRAM baseline and against hand-written
PMDK; recovery time; crash-consistency validation via the `--crash` harness
already used in `pracitce/`. NVMW as a first venue; CGO or PLDI for the full
version, where the contribution is framed as the type-directed pass rather than
the library.
