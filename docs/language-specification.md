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

1. One staging model combines deferred angle-header declarations, suffix-1
   compiler storage, imports, and control flow, `@` for compile-time
   evaluation, and `#` for inserting generated code.
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
  reinitialized before it can be used as a value again. Attempting to destroy
  an inactive value is invalid; section 6.4 explains who prevents that cleanup.
- **Residual** code is the runtime code left after compile-time work.
  **Residualization** produces that code; **materialization** writes a
  compiler-known value as valid HUC0 runtime syntax.

### 1.2 Design promises

HUC 0.1 makes these promises:

- `T*` is pointer-sized and has no ownership or lifetime tracking.
- Relocating an Advanced value transfers its representation and makes the
  whole source inactive, without source cleanup or required byte clearing.
- Destruction is deterministic and occurs at statically defined scope exits.
- A directly tracked local or parameter that may have been relocated cannot
  be used again until it is reinitialized; the compiler rejects such uses.
- Function arguments and subexpressions evaluate from left to right.
- An Advanced transfer is fixed, compiler-defined destructive relocation,
  never an overload-selected user call.
- Compilation resolves generic families and removes compiler-only constructs
  before backend code generation.
- Selected HUC1 syntax is residualized structurally. Generated text is parsed
  and checked when HUC1 inserts it; printed HUC0 is checked again by the
  independent HUC0 translator before any C is emitted.

HUC does not promise:

- prevention of dangling pointers, null access, invalid casts, buffer
  overflows, access to relocated storage through raw pointers or untyped
  addresses, or data races;
- runtime performance better than equivalent optimized C or C++;
- source or binary compatibility with C++;
- that arbitrary compile-time programs terminate.

## 2. Program structure

A staged source file has the extension `.huc1`; a runtime-only source file has
the extension `.huc0`. Source starts with a module declaration, followed by
imports and declarations. A root file and its imported sources form one
translation unit. Modules supply namespaces, not separate compilation or
specialization ownership. The HUC1 translator writes one printable, flattened
`.huc0` translation unit; the standalone HUC0 translator consumes that text.

```huc
module app.main;

import std.io;
import app.widget;

fn main() -> i32 {
    return 0;
}
```

Import processing is inclusion-like: process an imported source at its first
import, then resume the importing source. Unlike literal text substitution,
the imported source keeps its own namespace and lexical name lookup. HUC keeps
`module` and `import`; it does not add a HUC `#include` directive.

### 2.1 Modules

- A module name is a dot-separated sequence of identifiers.
- A source file starts with `module name;`. That declaration selects the
  namespace for following declarations. Another `module name;` starts a new
  section, selecting or reopening a namespace; it does not start another
  translation unit. Ordinary authored files normally need only one section.
- `import a.b;` binds the imported module under its last component, `b`.
- `import a.b as c;` binds the module to `c`.
- Unqualified wildcard imports do not exist in 0.1.
- Every top-level declaration is accessible through its namespace. A full
  canonical name uses `::`, for example `app::widget::Widget`; an import alias
  provides the shorter spelling. HUC 0.1 has no module visibility or `export`
  keyword. Imports load source and bind aliases; they are not access barriers.
- Resolve source imports with the configured module-to-file search paths.
  Process each imported source once per translation unit, in depth-first import
  order. A repeated import binds its alias without repeating declarations or
  compiler-stage effects. On return, restore the importer's namespace and
  lexical context. Conflicting aliases are diagnostics.
- Import cycles are a diagnostic in 0.1.

An imported function resolves unqualified names in its own lexical context,
not in the caller's locals. A generic may nevertheless receive any suitable
type available at its request, including a type declared by the importer. It
does not need a reverse import to name that type through its phase parameter.

Flattened HUC0 uses repeated `module name;` sections to preserve namespaces
while keeping declarations in forward processing order. Its first section
names the configured root, even if initially empty. Imports are resolved away;
the printer rewrites imported aliases to canonical names. Reopening a section
is only a change of naming scope, not another import or another execution of
previous declarations. The flattened file is ordinary valid HUC0, with no
private module bundle or hidden HUC1 state.

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
not through a C-style pointer representation. Its API and any alternate entry-point
signature belong to the separate standard-library design.

### 2.2 Declaration order

Runtime name resolution may refer to later top-level declarations in the
translation unit. This does not make future generated declarations available
to compiler execution. Ordinary `fn` bodies are parsed and prepared when
encountered; angle-header `fn` and `struct` bodies remain deferred until requested. Pending
module output becomes available only at its forward insertion boundary
(section 11). HUC0 runtime globals are
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
if if1
for for1
while while1
let let1
import import1
```

The suffix is part of the keyword: `let1` is not `let` followed by the integer
literal `1`. Functions and types use only `fn` and `struct`. An angle-bracket
header makes `fn`, `struct`, or `let` a deferred family, even when it is empty.
Suffix `1` marks compiler storage, imports, or control flow, not genericity.
HUC 0.1 supports only the listed numbered forms. A spelling such as `fn0`,
`fn1`, `fn2`, `struct1`, `struct2`, `let2`,
`if2`, or `import2` in the corresponding grammatical position
receives an “unsupported phase class” diagnostic.

There is no `var` keyword. Runtime variables use `let name: Type`; variable
families use `let name<T: auto> { ... }`; compiler-only variables use `let1`.
There are no compatibility aliases for the former numbered generic declarations
or suffix-2 compiler declarations. Section 9 defines the staging rules.

`in` is an ordinary identifier, not a reserved keyword. Neither `for` nor
`for1` has a range-based form; both use three-part, semicolon-separated headers.

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
subscripts, and the single type operands in casts and type queries
are not comma-separated lists. Semicolons remain required where the grammar
uses them; line breaks do not replace them.

### 3.4 Core intrinsic names

HUC intrinsics use unqualified compiler-known names, not `std::` names.
`construct_at(pointer, value)` and `destruct_at(pointer)`
use ordinary call syntax but cannot be overloaded or used as first-class
function values. Semantic analysis checks their operands and argument counts.
`as<T>` and `ptr_as<Target>` retain their explicit type operands. `ptr_as`
converts between raw-pointer types or between a raw pointer and `usize`.
`size_of<T>()` and `align_of<T>()` are target-layout queries with one type
operand and no value operands. `raw_storage<T, N>` is a built-in storage type,
not a phase family; `storage_ptr(pointer)` exposes its element storage.
`is_basic<T>()` queries the existing Basic/Advanced category at compile time.
These names likewise cannot be overloaded or used as function values.

Address-taking uses unary `&`, not a named intrinsic call. Section 6.7
specifies the manual-storage lifetime operations `construct_at` and `destruct_at`.

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
raw_storage<T, N>
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
regardless of the pointee's category. The built-in `raw_storage<T, N>` type
is always Advanced and has no `clone` or automatic element cleanup (section
6.7.5). A structure containing it is therefore Advanced too.

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

HUC accepts any number of `*` suffixes on a pointer type. For example,
`Widget*`, `Widget**`, and `Widget***` are nullable, non-owning raw-pointer
types. Each additional suffix points to the storage of the preceding pointer
type; it does not add ownership or lifetime tracking. Pointer dereferencing is
unchecked at every level.

The non-overloadable unary operator `&place` returns `T*` for a fixed inline
`T` place and `mod T*` for a writable inline `T` place. For a pointer slot it
adds one pointer level, so taking the address of a `Widget*` place produces
`Widget**`, or `mod Widget**` when the pointer slot is writable. Its operand
must be an addressable data place. It evaluates the place once without
copying, relocating, destroying, or extending the lifetime of its value. It
does not allocate or create a temporary, or change activation or cleanup state.
Literals, non-place results, types, and function symbols are not valid operands.

Forming a place through an invalid pointer is not made valid by taking its
address.
Binary `&` remains bitwise AND; `&&` remains logical AND. The parser distinguishes
prefix and binary `&` by expression position, not by whitespace.

```huc
let value: Widget = Widget(7);
let observer: Widget* = &value;
let mod writable: Widget = Widget(9);
let writable_observer: mod Widget* = &writable;
```

`&` accepts addressable locals, parameters, fields, elements, and dereferenced
data storage, including raw-pointer slots. `&pointer` addresses the pointer
slot, not its pointee. If an integer address is needed, write
`ptr_as<usize>(&place)` (section 5.5). This still evaluates the place once
and applies the normal address-taking checks before casting the pointer.

In particular, taking the address of a named Advanced root, or a field path
through it, requires its tracked state to be Active (section 6.12). To manually
reuse storage after a direct transfer, retain its pointer before the transfer,
or use `raw_storage`. Neither obtaining nor casting an address creates a live
value, extends storage lifetime, or changes automatic cleanup state.

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
`return` result, or is the value operand of `construct_at`. No temporary `T`
exists and no destructive relocation occurs in these contexts. This
guaranteed destination construction lets `init` observe
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
A mutable `raw_storage<T, N>` field instead creates empty storage, without
constructing elements (section 6.7.5).

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

Raw-pointer indexing is unchecked. Arrays, slices, and containers remain
library types. The built-in `raw_storage<T, N>` supplies aligned, contiguous
inline slots without constructing elements or managing their cleanup.
A HUC1 library can build `Array<T>`, `Slice<T>`, and `InlineArray<T, N>`
families that generate concrete HUC0 structure types using this facility.

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
value twice is therefore ill-formed. The first binding relocates the local,
so the second argument names an inactive value (section 6.12). For the same
reason, `f(x, x.size())` is ill-formed while `f(x.size(), x)` is valid.

`&&` and `||` short-circuit. Only the selected operand of `?:` is evaluated.
`?:` yields a value, not a place: an arm that names an Advanced local or
parameter relocates it into the result on that arm's path.

This also applies when the result is used only as a method receiver:

```huc
// a and b are live Advanced locals declared with binding mod.
(condition ? a : b).size(); // transfers the selected value, then observes it
// The result is destroyed at this semicolon. Both a and b are now MaybeActive.
```

To observe one of the existing values, select pointers or select the results
of the observations instead:

```huc
(condition ? &a : &b)->size(); // observes; neither value is transferred
condition ? a.size() : b.size(); // evaluates only the selected call
```

These are alternative examples with live inputs, not consecutive statements
after the consuming example. Fixed Advanced operands cannot be transferred.
There is no special exception based on whether `?:` appears in an initializer,
return, argument, or method receiver.

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
ptr_as<usize>(pointer) // raw data pointer to target-sized integer address
```

`ptr_as<Target>(value)` takes one value operand and evaluates it once. When
`Target` is a raw-pointer type, it accepts a raw pointer or a `usize` data
address. The target can include pointer chains and spelled `mod` permissions.
When `Target` is `usize`, the source must be a raw data pointer. Other target
types and other source/target combinations are rejected.

An unchanged `usize` obtained from `ptr_as<usize>(pointer)` can be converted
back to the original pointer type to recover the same address while the
storage remains valid. Targets must support this data-address round trip.
An integer address is not a portable serialized address. Arbitrary integers
or arithmetic on an address do not by themselves establish valid storage or
access rights. Neither direction transfers a pointee or changes cleanup state.

`ptr_as` is unchecked: the cast does not validate alignment, lifetime,
representation, mutation permissions, or access-type compatibility. For
example, casting a pointer slot's address to `Widget*` does not make that slot
a `Widget`; its bytes represent a pointer, not the pointed-to object. Byte
views through `u8*` or `c8*` follow section 5.3. No function-address conversion
is provided by this rule. The integer carries no mutation permission or
lifetime metadata. Writing actually fixed storage remains undefined behavior,
even after casting to a writable pointer. Byte access does not construct a
value, destroy a replaced resource, or cancel automatic cleanup. Numeric
narrowing is explicit. Representation reinterpretation of values (`bit_as`)
is deferred from 0.1.

## 6. Value transfer, copying, and destruction

### 6.1 Destructive-relocation contexts

For an Advanced type, ordinary binding transfers the value. This operation is
**destructive relocation**, not C++ move construction. It occurs only when an
Advanced value is supplied to a destination that stores that value:

- local or field initialization;
- assignment;
- argument binding to a by-value parameter;
- return-value binding;
- aggregate initialization;
- manual-storage initialization with `construct_at`.

Taking a value's address or calling a method does not transfer it. To observe
an inline value through a raw-pointer parameter, pass `&value` explicitly.

Whenever relocation from a named root is otherwise permitted, its source
storage must be writable, expressed by `mod` before the binding name.
Relocation through `*pointer` instead requires `mod T*`; the pointer binding
itself need not be writable. Section 6.4 defines the source cleanup rules and
further restricts direct subobject sources. Fresh unnamed results can
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
An assignment to an Advanced destination has type `void`, so it cannot be
used as an operand or argument.
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
cleared or replaced with a valid “empty” value. Constructing a new value in
the source storage starts a new lifetime there. Ordinary assignment can do
this for a root tracked as inactive, but not for an inactive indirect slot.

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

Using a local or parameter whose tracked state is Inactive or MaybeActive
is a compile-time error. Whole-value assignment does not read the old value
and is permitted subject to section 6.12. This check does
not track lifetime changes through raw-pointer aliases. Accessing inactive
storage outside the checked cases remains undefined behavior.

Relocation has two source forms:

- A whole named local or parameter, or a fresh temporary or result. These
  are **root places**. The compiler tracks their active state and suppresses
  cleanup after a direct transfer. It may use static control-flow facts or
  separate drop flags at joins. Straight-line transfers require no flag.
- A dereference `*pointer` of type `T`, reached through `mod T*`. This
  transfers a live Advanced pointee and makes its old slot inactive. The
  programmer manages source cleanup: the operation does not find or update
  automatic cleanup state through aliases, including aliases of locals or
  their fields. No runtime ownership lookup or per-element flags are added.

In both cases, the whole source value and all its fields become inactive.
No source destructor runs, no source bytes are cleared or initialized to a
replacement, and the destination receives the cleanup obligation. A source
pointer is evaluated once and its address value is not changed by extraction.
The pointee must be live, correctly aligned, and writable; null or otherwise
invalid pointers do not become valid relocation sources.

For example, a container can extract its last element with
`let result: File = *element_pointer;` and reduce its initialized length so
that the old slot receives no later cleanup. This rule depends on who manages
cleanup, not whether the allocation is on the stack or heap.

Passing `&local` to a function that extracts through its pointer does not
cancel the caller's automatic cleanup of `local`. Allowing cleanup to reach
that inactive storage is undefined behavior. The same applies to an extracted
field when its enclosing object is later destroyed. Any such slot must have
a new live value before pending cleanup reaches it. Ordinary assignment
through a pointer assumes a live destination and cannot be used to construct
into an inactive slot: it would first destroy the old value. Assignment to a
named local whose cleanup state was not updated has the same problem.
Use `construct_at` to initialize inactive manual storage (section 6.7).

Direct relocation from `value.field`, `pointer->field`, or `array[index]`
remains rejected. HUC does not track partially inactive aggregates. Explicitly
dereferencing a writable pointer to such a slot uses the programmer-managed
rule above, not automatic partial-lifetime tracking. `copy *pointer` remains
valid for a live pointee that supports cloning and leaves it active; ordinary
binding of a Basic pointee copies it instead of ending its lifetime.

Either eligible source form may replace a valid indirect destination. If
source and destination may alias, exact storage identity is checked before
destruction; exact self-relocation remains a no-op. `construct_at` initializes
inactive storage and `destruct_at` destroys a live value there (section 6.7).
Pointer extraction uses ordinary dereference syntax and needs no separate
intrinsic.

This is an unchecked lifetime operation even when the surrounding expression
looks like an ordinary read. `let working: T = *p`, `return *p`, `consume(*p)`,
and `construct_at(q, *p)` all extract an Advanced pointee in their value-binding
contexts. The same expressions copy Basic pointees. Calling `p->method()`
observes the pointee; `copy *p` requests an independent logical copy when
supported. A writable observer parameter alone does not promise to preserve
the pointee's lifetime: the function's contract must state any extraction and
the required source cleanup or reconstruction.

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
runs. Function parameters are local values and follow the same rule: active
by-value parameters are destroyed before the call returns to its caller.
On return, the result is bound first, then return-expression temporaries are
cleaned up, then the function's remaining locals and parameters.

HUC 0.1 has no exception unwinding. `panic` terminates the process after
implementation-defined diagnostic output; it need not run pending destructors.

#### 6.5.1 Temporary values

A temporary is an unnamed value created during expression evaluation, such
as a function result used as a method receiver or a discarded constructor
result. A temporary that has not been transferred elsewhere stays alive until
the end of its **full expression**. At that boundary, remaining active
temporaries are destroyed in reverse order of completed initialization.
Basic temporaries have no `drop`, but their lifetimes end at the same boundary.

A full expression includes its nested operands and argument evaluations.
Its boundary is defined by the enclosing construct:

| Construct | Temporary cleanup boundary |
|---|---|
| Expression statement, including assignment | After the whole expression finishes, including any destination replacement, before the next statement |
| Local declaration initializer | After the declared value is fully initialized, before the next statement |
| `return expression;` | After binding the return value, before cleanup of the function's remaining locals and parameters |
| `if`, `while`, or `for` condition | After computing and retaining the condition result, before entering the chosen branch/body or leaving the loop |
| `for` initializer or step expression | After that expression finishes, before evaluating the condition |
| Field initialization | After that field is initialized, before the next field initializer or the constructor body |

Each evaluation of a loop condition or step has its own boundary. The field
rule covers both declaration initializers evaluated during construction and
explicit constructor initializer entries. All arguments of one entry belong
to that entry's full expression; the field itself is not a temporary.
`construct_at` initializes its supplied destination, not an intermediate
temporary. Its helper temporaries belong to the enclosing full expression,
as for other call operands. The constructed value is not automatically
destroyed when that expression ends.

Parentheses, nested calls, individual arguments, and operands of `&&`, `||`,
and `?:` do not introduce earlier boundaries. Temporaries from evaluated
operands stay alive until the enclosing full expression ends. Unevaluated
operands create no values and require no cleanup. Calls made during an
expression have their own local and temporary cleanup inside the callee;
they do not end the lifetimes of the caller's unconsumed temporaries.

For example, assume `make_text()` returns an Advanced `Text`, `view()`
returns a non-owning `View` into it, and `print` uses the view only during
the call:

```huc
print(make_text().view()); // Text stays alive through print, then is destroyed

let view: View = make_text().view(); // Text is destroyed at this semicolon
print(view); // undefined behavior: accesses the destroyed Text's storage
```

Giving the owning value a name makes its lifetime explicit:

```huc
let text: Text = make_text();
let view: View = text.view();
print(view); // text is still alive
```

There is no lifetime extension for a pointer or view stored in a local,
returned, or passed to another function. The same applies to a view obtained
from a condition temporary: keeping that view does not keep its source alive
in the body. Raw-pointer lifetime mistakes remain unchecked.

This rule does not postpone destructive relocation. When an Advanced
temporary is transferred, its source becomes inactive immediately and
receives no cleanup at the full-expression boundary. The destination follows
its own lifetime. In particular, a value transferred into a by-value parameter
is destroyed by the callee unless transferred onward; returning an observer
of that parameter does not extend its lifetime.

Direct construction in a named local, field, parameter, or return result
creates no intermediate temporary. That destination follows its normal
lifetime, not the surrounding expression's temporary cleanup boundary.
An active discarded result, such as `make_text();`, is destroyed at the end
of that statement. The existing rules for `&` are unchanged: it does not
materialize a temporary or make `&make_text()` valid.

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

An Advanced value can be extracted with `*pointer` through `mod T*`, under
the programmer-managed cleanup rules in section 6.4. The old slot is inactive,
not an empty live value, and must be excluded from destruction until a new
value is constructed there. This operation does not deallocate the storage.

Two non-overloadable HUC0 intrinsics manage these lifetimes:

| Expression | Required state of the destination/pointee | Result |
|---|---|---|
| `construct_at(pointer, value)` | Inactive storage suitable for `T` | The same address as a `mod T*`, now pointing to a live `T` |
| `destruct_at(pointer)` | A live `T` | `void`; the slot becomes inactive |

Both infer `T` from a first operand of type `mod T*`. The pointer binding
itself may be fixed. `T` must be a complete runtime object type, not `void`
or `never`. Pointer slots are valid destinations when the supplied pointer has
the appropriate pointer-chain type. There is no additional type operand,
untyped-address overload, or `std::` spelling.

The pointer must be non-null and designate valid, writable storage of
sufficient size and alignment for `T`, within its allocation's lifetime.
A cast alone does not establish these conditions or end a previous value's
lifetime. Neither intrinsic allocates or deallocates the destination storage,
changes the pointer address, or adds automatic cleanup registration. A
constructor or `drop` body may of course manage resources of its own.

#### 6.7.1 Construction in supplied storage

`construct_at(pointer, value)` has exactly two operands. It evaluates the
pointer expression once, then initializes the designated slot from the value
expression, evaluated once. The second operand is an **initializer**, not an
ordinary by-value parameter that would first receive its own `T`.

Type checking and initialization follow the same rules as `let local: T = value`:

- a Basic value is copied and its source remains active;
- an eligible Advanced source is relocated, with the usual `mod` and source
  cleanup rules; `copy source` explicitly requests cloning;
- a direct `T(arguments)` expression constructs in the supplied slot, so
  `init` observes its final address. A direct function-result initializer
  likewise uses that slot as its result destination under the existing
  result-placement rules.

No old value is destroyed. The slot must be inactive when initialization
begins and is live after initialization completes normally. The intrinsic
then returns the destination pointer. It does not initialize a placeholder,
call `clone` implicitly, or clear a relocated source. A constructor call has
the usual argument binding and evaluation order. Other helper temporaries
follow section 6.5.1; no extra expression boundary is introduced.

Examples, each assuming `slot` is inactive before the call:

```huc
construct_at(slot, File(handle)); // direct construction in slot
construct_at(slot, existing); // Advanced transfer; existing requires binding mod
construct_at(slot, copy existing); // explicit clone into new storage
construct_at(number_slot, 42); // Basic initialization; number_slot is mod i32*
```

Use an explicit initializer: `construct_at(slot, T())` for a default
structure constructor, or a value such as `0` for a scalar. There is no
one-operand default-construction form or constructor-argument pack overload.

Constructing over a live value is undefined behavior, not replacement.
Unlike assignment, construction has no self-relocation no-op: the source
must be live and the destination inactive, so they cannot be the same slot.
Overlapping source and destination storage is also invalid.

#### 6.7.2 Destruction without deallocation

`destruct_at(pointer)` has exactly one operand and evaluates it once. It
performs the same destruction as automatic cleanup: run `T.drop`, if any,
then destroy inline fields in reverse declaration order. The slot becomes
inactive after destruction completes normally. Destroying a Basic value
invokes no user code but still ends that value's lifetime.

This is not a direct source-language call to `drop`; direct calls to that
method remain rejected. The intrinsic performs the complete cleanup,
including fields. It neither frees the slot's storage nor promises to clear
its bytes or null the supplied pointer. It returns `void`.

Destroying a null, uninitialized, already destroyed, or extracted slot is
undefined behavior. Repeated destruction is not a no-op. `construct_at` can
start a new lifetime in that slot before a later destruction.

#### 6.7.3 Automatic cleanup and unchecked aliases

Both intrinsics operate through raw pointers. Neither changes an aliased
local's tracked state or the automatic cleanup of an enclosing object,
even for a directly written `&local`. This is the same boundary as pointer
extraction; inlining does not change it.

- After `destruct_at(&local)`, the caller must construct a new live value
  before automatic cleanup or any other value access reaches that slot.
  Ordinary assignment still tries to destroy the old value and is not a
  substitute for `construct_at` here.
- Constructing through a pointer into a root already tracked as Inactive
  does not reactivate that root for named use or automatic cleanup. The
  programmer must manage the constructed value through pointers and destroy
  or extract it before releasing the storage.
- Destruction or extraction of an inline field does not cancel automatic
  field cleanup. Restore that field before enclosing-object use or cleanup
  requires it to be live.

In contrast, an Advanced value transferred from a directly named local into
`construct_at` updates that **source** local's tracked state normally. The
ordinary relocation-state checks and direct receiver/address-argument checks
also apply to the intrinsic operands; they do not establish the destination's
lifetime preconditions.

For example, the pointer remains usable as a storage address across this pair:

```huc
// slot points to one live, manually managed File.
destruct_at(slot); // close its resource; keep the backing storage
construct_at(slot, File(new_handle)); // create a new File in that storage
```

This is not clone-then-replace assignment: the old value is destroyed before
the initializer in the second statement runs. HUC has no exception unwinding;
a terminating panic need not perform pending cleanup.

Incorrect runtime lifetime, alignment, and storage conditions are unchecked
preconditions, not mandatory runtime tests. Static errors such as a read-only
pointer, wrong arity, incompatible initializer, or ineligible Advanced source
require diagnostics. Allocation APIs remain library choices; HUC0 has no
built-in allocating constructor expression.

#### 6.7.4 Target layout queries

`size_of<T>()` and `align_of<T>()` return `usize` constants for the selected
runtime target, never the compiler host. `T` must be a complete runtime object
type; `void`, `never`, phase-only types, and types with recursive inline layout
are rejected. Pointer types, including pointer chains, are complete even when
their ultimate pointee's layout is not yet known.

Size includes padding and is the stride of contiguous `T` slots. Alignment is
the byte alignment required for a `T` address. Both queries take no runtime
arguments, evaluate no values, and start or end no lifetimes. A pointer's size
is its own representation size, not its pointee's size. Layout arithmetic for
allocations must be checked for overflow before allocation or pointer use.
HUC1 may use these queries during specialization once `T` and the target are
known; HUC0 accepts the same queries on concrete types without staging.

`is_basic<T>()` takes the same complete runtime type operand and no value
operands, and returns a constant `bool`. It is true for Basic types and false
for Advanced types, including `raw_storage`. It reports the existing inferred
category; it does not declare a trait, opt a type into copying, or generate
`clone`. HUC1 can use `if1 (is_basic<T>())` to end a copied Basic source's
lifetime without trying to destroy an extracted Advanced source.

#### 6.7.5 Typed raw inline storage

`raw_storage<T, N>` reserves `N` contiguous slots for `T` inside the containing
object, with alignment `align_of<T>()` and size `N * size_of<T>()`. It adds no
tag, length, allocation, or per-element cleanup flags. `T` must be a complete
runtime object type. `N` must be positive and the total size must fit the
target's object-size limits and `usize`. Invalid layouts are compile-time
errors. HUC0 spells `N` as an integer literal; HUC1 can use a compile-time
`usize` expression and emits the resolved literal. Aliases preserve this type.
Zero-capacity libraries specialize to a representation with no storage field.

The zero-argument expression `raw_storage<T, N>()` creates the storage with
every element slot inactive. It calls no `T` constructor and need not initialize
any payload bytes. An omitted mutable local or field of this type has the same
initialization; fixed bindings require an initializer as usual. Its slots are
reserved for exactly `T`, including that type's pointer permissions. Casting
an address does not permit using the same slot for an unrelated type.

`storage_ptr(pointer)` takes exactly one pointer to a live `raw_storage<T, N>`
object and returns the address of its first element slot. A read-only storage
pointer produces `T*`; a writable storage pointer produces `mod T*`. The
operand is evaluated once. It must be non-null, aligned, and valid; the
operation neither reads the payload nor constructs an element. It works even
when all element slots are inactive. It has no type operand and cannot be
overloaded or used as a function value. Use `&storage` to address the storage
object itself, and `storage_ptr(&storage)` to address its element region.

```huc
let mod storage: raw_storage<File, 4> = raw_storage<File, 4>();
let slots: mod File* = storage_ptr(&storage);
construct_at(slots, File(handle)); // only slot 0 is now live
let file: File = *slots; // Advanced extraction; slot 0 is inactive again
// file has automatic cleanup; storage never closes a File by itself.
```

Pointer arithmetic within the region, including forming its one-past address,
is allowed independently of which slots are live. Reading a value, calling its
methods, or destroying it requires a live slot. `construct_at` instead requires
an inactive slot. Merely declaring a `T*`, casting bytes, or obtaining a slot
address never constructs a `T`.

Raw storage is always Advanced, even for Basic `T`. Ordinary whole-value
transfer relocates its representation and any live payload values to the
corresponding destination slots, leaving the source inactive without cleanup.
Inactive bytes may be transferred as representation without reading them as
initialized `T` values. No source clearing or element-state scan is required.
`copy storage` is rejected: a container's `clone` must create fresh storage and
explicitly copy its live elements. Pointers into relocated storage still point
to the old region and must not be used as observers of the new values.

Ending or replacing a raw-storage object's lifetime never invokes element
cleanup. Any live payload lifetimes end with it, potentially leaking resources.
The containing library type must destroy or extract its live elements before
releasing that storage. Its ordinary `drop` can do so; subsequent automatic
cleanup of the raw-storage field does not repeat element destruction. Storing
a length or tag is a library responsibility, not hidden compiler state.

One slot plus a presence flag is enough for a library inline optional. A sum
type can use separate raw-storage fields for its alternatives and a tag; this
uses more space than an overlapping union. Compact overlapping alternatives
and a general union syntax remain separate design work. These facilities do
not make ordinary `T` fields manually managed or allow direct Advanced field
moves.

#### 6.7.6 Allocated storage and container bookkeeping

Dynamic containers obtain suitably aligned, sufficiently large storage from
an allocator library. The allocator contract must permit constructing the
chosen `T` values there. `size_of<T>()` and `align_of<T>()` supply its layout
requirements; no allocation intrinsic is required. For a contiguous allocation,
the library establishes capacity for whole `T` slots before using element
pointer arithmetic. Zero capacity may use a null pointer; it permits no
dereference or pointer arithmetic on that null pointer.

A vector maintains `0 <= length <= capacity`, with exactly `[0, length)` live.
Push constructs at `data + length` before increasing length. Pop extracts an
Advanced last element or copies a Basic one, then removes that slot from the
live range. A copied Basic source still has a lifetime: use `destruct_at` to
end it before reusing the slot with `construct_at`. This calls no user code.
The same distinction applies when growing into a new allocation: Advanced
sources become inactive on transfer, while copied Basic sources must have
their lifetimes ended before the old region is reused. The old allocation
can then be freed without destroying already-transferred Advanced elements.

Clear destroys each live element exactly once and sets length to zero. The
library must not publish an unconstructed element or let cleanup reach an
extracted one, including on early returns. An ordinary inline-storage field
containing live `T` objects would receive automatic field cleanup; a
`raw_storage<T, N>` field deliberately does not. No hidden array of drop flags
is needed in either inline or allocated storage.

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
It can also extract the pointee. Such a function must document that it leaves
the slot inactive and requires the caller to manage source cleanup; the
pointer parameter does not carry the caller's automatic cleanup state.

### 6.10 Return values

Returning an Advanced local destructively relocates it into the caller's
result location and makes the source inactive. Direct constructor results
instead use guaranteed destination construction. Returning a Basic value
copies it. Returning `T*` never transfers or extends the pointee's lifetime.
The return value is bound before return-expression temporaries and active
locals are destroyed (section 6.5.1). Returning an observer into either does
not keep its source alive for the caller.

### 6.11 Value and pointer conversions

Values and pointers do not implicitly convert to each other. Use `&value`
to observe addressable inline storage and `*pointer` to access a live
pointee. Binding from `*pointer` copies a Basic value or relocates an Advanced
value. Advanced extraction requires `mod T*` and programmer-managed source
cleanup as described in section 6.4. Explicit `copy *pointer` may clone a
copyable pointee without making it inactive.

### 6.12 Checked use of relocated roots

Before an Advanced local or by-value parameter is used, its tracked state
must be Active. This is a required compile-time check in every compilation
mode, not an optional warning. It follows direct transfers and control flow;
it does not track lifetime changes through raw-pointer aliases.

#### Tracked state

The compiler tracks the state of every Advanced local and by-value parameter
at each point of a runtime function. For a constructor, this includes its
initializer entries, which run in field declaration order before the block.
`this` is a raw pointer and is not tracked; relocating `*this` is pointer
extraction (section 6.4). Based on the transfers it tracks, the analysis assigns
one of these states:

- **Active**: the value is live on every path considered;
- **Inactive**: the value has been transferred away on every path considered;
- **MaybeActive**: some paths leave the value live and others transfer it away.

Initialization, parameter binding, and assignment of a whole new value make
the place Active. Direct relocation from the place makes it Inactive. If the
destination of that relocation is indirect and may be the place's own
storage, the identity check of section 6.3 can skip the transfer at run
time, so the place is MaybeActive afterward. Where control-flow paths with
different states meet, the place is MaybeActive.

The analysis follows the evaluation order of section 5.1, including inside
expressions. Operands and arguments are processed left to right, and a
method receiver is processed before the arguments. Each by-value parameter
is bound before the next argument starts. An assignment evaluates its
destination first and stores after its right-hand side (section 6.3). In
this section, a *call* is a function, method, or intrinsic call, a constructor
expression `T(arguments)`, or a constructor initializer entry
`field(arguments)`.

#### Control flow

The checker follows the operations inside runtime conditions, including any
transfers, but does not predict whether a condition is true or false. Both
outcomes of every `if`, loop condition, `&&`, `||`, and `?:` are considered
possible, even for a constant condition. States are combined where paths meet.
An arm of `?:` that names an Advanced local or parameter transfers it only on
that arm's path. Short-circuit operands are likewise checked on the paths
that evaluate them.

Loops use the same rule, including paths back to an earlier point:

- The loop head combines entry state, after any `for` initializer, with the
  state returning from the body and from each `continue`. In a `for`, both
  returning paths run the step before reaching the head.
- The condition is checked at the head. Its resulting state feeds the body
  and the condition-false exit. A `for` without a condition has no
  condition-false exit, but can still be left with `break` or `return`.
- After the loop, the state combines the condition-false exit, if present,
  with every `break` that leaves that loop.

The analysis repeats around loop back edges until the states stop changing,
then checks uses against those states. A path ends at `return`, after its
operand is evaluated, or at a call whose result type is `never`, including
`panic`. Code unreachable by this analysis is not subject to this
relocation-state check; ordinary parsing, name resolution, and type checking
still apply.

#### Uses and reinitialization

A use of a tracked place whose state is Inactive or MaybeActive is
ill-formed, and the compiler must issue a diagnostic. A use is any operation
that reads or observes the value: relocating it, `copy`, calling a method on
it, selecting a field, or applying `&` to it or to a field path of it.
Storing into a field is checked twice. Selecting the field is a use when the
destination is evaluated, and the store is a use after the right-hand side
has been evaluated. So `x.count = consume_and_count(x)` is ill-formed.

The following are not uses and are permitted in any state:

- assigning a whole new value to the place. When the store happens, after
  the right-hand side has been evaluated, the old value is destroyed if it is
  still live, using a drop flag when the state is MaybeActive, and the place
  becomes Active. Uses on the right-hand side are still checked.
  `x = rebuild(x)` is valid when `x` starts Active: the right-hand side
  transfers `x`, and the store reactivates it without destroying an old value;
- compiler-inserted destruction at scope exit, which likewise destroys only a
  live value.

`ptr_as<usize>(&x)` still uses `x` through `&` and does not bypass this check.
A pointer saved before a transfer remains an address, but accessing an inactive
pointee through it is undefined behavior. It may be used to construct a new
value in suitable inactive storage; this does not reactivate the original
name or register automatic cleanup (section 6.7.3).

Fixed locals and parameters cannot be relocation sources (section 6.1), so
their tracked state stays Active after initialization. This does not prove
that unchecked pointer operations have preserved the actual value.

In the following examples, `Widget` and `Job` are Advanced types, for example
because they declare `drop`:

```huc
let mod widget: Widget = Widget();

if (condition) {
    receive(widget);
}

inspect(&widget); // error: widget is MaybeActive here
```

Reinitialize the value on the path that transferred it, or keep the use on
paths where the value is still live:

```huc
if (condition) {
    receive(widget);
    widget = Widget(); // Active again on this path
}

inspect(&widget); // valid: Active on every path
```

A transfer inside a loop is checked by the same rules. Here the next
iteration could try to submit a value that was already transferred:

```huc
let mod job: Job = Job(1);

while (more_work()) {
    submit(job); // error: job may have been transferred in an earlier iteration
}
```

Declaring `job` inside the loop body, or assigning a new `Job` after
`submit(job)`, makes each use Active. A transfer followed by `break` leaves
the value Inactive on that break path. After the loop it may be MaybeActive
if another exit, such as a zero-iteration path, leaves it Active. Transfers
in the condition also affect exit state; short-circuit paths may differ.

#### Checks within one call

Within one call, the checks above follow the argument order. `f(x, x.size())`
is ill-formed because `x` is relocated into the first parameter before
`x.size()` runs; `f(x.size(), x)` is valid. Passing the same Advanced local by
value twice is ill-formed for the same reason. Two additional checks reject
calls that could pass a pointer to a root that their arguments transfer:

- relocating the root of the call's receiver while the call's arguments are
  evaluated or bound, directly or inside a nested expression: `x.absorb(x)`,
  `x.absorb(wrap(x))`, and `x.part.absorb(x)`;
- relocating a local or parameter while a later argument is evaluated or
  bound, when an earlier argument of the same call is `&` applied to that
  value or to a field path of it: `f(&x, x)`, `f(&x.payload, x)`, and
  `f(&x, wrap(x))`. Parentheses and `ptr_as<...>(...)` around the earlier
  argument are looked through, and either arm of a `?:` counts, so
  `f(ptr_as<u8*>(&x), x)`, `f(ptr_as<usize>(&x), x)`, and
  `f(c ? &x : &y, x)` are ill-formed too.

The root of a receiver or assignment destination is found by removing
parentheses and field selections and by treating `(&r)->f` as `r.f` and
`*&r` as `r`. An assignment whose destination has root `r`, other than `r`
itself, is checked like a field store. So `(&x)->absorb(x)`,
`(&x)->count = consume_and_count(x)`, and `*(&x) = rebuild(x)` are
ill-formed. This recognition only widens what is rejected. It does not give
`*&r` the cleanup behavior of `r` (section 6.4). A receiver or destination
reached through any other pointer, a call result, or a `copy` result has no
tracked root for these additional checks. Thus `(copy x).absorb(x)` and
`x.next->absorb(x)` are not rejected by them. This does not prove that a raw
pointee remains valid; for example, `x.next` might point into `x` itself.

Only these direct forms are recognized. A pointer formed and used up inside
an argument is not rejected: `f(length_of(&x), x)` is valid. A pointer that
reaches the callee through another call or a constructor is not detected
either, as in `f(identity(&x), x)`. If the callee uses such a pointer, the
behavior is undefined when that use accesses the inactive source value.
Merely passing this check is not a lifetime guarantee for pointer arguments.

#### Staging and unchecked boundaries

The check applies to each runtime function after its body has been prepared,
including each residual generic `fn` specialization. `if1`, `for1`, and `while1` are
resolved first and are not branches. A generic `fn` body can therefore be valid for
one specialization and ill-formed for another. For example, a body that
passes the same argument by value twice is valid when `T` is Basic and
ill-formed when `T` is Advanced. The diagnostic names the specialization and
its generation and specialization chain.

The analysis needs no general alias tracking. It follows whole named locals
and parameters, plus the limited source forms listed above. It is conservative:
it can reject code whose unsafe path never runs, because it does not prove
runtime condition results or relationships between conditions.

Raw-pointer access is still unchecked. Accessing an inactive value through a
raw pointer, including one reconstructed from an integer address, is undefined
behavior (sections 8.1 and 14). Pointer extraction (section 6.4) does not update an
aliased local's tracked state, so even a later use of that local by name can
escape this check and have undefined behavior. The check is not a borrow
checker and does not make HUC memory-safe.

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

A function whose result type is `never` does not return. Its body must not
reach a `return` or its closing brace. `panic` has result type `never`.

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

Use `let1` for a compiler-only binding while preparing runtime code:

```huc
let1 element: compiler::Type =
    @compiler::find_type("app.Widget"); // future reflection API
let1 mod generated: usize = 0;
```

`let1` never creates runtime storage. `mod` before its name permits reassignment
during translation. Working locals inside an evaluated function can instead use
plain `let`; the call already supplies their compiler execution context (section
9.8). A generic `let` instead names a specialization family whose
selected body must residualize one ordinary runtime `let`. The
`compiler::Type` example illustrates the separately designed future reflection
library; the bootstrap compiler module initially exposes generated-code
values, the mode query, and explicit compile-time diagnostics.

### 7.3 Conditions and loops

`if`, `for`, and `while` are ordinary control flow in a runtime function.
Their compile-time forms are specified in section 9.

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

inspect(&widget); // error: widget may have been transferred
```

The source is inactive if the branch was taken, and active otherwise. After
the `if` it is MaybeActive, so using it there is a compile-time error
(section 6.12). Assigning a new value to it begins a new lifetime, after
which it may be used again. Cleanup still occurs exactly once, using static
control-flow knowledge or a drop flag where needed.

### 8.3 Data races

Concurrent conflicting access without the synchronization required by the HUC
memory model is undefined behavior. HUC does not infer thread-safety traits.
Atomic operations are provided by a standard library that maps to the target
platform's atomic memory model.

### 8.4 Optional diagnostics and sanitizers

Implementations may warn about likely dangling pointers, null dereferences, or
access to inactive storage through raw pointers. Debug modes may insert
checks. These facilities must not be presented as a language guarantee and
their absence must not change the meaning of a well-defined program.

The section 6.12 check on named locals and parameters is not one of these
optional facilities. It is required in every compilation mode.

## 9. Numbered phase semantics

### 9.1 Source levels and phase classes

Numbered keywords describe availability, not a request to run an arbitrary
number of compiler passes. HUC has two independently usable source levels:

```text
HUC1 source
    |
    | prepare code and evaluate compiler expressions
    v
printable flattened HUC0 source
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
| Generic families | `fn f<T: auto>`, `struct S<T: auto>`, `let v<T: auto>` | Parameterized declarations selected and prepared on request |
| Compiler storage and imports | `let1`, `import1` | Compiler bindings and dependencies, erased from HUC0 |
| Compiler-stage control | `if1`, `for1`, `while1` | Select or repeat code during preparation |

Genericity and compiler-only availability are separate. Angle headers defer
definitions; they do not make every instance compiler-only. Runtime demand
produces a concrete declaration. `@` evaluates an expression in the compiler
and uses its value; `#` inserts a generated `compiler::Code` value where the
`#` appears (section 11). A compiler call never inserts code into its caller.
Compiler-only types and operations cannot leak into runtime code. Code
preparation has two generation modes:
**program mode** prepares code for HUC0, and **evaluation mode** prepares a
private body for the compiler to execute. Both modes run during compilation;
section 11 defines their insertion and emission rules.

Every successfully prepared translation unit must be independently readable HUC0.
It contains no numbered declaration or control-flow keyword, `import1`, `@`,
`#`, unresolved family binder, specialization request, compiler-only value, or
compiler metadata handle. Both import forms are resolved away. The HUC1
translator writes one flattened file with namespace sections before the
HUC0-to-C17 translator begins.

Built-in casts such as `as<T>` and `ptr_as<Target>`, type queries, and
`raw_storage<T, N>` use angle operands without being phase-family requests.
No nominal type or ordinary function application with phase arguments survives
into HUC0.

### 9.2 Uniform generic families

Generic `struct`, `fn`, and `let` declarations use one primary/partial/full
specialization model. The primary's angle-bracket binder list follows the
family name. A generic `fn` then has a separate parenthesized ordinary-parameter
list; write `()` when it is empty. Those parameters are runtime inputs for a
runtime call and compiler values for an evaluated call. Bodies remain deferred
until requested, including zero-binder declarations such as `fn answer<>()`.

Primary families have binders and no angle pattern:

```huc
struct Cell<T: auto> {
    let mod element: T;
}

fn convert<T: auto>(mod value: T) -> T {
    return value;
}

let value<T: auto> {
    let value: T = T();
}
```

Partial specializations add an angle pattern:

```huc
struct Cell<T: auto><T*> {
    let mod element: T*;
}

fn convert<T: auto><T*>(value: T*) -> T* {
    return value;
}

let value<T: auto><T*> {
    let value: T* = null;
}
```

Declaration headers have no `if1` condition. Use `if1` inside the selected body
to choose code, members, or a resulting variable declaration. A body-level
condition does not participate in specialization selection.

Full specializations have an empty first list and a concrete second list:

```huc
struct Cell<><i32> {
    let mod element: i32;
}

fn convert<><i32>(value: i32) -> i32 {
    return value;
}

let value<><i32> {
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

After a declaration name, `<` starts a deferred header; `(` starts an ordinary
function's parameters. The first list always declares binders as `Name: Type`.
The optional second list always supplies the specialization pattern. Nonempty
binders with a pattern declare a partial specialization; empty binders with a
pattern declare a full specialization. An empty `<>` without a pattern declares
a zero-binder primary. A declaration such as `fn first<T>(...)` is invalid;
diagnose it with a hint for `<T: auto>` (binder) or `<><T>` (full specialization).
These choices require no type lookup. Parenthesize `<` and `>` comparisons in angle-list value expressions
and defaults, for example `fn choose<Big: bool = (8 > 4)>() -> bool { ... }`.
Nested generic type arguments still use ordinary angle brackets.

Requests use angle brackets uniformly:

```huc
let cell: Cell<u64> = Cell<u64>();
let result: u64 = convert<u64>(input);
inspect(value<u64>);
```

Angle tokens are lexed separately. Where `<` could instead begin a comparison,
the parser preserves an ambiguous angle postfix until name resolution
determines whether the left name denotes a generic family. Square brackets do
not request a compile-time specialization.

A function call, with or without `@`, may omit the entire angle list when all
binders are deduced or defaulted. Deduction unifies ordinary parameter types
with argument types. Explicit arguments and defaults supply the remaining
binders; a binder that cannot be determined is a diagnostic. `answer()` and
`answer<>()` select the same zero-binder primary. Calls never have two angle
lists: `convert<i32>(x)` requests the family, and selection finds its definition.

Zero-binder `struct S<>` and `let v<> { ... }` are also allowed. Type and variable
requests retain their angle list: `S<>`, construction `S<>()`, and `v<>`.
A generic structure or variable request supplies every non-defaulted binder.
Unlike an ordinary `fn` or `struct`, an unrequested zero-binder definition stays
opaque. `<>` is therefore useful even when there is no parameter to deduce.

### 9.3 Selection and residual declarations

Each generic structure, variable, or function name has one primary family in
a scope in 0.1; ordinary runtime `fn`
overloading remains a separate facility. In one scope, a function name is
either a non-generic overload set or one generic family, never both. Use a full
specialization such as `fn convert<><i32>(value: i32) -> i32 { ... }` for a
family's concrete case, not a competing `fn convert(value: i32)` overload.
The primary must precede its
specializations, and a specialization must be declared in the primary's
namespace in 0.1; this need not be a separately compiled source file.

In 0.1, generic `struct` families are module-scope declarations; generic `fn` families may
be declared at module scope or as type members; and generic `let` families may be
declared at module, function, or nested-block scope but never as fields. Local
generic `fn` or generic `struct` declarations and nested type families are deferred.

For a concrete request, selection proceeds as follows:

1. match phase argument count and the structural angle pattern;
2. bind pattern variables;
3. choose the unique structurally most-specific matching candidate.

An exact full pattern outranks a partial pattern. A partial pattern that accepts
a strict subset of another candidate's structural matches outranks that
candidate. The primary is the fallback. Declaration order never resolves a tie.
Equally specific or incomparable matching candidates are an ambiguity
diagnostic. Candidate filtering and ordering use no user-written conditions
and do not inspect candidate bodies.

An `if1` inside the chosen body can select an implementation or report an
invalid request with `compiler::error` (section 11.6). A false condition or an
error in that body never causes selection to retry another candidate. Shared
helpers can provide a general implementation where a selected specialization
needs its own fallback.

Only after selection does the translator parse and prepare the chosen body:

- a generic `struct` request produces a nominal concrete type; it is emitted
  only if runtime code needs it and all of its fields are runtime-compatible;
- a normal generic `fn` request produces one concrete runtime `fn` and a call to it;
- a generic `let` request produces exactly one concrete runtime `let` as specified in
  section 9.4.

Concrete specializations keep their family's namespace and lexical lookup,
but have no separate defining-module compilation boundary. The printer places
them at their family declaration positions, including methods within their
containing type and local variable specializations within their block.
This placement of concrete family definitions is distinct from module code
inserted with `#` in their bodies: that code follows the original runtime
request's top-level declaration (section 11), including for nested requests. The HUC0
printer assigns deterministic concrete names where multiple
specializations would otherwise collide and rewrites their requests to those
names. No unresolved family request may remain in HUC0.

For generic `fn`, completing a specialization for compile-time evaluation does not
by itself require emitting its runtime definition. Only specializations needed
by retained runtime code are written to HUC0; section 9.7 separates these two
steps. Ordinary `fn` declarations are not subject to this demand-driven rule.

Any generic family may be requested through an ordinary import from another
namespace. A runtime-needed concrete definition keeps the family's namespace;
the printed request uses its canonical qualified name, with no preserved import
or defining-module dependency barrier. Generated concrete names remain
deterministic implementation details.

This uniform model deliberately permits function-family partial
specialization. It does not use C++ substitution failure, and a failure while
parsing or checking the selected body is an ordinary diagnostic rather than a
reason to try a less-specific candidate.

### 9.4 Generic variable families

A non-generic `let` declares runtime storage. `let1` declares compiler-only
storage. A `let name<...>` family declares neither directly: its selected body
must produce one runtime variable.

A generic variable header has no result type because the chosen specialization
determines the complete runtime declaration:

```huc
let counter<Writable: bool> {
    if1 (Writable) {
        let mod counter: i32 = 0;
    } else {
        let counter: i32 = 0;
    }
}
```

Every selected path must leave:

1. exactly one ordinary, unnumbered `let`;
2. whose source-level identifier is the generic `let` family identifier;
3. and no other residual runtime declaration or statement.

Compiler-only setup, `let1`, and phase control are permitted in the selection
body because they disappear. A `#` insertion remains prohibited within a
generic `let` selection body in 0.1. Parsing generated text does not relax this
separate restriction on variable families.

Different specializations of one family may choose different value types,
initializers, inline values versus raw-pointer forms, pointee permissions,
and slot mutability. The family name is not itself a runtime variable; only a concrete
request such as `counter<true>` denotes one.

Generic `let` is permitted at module scope and at function or nested-block scope. It
is not permitted as an instance field. Every statically requested
specialization is emitted at the family declaration site. Consequently, a
function-local specialization initializes on every runtime execution that
reaches that site; preparation does not add a lazy runtime initialization flag.

### 9.5 Deferred bodies

Family bodies and phase-control bodies are deferred source
spans. Registration parses their headers, signatures, and patterns,
but does not lex or parse the associated braced body merely to discover its
end. Registering a deferred `fn` or `struct` is not a request to prepare or execute it.

A dedicated delimiter scanner locates the matching outer `}` while recognizing
only what is necessary to avoid false brace matches:

- nested braces;
- string and character literals;
- line comments;
- block comments.

The following bodies remain opaque until selected:

- every unchosen primary, partial, or full family body;
- every unrequested zero-binder body, including the methods and constructors
  inside an unrequested deferred type;
- the unselected branch of an `if1`;
- the body of a zero-iteration `for1` or `while1`.

No token, parse, name, type, ownership, or phase diagnostic may originate
inside such a skipped body. A skipped body is explicitly not required to be
valid HUC. The delimiter scanner may report only that it cannot locate the
outer closing delimiter. Once a body is selected, the translator lexes, parses,
resolves, and checks it normally in its inherited context.

Each executed phase-loop iteration prepares its selected body and performs
the iteration's compiler calls and insertions in order. No iteration is skipped
because an earlier iteration produced the same values or text.

On a runtime `fn` request, prepare the selected specialization in program mode:
run its compile-time operations and insertions once, where they appear, and
retain its ordinary statements. A compiler call prepares the selected body the
same way in evaluation mode, then executes it with the call's arguments
(section 9.7). Each compiler call gets a fresh evaluation-mode preparation;
section 12.2 distinguishes these calls from once-only program definitions.

An ordinary, non-generic `fn` is different: parse and prepare its body in program mode
when its declaration is encountered in the active source stream, even if it
is never called. A method is encountered when its containing type body is
processed. In a deferred type requested only for compiler use, methods and
constructors belong to that compiler preparation, not separate runtime roots.
A `fn` inside an unselected phase body is still skipped with that
body. Runtime forward-name checks may wait for declaration collection, but
the function's compiler-stage work is not postponed until a runtime call.

### 9.6 Compile-time control flow and insertion contexts

`if1`, `for1`, and `while1` execute while HUC1 prepares code. They are legal at all
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
with compiler-only `let1`, expression, and control statements. Such statements
must use only compiler-available values; they execute during preparation and
erase. Function and block bodies use the mixed HUC1 statement grammar.

```huc
let1 EnableTracing: bool = true;

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
parsed. A `for1` uses `for1 (initialization; condition; step)`: initialization
executes once, the condition is checked before each iteration, and the step
executes after each completed iteration. All three clauses use compiler-only
values. Like ordinary `for`, each clause is an optional expression; an omitted
condition means `true`. Declare compiler-only loop variables with `let1` before
the loop.

A `while1` reevaluates a compiler `bool`. Both phase loops prepare and insert
their body once per iteration and are bounded by the evaluator's resource
limits. A loop with zero iterations leaves its body opaque.

For example:

```huc
let1 mod index: usize = 0;
for1 (; index < 3; index += 1) {
    io::println(index);
}
```

The index and loop control disappear during preparation. The output contains
three calls with the arguments `0`, `1`, and `2`, not a runtime loop.

There are no `if2`, `for2`, or `while2` keywords. In a function executing in
the compiler, or an `@` or `#` operand, ordinary `if`, `for`, and
`while` already execute in the compiler environment.

### 9.7 Generic functions: residual and forced call modes

A generic `fn` has angle-bracket phase binders followed by parenthesized runtime
parameters:

```huc
fn choose<T: auto, Greater: bool = true>(mod left: T, mod right: T) -> T {
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

A call retained in runtime code requests a concrete runtime specialization:

```huc
let selected: i32 = choose<i32, true>(runtime_left, runtime_right);
```

The phase arguments and concrete runtime argument types determine the
specialization. In program mode, ordinary parameters are typed symbolic
expressions and remain runtime inputs. Ordinary `if`, `for`, and `while` that
depend on them remain in the generated `fn`. Compile-time control uses only compiler
values and is resolved while the function body is prepared.

Prefixing the call with `@` instead requires complete compile-time evaluation:

```huc
let folded: i32 = @choose<i32, false>(3, 5);
```

Every argument of a compiler call must be a compiler value. The call prepares
the selected body in evaluation mode, executes it in the evaluator with those
arguments, and returns the result. A compiler call never inserts code into its
caller; code reaches a body only through a `#` statement in that body
(section 11.2).

Preparation never sees ordinary argument values. Phase control, `@`
evaluations, and `#` insertions in a body run while that body is prepared,
once per occurrence, and may use only compiler-known values: phase binders,
`let1` storage, constants, and compiler queries. Ordinary parameters and `let`
locals exist only when the body executes: as symbolic runtime inputs after a
runtime request, and as concrete evaluator values during a compiler call.
Runtime requests and compiler calls follow the same preparation rules, with
their respective generation modes and currently visible compiler inputs.
Preparation does not depend on ordinary argument values.

Code that executes in the evaluator may use compiler-only values and call the
compiler library directly, without further `@` (section 9.9). Such code makes
the function compile-time-only: a runtime request rejects it and names the
operation that cannot survive.

The result of a compiler call may stay in compiler code; if it flows into
runtime code, it must be materializable as HUC0. Failure is a diagnostic, with
no fallback to runtime execution.

Preparing a body and emitting HUC0 are separate:

- A deferred `fn` used only in compiler calls has its body prepared and
  checked, but its runtime definition is not emitted merely for those calls.
  Local code inserted into it with `#` is part of its private body; inserted
  module code introduces no declarations in evaluation mode.
- A specialization referenced by retained runtime code is emitted at its
  family's declaration position, together with the specializations that runtime
  body needs. Module code inserted with `#` in its body instead follows the
  original requesting top-level declaration, in the body's own namespace.
- A later runtime request requires preparation in program mode and emission of
  that definition and its module output. A completed evaluation-mode call
  does not satisfy this requirement or mark module output as
  already emitted.
- A never-requested family body remains deferred. Ordinary unnumbered `if`
  branches in retained runtime code still count as runtime code; only removed
  phase branches and other erased compiler execution remove that demand.

Ordinary `fn` declarations encountered in the translation unit are prepared
immediately in program mode and pass through into HUC0, even if they are called
only at compile time or never called. Calling one from compiler code under
section 12.1 does not make its emission demand-driven. Later optimization may
remove unused runtime functions.

Code inserted with `#` belongs to the body that contains the `#`, in that
body's generation mode. Its module parts have a separate insertion anchor
(section 11).

### 9.8 Unified functions, types, and compiler storage

There is no separate `fn1` or `struct1` declaration. Use `fn` and `struct`,
with a deferred angle header for compiler helpers and compiler data types.
`<>` alone is enough; nonempty binders add parameterization. The same partial
and full specialization rules apply to compiler helpers, runtime functions, and
types.

Fields and working locals use plain `let`. Their storage belongs to runtime
code or the evaluator according to the use of the definition. `let1` still
declares explicit compiler storage, including module-level compiler state or
a variable needed while preparing retained runtime code. Function parameters
and fields do not require a numbered declaration.

```huc
struct BuildOptions<> {
    let tracing: bool;
    let inline_limit: usize;
}

fn add<>(left: i32, right: i32) -> i32 {
    let result: i32 = left + right;
    return result;
}

let1 answer: i32 = @add(20, 22);
```

`BuildOptions<>` is not inherently compiler-only: its scalar fields also permit
runtime storage. A type containing a compiler-only field, such as
`compiler::Text` or `compiler::Type`, cannot be stored at runtime. This applies
transitively through its fields. Compiler-only types may appear in a deferred
function's ordinary parameters and result for compiler calls; a runtime request
for such a signature is rejected. Merely registering it is not a runtime request.

A function body may contain `@` evaluations and `#` insertions; these run
while it is prepared and need not make its runtime definition invalid. Its
executed code may also use compiler-only values when the function runs in the
compiler. What must not survive a runtime request is a compiler-only value,
operation, or dependency in the retained signature, storage, or executable
body. An ordinary declaration without an angle header requests runtime
preparation immediately, so use `<>` when a definition is intended only for
compiler use. For example, `fn counter_decl() -> compiler::Code` without `<>`
is rejected, because its runtime signature would be compiler-only. Ordinary
methods and constructors inside a deferred type are processed in that type's
requested context.

Compiler-only values may be used by phase control, family arguments,
evaluation, reflection, and generation. None may leak into HUC0. A scalar-only
deferred function is allowed at runtime if its resulting definition is valid;
there is no replacement `consteval` marker. Compiler calls may reuse ordinary
functions under section 12.1; this does not make their runtime emission lazy.

### 9.9 The `@` evaluation-island operator

`@` goes before an expression and evaluates it at compile time. This is called
an evaluation island because the surrounding code may still be runtime code.
It does not mark a declaration and never forms part of a type.

The following illustrates the spelling when a compiler query API is available;
the query itself belongs to the future reflection library:

```huc
import1 compiler;

let1 function: compiler::Function =
    @compiler::find_function("render");
```

The declared type above is `compiler::Function`, not `@Function` and not
`@compiler::Function`.

The whole operand of `@` is compiler code: calls, operators, constructors, and
method calls inside it need no further `@`. The same holds in other places that
only hold compiler code: `if1`, `for1`, and `while1` clauses,
`let1` initializers, and the operand of `#`. A redundant `@` there
has no effect.

Code that executes in a compiler call is compiler code too, so a function used
that way calls the compiler library directly. In any function body, however,
`@` keeps its preparation meaning: it marks a value computed once while the
body is prepared, which cannot read the body's parameters or `let` locals
(section 9.7).

An evaluation island must not read a runtime-only value. Its result may be used
by compiler code without materialization. If it flows into residual runtime
code, it must have an exact HUC0 representation, such as a scalar, boolean,
floating value, or string literal accepted at that location. Compiler handles,
`compiler::Code` values, and compiler-only structures cannot be materialized as
runtime values.

A standalone phase-expression item has this form:

```huc
@compiler_check();
```

It is legal at module, type-body, function, and nested-block scope. The
expression must completely evaluate in the compiler and have type `void`, which
suits compile-time checks. The item itself is erased. A standalone expression
of type `compiler::Code` is rejected with a hint to use `#`.

`@` never inserts code into the code around it. When an evaluated function
inserts local code into its own body with `#`, that code stays in its private
body (section 11.3). Code enters a body only through a `#` statement there.

Compiler-supplied intrinsics have their specified behavior in the active
preparation; they have no user body to prepare privately. In particular,
`compiler::emitting()` reads the active generation mode, including as an `if1`
condition.

`@this` is reserved for a future contextual-reflection proposal. Bootstrap
HUC1 diagnoses it rather than treating runtime `this` as compiler metadata.

A generic `fn` call retained in runtime code requests a residual specialization.
Applying `@` selects the same family and specialization rules but requires
compiler execution. A bare call reached while code executes in the compiler
also executes there; lack of `@` alone does not create a runtime-emission
request.

### 9.10 Imports and residual HUC0

HUC1 has two import forms:

```huc
import app.data;
import1 compiler;
```

`import` exposes runtime declarations and generic families and processes the
imported source in the same translation unit under section 2.1. `import1`
exposes compiler-only declarations; it does not create a runtime dependency.
The compiler-supplied `compiler` namespace is bound this way without reading
a user source file. Both import forms disappear from flattened HUC0; retained
declarations use namespace sections and canonical references instead.
These are the two import forms in 0.1. Import cycles are rejected with the
complete cycle path.

Residualization removes the phase mechanism itself:

- unnumbered HUC0 syntax passes through after any selected nested phase
  constructs have been processed;
- generic structure, function, and variable requests become concrete declarations
  and references;
- generic family declarations, phase control, compiler-only declarations, `import1`,
  evaluation-island calls, and `#` statements disappear;
- a forced materializable result becomes ordinary HUC0 syntax;
- generated code parsed and checked under section 11 is printed at its
  forward-only insertion points in program mode; private evaluation-mode bodies
  are not printed merely because they were evaluated.

The emitted translation-unit text, rather than an internal HUC1 AST, is the contract
between rounds. Running the HUC0 translator on it must require no compatibility
mode and must reject any remaining numbered construct.

### 9.11 Identity, recursion, and determinism

A concrete family specialization is identified by:

- the primary family's stable declaration identity and definition hash;
- serialized explicit and deduced phase arguments;
- for a normal generic `fn` call, its concrete runtime argument types;
- the target ABI and enabled language features;
- hashes of imported or reflected declarations on which selection or preparation
  depended.

Runtime argument values are not part of a normal function-specialization key.
They are concrete evaluator inputs only for a forced generic call using `@`. Equivalent
requests within one program produce one concrete specialization.

The translator processes one translation unit in deterministic source and
first-import order. A program-mode generic `fn` request reserves its insertion anchor
and prepares the selected body, discovering nested requests as needed. A
worklist may implement this, but must preserve that observable ordering.
Compiler calls do not reserve a program insertion anchor. A `#` at module
scope uses the boundary after its own item. The
output depends on the complete runtime-needed specialization closure, including
requests from importing sources. A caller-defined type is a valid phase
argument; the family need not import the caller. Substitution carries canonical
type identity, while unqualified names still use the family's lexical context.

The translation unit records each program definition, its checked body,
generation origins, and ordered module output. Each ordinary function definition
or concrete generic `fn` specialization is prepared and publishes its
program-mode output once. Repeated runtime uses reference that definition;
they do not replay generation. This is definition identity, not reuse of a
compiler call's result. Different specializations and distinct `#` insertions
within one preparation remain
distinct requests; only identical resulting module declarations are shared
(section 11.3).

Each `#` statement runs while its containing body is prepared and inserts the
code its operand produces at that point. A function that builds code runs on
every compiler call, even with identical arguments. A caller may explicitly
store its result in `let1` and use that value again; every `#` still performs
its own insertion. In evaluation mode, inserted module code publishes nothing,
introduces no names, and does not consume the later program-mode emission
opportunity. Runtime calls execute prepared bodies; they do not rerun their
compile-time operations. Generation mode, compiler-state values, and dependency
fingerprints do not add inputs to name mangling.

An identical runtime function specialization requested while it is already being
constructed denotes a recursive reference to the in-progress function. Compiler
calls instead execute with the normal recursion/resource limits: an in-progress
evaluation is not a computed result. An
infinitely recursive concrete type layout is a diagnostic. A failed program
definition is recorded as failed while reporting that translation's errors;
it is not a reusable evaluation result. Each compiler call resolves its private
body against the declarations currently published and checked. It cannot see
pending output. Declaration availability is not part of name mangling.

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
type and declaration identities needed for specialization, and the
generated-code values, `#` insertion, generation-mode query, and explicit
compile-time diagnostics in section 11. This deliberately keeps the first
HUC1-to-HUC0 milestone independent of an
unfinished public compiler object model.

The implementation supplies a compiler-only module:

```huc
import1 compiler;
```

Its types and functions are ordinary qualified HUC names. Phase information
comes from compiler API contracts, compiler-only field types, `import1`, `let1`,
and checked use contexts; `@` is never used as a type marker.

The module's declarations come from a versioned, inspectable generated
interface unit loaded before user HUC1 modules, analogous to an always-included
header. The explicit `import1 compiler` still binds its namespace and makes the
phase dependency visible. The bootstrap interface declares only
`compiler::Code`, its three constructors, `compiler::emitting()`, and
`compiler::error(message)`; the object-model design remains separate.

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
- queries record their inputs for reproducibility and dependency reporting;
- structured changes, validation, and commits are explicit operations;
- there are no ambient mutable arrays named `types`, `functions`, or
  `variables`;
- only concrete runtime declarations and already materialized family
  specializations are enumerable as declarations; an uninstantiated family
  does not imply an infinite set of types or functions.

`compiler::Code` already exists in the bootstrap as generated text (section
11.2). A future structured generation API may extend it with typed syntax or
semantic builders, hygiene, and pre-commit checking. Its design is a separate language-library
proposal and does not replace the text-generation rules below.

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
behavior during code preparation. Neither bootstrap milestone
implements `@this`, and no 0.1 program may depend on it.

## 11. Text generation and insertion

### 11.1 Program mode and evaluation mode

Preparing a function body means resolving its phase arguments, selecting phase
branches, running its `@` evaluations and `#` insertions, and checking the
resulting code. There are two **generation modes**:

| Mode | Purpose | Local code from `#` | Module code from `#` |
|---|---|---|---|
| Program mode | Prepare code for the HUC0 program | Insert into that code | Insert forward at module scope |
| Evaluation mode | Prepare a private body for the compiler to execute | Insert into that private body | Produce no module declarations |

Both modes run during compilation. They do not describe when runtime code
executes. Retained modules, types, ordinary `fn` definitions, and runtime-needed
generic `fn` specializations are prepared in program mode. A compiler call to
any user `fn` prepares a private body in evaluation mode and then executes it.
Inserted local statements are parsed and checked before subsequent statements
use their names. No runtime function body or module output is published merely
for that evaluation request. An ordinary `fn` may independently need both
preparations because its runtime definition remains in HUC0 under section 9.7.

A `#` always inserts into the body that contains it, in that body's mode. A
compiler call prepares its callee's body separately and never changes where the
caller's code goes. A generic type request prepares fields inside that type,
never in its requester's block; only its module insertion anchor is inherited.
One nominal type specialization must have a consistent field definition and
layout across its compiler and runtime uses.

#### Querying the mode

The `compiler` module provides one query, `compiler::emitting()`. It returns a
compiler `bool`: `true` while the current body is being prepared in program
mode, and `false` in evaluation mode, including in code executing in a compiler
call. The value is fixed for one preparation. Structural specialization
selection never depends on the mode (section 9.3).

Phase control resolves the query before either branch is parsed, so one
definition can take a different path in each mode:

```huc
fn square_decl<>() -> compiler::Code {
    return compiler::module("fn square(x: i32) -> i32 { return x * x; }");
}

fn area<Scale: i32>(side: i32) -> i32 {
    if1 (compiler::emitting()) {
        #square_decl();
        return Scale * square(side);
    } else {
        return Scale * side * side;
    }
}
```

The emitted `area<2>` calls the generated `square`. The evaluated `area<2>`
never refers to it. Without the query, `@area<2>(3)` cannot use `square` while
that declaration is still pending or if it has never been emitted. A later
evaluation may use an already published, checked `square` (section 11.2).
The two bodies may differ by design, and the compiler does not compare them;
keeping their results consistent is the programmer's responsibility.

### 11.2 Code values and `#` insertion

Generated code is a compiler-only value of type `compiler::Code`. The bootstrap
`compiler` module provides three constructors:

```huc
compiler::local(text)           // code that takes the place of the `#`
compiler::module(text)          // declarations inserted at module scope
compiler::around(before, after) // local code with one hole for the next block
```

`+` joins code values in order. Constructing or joining code inserts nothing; a
function that builds code returns it. A code value contains at most one hole.
Text arguments are compile-time text: string literals or `compiler::Text`
values.

A `#` statement inserts a code value:

```huc
#counter_decl();
```

`#` begins an insertion statement; it is not an expression operator. Its
operand is compiler code (section 9.9) and must have type `compiler::Code`.
The statement is legal at module, type-body, function, and nested-block scope.
It runs while the enclosing body or module is prepared, once per occurrence,
and is then erased. A `#` inside a runtime loop therefore inserts its code once,
into the loop body. Phase loops prepare their bodies once per iteration, so a
`#` inside a `for1` body inserts once per iteration.

A **generation site** is the position of a `#` statement. An **insertion
point** is the destination of each part of its code. An **origin** records the
`#` site and the compiler call chain that built the code, for diagnostics. A
**module insertion anchor** is the top-level declaration or module item whose
completion releases queued module code. Keep the anchor separate from the
code's namespace: a request in `app` can queue declarations named in `library`
after an `app` function.

Inserted code uses the namespace and lexical context of its `#` site. Local
code takes the place of the `#` statement and must be valid there:

- at module scope, complete module declarations;
- in a type body, complete members;
- in a function or nested block, statements and local declarations.

Local text is parsed before the following source is processed. If the code has
a hole, the `#` statement must be followed by a braced source block, which
fills the hole. The text before the hole, that block, and the text after the
hole must together parse as complete items at that position, so the block's
names, `break`, and `continue` have the proper context. A missing block is a
diagnostic. A hole stands where a statement could, never inside an expression;
no other unmatched braces or partial fragments are accepted.

For example, a builder can wrap a caller's block in a loop without seeing that
block, assuming a `Vector` with `size()` and `data()`:

```huc
fn loop_forward<>(items: compiler::Text, item: compiler::Text, index: compiler::Text)
        -> compiler::Code {
    let c: compiler::Text = "__items_" + item;
    return compiler::around(
        "{ let " + c + ": auto = " + items + "; "
            + "let mod " + index + ": usize = 0; "
            + "for (; " + index + " < " + c + "->size(); " + index + " += 1) { "
            + "let " + item + ": auto = " + c + "->data() + " + index + ";",
        "} }",
    );
}

fn draw_all(widgets: Vector<Widget>*) -> void {
    #loop_forward("widgets", "item", "index"); {
        if (item->hidden()) {
            continue;
        }
        item->draw(index);
    }
}
```

The generated block declares `item` and `index` in its own scope, so two such
loops in one block do not collide, and `continue` runs the generated step.
Names are not made unique automatically; the helper name here is derived from
the caller's binding name.

Module code is queued in program mode for the `#` site's namespace, after its
module insertion anchor:

- An ordinary `fn` in the active source is prepared when encountered. Its
  anchor is its enclosing top-level declaration, even if the function is never
  called. For a method in an ordinary type, this is the containing top-level type.
- A first runtime request for generic `fn` inherits the anchor of the source being
  prepared. If `main` requests `A`, which requests `B`, module code inserted in
  either body follows `main`, not the family declarations or a private `B`
  preparation. Preparing nested functions does not release this queue early.
- A request made while preparing a concrete type uses the top-level declaration
  whose preparation originally requested that type; registration alone does
  not execute a deferred body. Methods reached inside a requested generic `struct`
  keep that inherited anchor rather than opening an earlier type-site boundary.
- A `#` at module scope uses the boundary immediately after its own item.

At that boundary, retain the completed requesting declaration, then parse and
publish queued module declarations in insertion order, before continuing with
the next original source item. Namespace sections preserve each declaration's
naming context even when the request crosses an import. Already published code
is not relocated to later requesting sites. This is neither append-to-end-of-file
generation nor permission to insert before previously processed source.

In evaluation mode, the module parts of inserted code are suppressed: they
insert no text, register no names, and mark nothing as already emitted.
Suppressed text is not parsed as a declaration. An evaluated body cannot rely
on suppressed or pending code. This is a timing rule, not a permanent ban on
using its own generated declarations. Already published and checked
declarations may be used by later evaluations, including evaluations of the
function whose body inserted them, subject to the ordinary evaluator
restrictions. An evaluation inside `main` cannot use code queued after `main`,
directly or through another function. Unresolved references are diagnostics,
not a reason to fall back to program mode. Section 11.1 shows how
`compiler::emitting()` gives an evaluated body its own path.

Neither placement supports backward insertion, moving one arbitrary scope
outward, or modifying previously prepared source.

### 11.3 Code owned by functions

A function that builds code returns it, and the caller's `#` decides where it
goes:

```huc
fn counter_decl<>() -> compiler::Code {
    return compiler::local("let mod generated_counter: i32 = 0;");
}

fn main() -> i32 {
    #counter_decl();
    generated_counter += 1;
    return generated_counter;
}
```

The declaration enters `main` because the `#` is in `main`. Calling
`counter_decl` inserts nothing, and builders compose as values, for example
`return counter_decl() + limit_decl();`. An imported builder works the same
way; its text is parsed in the context of the `#` site.

A function can also insert code into its own body. The `#` then runs while that
body is prepared, whether the function is later executed at runtime or in a
compiler call:

```huc
fn answer<>() -> i32 {
    #compiler::local("let result: i32 = 42;");
    return result;
}

fn use_answer() -> i32 {
    let folded: i32 = @answer(); // Private result; no result local in use_answer.
    let live: i32 = answer();   // Runtime definition contains its own result.
    return folded + live;
}
```

`@answer()` evaluates privately and inserts nothing into `use_answer`. Local
code inserted into a body must not be skipped when that body is evaluated;
doing so could change the computed result.

To compute a value and generate code together, return both, keep them in
compiler storage, and insert the code separately. Here `helper_name` and
`helper_source` stand for ordinary functions that build text:

```huc
struct Helper<> {
    let name: compiler::Text;
    let code: compiler::Code;

    fn init(name: compiler::Text, code: compiler::Code) : name(name), code(code) {
    }
}

fn make_helper<T: auto>() -> Helper<> {
    let name: compiler::Text = helper_name<T>();
    return Helper<>(name, compiler::module(helper_source<T>(name)));
}

let1 helper: Helper<> = @make_helper<Packet>();
#helper.code;

fn helper_label() -> c8* {
    return helper.name; // printed as a string literal
}
```

`make_helper` runs once. `#helper.code;` inserts its declarations after that
item, and `helper_label` receives a copy of the name. Keep values that later
compile-time code needs in `let1`: compile-time code cannot read runtime
storage, even when a compile-time value initialized it. Code values never enter
runtime storage. Inserting the same code twice inserts it twice: identical
module declarations are shared as described below, and a duplicated local
declaration is an ordinary error.

A definition that is both emitted and evaluated has separate preparations, and
each runs its `@` evaluations and `#` insertions. As with macros, a builder that
reads or changes mutable `let1` state can therefore return different code each
time. A builder whose result depends only on its arguments and on
`compiler::emitting()` gives predictable results.

For each translation unit, publish the program-mode output of one ordinary
function definition or concrete generic `fn` specialization once. Repeated runtime
uses reuse that definition and its ordered module output. An evaluation-mode
preparation does not consume this opportunity; later program-mode demand must
still produce the required output.

This rule does not discard a second `#` in one preparation, later phase-loop
iterations, different specializations, or separate `#` statements. All keep
their normal order. Code inserted into a retained type or at a module item
likewise belongs to that item's preparation, not to a process-global "already
emitted" flag.

A generated module-scope declaration whose tokens are identical to those of a
generated declaration already inserted in the same namespace in this
translation unit is not inserted again when its lexical naming context also
matches. Both requests share the existing declaration, and its origins record
both. Whitespace and comments do not affect the comparison. Identical text at
module scope is compared within the same canonical namespace and resolved
lexical naming context, not across different import aliases. Sharing must not
change a lookup. Any other duplicate remains an ordinary duplicate-declaration
error, reported with both origins.

### 11.4 Parsing, diagnostics, and source locations

Generated text must be HUC0 syntax: no numbered constructs, `import1`, `@`,
`#`, compiler handles, or unresolved phase-family requests. HUC1 lexes and parses
inserted text, resolves its names and types with surrounding source, and applies
the same runtime permission and lifetime checks. Code around a hole and the
source block that fills it are checked together. Ordinary generated names
are therefore available to subsequent HUC1 processing, not only to HUC0.

Module-level forward references remain allowed. Collect the relevant generated
declarations before final body resolution; an unknown-name diagnostic must not
be issued merely because a later module declaration has not yet been collected.
A compiler evaluation needs an available, checked definition. It cannot advance
past its active module insertion anchor to publish output that is still pending
there, even if the needed declaration was generated by a different function.
If evaluation is required to finish that anchor, waiting for its queued output
is a dependency cycle and must be diagnosed. Suppressed evaluation-mode output
cannot satisfy the dependency either. A previously published declaration is
available regardless of which function generated it. Insertion may not
silently change a name lookup, specialization choice, or compiler decision that
has already been finalized; reject a conflicting insertion instead of rerunning
earlier generation with different results.

HUC0 independently parses and validates the final emitted program. HUC1 and
HUC0 should share syntax and semantic-checking rules, not trust an unchecked
private object format. The printable HUC0 boundary remains unchanged.

Original and generated text occupy separate immutable source buffers. A source
location identifies its buffer and offset; inserting output does not renumber
earlier original locations. Imported files also retain their own buffers;
record their import chains and the active namespace separately from physical
locations. A syntax node may span generated code and an
original block, so its source information must allow multiple spans. The final
HUC0 output map connects printed ranges to their source origins.

Diagnostics should retain:

1. the offending original or generated span;
2. the `#` site and module insertion anchor;
3. the compiler call chain that built the code, and the specialization chain;
4. the generated HUC0 location, when an output file exists.

Errors in an untouched user block point primarily to that block. Errors in
generated text show the generated span and its origin chain. No automatic name
isolation or fresh-name API is introduced: generated identifiers use ordinary
lookup and can still collide. Code builders must form valid identifiers and
escape literals correctly.

### 11.5 Facilities not added by text generation

HUC 0.1 does not define `quote`, structural syntax values, expression capture,
`compiler::fresh`, splicing, `emit`, `gen`, `gen_global`, `gen_up`,
`gen_module`, `gen_huc0_module`, or a public `parse_huc` operation. Parsing
inserted HUC0 is compiler behavior, not a new user-callable parsing API.
Phase control prepares selected ordinary HUC syntax; text becomes code only
through `compiler::Code` values inserted with `#`. The bootstrap constructors
take compile-time text; accepting ordinary string values there is a planned
relaxation. Typed container dispatch, operator overloading, and extended `for`
headers remain separate features.

### 11.6 Explicit compile-time errors

`compiler::error(message)` reports a user-written diagnostic and fails the
current translation when execution reaches it. It accepts exactly one
compile-time text argument, using the same text forms as the code constructors
in section 11.2. Its static result type is `void`, but it never returns
successfully. It is a compiler operation, not a runtime function or a code value.

Use it inside `if1` to reject an invalid generic request without a condition
on the declaration header:

```huc
import1 compiler;

fn bounded_count<Count: usize>() -> usize {
    if1 (Count > 64) {
        @compiler::error("bounded_count requires Count <= 64");
    }
    return Count;
}
```

This check runs when the selected body is prepared, in either generation mode.
An unselected phase branch or an unrequested deferred body does not execute
the operation. Code already executing in the compiler can call
`compiler::error(message)` directly, including with a message computed from
its ordinary parameters. An untaken ordinary branch in such execution does
not report the error. It does not provide a runtime failure operation.

Report the supplied message at the operation's source location, with the
request site and compiler call/specialization chain. Failing a selected
specialization does not try another candidate. No HUC0 program is successfully
produced for a translation that reaches this operation.

## 12. Compile-time execution model

The language specifies observable results, not a particular evaluator. A
conforming implementation may interpret typed HUC IR, compile it to a sandboxed
VM, or use another equivalent technique.

Compile-time values have HUC value semantics. Raw addresses into the compiler
process are not materializable and must not be used as stable declaration or
type identities.

Deferred `fn` definitions can compute values, including `compiler::Code`
values, under the call rules in sections 9.7 and 9.9. Code executing in a
compiler call may use the compiler's supported text, metadata, and
compiler-resident structures, with plain `let` working locals and the call's
concrete arguments, calling compiler APIs directly. Compiler-only fields,
parameters, and results are permitted for these calls but not for runtime
requests. Compiler execution never grants access to runtime memory, arbitrary
foreign code, or untracked external inputs.

Ordinary runtime functions can also be reused under the narrower initial
scalar subset below. Calling one from compiler code does not relax that subset.
An unsupported runtime operation cannot be hidden inside a deferred wrapper.
Explicit compiler APIs retain their separately specified capabilities; they do
not make every runtime operation evaluable.

### 12.1 Ordinary functions in compiler execution

The evaluator may call an ordinary `fn` whose HUC definition is available.
Arguments must be compiler-available values; it cannot read a runtime value
merely because that value's type is supported. Within code that executes in
the compiler, use normal call syntax. From otherwise runtime source,
`@` requests complete evaluation under the same rules. No new function marker
or alternative declaration is required.

The initial value subset is `bool`, `i8`, `i16`, `i32`, `i64`, `isize`, `u8`,
`u16`, `u32`, `u64`, and `usize`; `void` is also allowed as a result. Aliases
are checked after resolving their underlying types. Parameters and results
must use this subset, as must locals and intermediate values actually created
during evaluation. Pointers, `c8`, floating-point values, structures, strings,
and manually managed storage are outside this initial subset. Being Basic is
not sufficient: raw pointers and many structures are Basic too.

Within that subset:

- Local `let` variables and parameters may be changed where `mod` permits it.
  Ordinary conditions, loops, and recursive calls execute in the evaluator.
- The restrictions follow the evaluated call chain, including ordinary and
  concrete generic `fn` callees; an unsupported operation cannot be hidden in a helper.
  A call through a declaration without an available HUC body is rejected.
  Arbitrary foreign calls and execution of native host code are not allowed;
  compiler intrinsics have only their separately specified translation-time
  behavior.
- Immutable scalar constants with compiler-known values may be read without
  accessing runtime storage. Reading or writing runtime storage, or accessing
  mutable shared compiler state, is rejected. Local working variables remain
  mutable; this rule does not prohibit `let1 mod` elsewhere in compiler code.
- There is no untracked access to files, environment variables, clocks,
  randomness, processes, networks, or other external state. This is broader
  than a ban on direct system calls. Future tracked compiler APIs remain a
  separate facility, not permission to execute arbitrary runtime APIs.
- Arithmetic follows HUC's target widths and rules, not the host compiler's
  native arithmetic. Unsigned arithmetic wraps as usual. Executed signed
  overflow, division by zero, invalid shifts, and other undefined operations
  must produce compile-time diagnostics instead of host undefined behavior.

Eligibility is checked along the executed path. An unsupported operation in
an untaken ordinary branch does not prevent evaluation. That branch must still
be valid HUC and undergo normal parsing and type checking; it does not receive
the opaque-body exemption of an unselected `if1`. Reaching an unsupported
operation or exceeding an evaluator resource limit is a diagnostic with the
call stack. There is no fallback to a runtime call.

For example:

```huc
fn count_digits(mod value: u32) -> u32 {
    let mod count: u32 = 1;
    while (value >= 10) {
        value /= 10;
        count += 1;
    }
    return count;
}

fn field_width<>(value: u32) -> u32 {
    return count_digits(value);
}

let1 width: u32 = @field_width(12345); // 5
```

The evaluator uses the function's prepared, checked evaluation-mode body.
Calling it from compiler code never inserts code at the caller. Local code
inserted into the body with `#` is parsed and checked before that body
executes. The scalar restrictions apply to the prepared body, including
inserted code. Compile-time operations that run while it is prepared may still
use compiler text and other compiler-only facilities. Inserted module code is
suppressed in evaluation mode (section 11.2); a declaration that only it would
provide cannot be used during evaluation.

### 12.2 Execution without caching

HUC 0.1 does not cache compiler-call results or evaluation-mode preparations,
and does not analyze functions to decide whether they can be cached. Each
compiler call prepares a fresh private body, then executes it with that call's
arguments. Repeated calls with identical arguments still run, including their
preparation-time operations and permitted changes to compiler state. This
applies to ordinary functions, deferred helpers, and code builders. Compiler
intrinsics execute their specified operation directly in the active context.

Preparation still cannot read ordinary arguments (section 9.7). The current
compiler storage and published declarations supply the inputs visible to each
new preparation. Explicit `let1` storage can retain a computed value; that is a
programmer-written variable, not automatic result reuse. Instruction and
call-depth limits apply across preparation and execution of the complete nested
call chain, including recursive calls.

For example, these identical calls both execute:

```huc
let1 mod next: usize = 0;

fn next_id<>() -> usize {
    next += 1;
    return next;
}

let1 first: usize = @next_id();  // 1
let1 second: usize = @next_id(); // 2
let1 saved: usize = first;      // 1; reads stored data without calling next_id
```

The compiler still records declarations, canonical types, in-progress requests,
and completed program definitions. Those records establish identity, detect
recursion, and ensure once-only program preparation and publication. They do
not allow it to skip compiler calls or their effects. Each compilation starts
fresh; frontend, staging, and build-artifact caches are outside 0.1.

Evaluating an ordinary function does not suppress its HUC0 definition. A
generic `fn` specialization used only at compile time need not emit a runtime
definition; a retained runtime reference requires program-mode preparation and
its module output even if evaluation mode has already completed the same
specialization. Section 9.7 defines these emission rules independently of
compiler-call execution.

## 13. Foreign interfaces

HUC's first stable interoperation boundary is C:

```huc
extern "C" fn huc_write_bytes(data: u8*, size: usize) -> isize;
```

An `extern "C"` function may use:

- built-in arithmetic types with documented target mappings;
- `void` as a return type;
- raw pointers, including pointer chains, to built-in types.

It must not expose HUC Advanced values with `drop`, unresolved generic types,
compiler-only types, or compiler metadata. Resource management across C boundaries is expressed by
documented functions returning or accepting raw pointers or scalar handles.
Copying or passing a raw pointer does not itself establish or release any
cleanup obligation.

C-layout structure attributes, opaque foreign handle declarations, and C
variadics are deferred from 0.1. A small C shim can flatten such an interface
to the supported scalar and pointer boundary.

The bootstrap compiler emits ISO C17 with explicit HUC construction,
relocation, and cleanup. Generated C is an implementation technique, not a
stable HUC binary ABI. C++ interoperability may be added later through C ABI
adapters or a separate binding/backend design; arbitrary C++ headers are not
directly consumable in 0.1. The compiler itself is implemented in C++20.

## 14. Undefined behavior summary

The following list is representative rather than exhaustive:

- invalid raw-pointer dereference or arithmetic;
- access to inactive Advanced storage outside the section 6.12 check,
  including through raw pointers or untyped addresses, or by name after
  pointer extraction that did not update an aliased local's tracked state;
- destroying an inactive slot after pointer extraction, including through
  automatic local or enclosing-object cleanup that extraction did not cancel;
- violation of an external function's contract;
- out-of-range unchecked indexing;
- signed overflow, invalid shifts, and integer division by zero;
- data races;
- accessing a value through an incompatible pointer type;
- writing actually fixed storage through a cast pointer or integer-address round trip;
- raw slot manipulation that violates ownership or cleanup obligations;
- `construct_at` into a live slot, or `destruct_at` on an inactive slot;
- reading uninitialized storage;
- returning or storing an observer and later using it after its pointee dies.

Because undefined behavior is part of HUC's low-level contract, a compiler may
optimize on the assumption that it does not occur.

## 15. Deliberate omissions from 0.1

The following require separate proposals:

- checked references or lifetime analysis;
- primitive fixed arrays;
- representation reinterpretation with `bit_as`;
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

The closest C++ equivalents are conceptual, not emitted C17 declarations:

| HUC | Approximate C++ |
|---|---|
| `T*` | `const T*` for a non-pointer `T` |
| `mod T*` | `T*` for a non-pointer `T` |
| `&value` | `std::addressof(value)` or `&value` |
| `ptr_as<usize>(pointer)` | A raw data pointer converted to `std::uintptr_t` |
| `copy a` | clone/copy construction, explicitly selected |
| Advanced binding from `*p` | Move construction from `std::move(*p)`, except HUC ends the source lifetime and requires manual source cleanup management |
| `construct_at(p, value)` / `destruct_at(p)` | `std::construct_at` / `std::destroy_at`, with HUC initialization and cleanup rules |
| `raw_storage<T, N>` | Typed, aligned storage with manually managed element lifetimes; no implicit element cleanup |
| `size_of<T>()` / `align_of<T>()` | `sizeof(T)` / `alignof(T)` |
| generic `fn` | a subset spanning templates and `constexpr` |
| `fn helper<>()` used only through `@` | Deferred compiler use, without a `consteval`-only guarantee |
| `if1` | roughly `if constexpr` |
| `if1 (compiler::emitting())` | roughly `if !consteval` |
| generic `struct` | type generation/monomorphization |
| `#code;` with a `compiler::Code` value | macro expansion or a build-time code generator, but checked as HUC0 at the insertion site |

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

fn choose<T: auto, Greater: bool = true>(mod left: T, mod right: T) -> T {
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
