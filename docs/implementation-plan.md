# HUC Implementation Plan

Status: controlling implementation plan

Language name: HUC, “Hassan’s Update on C”

Implementation language: C++20

First backend: C++20

License: MIT

This document records the decisions that must guide the specification rewrite
and the first two compiler milestones. Where the older language specification,
grammar, examples, or transpiler architecture disagree with this plan, this
plan takes precedence until those documents are reconciled.

## 1. Outcome

HUC will have two independently usable source levels and two independently
testable translators:

```text
HUC1 source
    |
    | huc1::expand
    v
printable per-module HUC0 source
    |
    | huc0::transpile
    v
C++20 headers + one root translation unit
```

The implementation order is:

1. **Milestone 1: HUC0 to C++20.**
2. **Milestone 2: HUC1 to HUC0.**

This order is intentional. HUC0 gives HUC1 a concrete, executable target and
prevents the staging implementation from hiding unresolved runtime semantics.

## 2. Product position

HUC is an unchecked, ahead-of-time systems language intended to offer simpler
C++-like value, object, and generic-programming semantics.

HUC is not a memory-safe Rust alternative. It deliberately permits:

- dangling and invalid raw pointers;
- null dereferences;
- unchecked indexing and pointer arithmetic;
- invalid adoption of allocations;
- use of inactive moved-from storage;
- data races;
- signed overflow and other specified undefined behavior.

The language provides deterministic cleanup for its unique-owner type, but it
does not infer or prove the safety of raw observers.

The initial release does not include:

- a borrow checker or lifetime annotations;
- inheritance or virtual dispatch;
- exceptions or stack unwinding;
- coroutines;
- user-defined move constructors or assignment operators;
- arbitrary implicit conversions or operator overloading;
- a stable HUC binary ABI;
- direct arbitrary C++ header interoperation;
- the full compiler reflection and structured-generation library.

## 3. Naming

The project and language use the uppercase spelling **HUC** and the working
expansion **Hassan’s Update on C**. The natural pronunciation is “huck.”

An older project named [**HuC**](https://github.com/uli/huc) is a
C compiler/toolchain for the PC Engine.
That does not block the working language name, but package names, executable
names, searchability, and trademark concerns must be reviewed before a public
0.1 release. Until then:

- repository and language name: `HUC`;
- driver name: `huc`;
- staged source extension: `.huc1`;
- runtime source extension: `.huc0`.

## 4. Normative language split

### 4.1 HUC0

HUC0 is the standalone runtime language. It may be written directly by a user
or generated from HUC1. The HUC0 translator applies the same parser and
semantic checker in either case.

HUC0 contains:

- modules and runtime imports;
- runtime functions and structures;
- `let` declarations;
- `mod` permissions;
- raw pointers `T*`;
- unique owners `T*&`;
- constructors, methods, and `drop`;
- runtime expressions and control flow;
- local `let auto` inference.

HUC0 does not contain:

- numbered declarations or control-flow keywords;
- `@`;
- family binders or specialization requests;
- compiler-only values or handles;
- raw generation operations;
- unresolved `auto` parameters or inferred function results.

### 4.2 HUC1

HUC1 is a strict staged extension of HUC0. A `.huc1` module may contain
unnumbered runtime code plus phase-1 and phase-2 constructs. HUC1 expansion
must produce printable `.huc0` modules.

The name HUC1 describes the staged source level. It does not mean suffix-2
constructs are excluded; phase-2 declarations are compiler-program facilities
consumed by the HUC1 translator.

## 5. HUC0 runtime semantics

### 5.1 Declarations and mutability

Every local, field, and global variable begins with `let`. Function parameters
do not use `let`.

HUC has no `var` keyword and no `const` keyword. Values are fixed by default,
and `mod` grants mutation.

```huc
let i32 answer = 42;       // fixed scalar
let i32 mod counter;       // writable scalar, initialized to zero
```

`mod` is layer-sensitive:

```huc
let T* observer;           // fixed pointer; immutable T through it
let mod T* observer;       // fixed pointer; mutable T through it
let T* mod observer;       // reseatable pointer; immutable T through it
let mod T* mod observer;   // reseatable pointer; mutable T through it

let T*& owner;             // fixed owner; immutable T through it
let mod T*& owner;         // fixed owner; mutable T through it
let T*& mod owner;         // movable/reseatable owner; immutable T
let mod T*& mod owner;     // movable/reseatable owner; mutable T
```

A containing object must itself be writable before a writable field or
writable-receiver method can be used. A `mod` field does not bypass a fixed
containing object.

### 5.2 Raw pointers

`T*` is:

- nullable;
- non-owning;
- unchecked;
- pointer-sized;
- freely copyable;
- never automatically destroyed.

HUC performs no lifetime tracking between a raw pointer and an owner.

### 5.3 Unique owners

`T*&` is a primitive unique owner, not a C++ reference declarator. General
`T&` does not exist.

The default representation is one address-sized word:

- null owns nothing;
- non-null owns exactly one dynamically allocated `T`;
- destruction drops the pointee and deallocates its storage;
- move copies the address and clears the source;
- owner arithmetic is prohibited;
- observation as a compatible raw pointer is implicit;
- conversion from raw pointer to owner is explicit.

Moving requires a writable source slot because moving modifies the source.

```huc
let Widget*& mod source = new Widget(7);
let Widget*& destination = source; // move; source becomes null
```

Attempting to move from a fixed owner is a compile-time diagnostic.

### 5.4 Copy and Move classification

Copy types include:

- scalar arithmetic and boolean types;
- raw pointers;
- structures whose fields are all Copy and which do not declare `drop`.

Move types include:

- owners;
- structures containing a Move field;
- structures declaring `drop`.

Synthesized moves copy Copy fields and move Move fields. A moved-from owner is
null. Other moved-from Move storage is inactive until explicitly reinitialized.
Obvious invalid use should be diagnosed, but HUC does not promise complete
flow-sensitive use-after-move prevention.

`copy expression` explicitly requests logical duplication:

- a Copy type is copied;
- a non-null owner allocates and copies its pointee;
- a Move structure must supply a read-only `fn clone() -> T`;
- a raw pointer copy duplicates only the address.

Move behavior cannot be overloaded.

### 5.5 Construction and initialization

HUC0 uses constructors rather than aggregate brace initialization:

```huc
let Widget value = Widget(7);
let Widget*& mod owner = new Widget(7);
```

A constructor is named `init`:

```huc
struct Widget {
    let i32 id;
    let i32 mod visits;

    fn init(i32 id) : id(id) {
        // visits defaults to zero
    }
}
```

Rules:

- constructor initializer entries must follow field declaration order;
- entries evaluate and initialize left-to-right;
- every fixed field must have a declaration initializer or constructor entry;
- omitted mutable scalar storage initializes to zero;
- omitted mutable pointers and owners initialize to null;
- omitted mutable structures invoke zero-argument construction;
- zero-argument `init` is synthesized only when all fixed fields are already
  satisfiable and all mutable fields can default-initialize;
- fixed locals and globals require an initializer;
- a fixed field without a declaration initializer becomes an obligation of
  every constructor;
- constructors are infallible in 0.1; fallible creation uses named factory
  functions and library result types later.

### 5.6 Methods and destruction

Methods are read-only by default. A trailing `mod` grants a writable receiver:

```huc
fn value() -> i32 { ... }
fn increment() mod -> void { ... }
```

A Move structure may define:

```huc
fn drop() mod -> void {
    // release non-field resources
}
```

`drop` cannot be called directly. It executes before fields are destroyed in
reverse declaration order.

HUC 0.1 has no exception unwinding. Process termination through panic need not
run pending cleanup.

### 5.7 Evaluation order

HUC evaluation is left-to-right for:

- operands;
- call arguments;
- constructor arguments;
- initializer expressions;
- aggregate elements if aggregate syntax is added later.

Earlier side effects complete before later operands begin. `&&` and `||`
short-circuit, and only the selected arm of `?:` executes.

The backend must insert temporaries rather than inherit a different C++ order.

### 5.8 Runtime overloading

Ordinary `fn` overloads are selected by:

1. exact type match;
2. lossless built-in numeric promotion;
3. permission-dropping pointer conversion;
4. owner-to-observer conversion.

User-defined implicit conversions do not participate. Overloads that differ
only in observation versus ownership consumption are prohibited because
ownership must not depend on subtle overload ranking.

## 6. HUC1 phase semantics

### 6.1 Number meaning

The suffix is part of the keyword:

- no suffix: a concrete declaration that survives in HUC0;
- suffix `1`: a parameterized family or structural phase operation that
  produces HUC0;
- suffix `2`: compiler-only storage, function, or type.

These are availability classes, not requests for repeated arbitrary passes.

### 6.2 Phase-1 families

The primary forms are:

```huc
struct1 Stack(auto T) {
    ...
}

fn1 convert(auto T)(T value) -> T {
    ...
}

let1(auto T) value {
    let T value = T();
}
```

Partial specialization adds an angle-bracket pattern after the binder list,
followed by an optional parenthesized predicate:

```huc
struct1 Stack(auto T)<T*>(predicate) { ... }

fn1 convert(auto T)<T*>(predicate)(T* value) -> T* { ... }

let1(auto T)<T*>(predicate) value {
    let T* value;
}
```

Full specialization uses an empty binder list:

```huc
struct1 Stack()<Widget> { ... }
fn1 convert()<Widget>(Widget value) -> Widget { ... }
let1()<Widget> value { let Widget value = Widget(); }
```

Requests use angle brackets:

```huc
let Stack<i32> stack = Stack<i32>();
let i32 result = convert<i32>(input);
use(value<i32>);
```

Angle tokens are lexed separately. Parsing preserves an ambiguous angle
postfix where necessary, and semantic name resolution decides whether it is a
family request or comparison expression.

### 6.3 Binders and deduction

`auto T` binds a compile-time entity whose kind is inferred from use. A binder
used in a type position must hold a type. Typed non-type parameters use normal
types:

```huc
struct1 Buffer(auto T, usize N) { ... }
```

Primary declarations may provide trailing defaults. Partial and full
specializations may not introduce defaults. Packs, when implemented, must be
last, and a pattern pack expansion must also be last.

Normal `fn1` calls may infer binders by unifying runtime parameter types with
runtime argument types. Non-deducible parameters must be supplied explicitly.

### 6.4 Specialization selection

Each `struct1` and `let1` name has one primary family per scope. Each `fn1`
name also has one primary family per scope in 0.1; normal runtime `fn`
overloading remains available.

The primary must precede its specializations. Initially, every specialization
must be declared in the defining module.

Selection:

1. Match argument count and structural angle pattern.
2. Bind pattern variables.
3. Evaluate the optional predicate as a pure compiler boolean.
4. Remove false candidates.
5. Choose the unique structurally most-specific viable candidate.

An exact/full pattern outranks a partial pattern. A more constrained structural
pattern outranks a pattern that accepts all of its matches. The primary is the
fallback. Predicates filter but never rank. Declaration order never breaks a
tie. Equally specific or incomparable viable candidates are ambiguous.

HUC intentionally applies partial specialization uniformly to structures,
functions, and variable families. This is inspired by C++ partial
specialization but does not copy C++ substitution failure or its restriction
against function-template partial specialization.

### 6.5 Deferred bodies

An unselected `if1` branch, zero-iteration phase-loop body, or unchosen family
body is not lexed or parsed.

A dedicated delimiter scanner only locates the matching outer `}` while
recognizing:

- nested braces;
- quoted strings and character literals;
- line comments;
- block comments.

No HUC token, name, or type diagnostic may originate inside an opaque skipped
body. The only possible failure is inability to locate its outer delimiter.
If a body is selected, it is then lexed and parsed normally.

### 6.6 `let1`

A `let1` header has no result type. Every selected path must leave:

- exactly one ordinary runtime `let`;
- with the same identifier as the family;
- with no other residual runtime declaration or statement.

Compiler-only setup and `if1`/`for1`/`while1` are permitted. Raw emission is
not permitted inside a `let1` selection body because it would obscure the
single-variable invariant.

Different specializations may choose different:

- value types;
- raw versus owning pointer forms;
- pointee permissions;
- slot mutability;
- initializers.

`let1` is permitted at module and function/block scope but not as an instance
field. Every statically requested specialization is emitted at the family
declaration site. A function-local specialization therefore initializes on
each runtime execution that reaches that declaration; no lazy runtime flag is
introduced.

### 6.7 Phase control

`if1`, `for1`, and `while1` execute during HUC1 expansion and are allowed at:

- module scope;
- structure/type-body scope;
- function scope;
- nested block scope.

At module or type-body scope, residual output must be declarations or members
valid at that insertion point. At function/block scope, residual declarations
and statements are allowed.

`for1` requires a finite compiler iterable. `while1` reevaluates a compiler
boolean and is bounded by interpreter resource limits. A zero-iteration body
remains opaque.

Inside `fn2`, ordinary `if`, `for`, and `while` already execute in compiler
phase. HUC therefore has no `if2`, `for2`, or `while2`.

### 6.8 Phase-2 declarations

Compiler-only variables are always spelled `let2`, including locals in `fn2`
and fields in `struct2`. Parameters inherit the phase of their enclosing
function and omit `let2`.

`fn2`, `struct2`, and `let2` never survive in HUC0.

### 6.9 The `@` operator

`@` has one grammatical and semantic role: it opens a compile-time evaluation
island.

It is never attached to a type:

```huc
let2 compiler::Function function =
    @compiler::find_function("render");
```

The type is `compiler::Function`, not `@Function`.

Within `fn2`, a predicate, phase-control evaluation, or an existing `@`
island, nested compiler calls execute normally and do not need repeated `@`.

A normal `fn1` call produces or calls a concrete runtime specialization. An
`@fn1_call(...)` requires complete compile-time evaluation and may embed only a
materializable result.

### 6.10 Imports

HUC1 has two import forms:

```huc
import app.data;
import2 compiler;
```

`import` exposes runtime declarations and phase-1 families and remains in the
corresponding HUC0 module. `import2` exposes compiler-only declarations and is
removed.

There is no `import1` in 0.1. Import cycles are rejected with the complete
cycle path.

### 6.11 Raw generation

The bootstrap `compiler` module exposes only:

```huc
compiler::emit_huc(text);
compiler::emit_huc_module(text);
```

Calls begin with `@` when they enter compiler execution:

```huc
@compiler::emit_huc("let i32 mod generated;");
```

`emit_huc` writes at the inherited call-site insertion cursor.
`emit_huc_module` appends a declaration to the call-site module’s generated
declaration buffer.

Nested compiler helper calls inherit the cursor. A helper does not generate
beside its own definition. There is no arbitrary “one scope above” operation.

The raw text must be HUC0, but the HUC1 translator does not parse it. The HUC0
translator is the first component that parses and validates emitted text.

Bootstrap HUC does not include:

- `gen`;
- `gen_global`;
- `quote`;
- `parse_huc`;
- structural syntax objects.

## 7. Future compiler library boundary

The full compiler/reflection library is a separate design project. The initial
architecture must nevertheless preserve a clean boundary for future types such
as:

- `compiler::Environment`;
- `compiler::Module`;
- `compiler::Declaration`;
- `compiler::Type`;
- `compiler::Function`;
- `compiler::Class`;
- `compiler::Statement`;
- `compiler::Expression`;
- `compiler::Instruction`;
- `compiler::Code`.

These will be ordinary qualified type names declared as phase-2 types by the
compiler module. Their names will not contain `2` and will never use `@` as a
type marker.

The future API should expose immutable semantic snapshots and stable IDs,
rather than internal C++ AST objects. Structured edits and commits will be
explicit. Only concrete phase-0 declarations and materialized family
specializations are enumerable; an infinite uninstantiated family is not.

## 8. Compiler organization

All compiler code is C++20.

The repository will contain:

```text
CMakeLists.txt
include/
  huc/common/
  huc/huc0/
  huc/huc1/
src/
  common/
  huc0/
  huc1/
  driver/
runtime/
  huc_runtime.hpp
tests/
  unit/
  golden/
  negative/
  execute/
```

Logical libraries:

- `huc_common`: sources, diagnostics, tokens, module graph, canonical IDs,
  target information, arenas, and common parser facilities;
- `huc0`: standalone HUC0 frontend, semantic analysis, HIR, MIR, ownership
  lowering, and C++20 emission;
- `huc1`: HUC1 indexing, deferred-body spans, phase checking, interpreter,
  specialization, and HUC0 residualization;
- `huc`: thin command-line driver.

`huc1` may reuse HUC0 parsing and type-resolution facilities for selected
runtime code and `fn1` deduction. It must not bypass the textual HUC0 boundary
when chaining the translators.

## 9. Public compiler interfaces

The architecture document will define value-oriented public APIs equivalent to:

```cpp
namespace huc0 {

struct TranspileRequest {
    std::filesystem::path root_module;
    std::vector<std::filesystem::path> import_roots;
    std::filesystem::path output_directory;
    TargetConfig target;
    bool write_source_maps = true;
};

struct TranspileResult {
    std::vector<Artifact> artifacts;
    std::vector<Diagnostic> diagnostics;
    DependencyGraph dependencies;
    bool succeeded;
};

TranspileResult transpile(const TranspileRequest&);

} // namespace huc0
```

```cpp
namespace huc1 {

struct ExpandRequest {
    std::filesystem::path root_module;
    std::vector<std::filesystem::path> import_roots;
    std::filesystem::path output_directory;
    TargetConfig target;
    PhaseLimits limits;
    bool write_source_maps = true;
};

struct ExpandResult {
    std::vector<Artifact> modules;
    std::vector<Diagnostic> diagnostics;
    DependencyGraph dependencies;
    bool succeeded;
};

ExpandResult expand(const ExpandRequest&);

} // namespace huc1
```

Exact container implementation may change, but ownership, inputs, outputs, and
the absence of thrown compiler errors are fixed. Fatal internal invariant
violations may terminate a debug build; user program errors are diagnostics.

The driver commands are:

```text
huc lower <root.huc0> --out-dir <cpp-dir>
huc stage <root.huc1> --out-dir <huc0-dir>
huc build <root.huc1> --out-dir <build-dir>
```

`lower` is implemented first. `stage` is implemented in Milestone 2. `build`
chains `stage` and `lower` through files, not an undocumented AST shortcut.

## 10. Milestone 1: HUC0 to C++20

### 10.1 Exit result

Given an acyclic `.huc0` module tree, the translator emits:

```text
build/cpp/
  huc_runtime.hpp
  app/main.huc.hpp
  app/main.huc.cpp
  app/main.huc.map
  dependency/module.huc.hpp
```

Each HUC module becomes a guarded generated header. A runtime import becomes:

```cpp
#include "dependency/module.huc.hpp"
```

The root `.cpp` includes the root header. Only the root `.cpp` is compiled, so
the initial backend produces one C++ translation unit.

### 10.2 Increment A: project and source infrastructure

Implement:

- CMake C++20 targets;
- source buffers and stable source IDs;
- UTF-8 byte-position tracking with ASCII identifiers initially;
- line/column lookup;
- structured diagnostics with primary and secondary spans;
- module-name to path resolution;
- import search paths;
- acyclic import graph validation;
- deterministic traversal independent of hash-map order.

Acceptance:

- the driver loads a module tree;
- missing modules and cycles have useful diagnostics;
- repeated runs produce byte-identical dependency order.

### 10.3 Increment B: lexer and parser

Implement a hand-written lexer and recursive-descent/Pratt parser.

Cover:

- module and import declarations;
- `export`;
- `let`;
- `struct`;
- `fn`;
- `init` and `drop`;
- pointer/owner and `mod` layering;
- blocks and runtime control flow;
- expressions and precedence;
- constructor and `new` syntax;
- local `let auto`;
- error recovery at declaration and statement boundaries.

The lexer emits individual `<` and `>` tokens and indivisible numbered keyword
tokens, even though HUC0 rejects numbered forms.

Acceptance:

- every normative HUC0 example parses;
- AST dumps are deterministic;
- malformed inputs produce multiple useful diagnostics without crashing;
- HUC1-only syntax is explicitly rejected as unavailable in HUC0.

### 10.4 Increment C: names and types

Implement:

- module, type, function, field, parameter, and local scopes;
- collect-before-check for module declarations;
- declaration-before-use for locals;
- canonical built-in, structure, raw-pointer, owner, and fixed-array types;
- local `auto` inference;
- method receiver permissions;
- constructor obligation checking;
- overload resolution using only specified built-in conversions;
- no user-defined implicit conversions;
- prohibition of ownership-only overload sets.

Acceptance:

- type and name errors point to both use and candidate declaration;
- `mod` cannot be gained implicitly;
- fixed storage cannot be assigned;
- fixed owners cannot be moved;
- all constructor field obligations are checked.

### 10.5 Increment D: typed HIR and sequenced MIR

HIR records:

- canonical type;
- resolved symbol;
- value use: read, observe, copy, move, or place;
- source origin;
- explicit conversion selected by HUC.

MIR records:

- ordered temporaries;
- places and assignments;
- calls and control-flow blocks;
- active/inactive Move state;
- cleanup edges;
- conditional drop flags only where control flow requires them;
- owner nulling after moves.

No semantic decision is deferred to C++ overload resolution.

Acceptance:

- HIR/MIR dumps expose every move and observation;
- argument and operand sequencing is explicit;
- cleanup is exactly once on every normal control-flow exit.

### 10.6 Increment E: scalar C++20 backend

Emit:

- deterministic generated identifiers;
- module namespaces;
- guarded headers;
- the root source file;
- scalar and raw-pointer declarations;
- structures, functions, constructors, methods, and runtime control flow;
- explicit temporaries preserving left-to-right evaluation;
- `#line` or sidecar source-map information where practical.

Generated C++ is an implementation artifact, not HUC’s semantic authority.

Acceptance:

- scalar examples compile with a selected C++20 compiler;
- execute tests match expected output and side-effect order;
- generated output is stable across repeated builds.

### 10.7 Increment F: ownership and lifecycle

Provide a private one-word owner helper. Its C++ type:

- contains only a pointer;
- has deleted copying;
- can release, reset, observe, and move;
- invokes generated HUC drop logic before deallocation;
- has size/alignment assertions.

MIR remains authoritative for inactive aggregate state and conditional cleanup.
The backend clears/reset fields after explicit cleanup so later C++ storage
destruction is a no-op rather than a second HUC destruction.

Implement:

- `new T(args)`;
- owner observation;
- owner move initialization and assignment;
- self-move as a no-op;
- `copy`;
- `clone`;
- user `drop`;
- reverse field cleanup;
- `adopt`, `release`, `reset`, and `swap` only if retained by the reconciled
  0.1 specification.

Acceptance:

- `sizeof(owner<T>) == sizeof(void*)`;
- straight-line owner moves require no hidden state;
- conditional aggregate moves use flags only when necessary;
- sanitizers find no backend double destruction;
- optimized operation counts match equivalent simple C++ owner code.

### 10.8 Milestone 1 completion gate

Milestone 1 is complete when:

- standalone `.huc0` programs compile and run through generated C++20;
- ownership, constructors, `drop`, `clone`, and left-to-right semantics have
  executable tests;
- the HUC0 grammar/specification match the implementation;
- diagnostics and output are deterministic;
- no HUC1 facility is required by the HUC0 compiler.

## 11. Milestone 2: HUC1 to HUC0

### 11.1 Exit result

Given `.huc1` modules, the stage translator writes a mirrored `.huc0` module
tree. `import` remains; `import2` and every phase declaration disappear.

The generated HUC0 is then accepted by the already completed Milestone 1
translator without a private compatibility mode.

### 11.2 Increment A: staged indexing and pass-through

Implement:

- `.huc1` module loading;
- `import2`;
- indexed primary families and specialization headers;
- stable declaration/family IDs;
- opaque body spans;
- pass-through of ordinary selected HUC0 syntax;
- per-module HUC0 output buffers and origin maps.

Acceptance:

- a phase-free HUC1 program expands to semantically equivalent HUC0;
- ordinary imports are preserved;
- `import2` is erased;
- skipped bodies are never sent to the lexer/parser.

### 11.3 Increment B: phase-2 interpreter

Interpret typed phase AST in process.

Values include:

- booleans and integers;
- floating values if required by phase expressions;
- immutable UTF-8 text;
- lists/tuples needed by `for1`;
- phase-2 structures;
- canonical type and stable semantic IDs as implementation values;
- symbolic runtime expressions for `fn1`.

Enforce:

- instruction limit;
- recursion limit;
- allocation limit;
- generated-byte limit;
- specialization-depth limit;
- deterministic iteration and serialization;
- no implicit filesystem, process environment, network, clock, or random
  access.

Acceptance:

- `let2`, `fn2`, and `struct2` run deterministically;
- failure includes the phase call stack;
- no host pointer is observable or used as a persistent key.

### 11.4 Increment C: phase control

Implement:

- `if1`;
- finite `for1`;
- resource-limited `while1`;
- module, type-body, function, and block insertion contexts;
- phase-only statements in declaration staging blocks;
- selected-body parsing on demand.

Acceptance:

- global and type-body selection emits declarations/members;
- local selection emits statements/declarations;
- a false branch or zero-iteration body may contain invalid HUC tokens without
  a diagnostic;
- an unmatched outer delimiter is still reported.

### 11.5 Increment D: family specialization

Implement:

- `struct1`;
- `fn1`;
- `let1`;
- `auto` and typed binders;
- type/value deduction;
- primary, partial, and full patterns;
- pure predicates;
- structural partial ordering;
- ambiguity diagnostics;
- recursion state: absent, in-progress, complete, failed;
- deterministic generated names and defining-module placement.

Acceptance:

- the same pattern system works for all three family kinds;
- only the selected body is parsed;
- recursive functions resolve through in-progress entries;
- infinitely recursive type layout fails;
- `let1` enforces exactly one correctly named residual `let`;
- function-local `let1` declarations materialize at the family site;
- generated HUC0 contains no unresolved family request.

### 11.6 Increment E: `@` and raw generation

Implement:

- compile-time evaluation islands;
- forced `fn1` evaluation;
- result materialization rules;
- the compiler-supplied `import2 compiler` module;
- `compiler::emit_huc`;
- `compiler::emit_huc_module`;
- inherited call-site insertion cursors;
- deterministic module-generation buffers.

The HUC1 translator writes raw generated text without parsing it. Chained
compilation invokes the public HUC0 translator on the written artifact.

Acceptance:

- `@` cannot appear on a type;
- nested compiler calls do not need repeated `@`;
- helpers emit at the call site, not definition site;
- raw HUC1 emitted as text is rejected by the HUC0 stage;
- diagnostics compose HUC0 location, emission call site, and phase call stack.

### 11.7 Milestone 2 completion gate

Milestone 2 is complete when:

- every numbered construct is removed from generated HUC0;
- generated modules are deterministic and independently readable;
- the public HUC0 translator accepts all successful HUC1 output;
- global/type/local phase control, specialization, and raw generation have
  golden and negative tests;
- the full two-stage driver compiles representative HUC1 programs to runnable
  C++20.

## 12. Source maps and diagnostics

Every node or emitted range carries an origin chain:

```text
C++ range
  -> HUC0 range
  -> HUC1 selected source or raw emission
  -> compile-time call site
  -> specialization request chain
```

Diagnostics should lead with the user-facing origin and show generated layers
as notes. Opaque skipped bodies have no diagnostic origin because their
contents are never parsed.

Source maps must use stable file IDs based on normalized module paths, not
process addresses.

## 13. Testing

### 13.1 Unit tests

Cover:

- tokens and numeric/string literals;
- `mod`, pointer, and owner declarators;
- parser recovery;
- canonical types;
- overload ranking;
- phase pattern matching and ordering;
- interpreter arithmetic and control flow;
- source-map composition.

### 13.2 Golden tests

Check exact:

- AST, HIR, and MIR dumps;
- HUC1-to-HUC0 module trees;
- HUC0-to-C++ module headers and root source;
- diagnostic text and source spans;
- stable generated names.

Golden tests must normalize platform-specific paths without normalizing away
semantic ordering.

### 13.3 Negative tests

Include:

- import cycles;
- malformed selected code;
- malformed but skipped code;
- phase leaks;
- move from fixed storage;
- mutation without `mod`;
- constructor obligation failures;
- ambiguous partial specialization;
- false predicates leaving no candidate;
- multiple residual declarations from `let1`;
- resource-limit exhaustion;
- numbered syntax emitted into HUC0.

### 13.4 Execute tests

Compile generated C++20 and test:

- scalar computation;
- argument side-effect ordering;
- constructors and field initialization order;
- owner observation and consumption;
- conditional moves;
- explicit copy/clone;
- user `drop` and reverse field cleanup;
- module imports;
- full HUC1-to-program examples.

Use ASan and UBSan to find compiler/backend mistakes. Tests must not treat
intentional HUC undefined behavior as a required result.

### 13.5 Fuzz and determinism tests

Fuzz:

- lexer;
- HUC0 parser;
- HUC1 delimiter scanner;
- selected-body parser;
- phase interpreter inputs;
- specialization pattern matcher.

Compile the same input repeatedly with different allocator and hash seeds and
require byte-identical HUC0/C++ artifacts.

## 14. Documentation work

Before a compiler feature is considered complete:

- rewrite `docs/language-specification.md` to use `let`, `mod`, constructors,
  HUC0/HUC1, and the finalized numbered semantics;
- replace `docs/huc.ebnf` with shared productions and separate HUC0/HUC1 start
  symbols;
- rewrite `docs/transpiler-architecture.md` around the two public libraries and
  corrected milestone order;
- update README examples;
- maintain `docs/use-cases-and-examples.md` as executable examples become
  supported;
- keep every example paired with its expected result or expansion.

Documentation and implementation disagreements are compiler bugs or
specification issues that must be resolved explicitly; the backend language is
never allowed to decide HUC semantics accidentally.

## 15. Commit strategy

The initial Git commit records:

- the current design drafts;
- this controlling plan;
- the detailed use-case guide;
- the MIT license;
- project ignore rules.

Implementation commits should be small and milestone-oriented:

1. common C++20 project skeleton;
2. HUC0 lexer/parser;
3. HUC0 semantic core;
4. HIR/MIR and scalar C++ output;
5. ownership/lifecycle;
6. HUC0 hardening;
7. HUC1 staging skeleton;
8. interpreter and phase control;
9. specialization;
10. raw generation and full pipeline.

Each commit must leave existing tests passing. Generated artifacts and IDE
state are not committed.
