# HUC Implementation Plan

Status: controlling implementation plan

Language name: HUC, “Hassan’s Update on C”

Implementation language: C17

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
    | huc1_expand
    v
printable per-module HUC0 source
    |
    | huc0_transpile
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
- use of inactive source storage after relocation;
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
- type aliases and fixed-signature C declarations;
- `let` declarations;
- `mod` permissions;
- raw pointers `T*`;
- unique owners `T&`;
- `addressof` and the core type-operand intrinsics;
- constructors, methods, `clone`, and `drop`;
- runtime expressions and control flow;
- local `let auto` inference.

HUC0 does not contain:

- numbered declarations or control-flow keywords;
- `@`;
- family binders or specialization requests;
- compiler-only values or handles;
- raw generation operations;
- unresolved `auto` parameters or inferred function results.

The angle operands in `as<T>`, `ptr_as<T*>`, and `adopt<T>` are
built-in type operands, not phase-family requests. They are the only angle
forms accepted by HUC0.

Primitive fixed arrays, `bit_as`, and primitive owner `swap` are deferred from
0.1. Containers and swapping can initially be supplied by libraries.

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
let T* observer;           // fixed pointer field; constructor obligation
let mod T* observer;       // fixed field; mutable T; constructor obligation
let T* mod observer;       // reseatable pointer field; defaults to null
let mod T* mod observer;   // reseatable field; mutable T; defaults to null

let T& owner;              // fixed owner field; constructor obligation
let mod T& owner;          // fixed field; mutable T; constructor obligation
let T& mod owner;          // movable/reseatable owner field; defaults to null
let mod T& mod owner;      // reseatable field; mutable T; defaults to null
```

These uninitialized spellings illustrate structure fields. A fixed local or
global requires an initializer. In a structure, each fixed field without a
declaration initializer must be supplied by every constructor; mutable pointer
and owner fields omitted by a constructor default to null.

Runtime globals in HUC0 are limited to drop-free Copy scalars and raw
pointers. Fixed globals require constant initializers. Mutable globals may omit
an initializer and become zero or null. Move globals, owners, structure
globals, runtime initialization code, and global destruction are deferred.

A fixed containing object prevents assignment to one of its field slots and
prevents writable access to an inline field subobject. It does not erase
permission explicitly carried through a pointer or owner field to a separate
pointee. Thus a read-only structure may follow a `mod T*` or `mod T&` field and
mutate that `T`, while it still cannot reseat the field or call a writable
method on an inline field. This is the layer-specific behavior shown above,
not an interior-mutation escape for inline storage.

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

`T&` is a primitive unique owner, not a C++ reference declarator. General
aliasing-reference types do not exist, and HUC has no `T&&` type. An owner may
be null even though the same spelling denotes a reference in C++.

HUC0 has no unary `&` expression. The non-overloadable
`addressof(inline_place)` intrinsic produces `T*` for a fixed inline `T` place
and `mod T*` for a writable inline `T` place. It rejects raw-pointer and owner
slots because their addresses would require a forbidden composed type.

HUC0 has only the raw-pointer constructor `T*` and the unique-owner constructor
`T&`. Neither constructor composes: `T**`, `T*&`, `T&*`, `T&&`, and equivalent
alias-hidden forms are diagnostics. Writing an owner expression in a `T*`
observer context accesses the owned pointee; it does not take the owner slot's
address. Direct owner-slot reseating therefore uses consume-and-return rather
than an aliasing owner reference. Binary `&` remains bitwise AND.

The default representation is one address-sized word:

- null owns nothing;
- non-null owns exactly one dynamically allocated `T`;
- destruction drops the pointee and deallocates its storage;
- relocation copies the address and clears the source;
- owner arithmetic is prohibited;
- observation as a compatible raw pointer is implicit;
- conversion from raw pointer to owner is explicit.

Relocating from a named source requires a writable source slot because
relocation modifies the source.

```huc
let Widget& mod source = new Widget(7);
let Widget& destination = source; // relocate; source becomes null
```

Attempting to relocate from a fixed owner is a compile-time diagnostic.

Fresh unnamed result places are intrinsically consumable. This includes
function results, `new` owner results, `copy`/`clone` results, and other
temporary Move values. They may bind onward without a source-level trailing
`mod`; only named source storage is checked for that permission. A direct
`T(arguments)` destination construction has no temporary result place.

### 5.4 Copy and Move classification

Copy types include:

- scalar arithmetic and boolean types;
- raw pointers;
- structures whose fields are all Copy and which declare neither `clone` nor
  `drop`.

Move types include:

- owners;
- structures containing a Move field;
- structures declaring `clone`;
- structures declaring `drop`.

`Move` is the source-level transfer category. Its operation is **destructive
relocation**, not C++ move construction. Relocating a non-owner value continues
that value in destination storage and ends the active value in the source
storage. No source `drop` or field cleanup runs.

Structural relocation proceeds in declaration order:

1. Visit each field in declaration order.
2. Copy a Copy field; recursively relocate a Move field.
3. After the final field, make the aggregate source place inactive.

Field fixedness constrains ordinary assignment, not compiler relocation of a
whole value. A writable containing source permits relocation of its fixed
fields.

The bytes of an inactive source need not be cleared or form a valid `T`.
Assigning a new value begins a new active lifetime there. A primitive owner is
the deliberate exception: relocating `T&` transfers its address and leaves the
source as a usable null owner.

Obvious invalid use should be diagnosed, but HUC does not promise complete
flow-sensitive use-after-relocation prevention.

`copy expression` explicitly requests logical duplication:

- a Copy type is copied;
- a non-null owner allocates and copies its pointee;
- a Move structure must supply a read-only `fn clone() -> T`;
- a Move array is copied elementwise when its element type supports `copy`;
- a raw pointer copy duplicates only the address.

Declaring `clone` opts an otherwise fieldwise-Copy structure into Move. This
prevents an ordinary binding from bypassing custom logical-copy behavior.
The `clone` contract is deep across the type's logical ownership boundary:
owned resources are duplicated or given another valid ownership share so the
two results can be destroyed independently. It is not a graph-wide copy;
non-owning raw pointers and deliberately shared external state may remain
shared.

Relocation cannot be overloaded. There is no HUC move constructor: one is not
needed to restore a destructible moved-from state because no non-owner source
object remains alive. Exact self-relocation assignment is a no-op.

An inline type whose invariant depends on its own address is not structurally
relocatable. HUC0 does not silently repair self-pointers. Such a value must be
kept in directly constructed fixed read-only storage, redesigned to compute
internal addresses, placed in a stable-address library container, or allocated
behind `T&` so relocation transfers only the owner word. Mutable
address-dependent objects should normally use `T&`; HUC0 has no first-class
pinning qualifier. Relocating a value that violates this contract is undefined
behavior.

The backend may replace structural relocation with a representation transfer
only when it proves the result equivalent. This is an optimization; it does not
create a user customization point.

Explicit non-owner relocation is restricted to entire compiler-tracked root
places: named locals and parameters; fresh temporary/result places; and
compiler-generated recursive relocation of a whole aggregate. HUC0 rejects
moving a non-owner Move value out through `*pointer`, `pointer->field`,
`value.field`, or `array[index]`. Leaving such a subobject inactive while its
owner or containing object remains active would make later cleanup invalid
without storing additional liveness state.

Primitive `T&` subobjects are exempt because relocation writes the source
owner to its active null representation. Copying/cloning an indirectly reached
value also remains valid. An eligible root source may replace an indirect
destination; if the two can alias, exact storage identity is checked before
destination destruction. A future compiler-backed raw-storage intrinsic may
serve container implementations, but no general indirect relocation operation
is part of HUC0.

### 5.5 Construction and initialization

HUC0 uses constructors rather than aggregate brace initialization:

```huc
let Widget value = Widget(7);
let Widget& mod owner = new Widget(7);
```

When `T(arguments)` directly initializes a local, field initializer, by-value
parameter, or `return` result, it constructs in that destination storage.
`new T(arguments)` constructs directly in the final allocation. These
forms have no intermediate `T` and perform no relocation; this is guaranteed
HUC behavior rather than optional backend copy elision.

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

An `init` body has an implicit writable `mod T* this`. All initializer entries
and defaults complete before the body begins. The body may assign mutable
fields but cannot assign a fixed field in place of its required initializer.

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

Fixed arrays destroy active elements in decreasing index order. Elementwise
copy and construction track an initialized prefix; a normal edge that abandons
that prefix cleans it from its highest active index down to zero.

A Move structure may customize explicit logical copying with exactly one
read-only method:

```huc
fn clone() -> T {
    // return an independent T without modifying this
}
```

Only an evaluated `copy` expression selects `clone`; ordinary binding never
does, and source code cannot call `clone` directly or take its address. Copy
assignment of a Move type is expressed as `destination = copy source`, which
clones to a temporary, destroys the old destination, and relocates the
temporary into place. Users cannot customize relocation initialization or
relocation assignment.

The complete rules, diagnostics, and side-by-side C++20 examples are in
[Value Semantics and Special Operations](value-semantics.md).

HUC 0.1 has no exception unwinding. Process termination through panic need not
run pending cleanup.

### 5.7 Evaluation order

HUC evaluation is left-to-right for:

- operands;
- call arguments;
- constructor arguments;
- initializer expressions;
- aggregate elements if aggregate syntax is added later.

For a call, each argument expression and its corresponding parameter
initialization complete before evaluation of the next argument begins. This
makes combinations such as `receive(owner, owner)` deterministic: observation
for the first parameter occurs before ownership transfer into the second.

Earlier side effects complete before later operands begin. `&&` and `||`
short-circuit, and only the selected arm of `?:` executes.

The backend must materialize each argument's typed by-value parameter value
before evaluating the next argument, rather than merely retaining expression
aliases or inheriting C++20's unspecified choice of argument order. Generated
typed holders or an explicit call frame may transport those already-bound
values into the C++ ABI call. Optimizers may still reorder pure computation
when the change is not observable.

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

fn1 convert(auto T)(T mod value) -> T {
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
    let T* mod value = null;
}
```

Full specialization uses an empty binder list:

```huc
struct1 Stack()<Widget> { ... }
fn1 convert()<Widget>(Widget mod value) -> Widget { ... }
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

`struct1` is module-scope in 0.1. `fn1` is allowed at module or type-member
scope. `let1` is allowed at module, function, or nested-block scope and never
as a field. Local `fn1`/`struct1` and nested type families are deferred.

The primary must precede its specializations. Initially, every specialization
must be declared in the defining module.

Any family may be requested from an importing module. Each generated concrete
declaration is emitted in its defining HUC0 module, and residual uses stay
qualified through the preserved import. HUC 0.1 has no module visibility.

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

A selected module/type body may also contain ordinary compiler-only
expressions and control statements, such as updating a `let2` loop index.
Their operands must be compiler-available and the statements erase. The
selected-body parser uses the inherited module, type, function, or block
context; skipped bodies remain opaque.

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

A standalone `@expression;` is an erased phase-expression item. It is legal at
module, type-body, function, and nested-block scope, must evaluate completely
at translation time, and must return `void`. Any raw-generation effect uses
the insertion cursor at that exact call site.

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
`emit_huc_module` appends a raw fragment containing module-scope declarations
to the call-site module’s generated declaration buffer. The HUC0 translator
enforces declaration boundaries because HUC1 does not parse the fragment.

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

The eventual compiler library is exposed through a versioned, inspectable
generated interface unit that the HUC1 frontend loads before user modules—the
language-level equivalent of an always-included header. `import2 compiler`
binds that already loaded interface under the `compiler` name. The bootstrap
interface declares only the two raw-emission operations; later versions add
the public handle types above without exposing internal C compiler objects.

The future API should expose immutable semantic snapshots and stable IDs,
rather than internal compiler AST objects. Structured edits and commits will be
explicit. Only concrete phase-0 declarations and materialized family
specializations are enumerable; an infinite uninstantiated family is not.

Reserve `@this` for a later contextual-reflection proposal. The intended
direction is the narrowest enclosing handle: `compiler::Function` in a
function/method/block, a class or type-declaration handle in a type body, and
`compiler::Module` at module scope. Broader context is reached through that
handle rather than injected as a universal environment. Bootstrap HUC1 must
diagnose `@this`; exact APIs and dependency semantics are a separate design.

## 8. Compiler organization

All compiler and command-line driver code is ISO C17. C++20 is the language of
the first generated backend, not the implementation language of the compiler.
The compiler must not require a C++ compiler to build itself.

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

The architecture document will define value-oriented C APIs equivalent to:

```c
#include <stdbool.h>
#include <stddef.h>

typedef struct Huc0TranspileRequest {
    const char *root_module;
    const char *const *import_roots;
    size_t import_root_count;
    const char *output_directory;
    HucTargetConfig target;
    bool write_source_maps;
} Huc0TranspileRequest;

typedef struct Huc0TranspileResult {
    HucArtifactArray artifacts;
    HucDiagnosticArray diagnostics;
    HucDependencyGraph dependencies;
    bool succeeded;
} Huc0TranspileResult;

Huc0TranspileResult huc0_transpile(const Huc0TranspileRequest *request);
void huc0_transpile_result_dispose(Huc0TranspileResult *result);
```

```c
typedef struct Huc1ExpandRequest {
    const char *root_module;
    const char *const *import_roots;
    size_t import_root_count;
    const char *output_directory;
    HucTargetConfig target;
    HucPhaseLimits limits;
    bool write_source_maps;
} Huc1ExpandRequest;

typedef struct Huc1ExpandResult {
    HucArtifactArray modules;
    HucDiagnosticArray diagnostics;
    HucDependencyGraph dependencies;
    bool succeeded;
} Huc1ExpandResult;

Huc1ExpandResult huc1_expand(const Huc1ExpandRequest *request);
void huc1_expand_result_dispose(Huc1ExpandResult *result);
```

Exact array and allocator APIs may change, but ownership, inputs, outputs, and
explicit result disposal are fixed. The public C API never uses `longjmp` for
ordinary errors. Fatal internal invariant violations may terminate a debug
build; user program errors are diagnostics.

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

- CMake C17 targets for the compiler and driver;
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
- `let`;
- type aliases;
- fixed-signature `extern "C"` declarations;
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
- canonical built-in, structure, raw-pointer, and owner types;
- canonical type aliases and alias-cycle diagnostics;
- local `auto` inference;
- method receiver permissions;
- constructor obligation checking;
- `extern "C"` signatures limited to built-in scalars, `void` returns, and
  one-level raw pointers to built-in types, with no owner, structure, array,
  phase type, or variadic boundary;
- overload resolution using only specified built-in conversions;
- no user-defined implicit conversions;
- prohibition of ownership-only overload sets.

Acceptance:

- type and name errors point to both use and candidate declaration;
- `mod` cannot be gained implicitly;
- fixed storage cannot be assigned;
- fixed owners cannot be relocated;
- non-owner Move subobjects cannot be explicit relocation sources;
- all constructor field obligations are checked.
- every accepted C-linkage signature has a documented target ABI mapping.

### 10.5 Increment D: typed HIR and sequenced MIR

HIR records:

- canonical type;
- resolved symbol;
- value use: read, observe, copy, relocate, or place;
- explicit `addressof` of an eligible inline place;
- source origin;
- explicit conversion selected by HUC.

MIR records:

- ordered temporaries;
- places and assignments;
- calls and control-flow blocks;
- active/inactive Move state;
- cleanup edges;
- conditional drop flags only where control flow requires them;
- owner nulling after relocations.

No semantic decision is deferred to C++ overload resolution.

Acceptance:

- HIR/MIR dumps expose every relocation and observation;
- argument and operand sequencing is explicit;
- cleanup is exactly once on every normal control-flow exit.

### 10.6 Increment E: scalar C++20 backend

Emit:

- deterministic generated identifiers;
- module namespaces;
- guarded headers;
- the root source file;
- a global C++ `int main()` adapter for the configured root module's sole HUC
  `fn main() -> i32`;
- scalar and raw-pointer declarations;
- fixed-signature C-linkage declarations;
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
- can release, reset, observe, and relocate;
- invokes generated HUC drop logic before deallocation;
- has size/alignment assertions.

MIR remains authoritative for inactive aggregate state and conditional cleanup.
The backend clears/reset fields after explicit cleanup so later C++ storage
destruction is a no-op rather than a second HUC destruction.

Implement:

- `new T(args)`;
- owner observation;
- owner relocation initialization and assignment;
- structural Copy/Move classification, including `clone` opting into Move;
- exact self-relocation as a no-op;
- `copy`;
- `clone`;
- clone-then-replace assignment;
- fixed structural relocation with no user relocation hook;
- user `drop`;
- reverse field cleanup;
- `adopt`, `release`, and `reset`.

`adopt<T>(raw)` has an unchecked compatible-allocation and exclusivity
precondition. `release(owner)` and `reset(owner)` require a named owner slot
with trailing `mod`.

Acceptance:

- `sizeof(owner<T>) == sizeof(void*)`;
- straight-line owner relocations require no hidden state;
- conditional aggregate relocations use flags only when necessary;
- raw-handle `drop` types relocate without duplicate cleanup;
- indirect non-owner move-out is rejected without adding owner liveness state;
- ordinary binding never invokes `clone`;
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
- deterministic generated names and defining-module placement;
- cross-module qualification for requested concrete specializations;
- a deterministic whole-graph request worklist that reaches a fixpoint.

A cross-module request in 0.1 may use only types already nameable through the
defining module's import closure. Requester-side specialization placement is
deferred.

Acceptance:

- the same pattern system works for all three family kinds;
- only the selected body is parsed;
- recursive functions resolve through in-progress entries;
- infinitely recursive type layout fails;
- `let1` enforces exactly one correctly named residual `let`;
- function-local `let1` declarations materialize at the family site;
- generated HUC0 contains no unresolved family request;
- each module output depends on the sorted closure of specialization keys
  assigned to that defining module.

### 11.6 Increment E: `@` and raw generation

Implement:

- compile-time evaluation islands;
- erased module/type/function/block phase-expression items;
- forced `fn1` evaluation;
- result materialization rules;
- the compiler-supplied `import2 compiler` module;
- `compiler::emit_huc`;
- `compiler::emit_huc_module`;
- inherited call-site insertion cursors;
- deterministic module-generation buffers;
- pure/effectful phase-call classification.

The HUC1 translator writes raw generated text without parsing it. Chained
compilation invokes the public HUC0 translator on the written artifact.
A normal family specialization is constructed once per specialization key;
its cache entry retains the residual declaration and ordered raw fragments.
Local/type fragments enter the declaration, while module fragments enter the
defining module buffer once. Repeated normal requests only reference it.
Actual phase-execution occurrences—`fn2` calls, forced `@fn1` calls, and
standalone phase-expression items—whose transitive body reaches raw emission
execute at every occurrence with that occurrence's inherited cursor and are
not served from a bare result cache.

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
- relocation from fixed storage;
- non-owner relocation from a pointee, field, or array element;
- `addressof` applied to a pointer or owner slot;
- mutation without `mod`;
- copying a Move type without `clone`;
- malformed or duplicate `clone` and `drop` methods;
- attempted custom move or relocation hooks;
- constructor obligation failures;
- ambiguous partial specialization;
- false predicates leaving no candidate;
- multiple residual declarations from `let1`;
- resource-limit exhaustion;
- numbered syntax emitted into HUC0.
- invalid C-linkage owners, structures, arrays, phase types, pointer chains,
  opaque handles, or variadics.

### 13.4 Execute tests

Compile generated C++20 and test:

- scalar computation;
- argument side-effect ordering;
- constructors and field initialization order;
- owner observation and consumption;
- conditional relocations;
- Copy-value binding and assignment;
- explicit copy/clone and clone-then-replace assignment;
- structural relocation, source deactivation, and self-relocation;
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

The design commits record:

- the current design drafts;
- this controlling plan;
- the detailed use-case guide;
- the MIT license;
- project ignore rules.

Implementation commits should be small and milestone-oriented:

1. common C17 project skeleton;
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
