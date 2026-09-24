# HUC 0.1 Language Specification

Status: design draft

Revision: 0.1

Target audience: compiler implementers, library authors, and language reviewers

> **Design update:** this document predates several accepted syntax decisions
> and is being updated in stages. The
> [controlling implementation plan](implementation-plan.md) and
> [value-semantics specification](value-semantics.md) take precedence. In
> particular, current HUC uses name-first `let name: Type` declarations,
> fixed-by-default storage, layer-specific `mod`, destructive value transfer,
> `fn clone() -> T`, and
> `fn drop() mod -> void`; it has no source-level `const`, `var`, `copy struct`,
> or C++ reference type.

## 1. Purpose and scope

HUC is an unchecked, ahead-of-time systems programming language. It is intended
to offer C and C++ levels of control and runtime cost through a smaller and more
regular language model.

HUC has two defining mechanisms:

1. A numbered phase model unifies ordinary runtime code, generic
   specialization, compile-time evaluation, reflection, and code generation.
2. A small value model copies Basic values and transfers Advanced values,
   with deterministic cleanup and unchecked observation through `T*`.

HUC is not a memory-safe language. In particular, HUC does not have a borrow
checker, lifetime parameters, mandatory bounds checks, or automatic data-race
prevention.

The 0.1 draft specifies enough of the language to build a bootstrap transpiler.
It intentionally postpones inheritance, exceptions, coroutines, a stable native
ABI, arbitrary user-defined implicit conversions, and general operator
overloading.

### 1.1 How to read the rules

The words **must**, **must not**, **shall**, **shall not**, and **undefined
behavior** state language requirements. “May” gives the implementation a
choice. “Diagnostic” means the implementation must reject the program and
report the relevant source location.

Other recurring terms:

- A **place** or **slot** is a storage location, such as a local or field.
- An **inline** value is a `T` stored directly, rather than a `T*`
  holding its address.
- A **pointee** is the object a pointer points to.
- **Reseating** a pointer means replacing the address stored in its slot.
- A **binding** gives a value to a destination, such as a local or parameter.
- An **active** slot holds a live value. An **inactive** slot must be
  reinitialized before it can be used as a value again and receives no cleanup.
- **Residual** code is the runtime code left after compile-time work.
  **Residualization** produces that code; **materialization** writes a
  compiler-known value as valid HUC0 runtime syntax.
- A **predicate** is a compile-time boolean condition used to filter
  specialization candidates.

### 1.2 Design promises

HUC 0.1 makes these promises:

- `T*` is pointer-sized and has no ownership or lifetime tracking.
- Relocating an Advanced value transfers its representation and makes the
  whole source inactive, without source cleanup or required byte clearing.
- Destruction is deterministic and occurs at statically defined scope exits.
- Function arguments and subexpressions evaluate from left to right.
- An Advanced transfer is fixed, compiler-defined destructive relocation,
  never an overload-selected user call.
- Compilation removes every phase-1 and phase-2 construct before backend code
  generation.
- Selected HUC1 syntax is residualized structurally. Bootstrap raw strings are
  parsed and type-checked by the independent HUC0 translator before any C is
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
restricted to drop-free Basic scalars and raw pointers. A fixed global requires
a constant initializer; an omitted initializer is allowed only for mutable
storage and produces zero or null. Advanced globals, structure globals,
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

Indentation, line width, and spacing around operators are not compiler-enforced
style rules. For example, `a + b`, `a+b`, and `a+ b` have the same meaning.
Whitespace must still separate tokens where needed; changing string contents
or the extent of a line comment is not a formatting-only change. The
[default formatting guide](formatting.md) is a convention, not extra grammar.

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

`in` is a reserved keyword used between a `for1` binding and its iterable.
It does not add an ordinary runtime range loop or a membership operator.

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

A numeric literal may contain a single `_` between adjacent digits valid for
that digit sequence's base. Separators are optional and do not change the
value or type. No group size is required. This rule applies to integer,
fractional, and exponent digit sequences; exponent digits are decimal.

```huc
let plain: i32 = 1000000;
let grouped: i32 = 1_000_000; // same value and type as plain
let mask: u32 = 0xDEAD_BEEFu32;
let bits: u8 = 0b1010_0011u8;
let fraction: f64 = 1_234.56_78;
let scaled: f64 = 1.0e1_0;
```

A separator cannot touch a base prefix, decimal point, exponent marker or
sign, or type suffix. Within a numeric literal, leading, trailing, or repeated
separators are errors: `0x_FF`, `1_`, `1__000`, `1_.0`, `1._0`, `1e_3`,
`1e+_3`, and `1_u32` are invalid numeric spellings. An identifier such as
`_count` remains an identifier, not a malformed number.

Adjacent string literals separated only by whitespace or comments concatenate
in source order into one compile-time string value. This is lexical
concatenation and performs no runtime allocation.

### 3.3 List punctuation

Comma-separated parameter, argument, phase-binder, specialization-pattern,
phase-request, and constructor-initializer lists permit one optional comma
after the last item. The rule is the same on one line or several lines:

```huc
fn add(left: i32, right: i32,) -> i32 {
    return left + right;
}

let first: i32 = add(1, 2);
let second: i32 = add(1, 2,);
```

An empty list has no comma: use `()`, not `(,)`. A trailing comma does not
create another argument or an omitted item. Conditions, grouped expressions,
subscripts, and the single type operand in `as<T>` and `ptr_as<T*>`
are not comma-separated lists. Semicolons remain required where the grammar
uses them; line breaks do not replace them.

### 3.4 Core intrinsic names

HUC intrinsics use unqualified compiler-known names, not `std::` names.
`slot_off(place)` uses ordinary call syntax but cannot be overloaded or used
as a first-class function value. Semantic analysis checks its operand and
argument count. `as<T>` and `ptr_as<T*>` retain their explicit type operands.

Address-taking uses unary `&`, not a named intrinsic call. The former
`addressof` and `std::slot_of` spellings are not aliases for
the new operations. The planned `construct_at` and `destruct_at` also have
unqualified names; their interfaces and lifetime rules remain deferred.

## 4. Type system

HUC checks types at compile time. Structure types are nominal: two separate
structure declarations define different types even if their fields match.
The core types are:

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
```

`usize` and `isize` have the target pointer width. Integer widths are exact.

### 4.1 Plain values

A plain `T` is stored inline wherever its containing object is stored. HUC does
not force ordinary values onto the heap.

Every value type is Basic or Advanced:

- **Basic**: ordinary binding copies the value and leaves the source usable.
- **Advanced**: ordinary binding transfers the value. This is destructive
  relocation: the source becomes inactive and receives no cleanup.

Assignment uses the same category rules, with destination cleanup when
required. Basic and Advanced are category names, not source keywords.

Built-in scalars and raw pointers are Basic. A structure is Basic when every
field is Basic and it declares neither `clone` nor `drop`. Otherwise it is
Advanced. Having `init()` alone does not make it Advanced. `T*` remains Basic
regardless of the pointee's category.

```huc
struct Point {
    let x: i32;
    let y: i32;
}

struct Buffer {
    let mod data: mod u8*;
    let size: usize;

    fn drop() mod -> void {
        probable_free(this->data);
        this->data = null;
    }
}
```

The compiler derives copying for Basic types field by field. Declaring `clone`
makes the type Advanced so an ordinary fieldwise copy cannot bypass custom
logical-copy behavior.

An Advanced value may be duplicated only by the explicit `copy` expression,
when it supports the clone protocol in section 6.6.

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

### 4.3 Values and observation

HUC has no C++-style lvalue or rvalue reference type. A value is stored inline;
a raw pointer observes a separately stored value. Forming or copying a raw
pointer does not transfer the pointee or take over its cleanup.

Resource-managing structures use the ordinary Advanced-value rules and
`drop`. A raw-pointer field receives no automatic pointee cleanup.

### 4.4 Pointer layering

HUC0 accepts at most one `*` suffix on a non-pointer base type.
`T**` is rejected, including equivalent types hidden by aliases.

The non-overloadable unary operator `&place` returns `T*` for a fixed inline
`T` place and `mod T*` for a writable inline `T` place. Its operand must be an
addressable, non-pointer data place. It evaluates the place once without
copying, relocating, destroying, or extending the lifetime of its value. It
does not allocate or create a temporary, or change activation or cleanup state.
Literals, non-place results, types, and function symbols are not valid operands.

`&` rejects raw-pointer slots because their typed addresses would require a
forbidden composed type. Forming a place through an invalid pointer is not
made valid by taking its address.
Binary `&` remains bitwise AND; `&&` remains logical AND. The parser distinguishes
prefix and binary `&` by expression position, not by whitespace.

```huc
let value: Widget = Widget(7);
let observer: Widget* = &value;
let mod writable: Widget = Widget(9);
let writable_observer: mod Widget* = &writable;
```

The separate compiler-known, non-overloadable intrinsic
`slot_off(place) -> usize` returns the untyped address of the storage slot
designated by `place`. It accepts addressable locals, parameters, fields,
elements, and dereferenced data storage, including raw-pointer slots. A pointer
operand designates its pointer slot, not its pointee. Forming a place through
an invalid pointer does not become valid merely because the intrinsic
returns an integer.

The expression identifying the slot is evaluated exactly once. The intrinsic
does not load, copy, relocate, or destroy the stored value, allocate or create a
temporary, extend storage lifetime, or change activation or cleanup state.
Literals, non-place results, types, and function symbols are not valid operands.
The returned integer may be passed to an ordinary function and converted to a
one-level raw pointer using `ptr_as` (section 5.5). `usize` is target-pointer-sized;
the value is a data-storage address, not a portable serialized address.

Both fixed and writable slots may have their addresses taken this way. The
compiler does not track mutation permissions through the integer and cast or
insert permission checks. Correct alignment, storage lifetime, access type,
valid representation, resource management, and cleanup remain the programmer's
responsibility. Writing an actually fixed slot is undefined behavior, even if
the reconstructed pointer has leading `mod`. Raw representation access does
not automatically destroy a replaced value or update compiler-maintained
cleanup state. It does not make inactive storage readable or permit duplicate
cleanup of a resource.

`slot_off` is an untyped escape hatch, not a typed reference. It does
not add composed pointer types or change ordinary typed `mod` checks.

Each `mod` keeps its own meaning inside a structure. A fixed structure
prevents assignment to its fields and mutation of values stored inline in
them. A `mod T*` field still allows mutation of the separate
object it points to. A read-only method may therefore mutate that pointee but
may not replace the address stored in the field.

### 4.5 Nullability

`T*` may contain null. It has a contextual conversion to `bool` in a
condition:

```huc
if (pointer) {
    pointer->run();
}
```

Nullability is intentionally not tracked in the type system.

### 4.6 Structures

`struct` declares a nominal runtime type. Fields are laid out in declaration
order, subject to target ABI alignment and padding.

```huc
struct Pair {
    let first: i32;
    let second: i32;

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
`return` result. No temporary `T` exists and no destructive relocation occurs in
these contexts. This guaranteed destination construction lets `init` observe
the final `this` address; it is not optional backend copy elision. The C17
backend may use explicit destination pointers to preserve this guarantee.

`init` has an implicit `this: mod T*` receiver. Constructor initializer entries
and default initialization establish every field before the body begins; the
body may then mutate fields declared with `mod` before their names. Fixed
fields are initialized by their declaration or initializer entry and cannot
be assigned in the body.

An omitted mutable scalar initializes to zero, a mutable raw pointer
to null, and a mutable structure through its zero-argument constructor. Fixed
fields without declaration initializers must be initialized by each constructor.

Methods receive an implicit non-owning `this`:

- `this: T*` for a read-only method, which is the default;
- `this: mod T*` for a method with trailing `mod`.

Calling a method through a raw pointer observes the pointee and never transfers it:

```huc
widget->update(); // observes the pointee; no relocation
```

### 4.6.1 Lifecycle method

An Advanced structure may declare exactly one lifecycle method:

```huc
fn drop() mod -> void {
    // release non-field resources
}
```

`drop` must be a runtime, zero-parameter method returning `void`. It cannot be
called directly; the compiler invokes it during destruction. Its implicit
receiver is `this: mod T*`. The body runs before fields are destroyed in reverse
order. HUC has no user-defined move constructor, relocation constructor,
assignment operator, or C++-style destructor overload. Explicit logical copying
is customized only by `clone`.

Declaring `drop` makes the structure Advanced.

### 4.7 Indexing and library containers

Raw-pointer indexing is unchecked. Arrays, slices, and inline fixed-capacity
storage are library types rather than additional primitive storage syntax in
0.1. A probable HUC1 standard library can provide `Array<T>`, `Slice<T>`, and
`InlineArray<T, N>` families that generate concrete HUC0 structure types.

Primitive `[N]T` fixed arrays are deferred.

### 4.8 Type aliases

```huc
type Size = usize;
```

An alias is another name for the same type, not a new type.

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
| Unary | unary `+`, `-`, `!`, `~`, dereference `*`, address-taking `&`, `copy`, and HUC1 `@` | Right |
| Postfix | call, index, `.`, `->`, `++`, `--`, and HUC1 family request | Left |

Unary `&` has the same precedence as dereference `*`. Postfix operations bind
more tightly: `&value.field` means `&(value.field)`. Binary `&` retains the
bitwise precedence shown above. Address-taking never creates a reference type.

### 5.1 Left-to-right evaluation

Operands, function arguments, constructor arguments, and initializer elements
are evaluated from left to right. Side effects of an earlier operand are
complete before evaluation of the next operand begins.

For example, `combine(first(), second())` finishes evaluating and binding
`first()` before evaluating `second()`. Passing the same Advanced local by
value twice uses an inactive source on the second binding; evaluation order
does not make that valid.

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

The compiler shall not infer non-aliasing merely because a structure manages
a resource; raw observers may exist.

### 5.4 Member access

`.` accesses a member of an inline value. `->` accesses a member through a raw
pointer. A method call observes its receiver rather than transferring it.

### 5.5 Casts

HUC 0.1 provides explicit casts:

```huc
as<T>(value) // checked by static conversion rules
ptr_as<T*>(pointer_or_address) // raw pointer or usize address reinterpretation
```

`ptr_as` accepts a raw pointer or a `usize` data address and produces the
specified one-level raw pointer type, including a leading `mod` where spelled.
It does not produce a composed pointer type. A `usize` obtained
from `slot_off(place)` can be converted back to a pointer addressing that
same storage while the storage remains valid. Targets must support this
data-address round trip. Arbitrary integer values do not establish valid
storage or access rights.

`ptr_as` is unchecked: the cast does not validate alignment, lifetime,
representation, mutation permissions, or access-type compatibility. For
example, casting a pointer slot's address to `Widget*` does not make that slot
a `Widget`; its bytes represent a pointer, not the pointed-to object. Byte
views through `u8*` or `c8*` follow section 5.3. No function-address conversion
is provided by this rule. Numeric narrowing is explicit. Representation
reinterpretation of values (`bit_as`) is deferred from 0.1.

## 6. Value transfer, copying, and destruction

### 6.1 Destructive-relocation contexts

For an Advanced type, ordinary binding transfers the value. This operation is
**destructive relocation**, not C++ move construction. It occurs only when an
Advanced value is supplied to a destination that stores that value:

- local or field initialization;
- assignment;
- argument binding to a by-value parameter;
- return-value binding;
- aggregate initialization.

Taking a value's address or calling a method does not transfer it. To observe
an inline value through a raw-pointer parameter, pass `&value` explicitly.

Whenever relocation from a named place is otherwise permitted, its source
storage must be writable, expressed by `mod` before the binding name.
Section 6.4 further restricts subobject sources. Fresh unnamed results can
be transferred directly: function results, evaluated `copy` results, and
other temporary Advanced values need no binding `mod`. Direct
`T(arguments)` destination construction has no temporary source place.

### 6.2 Destructive-relocation initialization

For an Advanced `T`:

```huc
let destination: T = source;
```

The source must be eligible under section 6.4. Its representation transfers
to the new destination, and the source becomes inactive without cleanup.
No user function is called by the transfer.

### 6.3 Destructive-relocation assignment

For an Advanced `T`, `destination = source`:

1. evaluates and retains the destination slot;
2. evaluates the source slot;
3. does nothing if both slots are identical;
4. destroys the destination's old value if it is active;
5. transfers the source representation and makes the source inactive.

Exact self-relocation is a defined no-op. Assignment cannot be overloaded.
Named methods express domain-specific operations such as merge, append, swap,
or replace.

### 6.4 Bitwise relocation of Advanced values

Relocating an Advanced value performs a fixed representation transfer:

1. the whole value's representation transfers to the destination;
2. the source value's lifetime ends, and its storage and all field subobjects
   become **inactive**;
3. the destination receives the cleanup obligation, with no source cleanup.

This is bitwise relocation, not recursive invocation of the individual fields'
transfer operations. No source field remains an independently active value,
and no source bytes need to be cleared.
The rule applies whether `clone`, `drop`, or an Advanced field makes the type
Advanced; ordinary binding of a Basic type still leaves its source active.

Field fixedness restricts ordinary field assignment; it does not prevent the
compiler from relocating that field as part of a writable containing source.

Ending the source lifetime does not invoke its `drop` method and does not
destroy or otherwise clean up its source fields. Cleanup belongs to the
destination, which is now the one active value. The inactive bytes need not be
cleared or replaced with a valid “empty” value. Initializing or
assigning a new value into the source storage starts a new lifetime there.

For relocation assignment, the compiler first checks for exact
self-relocation. If source and destination are the same storage, the operation
is a no-op. Otherwise it destroys the old active destination and then performs
the representation transfer above.

This algorithm cannot be overloaded. HUC 0.1 has no move constructor,
relocation constructor, relocation method, or move-assignment hook. Bitwise
relocation defines the behavior, not the machine instructions. It does not
require a memory-copy instruction or give padding bytes a defined meaning.
The backend may use bulk copies, loads and stores, or registers, or remove the
transfer entirely, as long as behavior, source inactivity, and cleanup stay
the same.

Using inactive storage before reactivation is undefined behavior. A compiler
should diagnose an obvious straight-line use, but HUC does not promise
flow-sensitive use-after-relocation prevention.

The implementation must still arrange exactly-once destruction. It may use
static control-flow facts or hidden drop flags at joins where a value is only
conditionally active. Such flags are an implementation detail, not runtime
lifetime checking. Straight-line relocations require no flag.

An explicit Advanced source must be a whole value whose active state
the compiler tracks directly: a named local or parameter, or a fresh temporary
or result. These are called root places.
Whole-value relocation includes subobjects in the same transfer. HUC0
rejects moving an Advanced value out through a pointer dereference,
field selection, or array indexing. Otherwise the containing object
could remain active and later try to clean an inactive subobject without
storing extra state to track which parts are still active.

An eligible root source may replace a valid indirect destination; if it may
alias that destination, exact storage identity is checked before destruction.
`copy *pointer` is valid when the pointee is live and supports cloning; it
leaves that pointee active. Containers manage backing storage and initialized
element ranges explicitly. The deferred raw-storage lifetime intrinsics are
limited to `construct_at` and `destruct_at`, with interfaces and detailed
semantics to be designed later; they do not broaden ordinary relocation-source
eligibility.

Inline values whose correctness depends on a stable address are an unchecked
boundary of this model. For example, destructive relocation does not repair a
self-pointer, an interior pointer into the source, or an address registered
with an external API. A program that later relies on such stale addresses has
undefined behavior. The programmer must instead keep the inline value in one
directly constructed fixed read-only storage location and never relocate it,
redesign the representation, or use storage whose address remains stable.
A mutable address-dependent object must likewise stay at its construction
address while those invariants are needed. HUC 0.1 has no pinning qualifier,
neither infers this restriction, nor provides a custom relocation hook.

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

- For a Basic type, it performs the ordinary copy.
- For an Advanced structure, it calls a method with the effective signature
  `fn clone() -> T`.
- For a raw pointer, it copies the address and never clones the pointee;
  copying a null raw pointer is valid.

```huc
struct Text {
    let bytes: ByteArray; // probable concrete HUC0 library type

    fn init(mod bytes: ByteArray) : bytes(bytes) {
    }

    fn clone() -> Text {
        return Text(copy this->bytes);
    }
}

let a: Text = make_text("hello");
let b: Text = copy a;
```

Declaring `clone` makes the type Advanced, even if all fields are Basic. The
explicit syntax exposes that logical copying may allocate or execute arbitrary
code.
Source code cannot call `clone` directly or take its address; it requests the
operation with `copy`. Destructive relocation itself remains fixed,
non-overloadable, and non-failing. Full assignment, exact self-relocation, and
backend rules are specified in
[Value Semantics and Special Operations](value-semantics.md).

A valid `clone` returns an independent logical duplicate: the original and
its copy must each be destructible without invalidating the other. Owned
resources must be duplicated or given another valid ownership share. This is
HUC's meaning of deep cloning; it need not copy every reachable object.
Raw `T*` fields are non-owning and normally copy only their addresses.
Deliberately shared or interned state may remain shared.

`clone` must not modify the source object's inline storage or leave it invalid.
It may allocate and update external bookkeeping, such as a reference count,
through an explicitly writable observer. The compiler does not prove that
custom code follows these rules.

### 6.7 Manually managed storage

Allocation and deallocation are explicit library or external operations.
Copying a raw pointer does not create cleanup responsibility. Structures that
manage resources must release them in `drop`; raw-pointer fields do not
destroy or deallocate their pointees.

The planned `construct_at` and `destruct_at` operations will address object
lifetimes in manually managed storage. Their interfaces remain deferred.
HUC 0.1 has no built-in allocating constructor expression.

### 6.8 Raw-pointer binding

Initialization, assignment, argument passing, and return of a compatible raw
pointer copy its address. The source stays unchanged, including when null.
Binding may drop pointee mutation permission but cannot add it. No allocation,
pointee copy, pointee destruction, or resource transfer occurs.

### 6.9 Function parameters

The parameter type communicates whether a function observes or receives a value:

```huc
fn inspect(widget: Widget*) -> void; // read-only observation
fn mutate(widget: mod Widget*) -> void; // writable observation
fn receive(widget: Widget) -> void; // copies Basic or consumes Advanced Widget
```

Pass `&widget` to observe an inline value. An Advanced by-value parameter
receives the transferred value and its cleanup obligation. An ordinary
`mod Widget*` can also let a function replace a caller's writable value,
subject to the usual source, destination, and lifetime rules.

### 6.10 Return values

Returning an Advanced local destructively relocates it into the caller's
result location and makes the source inactive. Direct constructor results
instead use guaranteed destination construction. Returning a Basic value
copies it. Returning `T*` never transfers or extends the pointee's lifetime.

### 6.11 Value and pointer conversions

Values and pointers do not implicitly convert to each other. Use `&value`
to observe addressable inline storage and `*pointer` to access a live
pointee. Ordinary Advanced move-out through dereferencing is rejected;
explicit `copy *pointer` may clone a copyable pointee.

## 7. Functions and control flow

### 7.1 Runtime functions

```huc
fn add(left: i32, right: i32) -> i32 {
    return left + right;
}
```

Parameters are by value. Raw pointer parameters observe. Advanced
parameters consume. There is no C++-style lvalue reference, rvalue reference,
reference collapsing, or perfect forwarding.

### 7.2 Local declarations

Runtime locals begin with `let` and use `let [mod] name: Type`. The brackets
here mean that `mod` is optional; they are not source punctuation. Fields,
globals, and parameters use the same name-before-type order, but parameters
omit `let`. A binding's `mod` precedes its name; `mod` inside `mod T*`
still controls access to the pointee, not the binding slot.

```huc
let mod count: i32 = 0;
let result: auto = compute();
let limit: i32 = 100;
```

`auto` requires an initializer and deduces exactly one type. It does not retain
a reference or expression-category qualifier. Write `: auto` explicitly where
inference is allowed. HUC 0.1 does not also accept an omitted `: Type`, the
old type-first declaration form, or a destructuring declaration. Runtime
parameters, fields, and globals require explicit types.

Compiler-only locals use `let2`:

```huc
let2 element: compiler::Type =
    @compiler::find_type("app.Widget"); // future reflection API
let2 mod generated: usize = 0;
```

`let2` never creates runtime storage. `mod` before its name permits reassignment
during translation. `let1` instead names a specialization family whose
selected body must residualize one ordinary runtime `let`. The
`compiler::Type` example illustrates the separately designed future reflection
library; the bootstrap compiler module initially exposes only raw emission.

### 7.3 Conditions and loops

`if`, `for`, and `while` are ordinary control flow in a runtime function.
Their phase-1 forms are specified in section 9.

Conditions and loop headers require parentheses. Every `if`, `else`, `while`,
and `for` body requires braces, even for one statement or an empty body.
An `else if` chain may omit an extra pair of braces around the nested `if`:

```huc
if (ready) {
    run();
} else if (waiting) {
    wait();
} else {
    stop();
}
```

These rules also apply when ordinary control flow executes in the compiler.
Phase-control bodies remain braced deferred bodies. A single unbraced statement
or a lone semicolon is not a loop body. Returning a value requires an explicit
`return expression;`; a final expression in a block is not an implicit return.

### 7.4 No implicit truthiness

`bool` is required by boolean operators and conditions. Raw pointers
have a built-in explicit contextual conversion to `bool`; integers do not.

### 7.5 Function overloading

0.1 permits overloading by arity and parameter types. Resolution uses:

1. exact type matches;
2. built-in lossless numeric promotions;
3. permission-dropping pointer conversion.

No user-defined implicit conversion participates. A tie is a diagnostic.

## 8. Lifetime and safety model

These rules define HUC's safety limits; they are language requirements, not
just advice.

### 8.1 Raw observers are unchecked

```huc
let mod saved: Widget*;

fn remember(widget: Widget*) -> void {
    saved = widget;
}

fn example() -> void {
    let widget: Widget = Widget();
    remember(&widget);
} // widget's lifetime ends

// Dereferencing saved later is undefined behavior.
```

The compiler does not infer or check a relationship between `saved` and
`widget`.

### 8.2 Conditional relocation

For an Advanced `Widget`:

```huc
let mod widget: Widget = Widget();

if (condition) {
    receive(widget);
}

// widget must not be read if the branch transferred it.
```

The source is inactive if the branch was taken, and active otherwise. Using
an inactive value is undefined behavior. Cleanup must still occur exactly
once, using static control-flow knowledge or a drop flag where needed.
Assigning a new value to the source begins a new lifetime there.

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
C17 source
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

Phase 1 produces runtime code; it does not add a third kind of runtime
storage. A normal request for a phase-1 family produces a concrete runtime
declaration. A forced call to a phase-1 function may instead run completely
during expansion. Phase-2 declarations exist only during expansion and can
never become runtime symbols.

Every successfully expanded module must be independently readable HUC0. It
contains no numbered declaration or control-flow keyword, `import2`, `@`,
unresolved family binder, specialization request, compiler-only value, or
compiler metadata handle. Ordinary `import` declarations remain. The HUC1
translator writes one HUC0 module for every residual runtime module before the
HUC0-to-C17 translator begins.

The built-in HUC0 forms `as<T>` and `ptr_as<T*>` use an angle-delimited type
operand but are not phase-family requests. No nominal type or ordinary function
application with phase arguments survives into HUC0.

### 9.2 Uniform phase-1 families

`struct1`, `fn1`, and `let1` use one primary/partial/full specialization model.
The binder list is always the first parenthesized list following the family
name, including for `let1`.
A `fn1` always has a separate runtime-parameter list after its phase header;
write that list as `()` when it is empty.

Primary families have binders and no angle pattern:

```huc
struct1 Cell(T: auto) {
    let mod element: T;
}

fn1 convert(T: auto)(mod value: T) -> T {
    return value;
}

let1 value(T: auto) {
    let value: T = T();
}
```

Partial specializations add an angle pattern and may add a pure compiler
predicate:

```huc
struct1 Cell(T: auto)<T*>(true) {
    let mod element: T*;
}

fn1 convert(T: auto)<T*>(true)(value: T*) -> T* {
    return value;
}

let1 value(T: auto)<T*>(true) {
    let value: T* = null;
}
```

For a `fn1`, the runtime-parameter list is mandatory. Without a predicate, the
first parenthesized list after the angle pattern is therefore the runtime list;
with a predicate, the predicate list precedes it. A `struct1` has no runtime
list. A `let1` places its name before the binder list, like the other families.

Full specializations have an empty binder list and a concrete angle pattern:

```huc
struct1 Cell()<i32> {
    let mod element: i32;
}

fn1 convert()<i32>(value: i32) -> i32 {
    return value;
}

let1 value()<i32> {
    let value: i32 = 0;
}
```

A phase binder is `Name: auto` or `Name: Type`, with an optional default where
permitted. Phase binders are not writable slots and do not take binding `mod`.
`T: auto` binds a compiler entity whose kind is inferred from its uses; a binder
used in a type position must hold a type. A typed binder such as `Count: usize`
binds a compiler value of that type. Primary declarations may give trailing
defaults. Partial and full specializations must not introduce defaults.
Open-ended phase packs are reserved for a later revision.

Requests use angle brackets uniformly:

```huc
let cell: Cell<u64> = Cell<u64>();
let result: u64 = convert<u64>(input);
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
let1 counter(Writable: bool) {
    if1 (Writable) {
        let mod counter: i32 = 0;
    } else {
        let counter: i32 = 0;
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
initializers, inline values versus raw-pointer forms, pointee permissions,
and slot mutability. The family name is not itself a runtime variable; only a concrete
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
let2 EnableTracing: bool = true;

if1 (EnableTracing) {
    fn trace_enabled() -> bool {
        return true;
    }
}

struct Counters {
    if1 (EnableTracing) {
        let mod traced_calls: u64;
    } else {
        let mod calls: u32;
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

The loop header uses `in`, not a second colon. For example, given a finite
compiler iterable `values` of integers:

```huc
for1 (let2 item: i32 in values) {
    io::println(item);
}
```

Use `let2 mod item: i32` for a writable iteration binding or `let2 item: auto`
to infer its type from the elements. This adds no runtime range-loop syntax.

There are no `if2`, `for2`, or `while2` keywords. Inside a `fn2`, predicate, or
existing compile-time evaluation island, ordinary `if`, `for`, and `while`
already execute in the compiler environment.

### 9.7 `fn1`: residual and forced call modes

A `fn1` always separates its phase binder list from its runtime parameter list:

```huc
fn1 choose(T: auto, Greater: bool = true)(mod left: T, mod right: T) -> T {
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
let selected: i32 = choose<i32, true>(runtime_left, runtime_right);
```

The phase arguments and concrete runtime argument types determine the
specialization. Runtime argument values are represented during expansion as
typed symbolic expressions and remain runtime inputs. Ordinary `if`, `for`, and
`while` that depend on those symbolic values are residualized into the
generated `fn`. Phase-1 control uses only compiler values and is resolved while
the specialization is built.

Prefixing the call with `@` instead requires complete compile-time evaluation:

```huc
let folded: i32 = @choose<i32, false>(3, 5);
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
    let2 tracing: bool;
    let2 inline_limit: usize;
}

fn2 add(left: i32, right: i32) -> i32 {
    let2 result: i32 = left + right;
    return result;
}

let2 answer: i32 = @add(20, 22);
```

A phase-2 type cannot appear in runtime storage, a runtime structure field, a
runtime function ABI, or emitted HUC0. A phase-2 value may be used by phase
control, predicates, family selection, forced evaluation, reflection, and
generation. The phase checker must reject every path by which it could leak
into runtime code.

### 9.9 The `@` evaluation-island operator

`@` goes before an expression and makes it run at compile time. This is called
an evaluation island because the surrounding code may still be runtime code.
It does not mark a declaration and never forms part of a type.

The following illustrates the spelling when a compiler query API is available;
the query itself belongs to the future reflection library:

```huc
import2 compiler;

let2 function: compiler::Function =
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
    "fn generated_answer() -> i32 { return 42; }",
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
    compiler::emit_huc("let mod generated_counter: i32 = 0;");
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

Raw text has no automatic name isolation (hygiene). Its identifiers use
ordinary HUC0 name lookup, so a generated name may clash with an existing name
or refer to the wrong declaration. Generator authors must escape literals and
create valid identifiers themselves. The first implementation accepts these
limits; the future structured API in section 10 is intended to address them.

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
extern "C" fn huc_write_bytes(data: u8*, size: usize) -> isize;
```

An `extern "C"` function may use:

- built-in arithmetic types with documented target mappings;
- `void` as a return type;
- one-level raw pointers to built-in types.

It must not expose HUC Advanced values with `drop`, phase-1/2 types, or
compiler metadata. Resource management across C boundaries is expressed by
documented functions returning or accepting raw pointers or scalar handles.
Copying or passing a raw pointer does not itself establish or release any
cleanup obligation.

C-layout structure attributes, opaque foreign handle declarations, pointer
chains, and C variadics are deferred from 0.1. A small C shim can flatten such
an interface to the supported scalar and one-level-pointer boundary.

The bootstrap compiler emits ISO C17 with explicit HUC construction,
relocation, and cleanup. Generated C is an implementation technique, not a
stable HUC binary ABI. C++ interoperability may be added later through C ABI
adapters or a separate binding/backend design; arbitrary C++ headers are not
directly consumable in 0.1. The compiler itself is implemented in C++20.

## 14. Undefined behavior summary

The following list is representative rather than exhaustive:

- invalid raw-pointer dereference or arithmetic;
- use of inactive Advanced storage;
- violation of an external function's contract;
- out-of-range unchecked indexing;
- signed overflow, invalid shifts, and integer division by zero;
- data races;
- accessing a value through an incompatible pointer type;
- writing actually fixed storage through an untyped slot address;
- raw slot manipulation that violates ownership or cleanup obligations;
- reading uninitialized storage;
- returning or storing an observer and later using it after its pointee dies.

Because undefined behavior is part of HUC's low-level contract, a compiler may
optimize on the assumption that it does not occur.

## 15. Deliberate omissions from 0.1

The following require separate proposals:

- checked references or lifetime analysis;
- primitive fixed arrays;
- representation reinterpretation with `bit_as`;
- requester-side placement of cross-module family specializations;
- inheritance and implicit subtype conversion;
- exceptions and stack unwinding;
- coroutines and async suspension;
- shared ownership as a primitive type;
- user-defined move/relocation constructors or assignment operators;
- arbitrary implicit conversions;
- user-defined operator overloading;
- dynamic runtime reflection;
- a stable HUC-to-HUC binary ABI;
- compile-time network access or untracked host execution.

Shared ownership, when needed, should initially be ordinary library types such
as the HUC1 families `Rc<T>`, `Arc<T>`, and `Weak<T>`. Staging gives their
specializations concrete HUC0 names. Those Advanced handles use normal destructive
relocation; copying them explicitly exposes reference-count changes.

## 16. C++ comparison

The closest C++ equivalents are:

| HUC | Approximate C++ |
|---|---|
| `T*` | `T*` |
| `&value` | `std::addressof(value)` or `&value` |
| `slot_off(place)` | a data-slot address converted to `std::uintptr_t` |
| `copy a` | clone/copy construction, explicitly selected |
| `fn1` | a subset spanning templates and `constexpr` |
| `fn2` | roughly `consteval` plus meta facilities |
| `if1` | roughly `if constexpr` |
| `struct1` | type generation/monomorphization |
| `compiler::emit_huc` | raw generated source or a build-time code generator |

This comparison is explanatory, not definitional. In C++, `std::move` merely
selects operations such as a possibly user-defined move constructor; the source
object remains alive and is destroyed later. A HUC Advanced transfer is
instead fixed destructive relocation: the source lifetime ends immediately,
without source `drop` or field cleanup. HUC does not inherit C++ value categories,
overload-selected moves, reference collapsing, preprocessor macros, template
substitution rules, or unspecified operand order.

## 17. Conformance and evolution

The bootstrap driver exposes:

```text
huc lower <root.huc0> --out-dir <c-directory>
huc stage <root.huc1> --out-dir <huc0-directory>
huc build <root.huc1> --out-dir <build-directory>
```

Every implementation must report its language revision and target triple.
Feature experiments must be opt-in and must not silently change 0.1 semantics.

The language specification, not emitted C behavior, is authoritative. If the
backend language has a different evaluation order, destruction rule, or name
lookup rule, the transpiler must generate code that preserves HUC semantics.
The planned bootstrap transpilers and command-line driver will be implemented
in C++20; the first HUC0 backend emits C17 source. Ownership and cleanup are
lowered explicitly before emission rather than supplied by backend RAII.

## Appendix A: Consolidated example

```huc
module demo.main;

import std.io as io;

struct Widget {
    let value: i32;

    fn init(value: i32) : value(value) {
    }

    fn clone() -> Widget {
        return Widget(this->value);
    }
}

fn inspect(widget: Widget*) -> void {
    io::println(widget->value);
}

fn consume(widget: Widget) -> void {
    inspect(&widget);
}

fn1 choose(T: auto, Greater: bool = true)(mod left: T, mod right: T) -> T {
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
    let mod first: Widget = Widget(7);
    inspect(&first);

    let mod second: Widget = first;
    let independent: Widget = copy second;

    let a: i32 = 3;
    let b: i32 = 5;
    io::println(choose<i32, true>(a, b));
    io::println(@choose<i32, false>(3, 5));

    consume(second);
    inspect(&independent);
    return 0;
}
```

At the end of `main`, `independent` is destroyed. The value from `second`
was destructively relocated into `consume` and destroyed there. `first` and
`second` are inactive and receive no cleanup.
