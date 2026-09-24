# HUC Implementation Plan

Status: controlling implementation plan

Language name: HUC, “Hassan’s Update on C”

Implementation language: C++20

First output language: C17

License: MIT

This document records the decisions for updating the specification and building
the first two compiler milestones. If the older specification, grammar,
examples, or architecture disagree with this plan, follow this plan until
those documents are updated.

Some terms used below:

- **Bootstrap**: the first implementation, with a deliberately limited scope.
- **Concrete type or function**: a type or function whose generic arguments
  have all been resolved.
- **Residual code**: runtime code left after compile-time work. Producing it
  is called **residualization**.
- **Lowering**: turning a language operation into simpler compiler operations.
- **AST**, **HIR**, and **MIR**: the syntax tree, high-level intermediate
  representation, and mid-level intermediate representation. Each records
  more of the decisions needed to generate code.

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
C17 headers + one root translation unit
```

The implementation order is:

1. **Milestone 1: HUC0 to C17.**
2. **Milestone 2: HUC1 to HUC0.**

This order is intentional. HUC0 gives HUC1 a concrete, executable target and
requires the runtime rules to be settled before staging is added.

C17 is the first output language because HUC already needs to lower lifetime
operations and evaluation order explicitly. Plain C structures, pointers, and
generated functions express those decisions without implicit special members.
The compiler itself remains C++20. C++ interoperability and other output
backends are later work, not requirements of the first emitter. The initial
output uses ISO C17 rather than requiring compiler-specific extensions.

## 2. Product position

HUC is an unchecked systems language compiled ahead of time. It aims to offer
C++-like values, objects, and generic programming with simpler rules.

HUC is not a memory-safe Rust alternative. It deliberately permits:

- dangling and invalid raw pointers;
- null dereferences;
- unchecked indexing and pointer arithmetic;
- use of inactive source storage after relocation;
- data races;
- signed overflow and other specified undefined behavior.

The language provides deterministic cleanup for active values, but it
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

## 4. The HUC0 and HUC1 language split

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
- unary `&`, `slot_off`, and the core type-operand intrinsics;
- constructors, methods, `clone`, and `drop`;
- runtime expressions and control flow;
- local `let name: auto` inference.

HUC0 does not contain:

- numbered declarations or control-flow keywords;
- `@`;
- family binders or specialization requests;
- compiler-only values or handles;
- raw generation operations;
- unresolved `auto` parameters or inferred function results.

The angle operands in `as<T>` and `ptr_as<T*>` are built-in type operands,
not phase-family requests. They are the only angle forms accepted by HUC0.
All HUC intrinsics have unqualified names, not `std::` names; they are not
ordinary standard-library functions.

Primitive fixed arrays and `bit_as` are deferred from 0.1. Containers and
swapping can initially be supplied by libraries.

### 4.2 HUC1

HUC1 extends HUC0 with staging. A `.huc1` module may contain
unnumbered runtime code plus phase-1 and phase-2 constructs. HUC1 expansion
must produce printable `.huc0` modules.

The name HUC1 describes the staged source level. It does not mean suffix-2
constructs are excluded. Phase-2 declarations run in the compiler and are
handled by the HUC1 translator.

## 5. HUC0 runtime semantics

### 5.1 Declarations and mutability

Every local, field, and global variable begins with `let`. Function parameters
do not use `let`. Declarations use `name: Type`, with an optional `mod` before
the name for writable storage. A `mod` inside the type still grants pointee
mutation. For example, `let mod pointer: mod Widget*` has both a writable
pointer slot and a writable pointee. Method `mod` stays after the parameter list.

Local inference uses `let name: auto = expression;`; the colon and `auto` are
required. Runtime parameters, fields, and globals require explicit types.
HUC0 does not also accept type-first declarations, omitted type annotations,
or destructuring.

HUC has no `var` keyword and no `const` keyword. Values are fixed by default,
and `mod` grants mutation.

```huc
let answer: i32 = 42; // fixed scalar
let mod counter: i32; // writable scalar, initialized to zero
```

Each `mod` controls the layer where it appears:

```huc
let observer: T*; // fixed pointer field; constructor obligation
let observer: mod T*; // fixed field; mutable T; constructor obligation
let mod observer: T*; // reseatable pointer field; defaults to null
let mod observer: mod T*; // reseatable field; mutable T; defaults to null
```

These uninitialized spellings illustrate structure fields. A fixed local or
global requires an initializer. In a structure, each fixed field without a
declaration initializer must be supplied by every constructor; mutable pointer
fields omitted by a constructor default to null.

Runtime globals in HUC0 are limited to drop-free Basic scalars and raw
pointers. Fixed globals require constant initializers. Mutable globals may omit
an initializer and become zero or null. Advanced globals, structure
globals, runtime initialization code, and global destruction are deferred.

A fixed object prevents assignment to its fields and mutation of values
stored inline in those fields. A `mod T*` field still allows
mutation of the separate object it points to (the pointee). The field itself
cannot be replaced, and an inline field cannot be used to call a writable
method. This keeps each layer's permissions separate; it does not allow
mutation of fixed inline storage.

### 5.2 Raw pointers

`T*` is:

- nullable;
- non-owning;
- unchecked;
- pointer-sized;
- freely copyable;
- never automatically destroyed.

HUC performs no lifetime tracking between a raw pointer and its pointee.

### 5.3 Address-taking and storage

General aliasing-reference types do not exist. HUC0 has only the raw-pointer
type constructor `T*`; `T**` and equivalent alias-hidden forms are
diagnostics.

The non-overloadable unary `&place` produces `T*` for a fixed inline
`T` place and `mod T*` for a writable inline `T` place. It evaluates
its addressable non-pointer operand once without creating a temporary,
transferring a value, extending a lifetime, or changing cleanup state. It
rejects pointer slots, non-place results, literals, types, and function symbols.
Binary `&` remains bitwise AND.

The non-overloadable `slot_off(place) -> usize` intrinsic separately exposes
the untyped address of any addressable data slot, including raw-pointer slots.
It evaluates the place once without loading or transferring its value and
does not extend storage lifetime or update cleanup state. Fixed slots are
accepted, but rvalues are not materialized. `usize` is target-pointer-sized;
`ptr_as<T*>(address)` can reconstruct a one-level raw pointer. Byte views
use `u8*` or `c8*`; the integer does not make incompatible typed access valid.

Mutation permissions are not tracked through the integer and cast. The
programmer must preserve storage lifetime, alignment, actual permissions,
valid representation, and cleanup obligations. Writing actually fixed storage
remains undefined behavior. Ordinary typed `mod` checks remain unchanged.

Copying a raw pointer, including a null pointer, leaves the source unchanged.
It never transfers, destroys, or extends the lifetime of its pointee.
Resource-managing structures use ordinary `drop` and Advanced relocation.
Allocation and deallocation are explicit library or external operations;
HUC0 has no built-in allocating constructor expression.

Relocating a named Advanced source requires binding `mod`. Fresh unnamed
function results, evaluated `copy` results, and other temporary Advanced
values can transfer without it. Direct `T(arguments)` construction in the
destination has no temporary source place.

### 5.4 Basic and Advanced classification

Basic values are copied by ordinary binding; Advanced values are transferred.
These are category names, not source keywords or declaration modifiers.

Basic types include:

- scalar arithmetic and boolean types;
- raw pointers;
- structures whose fields are all Basic and which declare neither `clone` nor
  `drop`.

Advanced types include:

- structures containing an Advanced field;
- structures declaring `clone`;
- structures declaring `drop`.

Classification is recursive through inline fields. Having `init()` alone does
not make a structure Advanced. A raw `T*` remains Basic even when its pointee
is Advanced. Explicit `copy` uses the existing `clone()` method when
duplicating an Advanced structure.

`Advanced` is the source-level transfer category. Its operation is **destructive
relocation**, not C++ move construction. Relocating an Advanced value continues
that value in destination storage and ends the active value in the source
storage. No source `drop` or field cleanup runs.

Relocation transfers the stored representation using these fixed rules:

1. Transfer the whole value's representation to the destination without
   invoking `clone`, `drop`, or any user transfer hook.
2. Make the entire source, including all its field subobjects, inactive.
3. Make the destination responsible for cleanup; do not clean up the source.

Source fields become inactive as part of the same transfer; they receive
no recursive transfer operation or cleanup, and need not be cleared. This
rule applies whether the type is Advanced because of `clone`, `drop`, or an
Advanced field. Ordinary Basic binding duplicates the value and leaves its
source active.

A fixed field cannot be assigned to directly, but it can be transferred as
part of a whole writable source value.

The bytes of an inactive source need not be cleared or form a valid `T`.
Assigning a new value begins a new active lifetime there.

Obvious invalid use should be diagnosed, but HUC does not promise complete
flow-sensitive use-after-relocation prevention.

`copy expression` explicitly requests logical duplication:

- a Basic type is copied;
- an Advanced structure must supply a read-only `fn clone() -> T`;
- an Advanced array is copied elementwise when its element type supports `copy`;
- a raw pointer copy duplicates only the address.

Copying a null raw pointer remains valid; dereferencing a null pointer does not.

Declaring `clone` opts an otherwise Basic structure into Advanced. This
prevents an ordinary binding from bypassing custom logical-copy behavior.
`clone` must duplicate owned resources or create another valid ownership
share, so the original and its copy can be destroyed independently. It need
not copy every reachable object: non-owning raw pointers and deliberately
shared external state may remain shared.

Relocation cannot be overloaded. There is no HUC move constructor: one is not
needed to restore a destructible moved-from state because no source
object remains alive. Exact self-relocation assignment is a no-op.

An inline value that needs its address to stay unchanged cannot be safely
relocated bitwise. HUC0 does not repair self-pointers. Such a value must be
kept in directly constructed fixed read-only storage, redesigned to compute
internal addresses, or placed in stable-address storage. Mutable
address-dependent objects must likewise remain at their construction address
while those invariants are needed; HUC0 has no first-class pinning qualifier.
Relocating a value that violates this contract is undefined behavior.

Bitwise relocation is the language rule, not an optimization of separate
field transfers. It does not require a memory-copy instruction or give padding
bytes a defined meaning. The backend may copy memory, use loads and stores or
registers, or remove the transfer when the program still behaves the same.
The source must become inactive, and cleanup must still happen exactly once.

Explicit relocation is restricted to whole values tracked directly
by the compiler: named locals and parameters, and fresh temporaries or results.
Moving a whole aggregate includes its subobjects in the same operation; it does not
move them individually out of an otherwise active object. HUC0 rejects
moving an Advanced value out through `*pointer`, `pointer->field`,
`value.field`, or `array[index]`. Leaving such a subobject inactive while its
containing object remains active would make later cleanup invalid
without extra state to track which parts are still active.

Copying/cloning an indirectly reached value remains valid when it is live
and supports logical copying. An eligible root source may replace an indirect
destination; if the two can alias, exact storage identity is checked before
destination destruction. No general indirect relocation operation is part of
HUC0.

Container implementations are responsible for backing storage and initialized
element ranges. The only additional raw-storage lifetime intrinsics planned
are `construct_at` and `destruct_at`; their interfaces and detailed
semantics are deferred to later standard-library design. This does not change
ordinary relocation-source eligibility.

### 5.5 Construction and initialization

HUC0 uses constructors rather than aggregate brace initialization:

```huc
let value: Widget = Widget(7);
```

When `T(arguments)` directly initializes a local, field initializer, by-value
parameter, or `return` result, it constructs in that destination storage.
These forms have no intermediate `T` and perform no relocation; this is guaranteed
HUC behavior rather than optional backend copy elision.

A constructor is named `init`:

```huc
struct Widget {
    let id: i32;
    let mod visits: i32;

    fn init(id: i32) : id(id) {
        // visits defaults to zero
    }
}
```

An `init` body has an implicit writable `this: mod T*`. All initializer entries
and defaults complete before the body begins. The body may assign mutable
fields but cannot assign a fixed field in place of its required initializer.

Rules:

- constructor initializer entries must follow field declaration order;
- entries evaluate and initialize left-to-right;
- every fixed field must have a declaration initializer or constructor entry;
- omitted mutable scalar storage initializes to zero;
- omitted mutable pointers initialize to null;
- omitted mutable structures invoke zero-argument construction;
- the compiler generates a zero-argument `init` only when all fixed fields have
  initializers and all mutable fields can default-initialize;
- fixed locals and globals require an initializer;
- every constructor must initialize any fixed field that has no declaration
  initializer;
- constructors cannot return recoverable errors in 0.1; creation that can
  report an error will use named factory functions and library result types
  later.

### 5.6 Methods and destruction

Methods are read-only by default. A trailing `mod` grants a writable receiver:

```huc
fn value() -> i32 { ... }
fn increment() mod -> void { ... }
```

An Advanced structure may define:

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

An Advanced structure may customize explicit logical copying with exactly one
read-only method:

```huc
fn clone() -> T {
    // return an independent T without modifying this
}
```

Only an evaluated `copy` expression selects `clone`; ordinary binding never
does, and source code cannot call `clone` directly or take its address. Copy
assignment of an Advanced type is expressed as `destination = copy source`, which
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
makes combinations such as `receive(copy value, value)` deterministic: cloning
for the first parameter finishes before transfer into the second.

Earlier side effects complete before later operands begin. `&&` and `||`
short-circuit, and only the selected arm of `?:` executes.

The backend must initialize each typed by-value parameter before evaluating
the next argument. Keeping only a pointer to the source expression is not
enough, and C's unspecified argument order cannot decide the order for HUC.
Generated typed storage or a call frame may pass these already-initialized
values to the C call. Optimizers may still reorder pure computation when the
program behaves the same.

### 5.8 Runtime overloading

Ordinary `fn` overloads are selected by:

1. exact type match;
2. lossless built-in numeric promotion;
3. permission-dropping pointer conversion.

User-defined implicit conversions do not participate. Values and pointers do
not implicitly convert to each other; observing an inline value requires `&`.

### 5.9 Source syntax and formatting

The source syntax follows the same small rules at both source levels:

- Numeric literals allow optional `_` separators between digits, with no
  required grouping size. Prefixes, decimal points, exponent markers and
  signs, and type suffixes cannot touch a separator.
- Comma-separated lists allow one optional final comma, including constructor
  initializer lists. Empty lists have no comma. Line breaks do not change
  whether a trailing comma is accepted.
- Conditions and loop headers retain parentheses. Control-flow bodies require
  braces, including single-statement bodies; `else if` chains are allowed.
- Semicolons, explicit `return`, and `//` comments remain. Interpolation,
  implicit returns, and additional declaration spellings are out of scope.
- The [formatting guide](formatting.md) defines the default style, not compiler
  acceptance. Operator spacing, indentation, and line width are not grammar.

A canonical formatter is future tooling, not a new prerequisite for either
bootstrap milestone.

## 6. HUC1 phase semantics

### 6.1 Number meaning

The suffix is part of the keyword:

- no suffix: a concrete declaration that survives in HUC0;
- suffix `1`: a parameterized family or structural phase operation that
  produces HUC0;
- suffix `2`: compiler-only storage, function, or type.

These numbers say where code and values are available; they do not ask the
compiler to run an arbitrary number of passes.

### 6.2 Phase-1 families

All three family declarations put their name before the binder list. Each
binder uses `name: Type` or `name: auto`; binders are not writable slots.

The primary forms are:

```huc
struct1 Stack(T: auto) {
    ...
}

fn1 convert(T: auto)(mod value: T) -> T {
    ...
}

let1 value(T: auto) {
    let value: T = T();
}
```

Partial specialization adds an angle-bracket pattern after the binder list,
followed by an optional parenthesized predicate:

```huc
struct1 Stack(T: auto)<T*>(predicate) { ... }

fn1 convert(T: auto)<T*>(predicate)(value: T*) -> T* { ... }

let1 value(T: auto)<T*>(predicate) {
    let mod value: T* = null;
}
```

Full specialization uses an empty binder list:

```huc
struct1 Stack()<Widget> { ... }
fn1 convert()<Widget>(mod value: Widget) -> Widget { ... }
let1 value()<Widget> { let value: Widget = Widget(); }
```

Requests use angle brackets:

```huc
let stack: Stack<i32> = Stack<i32>();
let result: i32 = convert<i32>(input);
use(value<i32>);
```

`<` and `>` are separate tokens. When the parser cannot tell whether they
form a family request or a comparison, it keeps both possibilities until name
resolution can decide.

### 6.3 Binders and deduction

`T: auto` binds a compile-time entity whose kind is inferred from use. A binder
used in a type position must hold a type. Typed non-type parameters use normal
types:

```huc
struct1 Buffer(T: auto, N: usize) { ... }
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

The compiler must not report token, name, or type errors inside a skipped
body. It may report only that it cannot find the body's closing delimiter.
If a body is selected, it is then lexed and parsed normally.

### 6.6 `let1`

A `let1` header has no result type. Every selected path must leave:

- exactly one ordinary runtime `let`;
- with the same identifier as the family;
- with no other residual runtime declaration or statement.

Compiler-only setup and `if1`/`for1`/`while1` are permitted. Raw emission is
not permitted inside a `let1` selection body because it would obscure the
rule that exactly one variable must remain.

Different specializations may choose different:

- value types;
- inline values versus raw-pointer forms;
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

`for1` spells its header `for1 (let2 item: Type in values)`. The binding may
use `mod` before its name or `: auto` for inference. The keyword `in` is
reserved for this role; it adds no runtime range-loop or membership operation.
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

`@` starts compile-time evaluation of an expression. This is called an
evaluation island: the marked expression runs in the compiler even when the
surrounding code is runtime code.

It is never attached to a type:

```huc
let2 function: compiler::Function =
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
result that can be written as a HUC0 value (a materializable result).

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
@compiler::emit_huc("let mod generated: i32;");
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

The future API should provide read-only snapshots of checked code and stable
IDs, not direct access to the compiler's internal AST objects. Structured edits
and commits will be explicit. Queries can list concrete runtime declarations
and family specializations already produced, not all the possible members of
an infinite family.

Reserve `@this` for a later contextual-reflection proposal. The intended
direction is the narrowest enclosing handle: `compiler::Function` in a
function/method/block, a class or type-declaration handle in a type body, and
`compiler::Module` at module scope. Broader context is reached through that
handle rather than injected as a universal environment. Bootstrap HUC1 must
diagnose `@this`; exact APIs and dependency semantics are a separate design.

## 8. Compiler organization

All compiler and command-line driver code is ISO C++20. The first generated
output is ISO C17; implementation and output languages are separate choices.

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
  huc_runtime.h
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
  lowering, and C17 emission;
- `huc1`: HUC1 indexing, deferred-body spans, phase checking, interpreter,
  specialization, and HUC0 residualization;
- `huc`: thin command-line driver.

`huc1` may reuse HUC0 parsing and type-resolution facilities for selected
runtime code and `fn1` deduction. It must not bypass the textual HUC0 boundary
when chaining the translators.

## 9. Public compiler interfaces

The architecture document will define value-oriented C++20 APIs equivalent to:

```cpp
namespace huc {

struct Huc0TranspileRequest {
    std::string root_module;
    std::vector<std::filesystem::path> import_roots;
    std::filesystem::path output_directory;
    TargetConfig target;
    bool write_source_maps = true;
};

struct Huc0TranspileResult {
    std::vector<Artifact> artifacts;
    std::vector<Diagnostic> diagnostics;
    DependencyGraph dependencies;
    bool succeeded = false;
};

[[nodiscard]] Huc0TranspileResult
huc0_transpile(const Huc0TranspileRequest& request);

struct Huc1ExpandRequest {
    std::string root_module;
    std::vector<std::filesystem::path> import_roots;
    std::filesystem::path output_directory;
    TargetConfig target;
    PhaseLimits limits;
    bool write_source_maps = true;
};

struct Huc1ExpandResult {
    std::vector<Artifact> modules;
    std::vector<Diagnostic> diagnostics;
    DependencyGraph dependencies;
    bool succeeded = false;
};

[[nodiscard]] Huc1ExpandResult
huc1_expand(const Huc1ExpandRequest& request);

} // namespace huc
```

Exact container choices may change, but ownership, inputs, outputs, and
RAII-managed results are fixed. Ordinary user-program errors are returned as
diagnostics rather than thrown as exceptions.

The driver commands are:

```text
huc lower <root.huc0> --out-dir <c-dir>
huc stage <root.huc1> --out-dir <huc0-dir>
huc build <root.huc1> --out-dir <build-dir>
```

`lower` is implemented first. `stage` is implemented in Milestone 2. `build`
chains `stage` and `lower` through files, not an undocumented AST shortcut.

## 10. Milestone 1: HUC0 to C17

### 10.1 Exit result

Given an acyclic `.huc0` module tree, the translator emits:

```text
build/c/
  huc_runtime.h
  app/main.huc.h
  app/main.huc.c
  app/main.huc.map
  dependency/module.huc.h
```

Each HUC module becomes a guarded generated header. A runtime import becomes:

```c
#include "dependency/module.huc.h"
```

The root `.c` includes the root header. Only the root `.c` is compiled, so
the initial backend produces one C17 translation unit. Module names and HUC
overloads become deterministic, collision-free C identifiers.

### 10.2 Increment A: project and source infrastructure

Implement:

- CMake C++20 targets for the compiler and driver;
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
- name-first `let name: Type` declarations and `mod name: Type` parameters;
- type aliases;
- fixed-signature `extern "C"` declarations;
- `struct`;
- `fn`;
- `init` and `drop`;
- `T*` and independent binding/pointee `mod` permissions;
- required braces for control-flow bodies, including `else if` chains;
- optional numeric digit separators and consistent optional trailing commas;
- expressions and precedence;
- constructor syntax;
- local `let name: auto`;
- error recovery at declaration and statement boundaries.

The lexer emits individual `<` and `>` tokens and indivisible numbered keyword
tokens, even though HUC0 rejects numbered forms.

Acceptance:

- every normative HUC0 example parses;
- AST dumps are deterministic;
- malformed inputs produce multiple useful diagnostics without crashing;
- HUC1-only syntax is explicitly rejected as unavailable in HUC0;
- old declaration spellings, malformed separators, empty comma items,
  and unbraced control-flow bodies receive syntax diagnostics;
- spacing and indentation choices do not change acceptance when tokens remain
  the same; a formatter is not needed to parse the examples.

### 10.4 Increment C: names and types

Implement:

- module, type, function, field, parameter, and local scopes;
- collect-before-check for module declarations;
- declaration-before-use for locals;
- canonical built-in, structure, and raw-pointer types;
- canonical type aliases and alias-cycle diagnostics;
- local `auto` inference;
- method receiver permissions;
- constructor obligation checking;
- `extern "C"` signatures limited to built-in scalars, `void` returns, and
  one-level raw pointers to built-in types, with no structure, array,
  phase type, or variadic boundary;
- overload resolution using only specified built-in conversions;
- no user-defined implicit conversions.

Acceptance:

- type and name errors point to both use and candidate declaration;
- `mod` cannot be gained implicitly;
- fixed storage cannot be assigned through ordinary typed operations;
- fixed Advanced values cannot be relocated;
- Advanced subobjects cannot be explicit relocation sources;
- all constructor field obligations are checked.
- every accepted C-linkage signature has a documented target ABI mapping.

### 10.5 Increment D: typed HIR and sequenced MIR

HIR records:

- canonical type;
- resolved symbol;
- value use: read, observe, copy, relocate, or place;
- unary `&` of an eligible inline place;
- untyped `slot_off` and explicit `usize`-to-raw-pointer reconstruction;
- source origin;
- explicit conversion selected by HUC.

MIR records:

- ordered temporaries;
- places and assignments;
- calls and control-flow blocks;
- active/inactive Advanced state;
- cleanup edges;
- conditional drop flags only where control flow requires them;
- whole-source deactivation without clearing source bytes.

No semantic decision is deferred to the C compiler or its ABI lowering.

Acceptance:

- HIR/MIR dumps expose every relocation and observation;
- argument and operand sequencing is explicit;
- cleanup is exactly once on every normal control-flow exit.

### 10.6 Increment E: scalar C17 backend

Emit:

- deterministic generated identifiers;
- mangled identifier prefixes for modules and overloads;
- guarded headers;
- the root source file;
- a C `int main(void)` adapter for the configured root module's sole HUC
  `fn main() -> i32`;
- scalar and raw-pointer declarations;
- fixed-signature C-linkage declarations;
- structures, functions, and runtime control flow;
- explicit temporaries preserving left-to-right evaluation;
- generated free functions for methods and constructor initialization, using
  explicit receivers and final-destination pointers where needed;
- `#line` or sidecar source-map information where practical.

Generated C is an implementation artifact, not HUC's semantic authority. HUC
fixedness is checked before emission; physical C storage must remain writable
where compiler-generated relocation or reinitialization requires it. Lowering
must respect C alignment, effective-type, aliasing, and arithmetic rules.

Acceptance:

- scalar examples compile with a selected C compiler in ISO C17 mode;
- execute tests match expected output and side-effect order;
- generated output is stable across repeated builds.

### 10.7 Increment F: value transfer and lifecycle

MIR determines cleanup edges, active/inactive value state, and conditional
drop flags. Emit cleanup on normal exits, early returns, loop exits, and
replacement of active destinations. An inactive source receives no cleanup
and need not be cleared. C scope exit does not perform HUC cleanup.

Implement:

- structural Basic/Advanced classification, including `clone` opting into Advanced;
- exact self-relocation as a no-op;
- `copy` and `clone`;
- clone-then-replace assignment;
- fixed bitwise relocation of Advanced values with no user transfer hook;
- user `drop`;
- reverse field cleanup;
- raw-pointer copying without pointee cleanup.

Acceptance:

- straight-line relocations require no hidden state;
- conditional relocations use flags only when necessary;
- raw-handle `drop` types relocate without duplicate cleanup;
- Advanced aggregates lower to whole-value `Relocate` without recursive
  field transfers or source clearing;
- copying fixed or writable raw pointers leaves their addresses unchanged;
- `auto` from a raw pointer stays raw, with no cleanup of its pointee;
- indirect Advanced move-out is rejected;
- ordinary binding never invokes `clone`;
- sanitizers find no backend double destruction;
- optimized operation counts match equivalent explicit C resource-management code.

### 10.8 Milestone 1 completion gate

Milestone 1 is complete when:

- standalone `.huc0` programs compile and run through generated C17;
- value transfer, constructors, `drop`, `clone`, and left-to-right semantics have
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

A cross-module request in 0.1 may use only types the defining module can
already name through its direct or indirect imports. Placing specializations
in the requesting module is deferred.

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
its cache entry keeps the generated runtime declaration and ordered raw
fragments.
Local/type fragments enter the declaration, while module fragments enter the
defining module buffer once. Repeated normal requests only reference it.
An actual `fn2` call, forced `@fn1` call, or standalone phase expression that
reaches raw emission, directly or through helpers, runs each time it is
evaluated. It uses that call's inherited insertion cursor. Reusing only a
cached return value would lose the generated output and is not allowed.

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
  programs through generated C17.

## 12. Source maps and diagnostics

Every node or emitted range carries an origin chain:

```text
C range
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
- optional digit separators in every supported base and numeric component;
- name-first declarations, pointer types, list commas, and mandatory control-flow braces;
- `mod` and pointer declarators;
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
- HUC0-to-C17 module headers and root source;
- diagnostic text and source spans;
- stable generated names.

Golden tests must normalize platform-specific paths without hiding differences
in ordering that affect program behavior.

### 13.3 Negative tests

Include:

- import cycles;
- malformed selected code;
- malformed but skipped code;
- phase leaks;
- relocation from fixed storage;
- relocation from a pointee, field, or array element;
- unary `&` applied to a pointer slot, non-place result, or function symbol;
- `slot_off` applied to a literal, temporary value, or function symbol;
- attempts to use former address-taking or `std::` intrinsic spellings;
- `ptr_as` with an invalid source type or a composed pointer target;
- mutation without `mod`;
- copying an Advanced type without `clone`;
- malformed or duplicate `clone` and `drop` methods;
- attempted custom move or relocation hooks;
- constructor obligation failures;
- ambiguous partial specialization;
- false predicates leaving no candidate;
- multiple residual declarations from `let1`;
- resource-limit exhaustion;
- numbered syntax emitted into HUC0.
- invalid C-linkage structures, arrays, phase types, pointer chains,
  opaque handles, or variadics.

### 13.4 Execute tests

Compile generated C17 and test:

- typed address-taking and untyped `slot_off` addresses with side-effecting
  place expressions evaluated once;
- scalar computation;
- argument side-effect ordering;
- constructors and field initialization order;
- raw-pointer observation and Advanced-value consumption;
- slot addresses for inline, pointer, and fixed storage;
- slot-address round trips and byte access without extra ownership operations;
- conditional relocations;
- Basic-value binding and assignment;
- explicit copy/clone and clone-then-replace assignment;
- copying null raw pointers;
- bitwise relocation, whole-source deactivation, and self-relocation;
- relocation of nested Advanced aggregates with exactly-once destination
  cleanup and no inactive-source cleanup;
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
require byte-identical HUC0/C artifacts.

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

Disagreements between documentation and implementation must be resolved as
compiler bugs or specification issues. The backend language must never decide
HUC behavior by accident.

## 15. Commit strategy

The design commits record:

- the current design drafts;
- this controlling plan;
- the detailed use-case guide;
- the MIT license;
- project ignore rules.

Implementation commits should be small and milestone-oriented:

1. common C++20 project skeleton;
2. HUC0 lexer/parser;
3. HUC0 semantic core;
4. HIR/MIR and scalar C17 output;
5. value transfer and lifecycle;
6. HUC0 hardening;
7. HUC1 staging skeleton;
8. interpreter and phase control;
9. specialization;
10. raw generation and full pipeline.

Each commit must leave existing tests passing. Generated artifacts and IDE
state are not committed.
