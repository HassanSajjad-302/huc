# HUC 0.1 Language Specification

Status: design draft

Revision: 0.1

Target audience: compiler implementers, library authors, and language reviewers

> **Reconciliation notice:** this document predates several accepted syntax
> decisions and is being rewritten incrementally. The
> [controlling implementation plan](implementation-plan.md) and
> [value-semantics specification](value-semantics.md) take precedence. In
> particular, current HUC uses `let`, fixed-by-default storage, layer-specific
> `mod`, `T&` unique ownership, `fn clone() -> T`, and
> `fn drop() mod -> void`; it has no source-level `const`, `var`, `copy struct`,
> or C++ reference type.

## 1. Purpose and scope

HUC is an unchecked, ahead-of-time systems programming language. It is intended
to offer C and C++ levels of control and runtime cost through a smaller and more
regular language model.

HUC has two defining mechanisms:

1. A numbered phase model unifies ordinary runtime code, generic
   specialization, compile-time evaluation, reflection, and code generation.
2. A small ownership model distinguishes unchecked observation (`T*`) from
   automatic unique ownership (`T&`).

HUC is not a memory-safe language. In particular, HUC does not have a borrow
checker, lifetime parameters, mandatory bounds checks, or automatic data-race
prevention.

The 0.1 draft specifies enough of the language to build a bootstrap transpiler.
It intentionally postpones inheritance, exceptions, coroutines, a stable native
ABI, arbitrary user-defined implicit conversions, and general operator
overloading.

### 1.1 Normative terminology

The words **must**, **must not**, **shall**, **shall not**, and **undefined
behavior** are normative. “May” grants an implementation choice. “Diagnostic”
means the implementation must reject the program and report the relevant source
location.

### 1.2 Design promises

HUC 0.1 makes these promises:

- `T*` is pointer-sized and has no ownership or lifetime tracking.
- `T&` is pointer-sized when the default allocator is used.
- Relocating a `T&` transfers its address and leaves the source as a usable
  null owner.
- Destruction is deterministic and occurs at statically defined scope exits.
- Function arguments and subexpressions evaluate from left to right.
- A Move transfer is fixed, compiler-defined destructive relocation, never an
  overload-selected user call.
- Compilation removes every phase-1 and phase-2 construct before backend code
  generation.
- Selected HUC1 syntax is residualized structurally. Bootstrap raw strings are
  parsed and type-checked by the independent HUC0 translator before any C++ is
  emitted.

HUC does not promise:

- prevention of dangling pointers, null access, invalid casts, buffer
  overflows, use of inactive storage after relocation, or data races;
- runtime performance better than equivalent optimized C or C++;
- source or binary compatibility with C++;
- that arbitrary compile-time programs terminate.

## 2. Program structure

A staged source file has the extension `.huc1`; a runtime-only source file has
the extension `.huc0`. Each contains one module declaration, zero or more
imports, and declarations. The HUC1 translator writes printable `.huc0`
modules, and the standalone HUC0 translator consumes only that runtime form.

```huc
module app.main;

import std.io;
import app.widget;

fn main() -> i32 {
    return 0;
}
```

The initial implementation may compile a whole program at once. Module
semantics are nevertheless specified now so that separate compilation can be
added without changing the language.

### 2.1 Modules

- A module name is a dot-separated sequence of identifiers.
- A source file must contain exactly one `module` declaration as its first
  non-comment declaration.
- `import a.b;` binds the imported module under its last component, `b`.
- `import a.b as c;` binds the module to `c`.
- Unqualified wildcard imports do not exist in 0.1.
- Every top-level declaration is accessible through a qualified module import;
  HUC 0.1 has no module visibility or `export` keyword.
- Import cycles are a diagnostic in 0.1.

An imported member is named with `::`:

```huc
import std.io as io;
io::println("hello");
```

The configured root module of an executable contains exactly one runtime entry
point with this bootstrap signature:

```huc
fn main() -> i32;
```

Program arguments will be exposed through an opaque standard-library value,
not through a C-style pointer chain. Its API and any alternate entry-point
signature belong to the separate standard-library design.

### 2.2 Declaration order

Top-level names are collected before bodies are checked, so a declaration may
refer to a later declaration in the same module. HUC0 runtime globals are
restricted to drop-free Copy scalars and raw pointers. A fixed global requires
a constant initializer; an omitted initializer is allowed only for mutable
storage and produces zero or null. Move globals, owners, structure globals,
runtime initialization code, and program-exit global destruction are deferred.

## 3. Lexical structure

Source text is UTF-8. The bootstrap compiler accepts ASCII identifiers:

```text
identifier := [_A-Za-z][_A-Za-z0-9]*
```

Later revisions may adopt Unicode identifier classes without changing program
semantics.

Whitespace separates tokens but is otherwise insignificant. A line comment
starts with `//`. A block comment starts with `/*` and ends at the next `*/`;
block comments do not nest.

### 3.1 Numbered keywords

The following phase-related keyword families are indivisible:

```text
fn fn1 fn2
struct struct1 struct2
if if1
for for1
while while1
let let1 let2
import import2
```

The suffix is part of the keyword: `fn1` is not `fn` followed by the integer
literal `1`. The unnumbered spelling is the runtime form; HUC does not spell it
with a `0` suffix. HUC 0.1 supports only the listed forms. A spelling such as
`fn0`, `fn3`, `if2`, or `import1` in the corresponding grammatical position
receives an “unsupported phase class” diagnostic.

There is no `var` keyword. Runtime variables use `let`, phase-1 variable
families use `let1`, and compiler-only variables use `let2`. The numbers
classify availability; they do not request an arbitrary number of compiler
passes. Section 9 defines their exact meaning.

### 3.2 Literals

HUC has:

- decimal, binary (`0b`), octal (`0o`), and hexadecimal (`0x`) integer
  literals;
- decimal and hexadecimal floating-point literals;
- `true` and `false`;
- UTF-8 string literals;
- byte/character literals;
- `null`.

Integer suffixes select a type, for example `12u32` or `0xffu8`. An unsuffixed
integer literal is assigned the first of `i32`, `i64`, or `u64` that represents
its value. An unsuffixed floating literal has type `f64`.

Adjacent string literals separated only by whitespace or comments concatenate
in source order into one compile-time string value. This is lexical
concatenation and performs no runtime allocation.

## 4. Type system

HUC is statically and nominally typed. The core types are:

```text
bool
c8
i8 i16 i32 i64 isize
u8 u16 u32 u64 usize
f32 f64
void
never
T
T*
mod T*
T&
mod T&
```

`usize` and `isize` have the target pointer width. Integer widths are exact.

### 4.1 Plain values

A plain `T` is stored inline wherever its containing object is stored. HUC does
not force ordinary values onto the heap.

Every value type has one of two transfer modes:

- **Copy**: binding or assignment duplicates the value.
- **Move**: binding or assignment destructively relocates the value. A
  non-owner source lifetime ends and its storage becomes inactive; `T&` is the
  special case whose source remains active as a null owner.

Built-in scalars and raw pointers are Copy. A structure is Copy when every
field is Copy and it declares neither `clone` nor `drop`. Otherwise it is Move.

```huc
struct Point {
    let i32 x;
    let i32 y;
}

struct Buffer {
    let mod u8* mod data;
    let usize size;

    fn drop() mod -> void {
        probable_free(this->data);
        this->data = null;
    }
}
```

The compiler derives Copy operations field by field. Declaring `clone` makes
the type Move so an ordinary fieldwise copy cannot bypass custom logical-copy
behavior.

A Move value may be duplicated only by the explicit `copy` expression, which
uses the clone protocol in section 6.6.

### 4.2 Raw pointers: `T*`

`T*` is a nullable, non-owning, unchecked raw pointer.

- Copying it copies one address.
- Destroying it does nothing.
- It may alias any number of other pointers.
- It may outlive the object to which it points.
- Pointer arithmetic is permitted when the pointee type is complete.
- Dereferencing null, dangling, misaligned, one-past-the-end, or otherwise
  invalid pointers is undefined behavior.

`T*` prohibits mutation of `T` through that pointer. `mod T*` permits mutation.
Neither form extends the pointee lifetime or implies exclusive access.

### 4.3 Unique owning pointers: `T&`

In HUC, `T&` is a primitive unique-owner type. It does not mean C++'s lvalue
reference, and HUC has no general reference type or `T&&` type.

An owner has these states:

- **non-null**: it exclusively owns one dynamically allocated `T`;
- **null**: it owns nothing.

Its representation under the default allocator is exactly one address-sized
word. No reference count, control block, lifetime table, or separate ownership
flag is present.

When a non-null owner is destroyed, HUC:

1. invokes the pointee's `drop` operation, if any;
2. destroys its fields in reverse declaration order;
3. returns its storage to the allocator that created it.

Destroying a null owner does nothing.

An owner cannot participate in pointer arithmetic. It converts implicitly to
a permission-compatible `T*` only in an observing context. The reverse
conversion is never implicit.

### 4.4 Pointer layering

Postfix `&` in a type is parsed as an owner constructor. Examples:

```text
T*      RawPtr<T>
T&      Owner<T>
```

HUC0 accepts at most one pointer-like suffix on a non-pointer base type.
`T**`, `T*&`, `T&*`, `T&&`, equivalent constructions hidden through aliases,
are diagnostics. HUC has no unary `&` expression.

The non-overloadable `addressof(inline_place)` intrinsic returns `T*` for a
fixed inline `T` place and `mod T*` for a writable inline `T` place. It rejects
raw-pointer and owner slots because their addresses would require a forbidden
composed type. Writing an owner expression where `T*` is expected instead
performs pointee observation; it never produces an address to the owner word.
Binary `&` remains bitwise AND.

`mod` permissions remain layer-specific through containment. A fixed
structure prevents assignment to its field slots and writable access to an
inline field subobject. It does not remove a leading `mod` carried through a
`mod T*` or `mod T&` field to a separately stored pointee. A read-only method
may therefore mutate that pointee but may not reseat the field.

### 4.5 Nullability

Both `T*` and `T&` may contain null. They convert explicitly to `bool` in a
condition:

```huc
if (owner) {
    owner->run();
}
```

Nullability is intentionally not tracked in the type system.

### 4.6 Structures

`struct` declares a nominal runtime type. Fields are laid out in declaration
order, subject to target ABI alignment and padding.

```huc
struct Pair {
    let i32 first;
    let i32 second;

    fn sum() -> i32 {
        return this->first + this->second;
    }
}
```

All structure members are public in 0.1. HUC has no member-visibility labels,
inheritance, or virtual dispatch.

Construction selects an unnumbered `fn init(...)`. A constructor expression
`T(arguments)` constructs directly in the destination when it directly
initializes a local, a constructor field entry, a by-value parameter, or a
`return` result. `new T(arguments)` constructs directly in the final
allocation. No temporary `T` exists and no destructive relocation occurs in
these contexts. This guaranteed destination construction lets `init` observe
the final `this` address; it is not optional C++ backend copy elision.

`init` has an implicit `mod T* this` receiver. Constructor initializer entries
and default initialization establish every field before the body begins; the
body may then mutate fields whose slots carry trailing `mod`. Fixed fields are
initialized by their declaration or initializer entry and cannot be assigned
in the body.

An omitted mutable scalar initializes to zero, a mutable raw pointer or owner
to null, and a mutable structure through its zero-argument constructor. Fixed
fields without declaration initializers remain constructor obligations.

Methods receive an implicit non-owning `this`:

- `T* this` for a read-only method, which is the default;
- `mod T* this` for a method with trailing `mod`.

Calling a method on an owner observes the pointee and never relocates the owner:

```huc
widget->update(); // no ownership binding, therefore no relocation
```

### 4.6.1 Lifecycle method

A Move structure may declare exactly one lifecycle method:

```huc
fn drop() mod -> void {
    // release non-field resources
}
```

`drop` must be a runtime, zero-parameter method returning `void`. It cannot be
called directly; the compiler invokes it during destruction. Its implicit
receiver is `mod T* this`. The body runs before fields are destroyed in reverse
order. HUC has no user-defined move constructor, relocation constructor,
assignment operator, or C++-style destructor overload. Explicit logical copying
is customized only by `clone`.

Declaring `drop` makes the structure Move.

### 4.7 Indexing and library containers

Raw-pointer indexing is unchecked. Arrays, slices, and inline fixed-capacity
storage are library types rather than additional primitive owner syntax in
0.1. A probable HUC1 standard library can provide `Array<T>`, `Slice<T>`, and
`InlineArray<T, N>` families that residualize to concrete nominal HUC0 types.

Primitive `[N]T` fixed arrays are deferred. This also avoids an ambiguous
`new[]`/`delete[]` distinction for `T&`: a `T&` always owns exactly one `T`.

### 4.8 Type aliases

```huc
type Size = usize;
```

Aliases do not create nominally distinct types.

## 5. Expressions and execution order

HUC uses C-family operators and precedence. The grammar appendix is normative
where this prose is silent.

From lowest to highest precedence:

| Level | Operators/forms | Associativity |
|---|---|---|
| Assignment | `=`, `+=`, `-=`, `*=`, `/=`, `%=` and bit/shift assignments | Right |
| Conditional | `?:` | Right |
| Logical OR / AND | `||`, then `&&` | Left at each level |
| Bitwise | `|`, then `^`, then binary `&` | Left at each level |
| Equality | `==`, `!=` | Left |
| Relational | `<`, `>`, `<=`, `>=` | Left |
| Shift | `<<`, `>>` | Left |
| Additive | `+`, `-` | Left |
| Multiplicative | `*`, `/`, `%` | Left |
| Unary | unary `+`, `-`, `!`, `~`, dereference `*`, `copy`, `new`, and HUC1 `@` | Right |
| Postfix | call, index, `.`, `->`, `++`, `--`, and HUC1 family request | Left |

There is no unary `&`. Binary `&` retains the bitwise precedence shown above.

### 5.1 Left-to-right evaluation

Operands, function arguments, constructor arguments, and initializer elements
are evaluated from left to right. Side effects of an earlier operand are
complete before evaluation of the next operand begins.

For example, if `take` consumes its parameters:

```huc
take(owner, owner);
```

the first parameter receives the allocation and clears `owner`; the second
parameter receives null. HUC never leaves this dependent on backend evaluation
order.

`&&` and `||` short-circuit. Only the selected operand of `?:` is evaluated.

### 5.2 Arithmetic

- Unsigned integer arithmetic wraps modulo \(2^N\).
- Signed integer overflow is undefined behavior.
- Division by zero is undefined behavior.
- A shift by a negative amount or by an amount not less than the left operand
  width is undefined behavior.
- Floating-point behavior follows the selected target's IEEE-754 support unless
  a compilation mode explicitly permits relaxed math.

### 5.3 Pointer operations

Raw pointer arithmetic and comparison have C-like preconditions. Pointers may
alias unless an API explicitly uses a future `restrict` facility. Accessing an
object through an incompatible pointer type is undefined except through `u8*`
or `c8*`.

The compiler shall not infer non-aliasing merely because an address originated
from `T&`; raw observers may exist.

### 5.4 Member access

`.` accesses a member of an inline value. `->` accesses a member through a raw
or owning pointer. A method call is not an ownership-binding context.

### 5.5 Casts

HUC 0.1 provides explicit casts:

```huc
as<T>(value)           // checked by static conversion rules
ptr_as<T*>(pointer)    // raw pointer reinterpretation
```

`ptr_as` is unchecked. Numeric narrowing is explicit. Representation
reinterpretation (`bit_as`) is deferred from 0.1.

## 6. Value transfer, copying, and destruction

### 6.1 Destructive-relocation contexts

**Move** is a type's implicit transfer category. The corresponding operation is
**destructive relocation**, not C++ move construction. Destructive relocation
occurs only when a Move value is supplied to a destination that stores that
value:

- local or field initialization;
- assignment;
- argument binding to a by-value parameter;
- return-value binding;
- aggregate initialization.

Merely naming, testing, dereferencing, or observing an owner does not relocate
it.

```huc
let mod Widget& mod owner = new Widget();

owner->update();         // observe
if (owner) {}            // test
let Widget* raw = owner; // observe
inspect(owner);          // observe if parameter is Widget*

let mod Widget& mod next = owner; // relocate; owner becomes null
consume(next);                   // relocate if parameter is Widget&
```

Whenever relocation from a named place is otherwise permitted, its source
storage must be writable, expressed by trailing `mod`. Section 6.4 further
restricts non-owner subobject sources; primitive owner fields and elements are
the in-band-null exception. A fresh unnamed result place is intrinsically
consumable: function results, `new` owner results, evaluated `copy` results, and
other temporary Move values may bind onward without an unspellable source
`mod`. Direct `T(arguments)` destination construction has no temporary source
place.

### 6.2 Owner destructive-relocation initialization

For:

```huc
let T& destination = source;
```

`source` must be a writable owner slot, such as a parameter or local with
trailing `mod`. The abstract operation is:

```text
destination.address = source.address
source.address = null
```

No user function is called by the transfer. Unlike a relocated non-owner
value, the source owner remains active and may immediately be tested, assigned,
destroyed, or relocated again. It simply owns nothing.

### 6.3 Owner destructive-relocation assignment

For:

```huc
destination = source;
```

where both expressions designate owners of the same type:

1. evaluate and retain the destination slot;
2. evaluate the source slot;
3. if both slots are identical, perform no operation;
4. destroy the destination's old pointee, if non-null;
5. copy the source address into the destination;
6. store null into the source.

Exact self-relocation is therefore a defined no-op.

Assignment cannot be overloaded. A named method must express domain-specific
operations such as merge, append, swap, or replace.

### 6.4 Structural relocation of other Move values

Relocating a non-owner Move value performs a fixed structural operation.
Structure fields transfer in declaration order:

1. a Copy component is copied into the corresponding destination component;
2. a Move component is recursively destructively relocated;
3. after all components have transferred, the source value's lifetime ends and
   its storage becomes **inactive**.

Field fixedness restricts ordinary field assignment; it does not prevent the
compiler from relocating that field as part of a writable containing source.

Ending the source lifetime does not invoke its `drop` method and does not
destroy or otherwise clean up its source fields. Cleanup belongs to the
destination, which is now the one active value. The inactive bytes need not be
cleared or placed into any valid sentinel representation. Initializing or
assigning a new value into the source storage starts a new lifetime there.

For relocation assignment, the compiler first checks for exact
self-relocation. If source and destination are the same storage, the operation
is a no-op. Otherwise it destroys the old active destination and then performs
the structural relocation above.

This algorithm is not overloadable. HUC 0.1 has no move constructor,
relocation constructor, relocation method, or move-assignment hook. A backend
may replace the fieldwise operation with a bitwise representation transfer only
when that is observably equivalent to the fixed semantics.

Using inactive storage before reactivation is undefined behavior. A compiler
should diagnose an obvious straight-line use, but HUC does not promise
flow-sensitive use-after-relocation prevention.

The implementation must still arrange exactly-once destruction. It may use
static control-flow facts or hidden drop flags at joins where a value is only
conditionally active. Such flags are an implementation detail, not runtime
lifetime checking. Straight-line relocations require no flag.

`T&` uses its null representation instead of an additional drop flag whenever
possible.

An explicit non-owner Move source must be an entire compiler-tracked root
place: a named local, parameter, or global; a fresh temporary/result place; or
the source of compiler-generated recursive whole-value relocation. HUC0
rejects moving a non-owner value out through a pointer, owner dereference,
field selection, or array indexing. Otherwise the containing object or owner
could remain active and later try to clean an inactive subobject without
having liveness state in its representation.

A primitive `T&` field or element is exempt because relocation stores null in
the source owner, which remains active. `copy *owner` is also valid because it
leaves the pointee active, as is relocating the whole owner. An eligible root
source may replace a valid indirect destination; if it may alias that
destination, exact storage identity is checked before destruction. Low-level
container relocation through raw storage requires a future compiler-backed
intrinsic rather than an implicit HUC0 operation.

Inline values whose correctness depends on a stable address are an unchecked
boundary of this model. For example, destructive relocation does not repair a
self-pointer, an interior pointer into the source, or an address registered
with an external API. A program that later relies on such stale addresses has
undefined behavior. The programmer must instead keep the inline value in one
directly constructed fixed read-only storage location and never relocate it,
redesign the representation, use a stable-address library container, or place
it behind `T&` so relocation transfers only the owner while the pointee's
address stays stable. A mutable address-dependent object should normally live
behind `T&`; HUC 0.1 has no pinning qualifier, neither infers this restriction,
nor provides a custom relocation hook.

### 6.5 Destruction order

Active automatic values are destroyed in reverse order of completed
initialization:

- on normal block exit;
- before a `return`;
- before `break` or `continue` leaves their scopes;
- when an initialized destination is overwritten.

Fields are destroyed in reverse declaration order after the user `drop` body
runs. Function parameters are local values and follow the same rule.

HUC 0.1 has no exception unwinding. `panic` terminates the process after
implementation-defined diagnostic output; it need not run pending destructors.

### 6.6 Explicit logical copying

`copy expression` requests a distinct logical value.

- For a Copy type, it performs the ordinary copy.
- For a Move structure, it calls a method with the effective signature
  `fn clone() -> T`.
- For `T&`, null produces a null owner. A non-null owner allocates a new `T`
  and initializes it with `copy *source`.
- For a raw pointer, it copies the address and never clones the pointee.

```huc
struct Text {
    let ByteArray bytes; // probable concrete HUC0 library type

    fn init(ByteArray mod bytes) : bytes(bytes) {
    }

    fn clone() -> Text {
        return Text(copy this->bytes);
    }
}

let Text a = make_text("hello");
let Text b = copy a;

let Widget& first = new Widget();
let Widget& second = copy first;
```

Declaring `clone` makes the type Move, even if all fields are Copy. The explicit
syntax exposes that logical copying may allocate or execute arbitrary code.
Source code cannot call `clone` directly or take its address; it requests the
operation with `copy`. Destructive relocation itself remains fixed,
non-overloadable, and non-failing. Full assignment, exact self-relocation, and
backend rules are specified in
[Value Semantics and Special Operations](value-semantics.md).

A valid `clone` returns an independent logical duplicate: resources owned by
the source must be duplicated or otherwise made independently destructible.
This is deep cloning across the type's ownership boundary, not recursive
duplication of every reachable object. Raw `T*` fields are non-owning and
normally copy only their addresses; deliberately shared or interned state may
remain shared. `clone` must not modify the source object's inline storage or
invalidate its invariants, but it may allocate and update external bookkeeping
such as a reference-count control block through an explicitly writable
observer. The compiler does not prove this semantic contract.

### 6.7 Allocation

```huc
let T& aggregate = new T(constructor_arguments);
```

`new T(...)` allocates suitably aligned storage, invokes the selected `init`,
and returns a unique owner. Constructor initializer obligations follow section
5 of the implementation plan. Named factory functions express validated or
fallible construction. Allocation failure terminates in 0.1. A later standard
library may provide a recoverable `try_new`.

Advanced allocation policies use ordinary library owner types. Stateful custom
deleters are deliberately not stored in primitive `T&`, preserving its
one-word representation.

### 6.8 Adoption, release, and reset

Low-level ownership transitions are explicit:

```huc
let T& mod owner = adopt<T>(raw); // raw must designate one compatible allocation
let T* raw = release(owner);      // owner becomes null
reset(owner);                     // destroy pointee and set null
```

Violating `adopt`'s allocation, type, or exclusivity preconditions is undefined
behavior. `release` transfers the cleanup obligation to the programmer.
`release(owner)` and `reset(owner)` require a named owner slot with trailing
`mod`. A primitive owner `swap` operation is deferred from 0.1; a library can
express it using `release`, `reset`, and destructive relocation.

### 6.9 Function parameters

The parameter type communicates transfer intent:

```huc
fn inspect(Widget* widget) -> void;      // read-only observation
fn mutate(mod Widget* widget) -> void;   // writable observation
fn consume(Widget& widget) -> void;      // consume
```

Supplying an owner to a raw-pointer parameter is an implicit observation.
Supplying it to an owner parameter destructively relocates it.

HUC 0.1 does not have an ownership-slot reference that lets a callee reseat the
caller's owner without consuming it. Consume-and-return is the normal form:

```huc
owner = transform(owner);
```

### 6.10 Return values

Returning a Move value destructively relocates it into the caller's result
location. A returned non-owner local becomes inactive. A returned local owner
becomes null, unless the implementation elides the transfer entirely.

Returning `T*` never transfers or extends ownership.

### 6.11 Ownership and overload resolution

An overload set must not contain two candidates whose corresponding parameter
types differ only between `T*`/`mod T*` and `T&`. Such a set would make
observation versus consumption depend on overload ranking and is a diagnostic.

Use different names when the ownership behavior differs.

## 7. Functions and control flow

### 7.1 Runtime functions

```huc
fn add(i32 left, i32 right) -> i32 {
    return left + right;
}
```

Parameters are by value. Raw pointer parameters observe. Owner and other Move
parameters consume. There is no C++-style lvalue reference, rvalue reference,
reference collapsing, or perfect forwarding.

### 7.2 Local declarations

Runtime locals begin with `let`:

```huc
let i32 mod count = 0;
let auto result = compute();
let i32 limit = 100;
```

`auto` requires an initializer and deduces exactly one type. It does not retain
a reference or expression-category qualifier.

Compiler-only locals use `let2`:

```huc
let2 compiler::Type element =
    @compiler::find_type("app.Widget"); // future reflection API
let2 usize mod generated = 0;
```

`let2` never creates runtime storage. A trailing `mod` permits reassignment
during translation. `let1` instead names a specialization family whose
selected body must residualize one ordinary runtime `let`. The
`compiler::Type` example illustrates the separately designed future reflection
library; the bootstrap compiler module initially exposes only raw emission.

### 7.3 Conditions and loops

`if`, `for`, and `while` are ordinary control flow in a runtime function.
Their phase-1 forms are specified in section 9.

### 7.4 No implicit truthiness

`bool` is required by boolean operators and conditions. Raw and owning pointers
have a built-in explicit contextual conversion to `bool`; integers do not.

### 7.5 Function overloading

0.1 permits overloading by arity and parameter types. Resolution uses:

1. exact type matches;
2. built-in lossless numeric promotions;
3. permission-dropping pointer conversion;
4. owner-to-observer conversion.

No user-defined implicit conversion participates. A tie is a diagnostic.
Section 6.11's ownership-only restriction applies before ranking.

## 8. Lifetime and safety model

This section is normative because rejecting an accidental safety claim is part
of HUC's contract.

### 8.1 Raw observers are unchecked

```huc
let Widget* mod saved;

fn remember(Widget* widget) -> void {
    saved = widget;
}

fn example() -> void {
    let Widget& owner = new Widget();
    remember(owner);
} // owner destroys Widget

// Dereferencing saved later is undefined behavior.
```

The compiler does not infer or check a relationship between `saved` and
`owner`.

### 8.2 Conditional owner relocation

```huc
let Widget& mod owner = new Widget();

if (condition) {
    let Widget& mod local = owner;
    consume(local);
}

use(owner);
```

At the final call, `owner` is non-null if the branch was not taken and null if
it was. HUC does not need borrow checking or a compile-time “possibly
relocated” rejection for this owner case. Dereferencing the null state is
undefined behavior.

### 8.3 Data races

Concurrent conflicting access without the synchronization required by the HUC
memory model is undefined behavior. HUC does not infer thread-safety traits.
Atomic operations are provided by a standard library that maps to the target
platform's atomic memory model.

### 8.4 Optional diagnostics and sanitizers

Implementations may warn about likely dangling pointers, null dereferences, or
inactive-value use. Debug modes may insert checks. These facilities must not be
presented as a language guarantee and their absence must not change the meaning
of a well-defined program.

## 9. Numbered phase semantics

### 9.1 Source levels and phase classes

Numbered keywords describe availability, not a request to run an arbitrary
number of compiler passes. HUC has two independently usable source levels:

```text
HUC1 source
    |
    | phase expansion
    v
printable per-module HUC0 source
    |
    | runtime transpilation
    v
C++20 source
```

The first translator must finish all compiler work and write ordinary HUC0.
The second translator consumes that text through the same public HUC0 parser
used for handwritten HUC0. There is no private staged object format between the
two translators.

| Class | Representative spellings | Meaning |
|---|---|---|
| Runtime | `fn`, `struct`, `let`, `if` | Concrete code or storage that survives in HUC0 |
| Phase 1 | `fn1`, `struct1`, `let1`, `if1` | A family or structural expansion operation that produces HUC0 |
| Phase 2 | `fn2`, `struct2`, `let2`, `import2` | Compiler-only declarations, values, and imports that never survive in HUC0 |

Phase 1 is a production boundary, not a third runtime storage universe. A
normal request for a phase-1 family produces a concrete runtime declaration. A
forced call to a phase-1 function may instead execute completely during
expansion. Phase-2 declarations exist only in the expansion environment and
can never become runtime symbols.

Every successfully expanded module must be independently readable HUC0. It
contains no numbered declaration or control-flow keyword, `import2`, `@`,
unresolved family binder, specialization request, compiler-only value, or
compiler metadata handle. Ordinary `import` declarations remain. The HUC1
translator writes one HUC0 module for every residual runtime module before the
HUC0-to-C++20 translator begins.

The built-in HUC0 forms `as<T>`, `ptr_as<T*>`, and `adopt<T>` use
an angle-delimited type operand but are not phase-family requests. No nominal
type or ordinary function application with phase arguments survives into HUC0.

### 9.2 Uniform phase-1 families

`struct1`, `fn1`, and `let1` use one primary/partial/full specialization model.
The binder list is always the first parenthesized list following the family
name, except that `let1` places its binder list immediately after the keyword.
A `fn1` always has a separate runtime-parameter list after its phase header;
write that list as `()` when it is empty.

Primary families have binders and no angle pattern:

```huc
struct1 Cell(auto T) {
    let T mod element;
}

fn1 convert(auto T)(T mod value) -> T {
    return value;
}

let1(auto T) value {
    let T value = T();
}
```

Partial specializations add an angle pattern and may add a pure compiler
predicate:

```huc
struct1 Cell(auto T)<T*>(true) {
    let T* mod element;
}

fn1 convert(auto T)<T*>(true)(T* value) -> T* {
    return value;
}

let1(auto T)<T*>(true) value {
    let T* value = null;
}
```

For a `fn1`, the runtime-parameter list is mandatory. Without a predicate, the
first parenthesized list after the angle pattern is therefore the runtime list;
with a predicate, the predicate list precedes it. A `struct1` has no runtime
list. A `let1` places its name after the optional predicate.

Full specializations have an empty binder list and a concrete angle pattern:

```huc
struct1 Cell()<i32> {
    let i32 mod element;
}

fn1 convert()<i32>(i32 value) -> i32 {
    return value;
}

let1()<i32> value {
    let i32 value = 0;
}
```

A phase binder is either `auto Name` or a normal type followed by a name.
`auto T` binds a compiler entity whose kind is inferred from its uses; a binder
used in a type position must hold a type. A typed binder such as `usize Count`
binds a compiler value of that type. Primary declarations may give trailing
defaults. Partial and full specializations must not introduce defaults.
Open-ended phase packs are reserved for a later revision.

Requests use angle brackets uniformly:

```huc
let Cell<u64> cell = Cell<u64>();
let u64 result = convert<u64>(input);
inspect(value<u64>);
```

Angle tokens are lexed separately. Where `<` could instead begin a comparison,
the parser preserves an ambiguous angle postfix until name resolution
determines whether the left name denotes a phase-1 family. Square brackets do
not request a phase-1 specialization.

A normal `fn1` call may omit deducible phase arguments. Deduction unifies its
runtime parameter types with the runtime argument types. Every non-deducible
binder must be supplied explicitly. A `struct1` or `let1` request always
supplies every non-defaulted binder in its angle list.

### 9.3 Selection and residual declarations

Each `struct1` and `let1` name has one primary family in a scope. Each `fn1`
name also has one primary family in a scope in 0.1; ordinary runtime `fn`
overloading remains a separate facility. The primary must precede its
specializations, and a specialization must initially be declared in the
primary's defining module.

In 0.1, `struct1` families are module-scope declarations; `fn1` families may
be declared at module scope or as type members; and `let1` families may be
declared at module, function, or nested-block scope but never as fields. Local
`fn1` or `struct1` declarations and nested type families are deferred.

For a concrete request, selection proceeds as follows:

1. match phase argument count and the structural angle pattern;
2. bind pattern variables;
3. evaluate the optional predicate as a pure compiler `bool`;
4. discard candidates whose predicate is false;
5. choose the unique structurally most-specific viable candidate.

An exact full pattern outranks a partial pattern. A partial pattern that accepts
a strict subset of another candidate's structural matches outranks that
candidate. The primary is the fallback. Predicates filter candidates but never
rank them, and declaration order never resolves a tie. Equally specific or
incomparable viable candidates are an ambiguity diagnostic. A false predicate
is not an error in the candidate body. A predicate may inspect its bound
arguments and immutable compiler metadata, but it must not mutate ambient
compiler state, perform raw generation, or access untracked external state.

Only after selection does the translator parse and expand the chosen body:

- a `struct1` request produces one nominal concrete runtime `struct`;
- a normal `fn1` request produces one concrete runtime `fn` and a call to it;
- a `let1` request produces exactly one concrete runtime `let` as specified in
  section 9.4.

Concrete specializations are emitted in their defining module. A local
variable-family specialization is emitted at the family declaration site in
its block. The HUC0 printer assigns deterministic concrete names where multiple
specializations would otherwise collide and rewrites their requests to those
names. No unresolved family request may remain in HUC0.

Any phase-1 family may be requested through an ordinary import from another
module. Its generated concrete declaration is emitted in the defining module,
and the requesting module's residual reference stays qualified through that
preserved import. Generated concrete names remain deterministic implementation
details.

This uniform model deliberately permits function-family partial
specialization. It does not use C++ substitution failure, and a failure while
parsing or checking the selected body is an ordinary diagnostic rather than a
reason to try a less-specific candidate.

### 9.4 `let1` variable families

`let` always declares runtime storage. `let2` always declares a compiler-only
value. `let1` does neither directly: it declares a variable family whose
selected body must residualize one runtime variable.

A `let1` header has no result type because the chosen specialization determines
the complete runtime declaration:

```huc
let1(bool Writable) counter {
    if1 (Writable) {
        let i32 mod counter = 0;
    } else {
        let i32 counter = 0;
    }
}
```

Every selected path must leave:

1. exactly one ordinary, unnumbered `let`;
2. whose source-level identifier is the `let1` family identifier;
3. and no other residual runtime declaration or statement.

Compiler-only setup, `let2`, and phase control are permitted in the selection
body because they disappear. Raw source generation is prohibited within a
`let1` selection body because the HUC1 translator could not enforce the
single-variable invariant without parsing emitted text.

Different specializations of one family may choose different value types,
initializers, raw versus owning pointer forms, pointee permissions, and slot
mutability. The family name is not itself a runtime variable; only a concrete
request such as `counter<true>` denotes one.

`let1` is permitted at module scope and at function or nested-block scope. It
is not permitted as an instance field. Every statically requested
specialization is emitted at the family declaration site. Consequently, a
function-local specialization initializes on every runtime execution that
reaches that site; expansion does not add a lazy runtime initialization flag.

### 9.5 Deferred bodies

Family bodies and phase-control bodies are deferred source spans. Indexing
parses a family header, phase condition, loop header, pattern, and predicate,
but does not lex or parse the associated braced body merely to discover its
end.

A dedicated delimiter scanner locates the matching outer `}` while recognizing
only what is necessary to avoid false brace matches:

- nested braces;
- string and character literals;
- line comments;
- block comments.

The following bodies remain opaque until selected:

- every unchosen primary, partial, or full family body;
- the unselected branch of an `if1`;
- the body of a zero-iteration `for1` or `while1`.

No token, parse, name, type, ownership, or phase diagnostic may originate
inside such a skipped body. A skipped body is explicitly not required to be
valid HUC. The delimiter scanner may report only that it cannot locate the
outer closing delimiter. Once a body is selected, the translator lexes, parses,
resolves, and checks it normally in its inherited context.

A phase loop whose body executes at least once may parse that body once and
reuse the parsed form for later iterations. This implementation choice does not
change insertion order or observable compiler effects.

### 9.6 Phase-1 control flow and insertion contexts

`if1`, `for1`, and `while1` execute during HUC1 expansion. They are legal at all
of these scopes:

| Scope at the phase construct | Legal residual output |
|---|---|
| Module | Module-scope declarations |
| Structure or other type body | Members valid in that type body |
| Function body | Runtime statements and block-scope declarations |
| Nested runtime block | Runtime statements and declarations valid in that block |

The selected output is inserted at the phase construct's source position and
must be legal in that exact insertion context. A function statement selected
at module scope, or a module declaration selected inside a block, is a
diagnostic when the selected body is parsed. Phase control does not implicitly
move output one scope outward.

A selected module/type body may interleave residual declarations or members
with compiler-only `let2`, expression, and control statements. Such statements
must use only compiler-available values; they execute during expansion and
erase. Function and block bodies use the mixed HUC1 statement grammar.

```huc
let2 bool EnableTracing = true;

if1 (EnableTracing) {
    fn trace_enabled() -> bool {
        return true;
    }
}

struct Counters {
    if1 (EnableTracing) {
        let u64 mod traced_calls;
    } else {
        let u32 mod calls;
    }
}

fn run() -> void {
    if1 (EnableTracing) {
        log_trace();
    }
}
```

An `if1` condition must be a compiler `bool`. Only its selected branch is
parsed. A `for1` iterates a finite compiler iterable and inserts one expansion
of its body per element. A `while1` reevaluates a compiler `bool` and is bounded
by the evaluator's resource limits. A loop with zero iterations leaves its body
opaque.

There are no `if2`, `for2`, or `while2` keywords. Inside a `fn2`, predicate, or
existing compile-time evaluation island, ordinary `if`, `for`, and `while`
already execute in the compiler environment.

### 9.7 `fn1`: residual and forced call modes

A `fn1` always separates its phase binder list from its runtime parameter list:

```huc
fn1 choose(auto T, bool Greater = true)(T mod left, T mod right) -> T {
    if1 (Greater) {
        if (left > right) {
            return left;
        }
        return right;
    } else {
        if (left < right) {
            return left;
        }
        return right;
    }
}
```

A normal call requests a concrete runtime specialization:

```huc
let i32 selected = choose<i32, true>(runtime_left, runtime_right);
```

The phase arguments and concrete runtime argument types determine the
specialization. Runtime argument values are represented during expansion as
typed symbolic expressions and remain runtime inputs. Ordinary `if`, `for`, and
`while` that depend on those symbolic values are residualized into the
generated `fn`. Phase-1 control uses only compiler values and is resolved while
the specialization is built.

Prefixing the call with `@` instead requires complete compile-time evaluation:

```huc
let i32 folded = @choose<i32, false>(3, 5);
```

In that mode every argument must be a compiler value. Ordinary control flow
executes in the phase evaluator, and the call must produce a result that can be
materialized as HUC0 at its use site. Failure to finish evaluation is a
diagnostic; it does not fall back to a runtime specialization.

### 9.8 Phase-2 declarations

`fn2`, `struct2`, and `let2` are meta-only:

- `fn2` declares a compiler function and has one compiler-parameter list;
- `struct2` declares a compiler-only data type;
- `let2` declares compiler-only storage.

Compiler-only variables are always spelled `let2`, including locals in `fn2`
and fields in `struct2`. Function parameters inherit the phase of their
enclosing function and do not begin with `let2`.

```huc
struct2 BuildOptions {
    let2 bool tracing;
    let2 usize inline_limit;
}

fn2 add(i32 left, i32 right) -> i32 {
    let2 i32 result = left + right;
    return result;
}

let2 i32 answer = @add(20, 22);
```

A phase-2 type cannot appear in runtime storage, a runtime structure field, a
runtime function ABI, or emitted HUC0. A phase-2 value may be used by phase
control, predicates, family selection, forced evaluation, reflection, and
generation. The phase checker must reject every path by which it could leak
into runtime code.

### 9.9 The `@` evaluation-island operator

`@` has exactly one grammatical role: it prefixes an expression and opens a
compile-time evaluation island. It does not annotate a declaration and never
forms part of a type.

The following illustrates the spelling when a compiler query API is available;
the query itself belongs to the future reflection library:

```huc
import2 compiler;

let2 compiler::Function function =
    @compiler::find_function("render");
```

The declared type above is `compiler::Function`, not `@Function` and not
`@compiler::Function`. A compiler call made from otherwise runtime or module
source must use `@` at the point execution enters the compiler. Within a
`fn2`, a specialization predicate, phase-control evaluation, or an already
active `@` island, nested compiler calls execute normally and do not repeat
`@`.

An evaluation island must not read a runtime-only value. Its result may be used
by compiler code without materialization. If it flows into residual runtime
code, it must have an exact HUC0 representation, such as a scalar, boolean,
floating value, or string literal accepted at that location. Compiler handles
and phase-2 structures cannot be materialized as runtime values.

A standalone phase-expression item has this form:

```huc
@compiler_call_returning_void();
```

It is legal at module, type-body, function, and nested-block scope. The
expression must completely evaluate in the compiler and return `void`. The
item itself is erased; raw-generation effects, if any, use the insertion
cursor for that exact source position. This permits call-site generation at
module or type scope without wrapping the call in a fake declaration.

`@this` is reserved for a future contextual-reflection proposal. Bootstrap
HUC1 diagnoses it rather than treating runtime `this` as compiler metadata.

A normal `fn1` call requests a residual specialization. Applying `@` to that
call selects the same family and specialization rules but requires the selected
function to execute completely during expansion.

### 9.10 Imports and residual HUC0

HUC1 has two import forms:

```huc
import app.data;
import2 compiler;
```

`import` exposes runtime declarations and phase-1 families. It remains as an
ordinary import in the corresponding generated HUC0 module when runtime code
depends on it. `import2` exposes compiler-only declarations and is removed.
There is no `import1` in 0.1. Import cycles are rejected with the complete cycle
path.

Residualization removes the phase mechanism itself:

- unnumbered HUC0 syntax passes through after any selected nested phase
  constructs are expanded;
- concrete `struct1`, normal `fn1`, and `let1` requests become unnumbered
  declarations and references;
- phase-1 family declarations, phase control, phase-2 declarations, `import2`,
  and evaluation-island calls disappear;
- a forced materializable result becomes ordinary HUC0 syntax;
- raw generated fragments from section 11 are written into their target module.

The emitted module text, rather than an internal HUC1 AST, is the contract
between rounds. Running the HUC0 translator on it must require no compatibility
mode and must reject any remaining numbered construct.

### 9.11 Identity, recursion, and determinism

A concrete family specialization is identified by:

- the primary family's stable declaration identity and definition hash;
- serialized explicit and deduced phase arguments;
- for a normal `fn1` call, its concrete runtime argument types;
- the target ABI and enabled language features;
- hashes of imported or reflected declarations on which selection or expansion
  depended.

Runtime argument values are not part of a normal function-specialization key.
They are concrete evaluator inputs only for a forced `@fn1` call. Equivalent
requests within one program produce one concrete specialization.

The translator discovers specialization requests across the complete module
graph with a deterministic worklist and expands them to a fixpoint. Each
defining module's generated HUC0 depends on the sorted closure of concrete keys
assigned to it, including requests first discovered in importing modules.
In 0.1, every type named by a cross-module request must already be nameable
through the defining module's own import closure. A requester-local type that
would introduce a reverse dependency is rejected; requester-side specialization
placement is deferred.

Raw-generation effects in a normal family specialization occur once while that
specialization is constructed. Its cache entry records the residual declaration
and ordered raw fragments: local/type fragments enter that declaration, while
module fragments enter the defining module's buffer once. Repeated normal
requests do not replay them. Effectful `fn2` calls, forced `@fn1` calls, and
standalone phase-expression items execute once per actual evaluation occurrence
at that occurrence's inherited insertion cursor.

An identical function specialization requested while it is already being
constructed denotes a recursive reference to the in-progress function. An
infinitely recursive concrete type layout is a diagnostic. A failed
specialization remains failed for the same key so diagnostics and selection do
not depend on request order.

The core compiler environment is deterministic. It has no implicit access to
wall-clock time, randomness, process environment, network, or arbitrary
filesystem state. A future build API may expose tracked external inputs, but
each such operation must register its identity and content hash in the
dependency graph.

An implementation may impose configurable instruction, recursion, allocation,
specialization-depth, and generated-byte limits. Exceeding a limit is a
compile-time diagnostic with the phase call stack. Deterministic source order,
candidate ordering, serialization, and generated naming are required.

## 10. Compiler environment and future reflection

### 10.1 Bootstrap boundary

The bootstrap HUC1 language does not standardize the full reflection and
structured-editing library. It supplies the phase evaluator, internal canonical
type and declaration identities needed for specialization, and the two raw
generation operations in section 11. This deliberately keeps the first
HUC1-to-HUC0 milestone independent of an unfinished public compiler object
model.

The implementation supplies a compiler-only module:

```huc
import2 compiler;
```

Its types and functions are ordinary qualified HUC names. Phase information
comes from `import2`, `let2`, `fn2`, and `struct2`; `@` is never used as a type
marker.

The module's declarations come from a versioned, inspectable generated
interface unit loaded before user HUC1 modules, analogous to an always-included
header. The explicit `import2 compiler` still binds its namespace and makes the
phase dependency visible. The bootstrap interface declares only the two raw
emission operations; the object-model design remains separate.

### 10.2 Future reflection model

A later compiler-library specification is expected to expose immutable
semantic snapshots and stable IDs for concepts such as:

```text
compiler::Environment
compiler::Module
compiler::Declaration
compiler::Type
compiler::Function
compiler::Class
compiler::Statement
compiler::Expression
compiler::Instruction
compiler::Code
```

These names are design directions, not a complete 0.1 API. In particular, this
document does not yet standardize their constructors, query methods, ownership,
or serialization.

The future environment should let compiler programs discover their complete
tracked compilation environment; inspect modules, declarations, types,
functions, statements, and expressions; ask explicit semantic questions; and
use the results to generate code. The model must obey these constraints:

- handles identify immutable snapshots rather than exposing mutable compiler
  AST nodes;
- IDs and metadata iteration order are stable for one translation;
- queries register dependencies so cache invalidation remains correct;
- structured changes, validation, and commits are explicit operations;
- there are no ambient mutable arrays named `types`, `functions`, or
  `variables`;
- only concrete runtime declarations and already materialized family
  specializations are enumerable as declarations; an uninstantiated family
  does not imply an infinite set of types or functions.

A future structured generation API may add typed syntax or semantic builders,
hygiene, and pre-commit checking. Its design is a separate language-library
proposal and does not change the bootstrap raw-text rules below.

### 10.3 Reserved contextual `@this`

A future proposal may make `@this` produce the narrowest compiler handle for
the lexical location:

- in a function, method, or nested block: `compiler::Function`;
- in a type body outside a method: `compiler::Class` or the corresponding
  future general type-declaration handle;
- at module scope: `compiler::Module`.

It would not return a universal environment object. A function handle could
deliberately navigate to its owning class or module and from there reach other
APIs. In a method, `@this` would mean the enclosing compiler function, while
ordinary `this` would remain the runtime `T*` receiver.

This is only a reserved direction. A separate design must settle exact handle
types, navigation, dependency tracking, identity, nested declarations, and
behavior during generated-code expansion. Neither bootstrap milestone
implements `@this`, and no 0.1 program may depend on it.

## 11. Bootstrap raw HUC0 generation

### 11.1 Operations

The bootstrap `compiler` module exposes exactly these raw generation
operations:

```huc
compiler::emit_huc(text);
compiler::emit_huc_module(text);
```

They are compiler operations. A call from otherwise runtime or module source
uses `@` to enter compiler execution:

```huc
@compiler::emit_huc_module(
    "fn generated_answer() -> i32 { return 42; }"
);
```

Inside `fn2`, phase control, a predicate, or an existing `@` island, ordinary
compiler calls do not repeat `@`. Predicates remain pure and cannot call either
raw-generation operation.

`compiler::emit_huc(text)` writes at the inherited call-site insertion cursor.
The text must be valid HUC0 for that cursor:

- at module scope, module declarations;
- in a type body, members;
- in a function or nested block, statements or block-scope declarations.

`compiler::emit_huc_module(text)` appends a raw fragment containing
module-scope declarations to the generated-declaration buffer of the call
site's module, regardless of the helper's lexical definition scope. It cannot
append a statement or type member. Because HUC1 does not parse the fragment,
the HUC0 translator enforces declaration boundaries and validity.

### 11.2 Inherited call-site cursors

A compiler helper inherits the insertion cursor of the call that entered it.
Generation therefore happens where the helper is used, not beside the helper's
definition:

```huc
fn2 declare_local_counter() -> void {
    compiler::emit_huc("let i32 mod generated_counter = 0;");
}

fn main() -> i32 {
    @declare_local_counter();
    generated_counter += 1;
    return generated_counter;
}
```

The generated `let` is inserted in `main` at the phase-call position. Nested
helpers inherit the same cursor until an explicit
`compiler::emit_huc_module` selects the call-site module buffer.

There is no operation meaning “emit one lexical scope above,” no mutation of an
arbitrary already-checked module, and no definition-site emission. A helper
that must produce a module declaration uses `emit_huc_module` explicitly.

Generation calls are applied in evaluator execution order. The translator
records the target module, insertion cursor, emitted bytes, emission call site,
and phase call stack for every fragment. Module buffers and output order must
be deterministic.

### 11.3 Text boundary and validation

Generation is intentionally raw in the bootstrap. The HUC1 translator tracks
and writes the supplied text but does not lex, parse, resolve, type-check, or
ownership-check it. The text is required to be HUC0: it must not contain
numbered constructs, `import2`, `@`, compiler handles, or another unresolved
HUC1 operation.

The public HUC0 translator is the first component that parses the assembled
module. Invalid emitted syntax, an illegal insertion kind, a duplicate name, a
type error, or an ownership error is therefore reported by the HUC0 round. A
chained diagnostic should show:

1. the location in generated HUC0;
2. the raw emission call site;
3. the compiler call stack that produced the fragment.

Raw text is not hygienic. Identifiers in it participate in ordinary HUC0 name
lookup and may collide or capture by textual spelling. Generator authors are
responsible for escaping literals and constructing valid identifiers. These
costs are accepted for the small bootstrap surface; the future structured API
in section 10 is intended to address them.

### 11.4 Facilities not in bootstrap HUC

HUC 0.1 does not define `quote`, structural syntax values, splicing, `emit`,
`gen`, `gen_global`, `gen_up`, `gen_module`, `gen_huc0_module`, or
`parse_huc`. An implementation must not silently treat an ordinary statement
inside `for1` as a raw-generation request. Phase control residualizes selected
ordinary HUC syntax; raw strings enter output only through the two explicit
`compiler` operations.

## 12. Compile-time execution model

The language specifies observable results, not a particular evaluator. A
conforming implementation may interpret typed HUC IR, compile it to a sandboxed
VM, or use another equivalent technique.

Compile-time values have HUC value semantics. Raw addresses into the compiler
process are not materializable and must not be used as persistent cache keys.

A `fn2` may call `fn1` in forced-evaluation mode. A `fn1` specialization may
call `fn2` for metadata computation. A `fn2` cannot call a runtime `fn` unless
that function is exposed as a compiler intrinsic with separately specified
translation-time behavior.

## 13. Foreign interfaces

HUC's first stable interoperation boundary is C:

```huc
extern "C" fn huc_write_bytes(u8* data, usize size) -> isize;
```

An `extern "C"` function may use:

- built-in arithmetic types with documented target mappings;
- `void` as a return type;
- one-level raw pointers to built-in types.

It must not expose `T&`, HUC Move values with `drop`, phase-1/2 types, or
compiler metadata. Ownership across C boundaries is expressed by documented
functions returning or accepting raw pointers and explicit `adopt`/`release`
at the HUC side.

C-layout structure attributes, opaque foreign handle declarations, pointer
chains, and C variadics are deferred from 0.1. A small C shim can flatten such
an interface to the supported scalar and one-level-pointer boundary.

The bootstrap compiler's generated C++20 backend output is an implementation
technique, not a promise that arbitrary C++ headers or ABIs are directly
consumable. The bootstrap compiler itself will be implemented in C17.

## 14. Undefined behavior summary

The following list is representative rather than exhaustive:

- invalid raw-pointer dereference or arithmetic;
- dereferencing a null owner after its ownership was relocated;
- use of inactive Move storage;
- double adoption or adoption of an incompatible allocation;
- violation of an external function's contract;
- out-of-range unchecked indexing;
- signed overflow, invalid shifts, and integer division by zero;
- data races;
- accessing a value through an incompatible pointer type;
- reading uninitialized storage;
- returning or storing an observer and later using it after its pointee dies.

Because undefined behavior is part of HUC's low-level contract, a compiler may
optimize on the assumption that it does not occur.

## 15. Deliberate omissions from 0.1

The following require separate proposals:

- checked references or lifetime analysis;
- primitive fixed arrays;
- representation reinterpretation with `bit_as`;
- a primitive owner `swap`;
- requester-side placement of cross-module family specializations;
- inheritance and implicit subtype ownership conversion;
- exceptions and stack unwinding;
- coroutines and async suspension;
- shared ownership as a primitive type;
- user-defined move/relocation constructors or assignment operators;
- arbitrary implicit conversions;
- user-defined operator overloading;
- dynamic runtime reflection;
- stateful deleters in `T&`;
- a stable HUC-to-HUC binary ABI;
- compile-time network access or untracked host execution.

Shared ownership, when needed, should initially be ordinary library types such
as the HUC1 families `Rc<T>`, `Arc<T>`, and `Weak<T>`. Staging gives their
specializations concrete HUC0 names. Those Move handles use normal destructive
relocation; copying them explicitly exposes reference-count changes.

## 16. C++ comparison

The closest C++ equivalents are:

| HUC | Approximate C++ |
|---|---|
| `T*` | `T*` |
| `T&` | `std::unique_ptr<T>` |
| `addressof(value)` | `std::addressof(value)` or `&value` |
| owner relocation, `let T& b = a` | `auto b = std::move(a)` |
| `inspect(a)` where parameter is `T*` | `inspect(a.get())` |
| `consume(a)` where parameter is `T&` | `consume(std::move(a))` |
| `copy a` | clone/copy construction, explicitly selected |
| `fn1` | a subset spanning templates and `constexpr` |
| `fn2` | roughly `consteval` plus meta facilities |
| `if1` | roughly `if constexpr` |
| `struct1` | type generation/monomorphization |
| `compiler::emit_huc` | raw generated source or a build-time code generator |

This comparison is explanatory, not definitional. In C++, `std::move` merely
selects operations such as a possibly user-defined move constructor; the source
object remains alive and is destroyed later. A HUC non-owner Move transfer is
instead fixed destructive relocation: the source lifetime ends immediately,
without source `drop` or field cleanup. The special `T&` source remains alive
as a usable null owner. HUC does not inherit C++ value categories,
overload-selected moves, reference collapsing, preprocessor macros, template
substitution rules, or unspecified operand order.

## 17. Conformance and evolution

The bootstrap driver exposes:

```text
huc lower <root.huc0> --out-dir <cpp-directory>
huc stage <root.huc1> --out-dir <huc0-directory>
huc build <root.huc1> --out-dir <build-directory>
```

Every implementation must report its language revision and target triple.
Feature experiments must be opt-in and must not silently change 0.1 semantics.

The language specification, not emitted C++ behavior, is authoritative. If the
backend language has a different evaluation order, destruction rule, or name
lookup rule, the transpiler must generate code that preserves HUC semantics.
The planned bootstrap transpilers and command-line driver will be implemented
in C17; the first HUC0 backend emits C++20 source.

## Appendix A: Consolidated example

```huc
module demo.main;

import std.io as io;

struct Widget {
    let i32 value;

    fn init(i32 value) : value(value) {
    }

    fn clone() -> Widget {
        return Widget(this->value);
    }
}

fn inspect(Widget* widget) -> void {
    io::println(widget->value);
}

fn consume(Widget& widget) -> void {
    inspect(widget);
}

fn1 choose(auto T, bool Greater = true)(T mod left, T mod right) -> T {
    if1 (Greater) {
        if (left > right) {
            return left;
        }
        return right;
    } else {
        if (left < right) {
            return left;
        }
        return right;
    }
}

fn main() -> i32 {
    let Widget& mod first = new Widget(7);
    inspect(first);

    let Widget& mod second = first;
    let Widget& independent = copy second;

    let i32 a = 3;
    let i32 b = 5;
    io::println(choose<i32, true>(a, b));
    io::println(@choose<i32, false>(3, 5));

    consume(second);
    inspect(independent);
    return 0;
}
```

At the end of `main`, `independent` destroys its cloned `Widget`. Ownership from
`second` was destructively relocated into `consume` and destroyed there.
`first` and `second` are usable null owners whose destruction does nothing.
