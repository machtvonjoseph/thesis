# Design: `persistent<T>` library and the persistifying Clang tool

*The "how" document. For the motivation and the research claim, see
[idea.md](idea.md). For the dated log of decisions and current status, see
[HANDOFF.md](HANDOFF.md). Library internals in depth:
[Docs/PersistentLib.md](Docs/PersistentLib.md). Original Phase 5 tool plan:
[Docs/clang-tool.md](Docs/clang-tool.md).*

---

## 1. System in one picture

```
   user source (persistence-unaware)          persistified source
   ────────────────────────────────           ───────────────────
   class Node { char* key; Node* next; };     class Node { ... };          (untouched)
   class Bucket { Node* head; int size; };    template<> class persistent<Node> {
                                                  pmem_string key;
   int main() {                                   pmem_ptr<persistent<Node>> next;
     persistent<Bucket>* b =                      static void* operator new(size_t);
         new persistent<Bucket>();                persistent(const char* w) {
     b->insert("alpha");                              transaction::run(pmem_pool(), [&]{...});
   }                                              }
                                              };
        ─── persist-tool ───▶                 template<> class persistent<Bucket> { ... };

                                              struct __pers_root {
                                                  pmem_ptr<persistent<Bucket>> b;
                                              };
                                              __pers_root* __root = pmem_root<__pers_root>();

                                              int main() {
                                                persistent<Bucket>* b =
                                                    pmem_get_or_create<persistent<Bucket>>(__root->b);
                                                b->insert("alpha");
                                              }
```

Two layers:

- **[`persistentLib/`](persistentLib/)** — a header-only runtime over
  `libpmemobj` / `libpmemobj++`. Supplies the vocabulary types the generated
  code is written in.
- **[`persist-clang-tool/`](persist-clang-tool/)** — a LibTooling
  source-to-source rewriter, two passes, that produces that generated code.

Neither is useful alone. The library exists so the tool has something small to
emit; the tool exists so nobody has to hand-write the specializations (which is
what [`concept/`](concept/) contains, as the reference for what correct output
looks like).

---

## 2. Runtime layer — `persistentLib/`

Three headers, ~450 lines total.

### 2.1 Pool lifecycle — [`pmem_allocator.hpp`](persistentLib/pmem_allocator.hpp)

One global pool per process, opened before `main` and closed after:

```cpp
inline PMEMobjpool*        global_pool;        // C handle, for pmemobj_* calls
inline pmem::obj::pool_base pmem_pool_handle;  // C++ handle, for transaction::run

__attribute__((constructor)) void pmem_alloc_init();   // open-or-create
__attribute__((destructor))  void pmem_alloc_fini();   // close
```

Path comes from `PERSISTENT_POOL_PATH`, defaulting to
`/mnt/pmem-emu/global_persistent_pool`; size is fixed at 1 GB. A *single* global
pool is a deliberate simplification: it makes `PersistentAllocator<T>` stateless
(all instances compare equal, which satisfies the STL allocator contract for
free) and makes `pmem_root<T>()` unambiguous. Multi-pool support would need a
pool identity threaded through every type — real work, no current payoff.

Constructor-order caveat: `pmem_alloc_init` is an ELF constructor, so any pmem
allocation from another translation unit's static initializer is ordering-
dependent. All current benchmarks allocate from `main` or later, and the
generated `__root` global is initialized by `pmem_root<__pers_root>()`, which
throws loudly if the pool is not up.

### 2.2 Allocation — transaction-aware by construction

```cpp
void* pmem_alloc(size_t size, size_t align);
void  pmem_free(void* ptr);
bool  pmem_contains(void* ptr);
```

Both `pmem_alloc` and `pmem_free` branch on `pmemobj_tx_stage()`. Inside a
transaction they use `pmemobj_tx_alloc` / `pmemobj_tx_free`, so allocations
made in a transaction that later aborts are reclaimed, and frees made in one
are deferred until commit. This is Phase 2.5 and it closes two symmetric holes
that a naive implementation has: leaked nodes on abort, and use-after-free on
abort. The object *lifecycle* is inside the atomic unit, not just field writes.

### 2.3 `persistent<T>` — [`persistenttype.hpp`](persistentLib/persistenttype.hpp)

Declared with an SFINAE slot and split into two specializations:

```cpp
template<typename T, template<typename> class Alloc = PersistentAllocator, typename E = void>
class persistent;
```

**Primitive specialization** (`is_fundamental || is_pointer`) holds a `T contents`
and is the place the durability guarantee actually lives:

```cpp
void store(T data) {
    int succ = pmemobj_tx_add_range_direct(&contents, sizeof(contents));
    if (succ != 0) throw std::runtime_error("Failed to add range to transaction");
    contents = data;
}
```

Every mutating path routes through `store()`, so the undo-log snapshot happens
before the write, and a write outside any transaction throws at the first stray
store rather than corrupting silently.

Three properties of this class are load-bearing and easy to break by accident:

1. **The mutable `operator T&()` is deleted; only `operator const T&() const`
   remains.** Handing out a writable `T&` would let a caller assign straight to
   `contents`, bypassing the snapshot. With it gone, `int& r = *p;` is a compile
   error and unlogged writes to a scalar are *inexpressible* rather than merely
   discouraged. This is the crash-consistency claim in §5 of [idea.md](idea.md).
2. **Member compound operators exist for exactly that reason.** `++`, `--`,
   `+=`, `-=`, `*=`, `/=`, `%=`, `&=`, `|=`, `^=`, `<<=`, `>>=` are all
   `store(load() op v)`. Before they existed, `count++` resolved via the
   conversion operator to a built-in `++` on a raw `int&` and silently skipped
   the log — a bug that survived four benchmarks undetected because nothing
   crashed at the wrong moment.
3. **`operator new` calls `pmem_alloc(sz, alignof(T))` directly, not
   `allocator_type::allocate(sz)`.** `operator new` receives *bytes*;
   `allocate` takes a *count* and multiplies by `sizeof(T)` internally. Routing
   one through the other over-allocated 4× for `int`, 8× for `double`, and was
   correct only for `sizeof(T) == 1`.

**Class specialization** (`!is_fundamental && !is_pointer`) inherits from `T`
and forwards constructor arguments — and deliberately has **no** `operator new`.
It is a fallback for types the tool never specialized, and a fallback that
silently puts data in pmem is *worse* than one that puts it in DRAM: without the
tool's other transformations (field wrapping, transaction wrapping) pmem
placement is half-correct, and half-correct durability is the failure mode this
project exists to eliminate. DRAM is at least honest.

For any type actually used persistently, the tool emits a **full explicit
specialization** that replaces this fallback wholesale (see §4.3, and the
rationale in §6.1).

### 2.4 `pmem_ptr<T>` — restart-stable pointers

A 16-byte wrapper over `PMEMoid` presenting `T*` semantics (`operator*`,
`operator->`, `get()`, `explicit operator bool()`, comparison with `nullptr`).
Every mutator calls `snapshot_if_pmem()` first:

```cpp
void pmem_ptr<T>::snapshot_if_pmem() {
    if (pmem_contains(&oid_)) {
        if (pmemobj_tx_add_range_direct(&oid_, sizeof(oid_)) != 0)
            throw std::runtime_error("pmem_ptr assignment outside txn.");
    }
}
```

The residency check is what lets *one* class serve both roles: as a field inside
a persistent object it logs, and as a transient DRAM handle it does not. Without
that check we would need two types and the tool would have to decide between
them per use site.

### 2.5 `pmem_string` — [`pmem_string.hpp`](persistentLib/pmem_string.hpp)

`PMEMoid data_` + `size_t size_`, characters in pmem, whole-string replacement
only. The design question worth recording is **why this type exists at all**,
since `pmem_ptr<persistent<char>>` already solves placement and restart
stability. The answer is logging granularity: per-character `store()` means one
`pmemobj_tx_add_range_direct` per byte, so a 9-character key costs 9 undo
entries with tens of bytes of metadata each. `pmem_string` snapshots its two
fields once and bulk-`memcpy`s the payload.

The general principle, which decides future container questions: **per-element
logging is correct when elements are meaningful units** (pointers in a bucket
array, counters) **and pathological only for bulk byte copies.** Arrays of
pointers therefore need no special type — nested `pmem_ptr` composition is the
right answer.

Read sites need no rewriting, because of `operator const char*() const`:
`strcmp`, `ostream <<`, and any `const char*` parameter work unchanged. That
conversion is safe where `persistent<T>::operator T&()` was not, precisely
because it is `const` — nothing can be written through it.

Four bugs were found and fixed before this type landed, all of which are worth
keeping in mind as a checklist for any future pmem-resident type: missing
`operator new`/`delete` (a standalone `new pmem_string` went to DRAM, so
`pmemobj_oid` returned `OID_NULL` and the string read back empty every run —
*silently*); self-assignment use-after-free in `assign()`; an unchecked
`pmem_alloc` return; and a destructor that cleared its own fields with unlogged
pmem writes, which on abort would restore an object pointing at nothing while
its payload leaked. Statement order in `assign()` is load-bearing:
throw-before-allocate, then copy-before-release.

Known limits, all deliberate: no small-string optimisation (every non-empty
string costs one allocation; an inline buffer would eliminate that for
YCSB-length keys, but the optimisation should be profile-driven), copy
constructor and copy assignment deleted, whole-string replacement only.

### 2.6 Root and find-or-create

```cpp
template<typename T> T* pmem_root();                              // PMDK pool root, typed
template<typename T, typename... Args>
T* pmem_get_or_create(pmem_ptr<T>& slot, Args&&... args);         // idempotent construction
```

`pmem_get_or_create` is the whole "is this the first run?" branch, hidden:
return the slot if set, otherwise allocate and assign inside an internal
`transaction::run`. It is the single reason user code contains no first-run
special case. This is also why the "Tier 3 named registry" idea was rejected —
the tool generates equivalent boilerplate from a `new persistent<T>(...)`
declaration using these primitives, so a registry would be a second mechanism
for the same job.

### 2.7 Runtime invariants, collected

| Invariant | Enforced by | Failure mode if violated |
|---|---|---|
| Every write to a persistent scalar is undo-logged | `store()`; deleted mutable `operator T&()` | compile error (write path) / throw (no active tx) |
| Every write to a pmem-resident pointer slot is logged | `pmem_ptr::snapshot_if_pmem` | throw |
| Allocations and frees join the enclosing transaction | `pmem_alloc` / `pmem_free` stage check | leak or dangling pointer on abort |
| Durable objects are reachable after restart | `__pers_root` + `pmem_root<T>()` | permanent leak |
| Pointers stored in pmem are position-independent | `pmem_ptr` (`PMEMoid`) | garbage deref on run 2 |
| Recovery replays unfinished transactions | PMDK on `pmemobj_open` | — (no custom code) |

---

## 3. The transformation contract

What the user is responsible for, and what is derived. Anything in the right
column appearing in user source is a bug in this document or in the tool.

| The user writes | The tool derives |
|---|---|
| ordinary classes with ordinary fields (`Node* next`, `char* key`, `int count`) | field wrapping: `pmem_ptr<persistent<Node>>`, `pmem_string`, `persistent<int>` |
| `persistent<T>* x = new persistent<T>(args);` at file scope or in `main` | the `__pers_root` struct, the `__root` global, and the `pmem_get_or_create` rewrite |
| method bodies as normal code | `template<> class persistent<T>` containing those bodies, transaction-wrapped |
| `persistent<T>*` explicitly on **free-function parameters** | *(nothing — see §6.4; this is a known concession)* |
| transaction boundaries at the call site where a *composite* atomic unit is wanted (e.g. Hanoi's pop+push pair) | per-method transaction boundaries |

Two obligations remain on the user and both are documented concessions rather
than oversights: free-function parameter types (§6.4), and composite atomicity
that spans more than one method call.

---

## 4. Tool architecture — `persist-clang-tool/`

Standard LibTooling shape, mirroring the numa tool it descends from:

```
main.cc  ──pass name──▶  FrontendAction  ──▶  ASTConsumer  ──▶  Transformer
                         (creates one consumer per source file)
```

[`src/main.cc`](persist-clang-tool/src/main.cc) registers two passes via a
`StringSwitch` over frontend-action factories — `root-setup` and
`recurse-persistent` — plus `all`, which runs them in order with a *fresh*
`ClangTool` per pass. The pass registry is the extension point; adding a pass is
a `.Case` plus an action/consumer pair.

### 4.1 Why two passes and not one, and why not three

The root struct must exist in the AST before the second pass can find the types
to specialize — `discoverTypesFromRoot()` literally looks for a `RecordDecl`
named `__pers_root` and reads its field types. So root-setup runs first, its
output is written to disk, and the recurse pass re-parses it.

There was originally a *third* pass (`PersistentHeapTyper`) that would re-parse
the recurse pass's output and do body rewriting. **It was deleted.** The flaw:
the recurse pass's output does not compile — the emitted specialization bodies
still reference raw types — so a downstream pass parses a degraded AST full of
stacked errors, its edits compound the breakage, and Clang's recovery
progressively loses ground. All body-level rewriting was folded into the recurse
pass, operating on the *original, clean* AST via per-method scratch `Rewriter`s
(§4.4). The general rule this bought: **never chain a pass onto output that does
not compile.** Two passes is the maximum the current design supports, and the
boundary between them is exactly the point where the intermediate output is
still valid C++.

### 4.2 Pass A — `root-setup` ([`rootSetup.{h,cc}`](persist-clang-tool/src/transformer/))

Finds every durable declaration, aggregates them into a pool root, and rewrites
the allocation sites.

**Discovery** uses two `RecursiveASTVisitor`s sharing one `isRootCandidate`
filter (translation-unit scope, or a local in `main`; never a `ParmVarDecl`):

- `PersistentDeclVisitor` — matches `VarDecl`s whose type strips to a
  `ClassTemplateSpecializationDecl` named `persistent`, recording
  `{typeName, varName, isPointer}`. If the decl has an inline `new` initializer
  it also records an `InitSite` with the `CXXNewExpr`'s source range and its
  constructor-argument text (`getDirectInitRange` + `Lexer::getSourceText`).
- `AssignmentInitVisitor` — the same thing for the split form
  `persistent<int>* counter; ... counter = new persistent<int>(0);`, matching a
  `BO_Assign` whose LHS is a `DeclRefExpr` to a root candidate. Note the filter
  is on the *target variable*, not the assignment's location, so a global
  assigned inside a helper function rather than `main` works for free.

**Emission** inserts before `main`:

```cpp
struct __pers_root {
    pmem_ptr<persistent<Bucket>> b;      // pointer decls
    persistent<int> counter;             // value decls (no pmem_ptr wrapper)
};
__pers_root* __root = pmem_root<__pers_root>();
```

and replaces each recorded `new` expression in place with
`pmem_get_or_create<persistent<T>>(__root->name, args...)`.

Root-insertion has **no numa analogue** — NUMA-typed data lives in arbitrary
memory, whereas persistent data must hang off a recoverable root. This pass is
entirely new machinery.

### 4.3 Pass B — `recurse-persistent` ([`recursePersistentTyper.{h,cc}`](persist-clang-tool/src/transformer/))

Emits a full `template<> class persistent<T>` for every user-defined `T`
reachable from the root.

**Idempotence, first.** `utils::findExistingPersistentSpecs` walks the TU for
complete `ClassTemplateSpecializationDecl`s of the `persistent` template and
returns the set of already-specialized type names, which seeds the skip set.
This is *AST-based*, not text-based, so re-running the tool on its own output is
a no-op rather than a duplication. `wrapFieldType` has the matching property one
level down via `isPersistenceAwareType` — a field already typed `pmem_string` /
`pmem_ptr<..>` / `persistent<..>` is emitted unchanged.

**Discovery and recursion.** `discoverTypesFromRoot()` reads `__pers_root`'s
fields, unwraps `persistent<T>` and `pmem_ptr<persistent<T>>` uniformly
(`utils::unwrapPersistentType` recurses through the `pmem_ptr` layer), and seeds
a worklist. The BFS drains that worklist; `wrapFieldType` grows it as it
encounters user-defined class types in fields. `specializeType(name)` is the
public re-entrant entry point, so a `new Foo()` discovered in a *method body*
(§4.4, Visitor 1) queues `Foo` for specialization in the same run even if no
field ever referenced it.

**The type mapping**, from `utils::wrapFieldType` — this table is the core of
the whole design:

| Original field type | Emitted type | Notes |
|---|---|---|
| already `pmem_string` / `pmem_ptr<..>` / `persistent<..>` | unchanged | idempotence |
| `char*`, `const char*`, `std::string` | `pmem_string` | checked **before** the pointer branch, since `char*` is a pointer |
| `U*` where `U` is a user class | `pmem_ptr<persistent<U>>` | queues `U` for specialization |
| `int*` and other pointer-to-primitive | `persistent<int*>` | library's primitive spec already accepts pointers |
| fundamental `F` | `persistent<F>` | |
| user class `C` by value | `persistent<C>` | queues `C` |
| anything else | original type, marked `/* unwrapped */` | visible escape hatch, deliberately ugly |

**Class emission** (`processClass`) buckets members by original access
specifier, so a user's `private` fields stay private in the specialization, and
emits fields before methods within each bucket. `operator new` / `operator
delete` are always public and always route through `pmem_alloc` / `pmem_free`.

**Method emission** (`processMethod`):

- signature from `utils::buildMethodSignature`, which renames constructors to
  `persistent` and destructors to `~persistent`;
- `= default` / `= delete` / declaration-only methods carried through
  unchanged;
- constructor member-initializer lists are **converted to body assignments**
  prepended inside the body — `Node(int v) : value(v), next(nullptr) {}` becomes
  `persistent(int v) { value = v; next = nullptr; ... }`. This is necessary
  because the wrapped field types are not constructible from the original
  initializer expressions, and it *works* because `pmem_ptr::operator=(nullptr_t)`
  and `persistent<T>::operator=(const T&)` exist and the whole conversion runs
  inside `pmem_get_or_create`'s transaction, so the per-field store hooks fire;
- the body is then **transaction-wrapped**. Nested transactions are flattened by
  PMDK, so wrapping unconditionally is safe even when the caller is already in
  one. `transaction::run` returns `void`, so non-void methods get an IIFE:

```cpp
int getCount(const char* word) {
    int __pers_ret = {};
    pmem::obj::transaction::run(pmem_pool(), [&]() {
        __pers_ret = [&]() -> int { ...original body... }();
    });
    return __pers_ret;
}
```

  This constrains the return type: it must be default-constructible,
  assignable, and not a reference. Violations are compile errors in the
  generated spec, which is the acceptable failure mode.

**Insertion** places each spec after the closing `;` of its class (found by
`Lexer::findNextToken` from the record's end location) via
`InsertTextAfterToken`. `utils::writeAllRewrites` then flushes *every* modified
`FileID`'s buffer, not just the main file — which is what makes the multi-file
`DS` benchmark work, landing `persistent<Stack>` into `Stack.hpp` rather than
into `main.cpp`. Each touched file gets `#include "persistenttype.hpp"`
prepended if `utils::fileIncludes` doesn't find it (a text scan — it does not
see through macros or `#if 0`, which is fine here).

### 4.4 Body rewriting — the visitors ([`visitors.{h,cc}`](persist-clang-tool/src/transformer/))

Four `RecursiveASTVisitor`s run per method inside `processMethod`, each against
`method->getBody()` on the *original* AST, writing into a throwaway scratch
`Rewriter`. The rewritten body text is then read back out with
`scratch.getRewrittenText(bodyRange)` and spliced into the spec. The main
rewriter is reserved for edits to real source; scratch rewriters produce text
that only ever appears inside generated specializations. That separation is what
makes it safe to rewrite aggressively.

| # | Visitor | Matches | Rewrites |
|---|---|---|---|
| 1 | `NewExprRewriter` | `CXXNewExpr` allocating a user class | `new T(...)` → `new persistent<T>(...)`; side effect: calls `specializeType(T)` |
| 2 | `LocalLHSRewriter` | `VarDecl` (not a parameter) whose initializer is such a `CXXNewExpr` | `T* v = ...` → `persistent<T>* v = ...` |
| 3 | `PmemPtrInterfaceRewriter` | `CK_LValueToRValue` cast over a `MemberExpr` reading a user-class-pointer field | appends `.get()` |
| 3b | same visitor, `VisitVarDecl` | local initialized from such a field read | `T* v = field;` → `persistent<T>* v = field.get();` |
| 4 | `CStringIdiomRewriter` | the three C-string write idioms on a string-like field | see below |

Visitors 1 and 2 are a matched pair — the same declaration gets both sides
rewritten, and Visitor 2 deliberately does *not* touch `T* v = field;` or
`T* v = f();`, which are different cases (the first belongs to Visitor 3b, the
second is unsupported, §6.4).

Visitor 3's `CK_LValueToRValue` gate is what makes it precise: it fires exactly
where a `pmem_ptr` field is *read* in raw-pointer context, and naturally skips
LHS-of-assignment positions where no read happens. `top = old->next;`,
`if (!top)`, `cur = cur->next;`, and `Node* old = top;` all fall out of the one
rule.

Visitor 4 handles the fields `wrapFieldType` remapped to `pmem_string`:

```
field = new char[n];   →  statement deleted   (pmem_string::assign allocates)
strcpy(field, src);    →  field = src;
delete[] field;        →  statement deleted   (pmem_string's destructor frees)
```

This pass is **mandatory, not cosmetic**, and the reason is subtle:
`pmem_string`'s implicit `operator const char*` means the first and third
idioms would otherwise *compile* against a `pmem_string` — the first copying
from an uninitialised DRAM buffer and leaking it, the third handing a pmem
payload to the DRAM `delete[]`. Two of the three would be silent corruption.
Only `strcpy` fails loudly on its own. `strncpy`, `strcat`, `sprintf`, and
`key[i] = c` are deliberately *not* handled so that they keep failing loudly
rather than being half-translated. The visitor keys off each field's *original*
declared type, so it needs no coordination with `wrapFieldType` — it runs
against the unmodified AST where the field is still `char*`.

### 4.5 Shared utilities ([`utils.{h,cc}`](persist-clang-tool/src/utils/))

`fileIncludes`, `buildMethodSignature`, `getMethodBodyText`,
`getCtorInitAssignments`, `findExistingPersistentSpecs`, `isUserClassPointerType`,
`isStringLikeType`, `isPersistenceAwareType`, `findCXXRecordDeclByName`,
`unwrapPersistentType`, `wrapFieldType`, `writeAllRewrites`. The convention
followed throughout: **visitor scaffolding lives in the transformer layer;
reusable AST predicates and text builders live in `utils/`.**

---

## 5. Driver workflow

```
./persistify.py <bench>     # source → Output/<bench>/
./run.py <bench> --fresh --runs 2
```

[`persistify.py`](persistify.py) stages the benchmark through three trees so
each pass's output is separately inspectable and the original source is never
touched:

```
<bench>/  ──copy──▶  persist-clang-tool/input/<bench>/         (pristine copy)
          ──copy──▶  persist-clang-tool/output/<bench>/        ──root-setup──▶ (rewritten in place)
                     └──copy──▶ output_recurse/<bench>/        ──recurse-persistent──▶
                                └──copy──▶ Output/<bench>/     (final)
```

[`run.py`](run.py) compiles from `Output/<bench>/` and runs it. `--fresh` wipes
`/mnt/pmem-emu/global_persistent_pool` to exercise first-run behaviour;
`--runs N` runs the binary N times so state accumulation is visible (a stack
growing `[30,20,10] → [30,20,10,30,20,10]` across invocations is the
persistence test). Single-file benchmarks go through `clang++` directly;
multi-file `DS` uses its own Makefile.

Benchmarks: `counter`, `stack`, `hanoi`, `DS`, `kvnode`, `kvnode2`, `kvstr`.
[`concept/`](concept/) holds the hand-transformed reference output for
comparison, and is also where the hand-written PMDK baselines for evaluation
live.

---

## 6. Design decisions and their rationale

### 6.1 Full specialization, not inheritance

`template<> class persistent<Stack>` re-declares every field and method rather
than inheriting from `Stack`. Inheritance would permit an implicit upcast to
`Stack&`, which would silently call `Stack`'s *non-persistent* methods on a
pmem-resident object — writes with no undo log, exactly the failure this design
exists to prevent. The generic class specialization in `persistenttype.hpp`
keeps inheritance only as a fallback for un-specialized types, and has no
`operator new` for the reason in §2.3.

### 6.2 Layout incompatibility is accepted, not worked around

The numa work relies on `numa<Node*,0>` being byte-identical to `Node*`, making
`reinterpret_cast<Stack*>(new numa<Stack,0>())` legal. Here
`pmem_ptr<persistent<Node>>` is 16 bytes against `Node*`'s 8, so that cast would
compile and read the wrong bytes — silent corruption. The consequence is
accepted rather than patched: **the persistent-typed universe is closed.**
Methods operate on persistent types throughout and never coerce back. Interop
with code that wants a plain `Stack*` needs an explicit deep copy, or nothing.

### 6.3 No compile-time transaction enforcement

A "transaction token" type threaded through every write would make a missing
transaction a compile error rather than a runtime throw. It was considered and
deferred: the API cost is `counter.store(5, tx)` at every call site, rippling
through all user code, in exchange for catching a condition the runtime check
already catches loudly and immediately at the first stray write. Revisit if the
runtime check ever lets a real bug through.

### 6.4 Destructive rewriting, and where it is refused

Visitors 1–3 rewrite *inside* method bodies, which is safe because that text
only ever lands in a generated specialization. A fourth visitor
(`SignaturePropagator`) was designed to rewrite free-function parameters
`void f(Stack*)` → `void f(persistent<Stack>*)` and then **deliberately
dropped**: rewriting a free function's signature destructively locks it into
persistent-only use and breaks every DRAM call site. It is the same flaw as the
local-reassignment problem, one level up.

The contract instead is that **the user types free-function parameters
explicitly** — see `moveDisks` in [`hanoi/user_hanoi.cpp`](hanoi/user_hanoi.cpp).
The principled fix is function-variant generation ("Approach 3": emit a
`persistent<T>*` variant of each free function and select per call site,
i.e. whole-program monomorphization). Out of scope until a benchmark needs it.

---

## 7. Known gaps and failure modes

Ordered roughly by how much they block the next milestone. Each is documented
where it lives in the code; none are silent.

**Structural — blocking a real workload**

1. **Arrays.** The gating item. `HashTable::table` is `HashNode**` allocated
   with `new HashNode*[n]`. Three sub-gaps: `wrapFieldType` has no array case
   (currently yields `persistent<HashNode**>` — a raw pointer stored in pmem,
   stale on run 2); `NewExprRewriter` skips `new[]` because the element type is
   not a user class; and `AssignmentInitVisitor` matches only a `DeclRefExpr`
   LHS, not `table[i] = new ...` (an `ArraySubscriptExpr`). Per §2.5's
   principle the right mapping is nested `pmem_ptr<pmem_ptr<persistent<HashNode>>>`,
   not a new array type. The element count lives in a sibling field the tool
   cannot associate automatically — the user supplies it. Note also that
   `processClass` emits only `operator new`/`delete`, not the `[]` forms, so
   array support needs that too.
2. **Multi-TU root emission.** `findMainLocation` inserts `__pers_root` before
   `main` in whichever TU has `main`. YCSB puts its persistent globals in a
   different TU. Needs the root split into a declaration in a generated shared
   header plus one definition, and the tool driven over a compilation database.
3. **Thread-safe `pmem_get_or_create`.** The check-then-create has no
   synchronisation; two threads racing on one slot double-allocate and leak.
   Either make the slot update a transactional compare-and-set, or document and
   enforce a single-threaded-init contract.

**Semantic — accepted concessions, documented in-code**

4. **Local reassignment from a non-persistent source** (`Node* p = new Node();
   p = some_dram_func();`) — no single LHS type accepts both. Fails to compile;
   the programmer splits the variable. Marker comment is in
   `LocalLHSRewriter::VisitVarDecl`. Needs Approach 3.
5. **Free-function parameters** — user-typed by contract (§6.4).
6. **Composite atomicity** spanning multiple method calls — user-written
   `transaction::run` at the call site, as in Hanoi's pop+push pair.

**Latent — small, silent, worth fixing before they bite**

7. **In-class field default initializers are dropped.** `Node* head = nullptr;`
   in `kvnode`'s `Bucket` does not appear in the generated spec. It is currently
   harmless *by coincidence*: `pmem_ptr`'s default constructor gives `OID_NULL`
   and `persistent<int>`'s gives `0`, matching. A non-default initializer
   (`int size = 8;`) would be silently lost. Listed among the `processMethod`
   gaps but deserves promotion — it is the only known *silent* gap in the list.
8. **`alignof(T)` in generated `operator new` refers to the original type**, not
   the specialization, whose field types differ (16-byte `PMEMoid`s where
   pointers were). It happens to be ≥ the requirement for every current
   benchmark; it is not guaranteed in general and should be checked.
9. **Overloaded operators are skipped entirely** by `processClass` (`TODO` in
   code), so a user class with `operator==` loses it in its specialization.
10. **Alias-only root slots.** `persistent<Stack>* st2; st2 = st;` gets a root
    slot it never uses.
11. **`buildInitializers()` in `rootSetup.cc` is dead code** — `applyRewrites`
    replaces new-expressions in place instead. Delete it or wire it up; leaving
    it invites someone to assume it runs.
12. **`findCXXRecordDeclByName` is not namespace-aware** (first match wins) and
    system-header filtering is deferred.
13. **Spec emission is not topologically ordered** — currently fine because
    specs are inserted next to their own class definitions, but fragile.
14. **`processMethod` gaps**: default arguments, variadic methods, templated
    members, ref-qualifiers, `explicit`/`constexpr`/`noexcept` specifiers,
    copy/move semantics for persistent types.

---

## 8. Where to look

| Question | File |
|---|---|
| What does correct output look like? | [`concept/<bench>/transformed_*.cpp`](concept/) beside [`<bench>/user_*.cpp`](kvnode/) |
| Library internals and per-decision history | [`Docs/PersistentLib.md`](Docs/PersistentLib.md) |
| Original Phase 5 pass plan | [`Docs/clang-tool.md`](Docs/clang-tool.md) |
| pmem emulation setup on this machine | [`Docs/EnvironmentSetup.md`](Docs/EnvironmentSetup.md) |
| Clobber-NVM structure (and why it needs LLVM 7) | [`Docs/clobber-nvm.md`](Docs/clobber-nvm.md) |
| Dated decision log, current status, next step | [`HANDOFF.md`](HANDOFF.md) |
| Why any of this | [`idea.md`](idea.md) |
