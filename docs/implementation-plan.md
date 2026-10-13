# HUC Implementation Plan

Status: controlling implementation plan

Language name: HUC, “Hassan’s Update on C”

Implementation language: C++20

First backend target: LLVM IR

License: MIT

This document controls implementation scope and sequencing. The language
specification defines semantics; this plan records how much of that design
the compiler implements and how it is built. The current direction replaces
the earlier C17 backend and mandatory HUC0-first milestone order.

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
printable flattened HUC0 source
    |
    | huc0_compile
    v
LLVM IR (.ll / .bc)
    |
    | LLVM optimization and target code generation
    v
object files -> linker -> executable
```

### 1.1 Specification and release scope

Make the specification as complete as possible rather than limiting design
work to what the first compiler can implement. Release 0.1 may support only
parts of that specification. No complete release feature list is fixed yet.
Keep an implementation-support list for each translator and target, identifying
what is specified, implemented, and tested. Unsupported features need explicit
diagnostics; implementing only part of the language does not permit changing
the semantics of supported features.

Existing deferred/proposed features still need their designs completed and
adopted. They may be specified before implementation reaches them. Neither
the full reflection design nor any other unfinished proposal is automatically
enabled by this planning change.

### 1.2 Implementation routes

The recommended first milestone is a thin end-to-end slice. Two routes remain
available; neither requires implementing the entire specification at once:

- **End to end (recommended):** implement both translators and the LLVM backend
  for a small, coherent feature slice. Start with scalar types, functions,
  arithmetic, branches, one fixed-signature C call, and an ordinary function
  tested both through `@` and as compiled code. This exercises shared MIR, the
  entry point, foreign ABI, and linking early.
- **Staging first:** implement a selected HUC1-to-HUC0 subset, with readable
  output and golden tests. Evaluating ordinary functions still requires shared
  HUC0 parsing, semantic analysis, HIR, and sequenced MIR for those features.
  This route postpones LLVM emission and native execution, not the shared
  semantic work. A pass-through-only demonstration can start with less, but
  does not demonstrate ordinary-function evaluation.

Sections 10 and 11 describe workstreams with shared dependencies, not independent
implementations or an enforced order. The interpreter in section 11.3 depends
on the frontend and HIR/MIR in sections 10.2 through 10.5 for its supported
subset; it does not depend on LLVM emission in section 10.6. Both routes retain
independently usable HUC0 and HUC1 translators and the written HUC0 boundary.
Choose the exact release subset when implementation begins. Matching `@` and
native results is useful differential testing, not an independent correctness
oracle: both paths can share a MIR bug. Keep separately specified expected
results and independently compiled C ABI fixtures too.

Both translators and the driver remain C++20 programs. HUC0 lowers directly
to LLVM IR; no C or C++ source emitter is required. HUC still implements
construction, relocation, evaluation order, and cleanup explicitly before
LLVM optimization. LLVM supplies optimization and target code generation,
not HUC semantics. Foreign ABI lowering, target layout, debug information,
runtime support, and linking remain explicit compiler responsibilities.

## 2. Product position

HUC is an unchecked systems language compiled ahead of time. It aims to offer
C++-like values, objects, and generic programming with simpler rules.

HUC is not a memory-safe Rust alternative. It deliberately permits:

- dangling and invalid raw pointers;
- null dereferences;
- unchecked indexing and pointer arithmetic;
- access to relocated storage through raw pointers or untyped addresses;
- data races;
- signed overflow and other specified undefined behavior.

The language provides deterministic cleanup for active values, but it
does not infer or prove the safety of raw observers.

The following remain outside the adopted core design or need separate designs;
none is made a release requirement by the completeness goal:

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

- module namespaces and once-per-translation-unit runtime imports, with one
  import block per file, `pub import`, and `as _`;
- `pub` declarations; other top-level declarations are private to their file;
- runtime functions and structures;
- type aliases and fixed-signature C declarations;
- `let` declarations;
- `mut` permissions;
- raw pointers `T*`;
- unary `&`, `construct_at`, `destruct_at`, and the core
  type-operand intrinsics;
- typed `raw_storage<T, N>`, `storage_ptr`, and compile-time `size_of`,
  `align_of`, and `is_basic` queries;
- constructors, methods, `clone`, and `drop`;
- runtime expressions and control flow;
- local `let name: auto` inference.

HUC0 does not contain:

- numbered declarations or control-flow keywords;
- `@`;
- family binders or specialization requests;
- compiler-only values or handles;
- text-generation operations;
- unresolved `auto` parameters or inferred function results.

The angle operands in `as<T>`, `ptr_as<Target>`, `size_of<T>()`, `align_of<T>()`,
and `is_basic<T>()` are built-in type operands, not phase-family requests.
`raw_storage<T, N>` is a built-in type form with a positive integer-literal
count in HUC0. These are the only angle forms accepted by HUC0.
All HUC intrinsics have unqualified names, not `std::` names; they are not
ordinary standard-library functions.

Primitive fixed arrays and `bit_as` are deferred pending separate designs.
Containers and swapping can initially be supplied by libraries. Raw storage
supplies their inline backing slots without becoming a general array or sum type.

### 4.2 HUC1

HUC1 extends HUC0 with staging. A `.huc1` module may contain
unnumbered runtime code plus compile-time and compiler-only constructs. HUC1 translation
must produce one printable, flattened `.huc0` translation unit.

HUC1 includes generic families, compiler-only declarations, and compiler-stage
control flow. HUC0 contains only the concrete runtime result. The suffix `1`
does not count interpreter passes or declare a generic family.

## 5. HUC0 runtime semantics

### 5.1 Declarations and mutability

Every local, field, and global variable begins with `let`. Function parameters
do not use `let`. Declarations use `name: Type`, with an optional `mut` before
the name for writable storage. A `mut` inside the type still grants pointee
mutation. For example, `let mut pointer: mut Widget*` has both a writable
pointer slot and a writable pointee. Method `mut` stays after the parameter list.

Local inference uses `let name: auto = expression;`; the colon and `auto` are
required. Runtime parameters, fields, and globals require explicit types.
HUC0 does not also accept type-first declarations, omitted type annotations,
or destructuring.

HUC has no `var` keyword and no `const` keyword. Values are fixed by default,
and `mut` grants mutation.

```huc
let answer: i32 = 42; // fixed scalar
let mut counter: i32; // writable scalar, initialized to zero
```

Each `mut` controls the layer where it appears:

```huc
let observer: T*; // fixed pointer field; constructor obligation
let observer: mut T*; // fixed field; mutable T; constructor obligation
let mut observer: T*; // reseatable pointer field; defaults to null
let mut observer: mut T*; // reseatable field; mutable T; defaults to null
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
stored inline in those fields. A `mut T*` field still allows
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
type constructor `T*`; the constructor composes, so `T**` and longer pointer
chains are valid non-owning types.

The non-overloadable unary `&place` produces `T*` for a fixed inline
`T` place and `mut T*` for a writable inline `T` place. It evaluates
its addressable operand once without creating a temporary, transferring a
value, extending a lifetime, or changing cleanup state. It adds one pointer
level for pointer slots and rejects non-place results, literals, types, and
function symbols.
Binary `&` remains bitwise AND.

An ordinary function can receive a slot's typed address from `&place`,
including a pointer to a pointer slot. `ptr_as<usize>(pointer)` converts a raw
data pointer to an integer address, and `ptr_as<T*>(address)` reconstructs a
raw pointer, including a pointer-chain type. Require exactly one operand;
permit pointer-to-pointer, pointer-to-`usize`, and `usize`-to-pointer casts,
and reject other pairs. Preserve the unchanged-address round trip for the target.
Byte views use `u8*` or `c8*`; casts do not make incompatible typed access valid.

Taking `&` of an Inactive or MaybeActive named root remains a diagnostic,
including when nested in a cast. Manual-storage code retains a pointer before
transfer or uses `raw_storage`. Casts do not preserve permission metadata or
update activation and cleanup state. The programmer must preserve storage
lifetime, alignment, actual permissions, valid representation, and cleanup
obligations. Writing actually fixed storage remains undefined behavior.

Copying a raw pointer, including a null pointer, leaves the source unchanged.
It never transfers, destroys, or extends the lifetime of its pointee.
Resource-managing structures use ordinary `drop` and Advanced relocation.
Allocation and deallocation are explicit library or external operations;
HUC0 has no built-in allocating constructor expression.

Relocating a named Advanced root requires binding `mut`. Extraction through
`*pointer` requires pointee `mut` and programmer-managed source cleanup;
the pointer binding itself can be fixed. Fresh unnamed
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

- built-in `raw_storage<T, N>`, with no implicit payload cleanup or cloning;
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
Assigning to a root tracked as inactive begins a new active lifetime there.
Inactive manual storage requires construction without old-value destruction;
ordinary indirect assignment assumes a live destination.

Using an Advanced local or parameter whose tracked state is Inactive or
MaybeActive is a required diagnostic in every compilation mode. The existing
drop-state analysis supplies that state. Whole-value assignment does not use
the old value, although reads in its operands are still checked. Scope-exit
cleanup skips inactive values. `&` is a checked use, including inside `ptr_as`.

The checker follows operations inside conditions without predicting their
results: every runtime branch outcome counts as possible. It follows
left-to-right argument binding, rejecting `f(x, x.size())`. A field store is
checked both before and after its right-hand side, rejecting
`x.count = consume_and_count(x)`.

Two additional checks recognize direct receiver and pointer-argument forms.
A call is rejected if argument evaluation transfers the root of its receiver
(`x.absorb(x)`), or transfers a root after an earlier argument took its address
or the address of one of its inline fields (`f(&x, x)`, `f(&x.payload, x)`).
The recognized spellings are defined in language specification section 6.12;
this is not general alias tracking. Pointer extraction does not update the
tracked state of aliased locals, and raw-pointer lifetime checks remain the
programmer's responsibility.

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

Relocation accepts two source forms:

- A whole named local or parameter, or a fresh temporary/result. The compiler
  updates that root place's active state and suppresses source cleanup.
- `*pointer` through `mut T*`. The source pointee becomes inactive, but the
  programmer must prevent later cleanup of its slot. The pointer expression
  is evaluated once; extraction does not change the pointer address, clear
  source bytes, or install a replacement value.

Pointer extraction never finds and cancels automatic cleanup through aliases,
including aliases of locals or their fields. Its semantics must not depend on
inlining or whether the compiler can recognize `&local`. Cleanup reaching an
inactive extracted slot is undefined behavior. Stack versus heap allocation
does not determine cleanup responsibility.

HUC0 still rejects direct moves from `pointer->field`, `value.field`, or
`array[index]`; it does not track partially inactive aggregates. Explicit
dereference of a writable pointer to such a slot uses programmer-managed
cleanup and does not disable automatic enclosing-object cleanup.

Copying/cloning an indirectly reached value remains valid when it is live
and supports logical copying. Either eligible source form may replace a valid
indirect destination; if the two can alias, exact storage identity is checked
before destination destruction. Basic pointee binding still copies, leaving
the source active.

Container implementations manage backing storage and initialized element
ranges. For example, a pop implementation extracts its last element through
a writable pointer and reduces the length, excluding that inactive slot from
later cleanup. No replacement value, per-element flag, or extraction intrinsic
is required. `construct_at(pointer, value)` initializes an inactive slot and
`destruct_at(pointer)` destroys a live value without freeing its storage.
Ordinary assignment is not a substitute for construction into an inactive slot.

Both are unqualified, non-overloadable runtime intrinsics using ordinary call
syntax. Infer the concrete `T` from a `mut T*` operand; do not add a type
argument, constructor-argument pack, or overload accepting an untyped address.
`construct_at` takes exactly two operands and returns the same `mut T*`;
`destruct_at` takes one and returns `void`. The pointer binding may be fixed,
but the pointee type must be complete and writable. Pointer-chain pointee
types are allowed when the destination storage is valid and writable.

Evaluate the destination pointer once before the initializer. Check the
initializer as for a local of type `T`, without introducing a by-value `T`
parameter: Basic values copy, eligible Advanced sources transfer, and explicit
`copy` clones. A direct constructor or function result initializes the supplied
slot as its final destination. No old value is destroyed. `destruct_at`
performs user `drop` and reverse field cleanup; for Basic values it ends the
lifetime without user code.

The caller supplies valid storage with sufficient size and alignment and
manages its live/inactive state. Null pointers, construction over a live value,
overlapping source/destination storage, and destruction of an inactive value
violate unchecked preconditions. No self-relocation no-op applies to
construction. Neither intrinsic deallocates destination storage, nulls the
pointer, or registers automatic cleanup. User constructors/destructors may
manage their own resources.

Do not update an aliased root's drop flags, even for `&local`. A directly
named Advanced source transferred into `construct_at` still becomes Inactive
normally. Destination cleanup remains manual. HIR must preserve this
distinction through inlining. See language specification section 6.7.

#### Raw storage and layout queries

Provide `raw_storage<T, N>` as a built-in runtime type with a positive literal
count in HUC0. HUC1 resolves compile-time counts before emission. The type
reserves `N * size_of<T>()` contiguous bytes aligned to `align_of<T>()`, with
no elements constructed, hidden tags, or cleanup flags. Require a complete
runtime `T` and diagnose invalid counts, recursive layout, and size overflow.
Empty-storage construction is `raw_storage<T, N>()`, also the default for an
omitted mutable binding or field.

The storage wrapper is always Advanced and has no `clone` or implicit element
destruction. Whole-wrapper relocation transfers live payloads and dormant
representation bytes without element scans. Library containers must track and
clean their live slots explicitly before releasing storage. A container's
ordinary `drop` performs that work; subsequent field cleanup does not repeat it.
Do not use a declared byte array with incompatible C typed accesses in lowering.

`storage_ptr` accepts one pointer to the wrapper and returns the first element
slot's typed address, preserving read-only/writable permission, without reading
or constructing a payload. `size_of<T>()`, `align_of<T>()`, and `is_basic<T>()`
take no value operands and fold to target `usize`, `usize`, and `bool` constants.
The category query reflects existing inference, not a type marker or trait.
Use it in generic pop/growth code to destroy copied Basic source values before
slot reuse, while skipping destruction of extracted Advanced sources.

One-slot storage plus a library flag supports an inline optional without a
default `T`; separate alternative fields plus a tag support a larger inline
sum representation. Overlapping unions are not introduced by this facility.
Dynamic backing allocations remain library operations with checked size
arithmetic and explicit alignment contracts. The controlling details are in
language specification sections 6.7.4–6.7.6.

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
    let mut visits: i32;

    fn init(id: i32) : id(id) {
        // visits defaults to zero
    }
}
```

An `init` body has an implicit writable `this: mut T*`. All initializer entries
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
- constructors cannot return recoverable errors; creation that can
  report an error will use named factory functions and library result types
  later.

### 5.6 Methods and destruction

Methods are read-only by default. A trailing `mut` grants a writable receiver:

```huc
fn value() -> i32 { ... }
fn increment() mut -> void { ... }
```

An Advanced structure may define:

```huc
fn drop() mut -> void {
    // release non-field resources
}
```

`drop` cannot be called directly. It executes before fields are destroyed in
reverse declaration order.

Library arrays and vectors should destroy their live prefix in decreasing
index order. Their own code tracks partial initialization and cleans that
prefix on normal exits. Raw-storage fields provide no automatic element
cleanup; this does not introduce primitive fixed arrays.

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

HUC has no exception unwinding. Process termination through panic need not
run pending cleanup.

Unconsumed temporaries remain alive until their full expression finishes,
then are destroyed in reverse order of completed initialization. An ordinary
statement cleans them up at its end. A condition cleans them up after saving
its result and before entering the branch or loop body. Each `for` initializer,
individual update expression, and field initializer has its own boundary.
A loop's header binding survives iterations; body locals are cleaned up before
updates, including on `continue`. Loop exit cleans up an active header value,
even if the body never ran. A return binds its result before temporary cleanup,
then cleans up remaining locals and parameters.

Arguments, nested calls, and short-circuit operands do not create earlier
boundaries for the caller's temporaries. A value consumed into a by-value
parameter instead follows the parameter lifetime and is cleaned up before
that call returns unless transferred onward. No observer extends a source
lifetime. Language specification section 6.5.1 defines the complete rule.

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
The conditional produces a value: selecting an Advanced local transfers it,
even when the result is only a method receiver. Its unconsumed result is
destroyed at the enclosing full-expression boundary. Selecting pointers with
`(condition ? &a : &b)->method()` observes without transferring either value.

The backend must initialize each typed by-value parameter before evaluating
the next argument. Keeping only a pointer to the source expression is not
enough; parameter binding must already be sequenced in MIR.
Generated typed storage or a call frame may pass these already-initialized
values to the LLVM call. Optimizers may still reorder pure computation when the
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
- `for (initialization; condition; updates)` accepts one `let` binding or
  expression in its initializer, one condition expression, and an ordered
  comma-separated update list. All three clauses are optional; a missing
  condition means `true`. The binding is loop-local and the body is a nested
  scope. Updates run after body cleanup on completion or `continue`, but not
  on `break` or `return`. Each update is a separate full expression. This
  introduces neither a general comma operator nor a range-based loop.
- Semicolons, explicit `return`, and `//` comments remain. Interpolation,
  implicit returns, and additional declaration spellings are out of scope.
- The [formatting guide](formatting.md) defines the default style, not compiler
  acceptance. Operator spacing, indentation, and line width are not grammar.

A canonical formatter is future tooling, not a prerequisite for either
implementation route.

## 6. HUC1 phase semantics

### 6.1 Number meaning

The suffix is part of the keyword:

- no suffix: an ordinary declaration; an angle header defers a family definition;
- suffix `1`: compiler-only storage and imports, or
  compiler-stage control flow;
- `<...>`: generic binders, specialization patterns, or specialization requests;
- `@`: entry into compiler execution.

There is one numbered compiler stage, not an arbitrary number of passes.
Former `fn1`/`struct1`, suffix-2 spellings, and parenthesized generic binders are rejected rather
than accepted as compatibility aliases.

### 6.2 Generic families

All three family declarations put their name before an angle-bracket binder list. Each
binder uses `name: Type` or `name: auto`; binders are not writable slots.

The primary forms are:

```huc
struct Stack<T: auto> {
    ...
}

fn convert<T: auto>(mut value: T) -> T {
    ...
}

let value<T: auto> {
    let value: T = T();
}
```

Partial specialization adds a second angle-bracket list for the pattern.
The body follows the completed header, with no declaration-level condition:

```huc
struct Stack<T: auto><T*> { ... }

fn convert<T: auto><T*>(value: T*) -> T* { ... }

let value<T: auto><T*> {
    let mut value: T* = null;
}
```

Full specialization uses an empty binder list followed by a concrete pattern:

```huc
struct Stack<><Widget> { ... }
fn convert<><Widget>(mut value: Widget) -> Widget { ... }
let value<><Widget> { let value: Widget = Widget(); }
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

Declaration headers are simpler: after the name, `<` starts a deferred family.
The first angle list always declares binders; the second is an optional pattern.
`<>` alone is a zero-binder primary, `<><i32>` a full specialization, and
`<T: auto><T*>` a partial specialization. Parenthesize `<` and
`>` comparisons inside angle-list values or defaults. These declaration choices
do not need name lookup. Keep generic bodies deferred until selected.
Diagnose declaration `<T>` with a hint for `<T: auto>` or `<><T>`. Calls keep
one optional argument list: omit it when arguments are deduced/defaulted or
there are no binders. `f()` and `f<>()` must select the same zero-binder family.
Zero-binder type and variable requests remain `S<>` and `v<>`.

### 6.3 Binders and deduction

`T: auto` binds a compile-time entity whose kind is inferred from use. A binder
used in a type position must hold a type. Typed non-type parameters use normal
types:

```huc
struct Buffer<T: auto, N: usize> { ... }
```

Primary declarations may provide trailing defaults. Partial and full
specializations may not introduce defaults. Packs, when implemented, must be
last, and a pattern pack expansion must also be last.

Normal generic `fn` calls may infer binders by unifying runtime parameter types with
runtime argument types. Compiler calls also deduce from an expression parameter's
represented type (section 6.8.1). Non-deducible parameters must be supplied explicitly.

### 6.4 Specialization selection

Each generic `struct` and generic `let` name has one primary family per scope. Each generic `fn`
name also has one primary family per scope; normal runtime `fn`
overloading remains available. A name in one scope is either a non-generic
overload set or one generic family, never both; concrete family cases use full
specializations rather than ordinary overload declarations.

Generic structures are module-scope. Generic functions are allowed at
module or type-member scope. Variable families are allowed at module, function,
or nested-block scope and never as fields. Local generic functions/structures
and nested type families are deferred.

The primary must precede its specializations. Every specialization
must be declared in the primary's namespace, in an owning file: the file that
declares the family, or a file that declares a type in the pattern when no
binder comes before that type in a left-to-right reading of the pattern,
including nested arguments. Another owning file reopens the family's
namespace to specialize it; the family must be `pub`. When a file finishes,
check its specializations against the same family's specializations in the
files it depends on: two that can match one request, with neither more
specific and no specialization for exactly their overlap, are an error in
that file. A specialization must also precede every request whose selection
it would change, by being chosen or by making the choice ambiguous; declaring
one afterwards is an error. The
[compilation model](compilation-model.md), section 9.5, explains these rules.

Any family may be requested from an importing source with a caller-defined
type; no reverse import is required. A concrete definition keeps its family's
namespace and declaration position, not separate compilation ownership. Its
body is checked in the file that wrote the selected definition: the family's
file for the primary, or the owning file for a specialization, with that
file's imports, aliases, private declarations, and `#` ownership. Flattened
HUC0 uses canonical references and namespace sections. Module-level
generated output has a separate requesting-site anchor (section 6.11). A
caller's private type may be a phase argument, because the family never
names it.

Selection:

1. Match argument count and structural angle pattern.
2. Bind pattern variables.
3. Choose the unique structurally most-specific matching candidate.

An exact/full pattern outranks a partial pattern. A more constrained structural
pattern outranks a pattern that accepts all of its matches. The primary is the
fallback. Declaration order never breaks a tie. Equally specific or
incomparable matching candidates are ambiguous. Selection executes no user
conditions. After selection, `if1` inside the body may choose an implementation
or report an invalid request with `compiler::error`. It cannot ask selection
to try another candidate; an error in the chosen body remains an error.

Patterns support C++'s structural forms: concrete types and values, binders,
pointers whose `mut` must agree, family applications, `raw_storage<T, N>`,
repeated binders that match only identical arguments, and family binders such
as `C<T>`. Ranking stays "matches a strict subset". Constraint-based and
substitution-failure forms are not supported.

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

### 6.6 Generic variable families

A generic `let` header has no result type. Every selected path must leave:

- exactly one ordinary runtime `let`;
- with the same identifier as the family;
- with no other residual runtime declaration or statement.

Compiler-only setup and `if1`/`for1`/`while1` are permitted. Text generation
is not permitted inside a generic `let` selection body because it would obscure the
rule that exactly one variable must remain.

Different specializations may choose different:

- value types;
- inline values versus raw-pointer forms;
- pointee permissions;
- slot mutability;
- initializers.

Generic `let` is permitted at module and function/block scope but not as an instance
field. Every statically requested specialization is emitted at the family
declaration site. A function-local specialization therefore initializes on
each runtime execution that reaches that declaration; no lazy runtime flag is
introduced.

### 6.7 Phase control

`if1`, `for1`, and `while1` execute while HUC1 prepares code and are allowed at:

- module scope;
- structure/type-body scope;
- function scope;
- nested block scope.

At module or type-body scope, residual output must be declarations or members
valid at that insertion point. At function/block scope, residual declarations
and statements are allowed.

A selected module/type body may also contain ordinary compiler-only
expressions and control statements, such as updating a `let1` loop index.
Their operands must be compiler-available and the statements erase. The
selected-body parser uses the inherited module, type, function, or block
context; skipped bodies remain opaque.

`for1` uses the same three-part header as ordinary `for`:
`for1 (initialization; condition; updates)`, with `let1` instead of `let` for
a header declaration. Its binding is initialized once and scoped to the loop;
it creates no runtime scope or storage. Clauses execute at compile time; an
omitted condition means `true`, and updates form the same ordered expression
list. A predeclared compiler variable remains valid too. `while1` reevaluates
a compiler boolean. Both phase
loops are bounded by interpreter resource limits, and a zero-iteration body
remains opaque. Neither `for` nor `for1` has a range-based form; `in` is not
reserved.

During compiler execution, ordinary `if`, `for`, and `while` already execute in compiler
phase. Such a `for` uses plain `let` for a header binding, including in selected
module/type bodies. HUC therefore has no `if2`, `for2`, or `while2`.

### 6.8 Unified declarations and compiler storage

Use only `fn` and `struct`; an angle header defers the definition, including
an empty `<>`. `let1` marks explicit compiler storage. Fields and working
locals use plain `let` and live in the evaluator or runtime according to use.
Compiler-only field types propagate to the containing type. Deferred functions
may accept/return compiler-only values for compiler calls, but not runtime calls.
A deferred function/type without compiler-only types is runtime-capable if its
prepared definition is valid. No separate compiler-only declaration marker is
inferred.

A `let1` is typed, scoped compiler storage. A module-level declaration has one
identity with values in each importing module's compiler state. A function-local
one has a lifetime per preparation or call. Compile-time code can assign a
`let1 mut`, including an imported one, subject to existing phase and mutation
rules. Ordinary evaluated functions retain section 12.1's restrictions on naming
mutable compiler storage; they may modify objects through received pointers.

Each module begins from its own ordered dependencies, not its importer's state.
Merge completed imported states by logical object identity, with the later
whole-object value winning even if unchanged. Pointers, structures, and
allocations follow the same rule. Writes affect only the active state, and
compiler calls inherit that state. Completed dependencies remain unchanged.
No separate configuration syntax or type category is required.

Reads made during preparation are fixed into the prepared body; executing
compiler code reads current values. Runtime definitions and canonical types
are prepared using their defining module's state, not requester overrides.
Evaluation-mode calls use the caller's active state. Existing runtime
definitions are not regenerated or renamed when an importer changes a value.

Every late program-mode preparation gets a fresh view of its defining module's
completed state, including when requested inside another preparation. Compiler
calls within a preparation share its view. A module-level object absent from
the active state may be read from its declaring module's completed state,
read-only. Do not borrow from unfinished modules or bypass existing lifetime
records. Borrowed entries are not merged or cleaned up as locally owned values.

A function-local `let1` lasts until preparation completes in program mode,
or until its compiler call finishes in evaluation mode. Save completed module
states through translation. After output, clean each in a private view, in
reverse module-processing order, with reverse initialization-order cleanup of
its remaining live values and leak checks. Cleanup cannot change other saved
states. State merging itself invokes no `clone` or `drop`; actual HUC assignment
and destruction still do. Redeclaring a `let1` remains an error.

Register angle-header signatures without parsing or preparing their bodies.
Parse the selected body on request; an unused body remains opaque. Each
compiler call prepares a fresh private body and performs its `#` insertions
and other compiler operations again.
Ordinary `fn` bodies are instead parsed and prepared in program mode as they
are encountered, even if never called. Runtime forward-name checks can wait
for declaration collection; compiler-stage work is not postponed until a call.

A compiler evaluation can call an ordinary `fn` without another `@` when the
call meets specification section 12.1. No marker is needed: ordinary value
operations compute what they would at run time, or evaluation fails.
Compiler-state reads or queries of generation mode or binding kind, directly
or through callees, can select different behavior; do not require those
branches to be equivalent. Support the runtime language on available HUC
bodies: every scalar type including `c8` and floats, Basic and Advanced
structures with constructors, methods, `copy`, relocation, and `drop`, pointers
to compile-time objects, raw storage with `construct_at` and `destruct_at`, and
compile-time allocation (specification section 12.3).
Locals in the ordinary function retain their `let` spelling and use
evaluator-owned storage during the call.

Check executed operations transitively. Untaken ordinary branches remain
normally parsed and type-checked, but do not execute unsupported operations.
Reject runtime-storage access, naming `let1 mut` from an ordinary function,
foreign or native calls, untracked external effects, and observing numeric
addresses. Every undefined operation, including null, dangling, out-of-bounds,
uninitialized, and inactive-slot access, is a diagnostic. Target arithmetic,
strict IEEE floating point, and evaluator limits determine results and
diagnostics; failed evaluation never falls back to runtime.

### 6.8.1 Argument-driven bindings and text extraction

Keep ordinary parameter syntax. Each compiler-call argument determines a value
or expression binding; no declaration-level marker or family kind map is
needed. `let1` values, literals, and actual evaluator-owned `let` objects supply
values. Runtime-dependent expressions, including references to runtime globals,
supply descriptions, not objects. Use the structural classification in
specification section 9.7.1, before execution
or optimization. Calls without runtime inputs are value computations; failed
evaluation never falls back to capture. An explicit inner `@` requests compiler
evaluation of that subexpression, whose calls can themselves bind expressions.

Bind descriptions by resolving their types without executing runtime effects
or changing source lifetime/cleanup state. Deduce from the represented type and
apply ordinary structural specialization, with exact substituted-type matching
and no hidden conversion. Binding kind adds neither overload ranking nor symbol
mangling. An expression binding has no writable slot, even on a `mut` parameter;
pointee permissions remain part of the represented type. Runtime calls continue
to bind values, and constructors/receivers/intrinsics keep their own contracts.

`compiler::is_expr_param(parameter)` is a compiler-known, non-overloadable bool
query on a parameter of the current function, optionally parenthesized. It is
available during preparation and compiler execution. Supply per-call binding
metadata before preparing `if1`; the unselected branch is not parsed or checked.
For runtime preparation it returns false for all ordinary parameters. This
reveals neither parameter values nor expression text. Two compiler calls of
the same specialization can prepare different branches; no caching is added.

Use `compiler::expr(parameter)` during execution to obtain parenthesized
`compiler::Text` from an expression binding. It is not a first-class function or
public AST handle. Allow unchanged forwarding, preserving either binding kind.
Reject reads, represented-object method calls, assignment, and storage of an
expression-bound parameter. Its type and kind are available during preparation,
but its text is not: `@compiler::expr(parameter)` inside its function body is
too early. The query's false result also does not make a runtime value readable
by preparation. Do not confuse evaluator-owned plain `let` objects with runtime
bindings, even when both have the same type and declaration spelling.

Compiler-known components within captured runtime expressions must be
materializable and are rendered using normal HUC1 substitutions. Runtime
locals known to an optimizer remain runtime expressions.

This works in `#`, `@`, `let1` initializers, and nested compiler calls. Code can
be built, stored, and inserted later; it is not tied to a direct `#` call.
`compiler::expr` produces plain expression text for existing `Code` constructors.
Resolved family references use the specialization engine's canonical concrete
names. Insertion resolves those names through the existing definition records;
only a retained program reference creates runtime demand. Capturing or keeping
unused text must not emit the referenced function. Final HUC0 includes its
needed definitions and needs no private name table.
Only insertion parses/checks generated fragments. Names resolve at insertion,
without bound-reference preservation or automatic hygiene. Capture-specific
runtime lifetime checks, temporary bans, and lifetime extensions are not added.
Generated bindings can own temporary containers; repeated inserted text can
repeat side effects. Ordinary runtime calls and scalar materialization stay
unchanged. Specification section 9.7.1 defines the full contract.

### 6.9 The `@` operator

`@` starts compile-time evaluation of an expression. This is called an
evaluation island: the marked expression runs in the compiler even when the
surrounding code is runtime code.

It is never attached to a type:

```huc
let1 function: compiler::Function =
    @compiler::find_function("render");
```

The type is `compiler::Function`, not `@Function`.

The whole operand of `@` is compiler code, so nested value calls need no further `@`.
The same holds in phase-control clauses, `let1` initializers, `#`
operands, and code executing in a compiler call. Inside a function body, `@`
marks a value computed once while the body is prepared; it cannot read the
body's parameters or `let` locals. Ordinary runtime-capable calls inside an
evaluated body follow the ordinary-function rules above.

Argument-driven expression binding permits describing code being prepared
without reading its runtime value. At a call's argument boundary, a runtime-
dependent expression is captured unless an explicit inner `@` requests its
evaluation. This does not let a body's own preparation inspect parameter values
or text. Only the binding-kind metadata is available through
`compiler::is_expr_param(parameter)` before the call executes.

A standalone `@expression;` is an erased phase-expression item. It is legal at
module, type-body, function, and nested-block scope, must evaluate completely
at translation time, and must have type `void` or `never`. It never inserts
code; a
`compiler::Code` result is rejected with a hint to use `#`. Every `@` call,
including one in an initializer, argument, condition, or return expression,
uses a private evaluation body and inserts nothing into its caller. Compiler
intrinsics query or operate on the active preparation without a private
user-function body; `compiler::emitting()` must still see that active mode.

A generic `fn` call retained in runtime code produces or calls a concrete runtime specialization. An
`@family<arguments>(...)` requires complete compile-time evaluation and may embed only a
result that can be written as a HUC0 value (a materializable result).

Completing a generic `fn` for compiler execution does not require emitting its runtime
definition. Track runtime demand separately and emit the specialization when
retained runtime code needs it, including a later request after compile-time
evaluation. A never-requested family body stays deferred. Ordinary `fn`
declarations in retained modules still pass through even when only evaluated
at compile time or never called; unused-function removal is a later optimization.

The same ordinary-function evaluator is available through explicit `@`
entry. Prepare and evaluate the callee privately in evaluation mode. Deferred
compiler calls can use their concrete ordinary inputs during execution, but
not during preparation. These are evaluator inputs, not family binders, and
never affect runtime mangling. Each compiler call gets a fresh private
preparation and executes again; no results or evaluation preparations are
cached. Runtime preparation still has symbolic parameters and
rejects phase work requiring their unknown values. Local generated code is
parsed into the private body. Module
emission evaluates its arguments but introduces no declarations in evaluation
mode. Never replay the callee's generation into its evaluator caller. A later
program-mode request still needs its own preparation and module output.

### 6.10 Imports

HUC1 has two import forms, and an ordinary import may also re-export:

```huc
import app.data;
import1 compiler;
pub import app.shapes;
```

Keep `module` and `import` syntax. Each source file is a module with its own
namespace, lookup environment, and private declarations; modules are not
separately compiled. `import` exposes a module's `pub` declarations of every
kind under their phase rules; process the imported source at its first import
in deterministic depth-first order, then resume the importer with its lexical
context restored. A source is processed once per translation unit; a later
import from another file merges its completed state without repeating
declarations or execution. `import1` exposes compiler-only declarations,
including the supplied compiler interface, without creating a runtime
dependency. `pub import` also re-exports the module.

Each file has one import block, directly after its first `module`
declaration. Importing one module twice in a file, by any form or alias, is a
diagnostic, and `as _` binds no alias. A file may name only its visible
modules (itself, its direct imports, and modules reached through `pub import`
edges) and only `pub` declarations of other files. A later namespace section
may reopen the file's own namespace, or a visible module's namespace only to
add specializations. A module saves its completed compiler state for importers
to merge without changing it (section 6.8). The
[compilation model](compilation-model.md) explains these rules with examples.

A root and its imports form one translation unit. Both import forms disappear
from flattened HUC0. Repeated `module name;` sections select or reopen naming
scopes, preserving forward insertion across namespace boundaries; canonical
`a::b::name` references replace imported aliases. The first section names the
root. Original and imported sources retain their own buffers and locations.
The same namespace-section grammar is accepted by the standalone HUC0 parser;
no `#include`, general `namespace` keyword, or private bundle format is added.
`lower --flattened` reads such a file without the source import, `pub`, and
ownership checks, which were made before flattening.

Import cycles are rejected with the complete cycle path.

### 6.11 Text generation and generation modes

Generated code is a compiler-only value of type `compiler::Code`. The bootstrap
`compiler` module provides three constructors, and `+` joins values in order:

```huc
compiler::local(text)
compiler::module(text)
compiler::around(before, after)
```

A `#` statement inserts a code value where it appears:

```huc
#compiler::local("let mut generated: i32;");
```

Program mode prepares HUC0 program code; evaluation mode prepares a private
body for compiler execution. Both run during compilation. A `#` always inserts
into the body that contains it, in that body's mode; a compiler call prepares
its callee separately and never inserts into its caller. The two modes are not
different mangled function names. The bootstrap query
`compiler::emitting()` returns `true` in program mode and `false` in
evaluation mode, so `if1` can choose a mode-specific branch. Structural
specialization selection does not evaluate conditions.

A `#` runs while its body is prepared, once per occurrence; inside a runtime
loop it inserts once, into the loop body. Local code takes the place of the
`#` statement and is parsed before subsequent source. If the code has a hole,
the next braced source block fills it, and the text around the hole and that
block must together form complete items. Other incomplete fragments are
rejected.

In program mode, module code is queued after the **module insertion anchor**:
the original requesting top-level declaration, or the `#` item itself at module
scope. An ordinary `fn` is its own anchor (its containing top-level type for a
method) and is prepared when encountered. A first runtime generic `fn` request
inherits the active anchor; nested requests, including methods prepared inside
a requested generic `struct`, keep it. Thus `main -> A -> B` queues module code
after `main`, not after the deferred families or a completed private
preparation of `B`.

Inserted code uses the namespace and lexical context of its `#` site,
independently of the anchor's namespace. Finish the anchor's compiler-stage
work before parsing and publishing its ordered module queue, then resume the
following source. No insertion prints before its anchor or appends all output
to the file's end. Local code stays within the body that contains the `#`.

Module paths in inserted text are canonical names: import aliases are not
consulted for them, and the module must be visible from the file that
contains the `#`. Inserted text may name only `pub` declarations of other
files, and declarations it adds are private unless the text says `pub`. A
source block that fills a hole is authored code and keeps its aliases.
A module's names are fixed when it finishes: a declaration that one of its
deferred bodies generates into it later can be named only by the preparation
that generated it, or by one that generates the identical declaration
(specification section 11.2). An evaluation inserts no module code, so it
cannot use such a declaration; a body that needs one at compile time takes
its own path with `compiler::emitting()`.

In evaluation mode, module code is not inserted or parsed and creates no
names. Pending code is unavailable to all evaluations until publication,
including through nested function calls. Already published, checked code can be
used by later evaluations, including the function whose body inserted it,
unless it was generated after its module finished. Apart from that case,
this is an availability rule, not an origin-based ban.

HUC1 parses generated HUC0 and checks it with surrounding source. Numbered
constructs, `@`, and `#` in inserted text are errors, not recursive staging.
Module forward references wait for declaration collection; compiler evaluation
needs an available checked definition. Missing definitions and
generation/evaluation cycles diagnose, as do insertions that would change
finalized compiler decisions. The independent HUC0 translator still checks the
final printed artifact.

Publish program-mode module output once per translation unit and owning
function or specialization. Distinct `#` statements and phase-loop iterations
within that preparation all remain. A generated module declaration with the
same tokens and lexical naming context as one already inserted in that
namespace is inserted once and records both origins; other duplicates
diagnose. Evaluation-mode completion never marks program output published.
Keep program definition/publication records, immutable source buffers, and origin
chains linking generated spans to their `#` site, anchor, import chain, and
building call chain. Preparations never depend on ordinary argument values;
the separate per-call binding-kind metadata is available to `is_expr_param`.
Every compiler call gets a fresh evaluation-mode body, using currently
published, checked declarations; it cannot publish pending output early.
That state never enters name mangling. No caching or cache-eligibility analysis
is part of the current design. Definition identity, recursion tracking, and
once-only program publication remain required.

`compiler::error(message)` is also part of the bootstrap module. It accepts
one compile-time text argument and has result type `never`: it fails
translation rather than returning. Run it whenever compiler execution reaches
it, in either generation mode. Report the message, operation location, request
site, and compiler call/specialization chain. An unselected phase body or an
untaken ordinary compiler branch does not execute it. Use `if1` with an
`@compiler::error(...)` statement to validate a selected body's phase arguments;
do not treat the error as permission to select another definition.

Bootstrap HUC does not include:

- `gen`;
- `gen_global`;
- `quote`;
- `parse_huc`;
- structural syntax objects.

Typed expression arguments use section 6.8.1 and yield plain text. They do not
add a first-class expression object, typed code placeholder, or structural splice.

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
- `compiler::Instruction`.

`compiler::Code` already exists in the bootstrap as generated text; a structured
API may extend it. These will be ordinary qualified type names declared as
compiler-only types by the compiler module. Their names will not contain `2`
and will never use `@` as a type marker.

The eventual compiler library is exposed through a versioned, inspectable
generated interface unit that the HUC1 frontend loads before user modules—the
language-level equivalent of an always-included header. `import1 compiler`
binds that already loaded interface under the `compiler` name. The bootstrap
interface declares `compiler::Code`, its three constructors, `compiler::expr`,
`compiler::is_expr_param`,
`compiler::emitting()`, `compiler::error(message)`, `compiler::allocate`, and
`compiler::deallocate`; later versions add the
public handle types above without exposing internal compiler objects.

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

Both translators and the command-line driver are C++20 programs. HUC1 emits
HUC0 source; HUC0 emits LLVM IR through LLVM's C++ APIs. Pin a supported LLVM
version and keep LLVM-specific ownership and APIs behind the backend boundary.
Implementation and output languages remain separate choices.

Writing textual `.ll` and invoking a pinned LLVM tool such as `llc` is an
optional bootstrap alternative if linking LLVM becomes an initial obstacle.
It is still direct LLVM lowering, not a C-source backend, and does not remove
HUC's ABI or semantic responsibilities. The planned path remains in-process
LLVM through its C++ APIs, which can also print `.ll` for inspection and golden
tests. Keep either transport behind the same backend boundary.

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
  support/
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
  lowering, and LLVM IR generation, verification, and object emission;
- `huc1`: HUC1 indexing, deferred-body spans, phase checking, interpreter,
  specialization, and HUC0 residualization;
- `huc`: thin command-line driver.

`huc1` may reuse HUC0 parsing and type-resolution facilities for selected
runtime code and generic `fn` deduction. It must not bypass the textual HUC0 boundary
when chaining the translators.

## 9. Public compiler interfaces

The architecture document will define value-oriented C++20 APIs equivalent to:

```cpp
namespace huc {

struct Huc0CompileRequest {
    std::string root_module;
    bool flattened = false; // a complete flattened program
    std::vector<std::filesystem::path> import_roots;
    std::filesystem::path output_directory;
    TargetConfig target;
    bool write_source_maps = true;
};

struct Huc0CompileResult {
    std::vector<Artifact> artifacts;
    std::vector<Diagnostic> diagnostics;
    DependencyGraph dependencies;
    bool succeeded = false;
};

[[nodiscard]] Huc0CompileResult
huc0_compile(const Huc0CompileRequest& request);

struct Huc1ExpandRequest {
    std::string root_module;
    std::vector<std::filesystem::path> import_roots;
    std::filesystem::path output_directory;
    TargetConfig target;
    PhaseLimits limits;
    bool write_source_maps = true;
};

struct Huc1ExpandResult {
    std::optional<Artifact> source;
    std::vector<Artifact> sidecars;
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
huc lower <root.huc0> --out-dir <llvm-dir>
huc stage <root.huc1> --out-dir <huc0-dir>
huc build <root.huc1> --out-dir <build-dir>
```

`stage` may be implemented before native lowering, or both commands may grow
together. `build` chains them through files, then emits and links object code;
it does not use an undocumented AST shortcut. A command not yet implemented
for the selected release subset reports that limitation explicitly.

## 10. HUC0-to-LLVM workstream

### 10.1 Exit result

Given a `.huc0` translation unit with acyclic source imports, or a flattened
file with namespace sections, the translator emits:

```text
build/llvm/
  app/main.ll
  app/main.bc       # optional bitcode serialization
  app/main.huc.map
```

One HUC translation unit produces an LLVM module with deterministic symbols
for namespace-qualified declarations and overloads. Source imports do not
create object-file boundaries. The target triple and data layout are explicit.
Object emission uses LLVM; linking is a separate driver step. Neither generated
C headers nor an external C compiler is part of HUC lowering.

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
- deterministic depth-first import traversal, processing each source once,
  through a module-loading table;
- namespace sections, alias binding, and restoration of importer context;
- one import block per file, duplicate-import diagnostics, `as _`, and
  re-export edges from `pub import`;
- source input for `lower` by default, and one complete flattened program
  with `lower --flattened`.

Acceptance:

- the driver loads the root and imports as one translation unit;
- missing modules, cycles, late imports, and duplicate imports have useful
  diagnostics;
- repeated runs produce byte-identical dependency order.

### 10.3 Increment B: lexer and parser

Implement a hand-written lexer and recursive-descent/Pratt parser.

Cover:

- module sections, one leading import block, `pub import`, and `as _`;
- `pub` before top-level declarations;
- name-first `let name: Type` declarations and `mut name: Type` parameters;
- type aliases;
- fixed-signature `extern "C"` declarations;
- `struct`;
- `fn`;
- `init` and `drop`;
- `T*` and independent binding/pointee `mut` permissions;
- required braces for control-flow bodies, including `else if` chains;
- C-style `for` headers with a loop-local initializer binding and an ordered
  update list; reject declaration lists and a general comma operator;
- optional numeric digit separators and consistent optional trailing commas;
- expressions and precedence;
- constructor syntax;
- `raw_storage<T, N>` type and zero-argument construction syntax, including
  nesting and aliases, plus the layout/category query type operands;
- local `let name: auto`;
- error recovery at declaration and statement boundaries.

The lexer emits individual `<` and `>` tokens and indivisible numbered keyword
tokens, even though HUC0 rejects numbered forms.

Acceptance:

- every normative HUC0 example parses;
- AST dumps are deterministic;
- generic declarations use angle binders and partial/full patterns, with the
  body immediately after the header; declaration-level `if1` conditions are rejected;
- a zero-binder primary `fn f<>()` stays deferred; non-generic `fn f()` does not;
- old parenthesized generic binders, removed `fn1`/`struct1`, and suffix-2 keywords
  are rejected; a full specialization without its empty first list is diagnosed;
- malformed inputs produce multiple useful diagnostics without crashing;
- HUC1-only syntax is explicitly rejected as unavailable in HUC0;
- old declaration spellings, malformed separators, empty comma items,
  and unbraced control-flow bodies receive syntax diagnostics;
- spacing and indentation choices do not change acceptance when tokens remain
  the same; a formatter is not needed to parse the examples.

### 10.4 Increment C: names and types

Implement:

- module, type, function, field, parameter, and local scopes;
- one global declaration registry with a separate lookup environment per
  file: visible modules, `pub` checks, and namespace-section ownership;
- collect-before-check for module declarations;
- declaration-before-use for locals;
- canonical built-in, structure, and raw-pointer types;
- canonical raw-storage types keyed by element type and count, with target
  size/alignment validation and Advanced classification without element drop;
- canonical type aliases and alias-cycle diagnostics;
- local `auto` inference;
- constant `size_of<T>()`, `align_of<T>()`, and `is_basic<T>()`, rejecting
  invalid types and value operands; permission-preserving `storage_ptr`;
- method receiver permissions;
- constructor obligation checking;
- `extern "C"` signatures limited to built-in scalars, `void` returns, and
  raw pointers, including pointer chains, to built-in types, with no structure, array,
  phase type, or variadic boundary;
- overload resolution using only specified built-in conversions;
- no user-defined implicit conversions.

Acceptance:

- type and name errors point to both use and candidate declaration;
- naming a loaded but invisible module, or another file's private
  declaration, is diagnosed; `main` needs no `pub`;
- `mut` cannot be gained implicitly;
- fixed storage cannot be assigned through ordinary typed operations;
- fixed Advanced values cannot be relocated;
- direct Advanced field/array-element moves are rejected;
- pointer extraction requires `mut T*` and accepts a fixed pointer binding;
- all constructor field obligations are checked.
- every accepted C-linkage signature has a documented target ABI mapping.

### 10.5 Increment D: typed HIR and sequenced MIR

HIR records:

- canonical type;
- resolved symbol;
- value use: read, observe, copy, relocate, or place;
- unary `&` of an eligible data place, including pointer slots;
- raw-storage creation, element-address calculation, and folded target layout
  and category queries;
- explicit raw-pointer/`usize` casts in both directions;
- source origin;
- explicit conversion selected by HUC.

MIR records:

- ordered temporaries;
- full-expression boundaries and the temporaries belonging to each boundary;
- places and assignments;
- calls and control-flow blocks;
- active/inactive Advanced state;
- cleanup edges;
- conditional drop flags only where control flow requires them;
- whole-source deactivation without clearing source bytes;
- tracked-root versus programmer-managed indirect source cleanup.

No HUC semantic decision is deferred to LLVM. Foreign ABI lowering is an
explicit part of the backend, separate from HUC lifetime decisions.

Acceptance:

- HIR/MIR dumps expose every relocation and observation;
- argument and operand sequencing is explicit;
- tracked-root cleanup is exactly once on every normal exit when user code
  respects indirect extraction and other unchecked lifetime contracts.

### 10.6 Increment E: scalar LLVM backend

Emit:

- an LLVM module with deterministic symbols, target triple, and data layout;
- concrete scalar, pointer, aggregate, function, and control-flow IR;
- a platform entry-point adapter for the root module's HUC `fn main() -> i32`;
- target-correct declarations and calls for the supported C ABI subset;
- explicit instruction and parameter-binding order from MIR;
- strict floating-point lowering by default, without contraction or
  reassociation, using ordinary LLVM operations for the fixed default
  environment; relaxed flags require an explicit HUC mode;
- initialization, clone, and drop functions with explicit receivers and final
  destinations where needed;
- source-origin sidecars and LLVM debug information as support is added.

HUC permissions, ownership, and lifetime checks happen before LLVM emission.
Do not attach LLVM alias, overflow, lifetime, or pointer attributes merely
because a source binding is fixed or Advanced. Each assertion must follow
from HUC semantics. Verify generated IR before optimization and verify
optimized IR in tests. A verifier pass is necessary but does not prove that
the lowering preserves HUC behavior.

Document justified optimization information as well as prohibitions:

- use `nsw` for signed addition, subtraction, and multiplication whose overflow
  is undefined in HUC, at the width of the source operation;
- preserve unsigned wrapping without `nuw`, unless a separate proof rules out
  overflow; do not apply signed-overflow promises to unrelated lowering steps;
- use `inbounds` on address calculations only when HUC's object bounds and
  offset rules establish LLVM's full requirements, including intermediate
  offsets; a pointer cast alone establishes no such guarantee;
- consider type-based alias metadata later, after spelling out compatible
  accesses and the `u8`/`c8` byte-access exceptions. Resource ownership alone
  never justifies `noalias`.

Use ordinary `fadd`, `fmul`, and related operations without fast-math flags in
the default mode. Explicitly configure the backend against contraction and
reassociation rather than relying on tool defaults. Constrained intrinsics
are not required for this fixed environment; reconsider them only if dynamic
rounding or observable floating-point exceptions are added. Initially support
targets that can meet the contract. In particular, require SSE2 for an initial
32-bit x86 floating-point implementation, or diagnose that target as unsupported;
do not silently fall back to excess-precision x87 arithmetic or relaxed math.

The C boundary needs target-correct lowering even for scalars: small integers
and booleans can need sign/zero extension and parameter/result attributes.
Test calls and returns in both directions against independently compiled C
fixtures. Clang's `CodeGenABITypes` is a possible later reuse path, not a
standalone replacement for this work: it requires Clang AST types and a
code-generation context. Broader Clang integration remains a separate design.

Acceptance:

- scalar examples pass LLVM verification and compile to objects for the
  selected target;
- execute tests match expected output and side-effect order;
- C ABI tests cover booleans, narrow signed/unsigned integers, pointers, and
  floating-point arguments and results at unoptimized and optimized settings;
- strict floating-point tests include a case that would change under fused
  multiply-add and a case that would change under reassociation;
- generated output is stable across repeated builds.

### 10.7 Increment F: value transfer and lifecycle

MIR determines cleanup edges, active/inactive value state, and conditional
drop flags for compiler-managed roots. Emit cleanup on normal exits, early
returns, loop exits, and replacement of active destinations. A directly
relocated root receives no cleanup and need not be cleared. Pointer extraction
does not update automatic cleanup through aliases; programmers prevent
cleanup of the old slot. LLVM supplies no automatic HUC destruction.

Implement:

- structural Basic/Advanced classification, including `clone` opting into Advanced;
- exact self-relocation as a no-op;
- `copy` and `clone`;
- clone-then-replace assignment;
- fixed bitwise relocation of Advanced values with no user transfer hook;
- user `drop`;
- reverse field cleanup;
- raw-pointer copying without pointee cleanup;
- `construct_at` and `destruct_at` with manual destination lifetime and cleanup.

Acceptance:

- straight-line relocations require no hidden state;
- conditional relocations use flags only when necessary;
- unconsumed temporaries survive until their full-expression boundary,
  then clean up in reverse order, independently of backend storage placement;
- consumed temporaries and directly constructed destinations receive no
  duplicate cleanup at expression boundaries;
- every named use of an `Inactive` or `MaybeActive` root is rejected, including
  uses in later arguments of the same call and self-receiver or earlier-`&`
  relocations, while reinitialization of such a root is accepted;
- raw-handle `drop` types relocate without duplicate cleanup;
- Advanced aggregates lower to whole-value `Relocate` without recursive
  field transfers or source clearing;
- copying fixed or writable raw pointers leaves their addresses unchanged;
- `auto` from a raw pointer stays raw, with no cleanup of its pointee;
- extraction through writable pointers is accepted without source clearing
  or alias-driven changes to automatic cleanup state;
- direct field/array-element moves and read-only pointer extraction are rejected;
- manual construction has no old-value drop, intermediate value parameter,
  or automatic destination cleanup registration;
- manual destruction performs full drop/field cleanup without deallocation;
- neither intrinsic updates automatic cleanup state through aliases;
- ordinary binding never invokes `clone`;
- cleanup counters and, where enabled, instrumented runtime tests find no
  backend double destruction;
- optimized operation counts match equivalent explicit C resource-management code.

### 10.8 HUC0 subset completion gate

The chosen HUC0 implementation subset is complete when:

- standalone `.huc0` programs in that subset compile and run through LLVM;
- each included value/lifecycle feature and left-to-right semantics have
  executable tests;
- the supported subset matches the specification, and unsupported features
  produce clear diagnostics;
- diagnostics and output are deterministic;
- no HUC1 facility is required by the HUC0 compiler.

## 11. HUC1-to-HUC0 workstream

### 11.1 Exit result

Given a `.huc1` root and its imports, the stage translator writes one flattened
`.huc0` file with namespace sections. Both import forms and every phase
declaration disappear; imported aliases become canonical qualified references.

The generated HUC0 is checked using the shared HUC0 frontend for the supported
subset. Ordinary-function evaluation also requires that frontend's HIR and MIR;
section 11.3 depends on sections 10.2 through 10.5, not on LLVM emission. This
workstream can precede native code generation. Once the backend is available,
the same output must compile through the public HUC0-to-LLVM translator without
a private compatibility mode.

### 11.2 Increment A: staged indexing and pass-through

Implement:

- `.huc1` module loading with file import blocks, `import1`, `pub import`,
  visible-module checks, and `pub` checks;
- indexed primary families and specialization headers;
- stable declaration/family IDs;
- module-state records separate from shared declaration identities;
- opaque body spans;
- pass-through of ordinary selected HUC0 syntax;
- flattened HUC0 output with namespace sections and origin maps.

Acceptance:

- a phase-free HUC1 program translates to semantically equivalent HUC0;
- zero-binder functions/types remain deferred until requested; ones without
  compiler-only types are runtime-capable if requested, while compiler-only
  fields/signatures cannot enter runtime code;
- ordinary imports are processed once and their namespace identities preserved;
- `import1` is erased;
- skipped bodies, including unrequested zero-binder bodies, are never sent to the lexer/parser;
- ordinary `fn` bodies are prepared when encountered, even if uncalled.

### 11.3 Increment B: compile-time interpreter

Interpret typed IR in process. Lower ordinary bodies to the sequenced MIR of
section 10.5, extended with compiler-only values, so evaluation order,
relocation, and cleanup match the LLVM backend.

Values include:

- booleans, integers, `c8`, and IEEE `f32`/`f64` with strict rounding;
- immutable UTF-8 text;
- indexed lists/tuples for compiler-only computations;
- structures, both compiler-only and runtime-capable;
- pointers as a block ID plus a path to a slot or subobject, never a host
  address;
- canonical type and stable semantic IDs as implementation values;
- symbolic runtime expressions for generic `fn`.

Every compile-time object lives in a block: a frame slot, a `let1` object, a
string literal, or an allocation. Each block records its type, which slots
hold live values, and its logical object identity. Resolve pointers through the
active module's block map. Save completed states and merge them in source import
order, replacing conflicting whole-block entries with the later values,
including unchanged ones. Direct and aliased writes change only the active
version.
Structures, pointers, and allocations use the same rules; no configuration
keyword or scalar-only subset is introduced. Sharing unchanged representations
is an implementation choice, not language-level cloning.

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

- explicit `let1` storage and compiler calls of deferred functions/types run deterministically;
- `let1 mut` assignments follow processing order within their module, and
  redeclaring a `let1` in the same scope is an error;
- assignment, pointer writes, methods, and compiler calls can change imported
  objects in the active state, without altering dependencies or siblings;
- reversing the root's imports of two modules that only read a shared module
  leaves all compile-time values unchanged;
- diamond imports merge complete values, including unchanged inherited values,
  without rerunning initializers; independent increments of 3 both yield 4;
- aliases resolve to the same object version within a state; whole structures
  and allocation blocks are selected rather than fields combined;
- inactive/deallocated entries remain meaningful after merging, and invalid
  accesses or ownership operations still diagnose;
- runtime definitions and canonical types use their defining module's state;
  evaluation-mode calls use the active caller state; mangling does not gain
  hidden state arguments;
- a specialization may read its completed defining module's missing object
  through read-only fallback; writes through any alias diagnose, and missing
  unfinished state or invalid active lifetime entries remain diagnostics;
- two late preparations that increment a saved counter each see the same
  starting value; they never inherit each other's changes;
- teardown cleans each state's remaining live values in a private view,
  including inherited values, without changing saved dependency results;
- ordinary `fn` calls from compiler execution use available checked HUC
  bodies, structures, methods, pointers, raw storage, floats, target-width
  arithmetic, local mutation, and shared call/instruction limits;
- unsupported signatures and reached operations fail; an untaken ordinary
  branch is still checked as HUC but its unsupported operations are not executed;
- mutable globals, runtime memory, `let1 mut` names, and arbitrary foreign calls
  cannot be evaluated through an ordinary function, though it may change a
  `let1` object through a pointer it receives;
- null, dangling, out-of-bounds, uninitialized, and inactive-slot accesses,
  pointer-to-integer conversion, and ordering of unrelated pointers are
  diagnostics with the evaluation call stack;
- a function-local `let1` assigned by its executing body stays alive until the
  compiler call finishes, then is destroyed with `drop` running in the
  compiler; module-level ones are destroyed at the end of translation;
- local generated code is parsed and can run in evaluation mode;
- repeated identical compiler calls prepare and execute again, preserving all
  permitted compiler-state effects; there is no caching analysis;
- failure includes the phase call stack;
- no host pointer is observable or used as a persistent key.

### 11.4 Increment C: phase control

Implement:

- `if1`;
- three-part, resource-limited `for1`;
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

- generic `struct`;
- generic `fn`;
- generic `let`;
- `auto` and typed binders;
- type/value deduction;
- primary, partial, and full patterns;
- pattern forms: pointers with `mut`, family applications, `raw_storage`,
  repeated binders, and family binders;
- structural partial ordering;
- errors for a specialization declared after a request whose selection it
  would change;
- ambiguity diagnostics;
- owning-file placement with the left-to-right condition, and the end-of-file
  check of a file's specializations against those in its dependencies;
- recursion state: absent, in-progress, complete, failed;
- deterministic generated names and lexical namespace preservation;
- cross-module qualification for requested concrete specializations;
- a deterministic translation-unit request worklist preserving import/source order.

A request from an importer may supply its own type without a reverse import.
Substitution carries canonical identities; it does not change where body
names are looked up, which is the file that wrote the selected definition.
Specialization definitions retain family positions, while generated module
output follows the original request's anchor.

Acceptance:

- the same pattern system works for all three family kinds;
- library traits such as `is_same` and `is_pointer` work through repeated
  binders and `mut`-sensitive pointer patterns, ranked by strict subset;
- a later specialization that would change an earlier selection is rejected,
  while a later, less specific one is accepted;
- a specialization in a file that owns neither the family nor a qualifying
  pattern type, or with a binder before its file's own type, is rejected;
- two files' specializations that can match one request, with neither more
  specific, are diagnosed in the later file unless it also writes the
  overlap;
- only the selected body is parsed;
- recursive functions resolve through in-progress entries;
- compile-time completion and runtime emission demand are tracked separately;
- a runtime-needed specialization brings in its retained runtime dependencies,
  even if it was first evaluated at compile time;
- infinitely recursive type layout fails;
- generic `let` enforces exactly one correctly named residual `let`;
- function-local generic `let` declarations materialize at the family site;
- generated HUC0 contains no unresolved family request;
- flattened output depends on the full runtime-needed specialization closure
  and source-ordered publication boundaries;
- imported generics accept caller-defined types, including private ones,
  without separate-module barriers.

### 11.6 Increment E: `@` and parsed text generation

Implement:

- compile-time evaluation islands;
- erased module/type/function/block phase-expression items;
- forced generic `fn` evaluation;
- ordinary `fn` evaluation from compiler code and explicit `@` entry;
- result materialization rules;
- the compiler-supplied `import1 compiler` module;
- argument-driven value/expression bindings, exact-type deduction, unchanged
  forwarding, `compiler::is_expr_param(parameter)` available to `if1`, and
  `compiler::expr(parameter)` returning expression text during execution;
- `compiler::Code` with `local`, `module`, and `around` constructors and `+`;
- `#` insertion statements, including holes filled by the next block;
- `compiler::emitting()` and `compiler::error(message)`;
- `compiler::allocate` and `compiler::deallocate`, with allocations that can
  persist in `let1` storage, leak checks at the end of each top-level
  evaluation and of the translation unit, and rejection of retained runtime
  calls;
- generation modes and forward insertion points;
- immutable generated buffers parsed before following source;
- ordered module output after the original request's top-level anchor;
- fresh private preparation for each compiler call, program definition records,
  once-per-owner publication, and shared identical generated declarations.

HUC1 parses and checks inserted HUC0 together with original source. Reuse the
runtime frontend rules while retaining a printable, independently checked HUC0
boundary. A code value is built before its `#` inserts it; queued declarations
are not visible to the call that built them. Code with a hole joins the next
source block, establishing names and loop context before that block is checked.
Generated loops use the ordinary C-style header from the runtime frontend;
this increment needs no generator-specific loop syntax. Keep `compiler::around`
for wrapping an original block in loops, scoped resources, or other statements.
It is not limited to filling a loop body, and it does not make its suffix run
on early exit. Cleanup must use ordinary value destruction. No range-based
loop syntax is introduced.

Program-mode preparation retains a checked body and ordered module output;
each compiler call prepares a fresh private evaluation-mode body and suppresses
module output without consuming publication state. A later runtime request must
obtain the program definition even if evaluation has already succeeded.
Repeated runtime uses do not rerun a definition's `#` insertions. Code builders
execute on every compiler call, like other functions; insertion is the effect
of a `#` statement. An explicit `let1` may store the returned code for later
use, and each `#` still inserts it. Do not implement result/preparation caches,
frontend/build-artifact caches, or function classification for caching.

Acceptance:

- `@` cannot appear on a type;
- expression arguments resolve their represented type without executing calls,
  reading storage, or producing transfer/cleanup effects;
- one ordinary parameter accepts value and expression bindings on different
  compiler calls; kinds come from arguments, not callee use or trial execution;
- literals, `let1` objects, and evaluator-owned plain `let` objects bind as
  values; runtime-dependent arguments bind as expressions regardless of folding;
- compound arguments, explicit inner evaluation, and no-runtime-input calls
  follow the classification rules; unsupported value evaluation diagnoses;
- `is_expr_param` supports mixed bindings in one call and `if1` discards the
  incompatible branch, without exposing argument contents to preparation;
- runtime preparation resolves the query to false without a runtime tag or
  changes to symbol identity; ordinary eager functions still emit normally;
- generic deduction and partial/full specialization select container-specific
  builders, including writable versus read-only pointer types;
- nested forwarding retains the original expression, and text can be retained
  in `let1` after a builder returns;
- captured expressions receive normal HUC1 substitutions and parenthesization,
  with origin records but no hidden binding-preservation metadata at insertion;
- runtime globals trigger expression binding without a read, including globals
  whose initializers are constants;
- concrete family names in captured text resolve on insertion, including in a
  different namespace; unused text creates no runtime definition demand, and
  retained references emit definitions readable by the independent HUC0 frontend;
- inserted text uses insertion-site lookup, including valid same-type shadowing,
  and missing names diagnose at insertion rather than code construction;
- generated bindings evaluate container expressions once and destroy owned
  temporaries normally; repeated insertion can execute an expression repeatedly;
- invalid binding-query operands, `compiler::expr` on a value binding or during
  preparation, and direct represented-object operations diagnose; no compiler
  query or expression-binding metadata survives in HUC0;
- nested calls inside an `@` or `#` operand, and compiler API calls in code
  executing in a compiler call, need no further `@`; inside a body, `@` cannot
  read parameters or `let` locals;
- every `@` call uses a private body and inserts nothing into its caller; a
  standalone `@` must have type `void` or `never`, and a `compiler::Code` result is
  rejected with a hint to use `#`;
- `fn answer<>()` with a generated local works as both `@answer()` in a value
  expression and `answer()` at runtime, without introducing caller locals;
- preparation never depends on ordinary argument values or expression text;
  only types and binding-kind metadata are available before execution;
- a compile-time-only generic `fn` request does not emit a runtime definition, while a
  later retained runtime reference requires one;
- ordinary `fn` declarations are prepared when encountered and remain in HUC0
  even when called only at compile time or not called;
- a `#` inserts into the body that contains it; a code builder called from
  another body inserts nothing until that body's `#`;
- nested local/module output is forward-only and ordered; an ordinary method's
  output follows its type, while deferred requests inherit their original anchor;
- `main -> A -> B` keeps all pending module output unavailable until after
  `main`, even to a compiler evaluation through a different function;
- module suppression evaluates arguments but registers no names or publication;
- `compiler::emitting()` selects mode-specific `if1` branches;
  evaluation after publication can use generated declarations,
  including those generated by the evaluated function itself;
- repeated runtime uses publish once while distinct emissions and iterations
  remain distinct; identical generated declarations are inserted once, and other
  duplicates are errors;
- code with a hole joins the following block, with correct `break` and
  `continue` handling, generated names scoped to the inserted block, and
  diagnostics for missing blocks and second holes;
- generated HUC1 syntax is rejected by HUC1, and standalone HUC0 still rejects it;
- forward module references resolve after collection; missing/cyclic compile-time
  dependencies and retroactive changes diagnose;
- diagnostics preserve mixed original/generated spans without renumbering input;
- `compiler::error` reports user text and call/request origins in both modes;
  untaken compiler branches do not report it, and selected-body errors never
  cause specialization fallback;
- malformed error calls (wrong arity or non-text/runtime arguments) are rejected;
- repeated identical compiler calls, including code builders, run again;
  explicitly stored results and once-only program definitions remain distinct.

### 11.7 HUC1 subset and end-to-end completion gates

The chosen staging subset is complete when:

- every numbered construct is removed from generated HUC0;
- flattened generated source is deterministic and independently readable;
- the shared HUC0 frontend accepts all successful HUC1 output within the
  declared subset, with no private extensions;
- each included staging feature has golden and negative tests.

End-to-end completion additionally requires the full driver to compile and run
representative HUC1 programs through HUC0 and LLVM. A staging-first milestone
may finish before that integration gate; it must not claim native execution
coverage that has not yet been tested.

## 12. Source maps and diagnostics

Every node or emitted range carries an origin chain:

```text
LLVM instruction/debug location
  -> HUC0 range
  -> HUC1 selected source or generated text
  -> compile-time call site
  -> specialization request chain
```

Diagnostics should lead with the user-facing origin and show generated layers
as notes. Opaque skipped bodies have no diagnostic origin because their
contents are never parsed.

Original and generated text use separate immutable buffers. Insertion changes
logical parse order, not earlier source offsets. A node can have multiple
spans, such as a generated loop header with an original body. Imports also
retain their buffers; record import chains, namespaces, and module insertion
anchors separately from offsets. Evaluation-mode
bodies have source origins even when they have no printed HUC0 range.

Source maps must use stable file IDs based on normalized module paths and
deterministic generated-buffer identities, not process addresses.

## 13. Testing

### 13.1 Unit tests

Cover:

- tokens and numeric/string literals;
- optional digit separators in every supported base and numeric component;
- name-first declarations, pointer types, list commas, and mandatory control-flow braces;
- `mut` and pointer declarators;
- parser recovery;
- canonical types;
- overload ranking;
- phase pattern matching and ordering;
- interpreter arithmetic and control flow;
- source-map composition.

### 13.2 Golden tests

Check exact:

- AST, HIR, and MIR dumps;
- HUC1-to-HUC0 flattened translation units;
- HUC0-to-LLVM unoptimized textual IR and selected optimized IR;
- diagnostic text and source spans;
- stable generated names.

Golden tests must normalize platform-specific paths without hiding differences
in ordering that affect program behavior.

### 13.3 Negative tests

Include:

- import cycles;
- an import after a declaration or a later `module` line, and a module
  imported twice in one file;
- naming a loaded but invisible module, another file's private declaration,
  or a module through an alias declared only in another file;
- generated text that uses an import alias as a module path, or names a
  builder's private helper;
- a foreign namespace section that adds an ordinary declaration, or a
  section for a namespace with no visible module;
- a specialization outside an owning file, a binder before the file's own
  type, and an unresolved clash between two files' specializations;
- writes that violate `mut`, and stale, inactive, or deallocated accesses in
  the active compiler state, including after merging imports;
- another file, or a later evaluation, naming a declaration that a deferred
  body generated into its module after that module finished;
- invalid lifetime operations or leaks during a state's private teardown;
- module code from a `#` in a specialization written in another file's
  namespace;
- malformed selected code;
- malformed but skipped code;
- phase leaks;
- relocation from fixed storage;
- use of an Advanced local or parameter after an unconditional relocation;
- use after a relocation in one branch, in a short-circuit operand, in a `?:`
  arm, or in a loop body without reinitialization;
- `f(x, x.size())`, passing one local by value twice, `x.absorb(x)`,
  `x.absorb(wrap(x))`, `f(&x, x)`, and `x.count = consume_and_count(x)`,
  alongside accepted near-misses such as `f(x.size(), x)`,
  `f(length_of(&x), x)`, and `x = rebuild(x)`;
- Advanced extraction through a read-only pointer;
- direct relocation from a field or array element;
- unary `&` applied to a non-place result, literal, type, or function symbol;
- taking `&` of an Inactive or MaybeActive root, including inside a cast;
- wrong arity, non-pointer/read-only operands, or invalid pointee types for
  `construct_at` and `destruct_at`;
- invalid initializers and ineligible Advanced sources in `construct_at`;
- overloads or first-class uses of either lifetime intrinsic;
- attempts to use former address-taking or `std::` intrinsic spellings;
- `ptr_as` with incorrect arity or a source/target pair outside the three
  supported pointer/pointer and pointer/`usize` cases;
- zero or invalid raw-storage counts, layout overflow, recursive inline layout,
  `copy` of raw storage, and invalid `storage_ptr` operands;
- incorrect query arity and incomplete/non-runtime query types;
- mutation without `mut`;
- copying an Advanced type without `clone`;
- malformed or duplicate `clone` and `drop` methods;
- attempted custom move or relocation hooks;
- constructor obligation failures;
- ambiguous partial specialization;
- declaration-level specialization conditions;
- a reached `compiler::error`, including in a selected specialization with an
  otherwise valid primary fallback;
- multiple residual declarations from generic `let`;
- compile-time undefined behavior: null, dangling, out-of-bounds,
  uninitialized, and inactive-slot access, constructing over a live value,
  and destroying an inactive slot;
- compile-time pointer-to-integer conversion, ordering of unrelated pointers,
  and viewing an object of another type through `u8*` or `c8*`;
- an ordinary function naming `let1 mut` storage during evaluation;
- a reached foreign call or runtime-storage access during evaluation;
- `compiler::deallocate` with the wrong pointer or count or a live slot,
  allocations leaked at the end of a top-level evaluation, and a retained
  runtime call to either allocation operation;
- resource-limit exhaustion;
- numbered syntax emitted into HUC0.
- invalid C-linkage structures, arrays, and phase types,
  opaque handles, or variadics.

### 13.4 Execute tests

Verify LLVM IR, compile to objects, link, and test the implemented subset:

- typed address-taking and explicit integer-address casts with side-effecting
  operands evaluated once;
- scalar computation;
- argument side-effect ordering;
- temporary receivers surviving an outer observing call and then dropping
  in reverse order of completed initialization;
- cleanup boundaries for discarded results, local and field initializers,
  conditions on both outcomes, and `for` initialization and each update expression;
- loop-local header bindings, left-to-right multiple updates, and `continue`
  cleaning up body locals before updates without destroying header storage;
- `break` and `return` skipping updates while cleaning up active header
  storage, and header cleanup when the first condition is false;
- evaluated-path-only cleanup for short-circuit operands and `?:` arms;
- return-result binding before temporary cleanup, followed by local and
  parameter cleanup, with no cleanup of the value transferred to the caller;
- constructors and field initialization order;
- raw-pointer observation and Advanced-value consumption;
- raw inline storage starting with no live elements, partial initialization,
  Advanced pop, Basic copy-and-retire, clear, and container relocation;
- one-slot optional storage without any default construction of its payload;
- generic `if1 (is_basic<T>())` selecting Basic source destruction only;
- target query results matching LLVM data layout, including cross targets;
- slot addresses for inline, pointer, and fixed storage;
- slot-address round trips and byte access without extra ownership operations;
- conditional relocations;
- Basic-value binding and assignment;
- explicit copy/clone and clone-then-replace assignment;
- copying null raw pointers;
- bitwise relocation, whole-source deactivation, and self-relocation;
- relocation of nested Advanced aggregates with exactly-once destination
  cleanup and no inactive-source cleanup;
- extraction from manually managed slots with initialized-range bookkeeping,
  including a pop-style return without cloning or a replacement value;
- single evaluation of pointer sources with unchanged pointer addresses and
  no source clearing or per-element flags;
- single evaluation and pointer-before-initializer ordering for `construct_at`;
- manual construction from Basic, relocated Advanced, cloned, constructor,
  and function-result initializers, including final `this` address checks;
- construct/extract/reconstruct/destruct cycles in manually managed slots;
- manual destruction of nested fields, unchanged pointer addresses, and
  continued use of the backing allocation after the value is destroyed;
- both alias cases: restoring a destroyed automatic local before its cleanup,
  and manually cleaning a value constructed in a root tracked as Inactive;
- user `drop` and reverse field cleanup;
- module imports;
- the same ordinary functions producing equal results through `@` and at run
  time, covering structures, methods, pointers, raw storage, and strict IEEE
  floating point;
- a library container whose allocator selects compile-time allocation, built
  across several compiler calls in a `let1 mut` and destroyed at the end of
  translation;
- full HUC1-to-program examples.

Build the C++20 compiler itself with ASan and UBSan in development and CI to
detect bugs in the compiler implementation, including its backend code.

Instrumenting generated HUC programs is separate work. ASan needs LLVM's
instrumentation pass, the appropriate IR annotations, and its target runtime.
UBSan-style checks need explicit HUC lowering before optimization can exploit
the unchecked operation: feeding HUC-generated IR to Clang with
`-fsanitize=undefined` does not recreate Clang's source-level checks. Track this
support separately rather than treating sanitizer flags as complete coverage.
Keep explicit cleanup counters and expected-result tests even when sanitizers
are enabled. Tests must not treat intentional HUC undefined behavior as a
required result.

### 13.5 Fuzz and determinism tests

Fuzz:

- lexer;
- HUC0 parser;
- HUC1 delimiter scanner;
- selected-body parser;
- phase interpreter inputs;
- specialization pattern matcher.

Compile the same input repeatedly with different allocator and hash seeds and
require byte-identical HUC0 and normalized LLVM IR artifacts for the same
target, options, and pinned LLVM version.

## 14. Documentation work

Before a compiler feature is considered complete:

- rewrite `docs/language-specification.md` to use `let`, `mut`, constructors,
  HUC0/HUC1, and the finalized numbered semantics;
- replace `docs/huc.ebnf` with shared productions and separate HUC0/HUC1 start
  symbols;
- rewrite `docs/compiler-architecture.md` around the two public libraries and
  LLVM backend and the selected implementation route;
- update README examples;
- maintain `docs/use-cases-and-examples.md` as executable examples become
  supported;
- keep every example paired with its expected result or generated output.

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

Implementation commits should be small and feature-oriented. Start with common
C++20 source, diagnostic, and frontend infrastructure, then preferably the thin
end-to-end slice from section 1.2. Staging first remains an alternative, but
ordinary-function evaluation still depends on shared semantic analysis and
HIR/MIR. Grow those layers for the chosen features alongside staging; LLVM
emission can follow later on that route. The order of these sections is not a
release schedule.

Each completed feature should update the implementation-support list with its
tests and supported targets. Do not label the whole specification implemented
because one pipeline example works.

Each commit must leave existing tests passing. Generated artifacts and IDE
state are not committed.
