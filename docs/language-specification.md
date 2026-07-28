# HUC 0.1 Language Specification

Status: design draft

Revision: 0.1

Target audience: compiler implementers, library authors, and language reviewers

## 1. Purpose and scope

HUC is an unchecked, ahead-of-time systems programming language. It is intended
to offer C and C++ levels of control and runtime cost through a smaller and more
regular language model.

HUC has two defining mechanisms:

1. A numbered phase model unifies ordinary runtime code, generic
   specialization, compile-time evaluation, reflection, and code generation.
2. A small ownership model distinguishes unchecked observation (`T*`) from
   automatic unique ownership (`T*&`).

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
- `T*&` is pointer-sized when the default allocator is used.
- Moving a `T*&` transfers its address and clears the source to null.
- Destruction is deterministic and occurs at statically defined scope exits.
- Function arguments and subexpressions evaluate from left to right.
- A move is a compiler-defined transfer, never an overload-selected user call.
- Compilation removes every phase-1 and phase-2 construct before backend code
  generation.
- Generated code is represented structurally and is type-checked after
  generation.

HUC does not promise:

- prevention of dangling pointers, null access, invalid casts, buffer
  overflows, use-after-move, or data races;
- runtime performance better than equivalent optimized C or C++;
- source or binary compatibility with C++;
- that arbitrary compile-time programs terminate.

## 2. Program structure

A source file has the extension `.huc` and contains one module declaration,
zero or more imports, and declarations.

```huc
module app.main;

import std.io;
import app.widget;

export fn main() -> i32 {
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
- `import a.b;` binds the exported module under its last component, `b`.
- `import a.b as c;` binds the module to `c`.
- Unqualified wildcard imports do not exist in 0.1.
- Top-level declarations are module-private unless marked `export`.
- Import cycles are a diagnostic in 0.1.

An imported member is named with `::`:

```huc
import std.io as io;
io::println("hello");
```

An executable contains exactly one exported runtime entry point with one of
these signatures:

```huc
export fn main() -> i32;
export fn main(i32 argc, const c8** argv) -> i32;
```

### 2.2 Declaration order

Top-level names are collected before bodies are checked, so a declaration may
refer to a later declaration in the same module. Runtime global initialization
with non-constant expressions is not supported in 0.1. Compile-time constants
and zero-initialized runtime globals are permitted.

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

The following are indivisible keywords:

```text
fn fn1 fn2
struct struct1 struct2
if if1
for for1
while while1
let1 var1 where1
```

The suffix is part of the keyword: `fn1` is not `fn` followed by the integer
literal `1`. HUC 0.1 supports only the listed suffixes. A spelling such as
`fn3` in declaration position receives an “unsupported phase class”
diagnostic.

The numbers classify availability; they do not request an arbitrary number of
compiler passes. Section 9 defines their exact meaning.

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
const T
T*
const T*
T*&
```

`usize` and `isize` have the target pointer width. Integer widths are exact.

### 4.1 Plain values

A plain `T` is stored inline wherever its containing object is stored. HUC does
not force ordinary values onto the heap.

Every value type has one of two transfer modes:

- **Copy**: binding or assignment duplicates the value.
- **Move**: binding or assignment relocates the value and deactivates the
  source.

Built-in scalars, enums, function handles, and raw pointers are Copy. A
user-defined structure is Move unless declared `copy struct`.

```huc
copy struct Point {
    i32 x;
    i32 y;
}

struct Buffer {
    u8*& data;
    usize size;
}
```

A `copy struct` is valid only if every field is Copy and the type has no
`drop` method. The compiler derives the copy operation field by field.

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

`const T*` prohibits mutation of `T` through that pointer. It does not extend
the pointee lifetime and does not imply exclusive access.

### 4.3 Unique owning pointers: `T*&`

In HUC, `T*&` is a primitive unique-owner type. It does not mean C++'s
“lvalue reference to a pointer,” and HUC has no general `T&` type in 0.1.

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
`T*` or `const T*` only in an observing context. The reverse conversion is
never implicit.

### 4.4 Pointer layering

`*&` is parsed as an owner constructor. Examples:

```text
T*      RawPtr<T>
T*&     Owner<T>
T**     RawPtr<RawPtr<T>>
T**&    Owner<RawPtr<T>>
```

The characters `*&` must be adjacent. There is no postfix reference declarator
and therefore no C++ declarator ambiguity.

### 4.5 Nullability

Both `T*` and `T*&` may contain null. They convert explicitly to `bool` in a
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
    i32 first;
    i32 second;

    fn sum() -> i32 {
        return this->first + this->second;
    }
}
```

Members are public by default. `private:` and `public:` sections are supported.
Inheritance and virtual dispatch are not part of 0.1.

Methods receive an implicit non-owning `this`:

- `T* this` for an ordinary method;
- `const T* this` for a method declared `const`.

Calling a method on an owner observes the pointee and never moves the owner:

```huc
widget->update(); // no ownership binding, therefore no move
```

### 4.6.1 Lifecycle method

A Move structure may declare exactly one lifecycle method:

```huc
fn drop() -> void {
    // release non-field resources
}
```

`drop` must be a runtime, zero-parameter method returning `void`. It cannot be
called directly; the compiler invokes it during destruction. Its implicit
receiver is `T* this`. The body runs before fields are destroyed in reverse
order. HUC has no user-defined move constructor, copy constructor, assignment
operator, or C++-style destructor overload.

A `copy struct` cannot declare `drop`.

### 4.7 Arrays and slices

Fixed arrays use `[N]T`, where `N` is a compile-time `usize`. Indexing is
unchecked in 0.1.

Dynamic arrays are library types rather than a second primitive owner syntax.
The standard `Array[T]` owns its allocation; `Slice[T]` contains a raw pointer
and a length. Neither primitive indexing nor `Slice[T]` indexing is required
to check bounds in the unchecked HUC profile.

This avoids an ambiguous `new[]`/`delete[]` distinction for `T*&`: a `T*&`
always owns exactly one `T`.

### 4.8 Type aliases

```huc
type Size = usize;
```

Aliases do not create nominally distinct types.

## 5. Expressions and execution order

HUC uses C-family operators and precedence. The grammar appendix is normative
where this prose is silent.

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
from `T*&`; raw observers may exist.

### 5.4 Member access

`.` accesses a member of an inline value. `->` accesses a member through a raw
or owning pointer. A method call is not an ownership-binding context.

### 5.5 Casts

HUC 0.1 provides explicit casts:

```huc
as[T](value)           // checked by static conversion rules
bit_as[T](value)       // same size, representation reinterpretation
ptr_as[T*](pointer)    // raw pointer reinterpretation
```

`bit_as` and `ptr_as` are unchecked. Numeric narrowing is explicit.

## 6. Value transfer, copying, and destruction

### 6.1 Ownership-binding contexts

A move occurs only when a Move value is supplied to a destination that stores
that value:

- local or field initialization;
- assignment;
- argument binding to a by-value parameter;
- return-value binding;
- aggregate initialization.

Merely naming, testing, dereferencing, or observing an owner does not move it.

```huc
Widget*& owner = new Widget{};

owner->update();        // observe
if (owner) {}           // test
Widget* raw = owner;    // observe
inspect(owner);         // observe if parameter is Widget*

Widget*& next = owner;  // move
consume(next);          // move if parameter is Widget*&
```

### 6.2 Owner move initialization

For:

```huc
T*& destination = source;
```

the abstract operation is:

```text
destination.address = source.address
source.address = null
```

No user function is called by the transfer.

### 6.3 Owner move assignment

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

Self-move is therefore a defined no-op.

Assignment cannot be overloaded. A named method must express domain-specific
operations such as merge, append, swap, or replace.

### 6.4 Moving other values

Moving a non-owner Move value relocates its representation to the destination.
The source storage becomes **inactive** and is not destroyed. Assigning a new
value to that storage reactivates it.

Using inactive storage before reactivation is undefined behavior. A compiler
should diagnose an obvious straight-line use, but HUC does not promise
flow-sensitive use-after-move prevention.

The implementation must still arrange exactly-once destruction. It may use
static control-flow facts or hidden drop flags at joins where a value is only
conditionally active. Such flags are an implementation detail, not runtime
lifetime checking. Straight-line moves require no flag.

`T*&` uses its null representation instead of an additional drop flag whenever
possible.

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
  `fn clone() const -> T`.
- For `T*&`, null produces a null owner. A non-null owner allocates a new `T`
  and initializes it with `copy *source`.
- For a raw pointer, it copies the address and never clones the pointee.

```huc
struct Text {
    Array[u8] bytes;

    fn clone() const -> Text {
        return Text{copy this->bytes};
    }
}

Text a = make_text("hello");
Text b = copy a;

Widget*& first = new Widget{};
Widget*& second = copy first;
```

The explicit syntax exposes that logical copying may allocate or execute
arbitrary code. Move itself remains non-overloadable and non-failing.

### 6.7 Allocation

```huc
T*& aggregate = new T{field_initializers};
```

`new T{...}` allocates suitably aligned storage, aggregate-initializes one `T`,
and returns a unique owner. Every field without a default initializer must be
provided. Constructor overloading is not part of 0.1; named factory functions
express validated or fallible construction. Allocation failure terminates in
0.1. A later standard library may provide a recoverable `try_new`.

Advanced allocation policies use ordinary library owner types. Stateful custom
deleters are deliberately not stored in primitive `T*&`, preserving its
one-word representation.

### 6.8 Adoption, release, reset, and swap

Low-level ownership transitions are explicit:

```huc
T*& owner = adopt[T](raw); // raw must designate one compatible allocation
T* raw = release(owner);   // owner becomes null
reset(owner);              // destroy pointee and set null
swap(first, second);       // exchange addresses
```

Violating `adopt`'s allocation, type, or exclusivity preconditions is undefined
behavior. `release` transfers the cleanup obligation to the programmer.

### 6.9 Function parameters

The parameter type communicates transfer intent:

```huc
fn inspect(const Widget* widget) -> void; // observe
fn mutate(Widget* widget) -> void;        // observe and mutate
fn consume(Widget*& widget) -> void;      // consume
```

Supplying an owner to a raw-pointer parameter is an implicit observation.
Supplying it to an owner parameter moves it.

HUC 0.1 does not have an ownership-slot reference that lets a callee reseat the
caller's owner without consuming it. Consume-and-return is the normal form:

```huc
owner = transform(owner);
```

### 6.10 Return values

Returning a Move value moves it into the caller's result location. Returning a
local owner leaves the local null or permits the implementation to elide the
transfer entirely.

Returning `T*` never transfers or extends ownership.

### 6.11 Ownership and overload resolution

An overload set must not contain two candidates whose corresponding parameter
types differ only between `T*`/`const T*` and `T*&`. Such a set would make
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

Runtime locals use type-first declarations:

```huc
i32 count = 0;
auto result = compute();
const i32 limit = 100;
```

`auto` requires an initializer and deduces exactly one type. It does not retain
a reference or expression-category qualifier.

Compile-time locals use `let1` or `var1`:

```huc
let1 Type element = type_of(value);
var1 usize generated = 0;
```

`let1` cannot be reassigned. `var1` is mutable during translation. Neither
creates runtime storage.

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
3. owner-to-observer conversion.

No user-defined implicit conversion participates. A tie is a diagnostic.
Section 6.11's ownership-only restriction applies before ranking.

## 8. Lifetime and safety model

This section is normative because rejecting an accidental safety claim is part
of HUC's contract.

### 8.1 Raw observers are unchecked

```huc
Widget* saved;

fn remember(Widget* widget) -> void {
    saved = widget;
}

fn example() -> void {
    Widget*& owner = new Widget{};
    remember(owner);
} // owner destroys Widget

// Dereferencing saved later is undefined behavior.
```

The compiler does not infer or check a relationship between `saved` and
`owner`.

### 8.2 Conditional moves

```huc
Widget*& owner = new Widget{};

if (condition) {
    Widget*& local = owner;
    consume(local);
}

use(owner);
```

At the final call, `owner` is non-null if the branch was not taken and null if
it was. HUC does not need borrow checking or a compile-time “possibly moved”
rejection for this owner case. Dereferencing the null state is undefined
behavior.

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

### 9.1 The three phase classes

The suffix is a declaration's **phase class**, not a literal pass number:

| Class | Spelling | Meaning |
|---|---|---|
| Runtime | no suffix | Exists in the residual executable |
| Staged | suffix `1` | Runs during translation to evaluate or specialize runtime code |
| Meta-only | suffix `2` | Exists and runs only inside the translation environment |

The temporal order is: phase-2/meta and phase-1/staging work is completed,
producing a phase-0/runtime program; the runtime program then executes.

Class 1 is a bridge rather than a third storage universe. A class-1 declaration
may be fully evaluated at translation time or specialized into a concrete
runtime declaration. A class-2 declaration can never become a runtime symbol.

### 9.2 Functions

| Keyword | Translation-time call | Runtime symbol |
|---|---:|---:|
| `fn` | No | One ordinary function |
| `fn1` | Yes | Concrete specializations only |
| `fn2` | Yes | Never |

`fn` bodies contain ordinary runtime code.

`fn1` defines a staged function. A normal call from runtime code requests a
specialization. A call prefixed by `@` requests complete compile-time
evaluation.

`fn2` defines a meta-only function. It may manipulate compile-time values,
reflection metadata, and syntax, but it is never emitted.

Parameter-list interpretation is phase-specific:

- `fn` has one runtime parameter list.
- `fn1` with one list treats it as the runtime/symbolic list.
- `fn1` with two adjacent lists treats the first as compile-time and the second
  as runtime/symbolic.
- every `fn2` parameter is compile-time and `fn2` has one list.

Consequently, a staged function with compile-time arguments and no runtime
arguments is written with an explicit empty second list:

```huc
fn1 table(usize count)() -> Syntax[Decl] {
    // ...
}
```

### 9.3 Staged function parameters

A staged function may have a compile-time parameter list followed by a runtime
parameter list:

```huc
fn1 select(bool reverse = false)(auto left, auto right) -> auto {
    if1 (reverse) {
        if (left < right) {
            return right;
        }
        return left;
    } else {
        if (left < right) {
            return left;
        }
        return right;
    }
}
```

Compile-time arguments use square brackets:

```huc
i32 result = select[true](a, b);
```

Square brackets deliberately replace C++ template angle brackets. This keeps
compile-time arguments distinct from comparisons. In expression position,
`name[arguments](...)` is a compile-time-argument call; calling an element
selected by runtime indexing is not supported in 0.1. In type position, square
brackets always provide compile-time type arguments.

If a `fn1` has no explicit compile-time parameters, the first parameter list is
its runtime list:

```huc
fn1 max(auto left, auto right) -> auto { ... }
i32 value = max(a, b);
```

At a normal runtime call:

- explicit compile-time argument values are part of the specialization key;
- every `auto` runtime argument's concrete type is part of the key;
- runtime values are represented to the staging evaluator as typed symbolic
  expressions;
- the result is a call to one concrete residual function.

### 9.4 Forced compile-time evaluation

`@expression` requires complete translation-time evaluation:

```huc
let1 i32 a = 3;
let1 i32 b = 5;
i32 runtime_value = @max(a, b); // embeds the literal 5
```

Every dynamic input to a forced expression must be a compile-time value.
Failure to fully evaluate is a diagnostic.

Inside a `fn2` body or an already forced evaluation, calls to other `fn2`
functions do not require an additional `@`.

The result of `@expression` must be materializable in its use site. Integers,
booleans, floating values, strings, concrete types, and quoted syntax can be
materialized. Compiler handles such as `Field` cannot become runtime values.

### 9.5 Ordinary flow inside `fn1`

During runtime specialization, ordinary `if`, `for`, and `while` using symbolic
runtime data are residualized into the concrete runtime body.

During forced evaluation, ordinary flow executes in the compile-time
interpreter and therefore requires concrete values.

This rule lets one `fn1` serve both roles:

```huc
fn1 max(auto a, auto b) -> auto {
    if (a > b) {
        return a;
    }
    return b;
}

i32 x = max(runtime_a, runtime_b); // emits a concrete runtime function
i32 y = @max(3, 5);                // evaluates to 5 during translation
```

### 9.6 Phase-1 control flow

`if1`, `for1`, and `while1` execute during translation even while a `fn1` is
being specialized.

```huc
if1 (condition) {
    // selected and processed during translation
} else {
    // discarded during translation
}
```

- An `if1` condition must be a compile-time `bool`.
- Both branches must be syntactically valid.
- Only the selected branch is name-resolved and type-checked.
- The discarded branch contributes no names, code, or diagnostics beyond
  lexical and parsing errors.

`for1` iterates a finite compile-time iterable. Its runtime statements are
generated once per iteration. `while1` requires a compile-time condition and is
subject to the evaluator's resource limits.

The `1` forms are valid at module, structure, and function scope where their
generated declarations or statements would otherwise be valid.

There are no `if2`, `for2`, or `while2` keywords in 0.1. Inside `fn2`, ordinary
control flow already executes only in the meta environment.

### 9.7 Staged and meta-only types

`struct1` declares a type generator:

```huc
struct1 Stack(Type T, usize inline_capacity = 0) {
    Array[T] elements;

    if1 (inline_capacity > 0) {
        [inline_capacity]T inline_elements;
    }
}

Stack[i32, 16] values;
```

Each distinct compile-time argument tuple creates one nominal concrete runtime
type. Its fields and methods remaining after phase-1 processing are checked and
emitted.

`struct2` declares a compile-time-only data type. Instances may be stored in
`let1`/`var1` variables and passed to `fn1`/`fn2`, but cannot appear in runtime
storage or a runtime function ABI.

An ordinary `struct` is one runtime type. Its declaration may contain `fn1` and
`fn2` methods and `if1`-selected members, but its runtime layout must be fully
determined by the end of translation.

### 9.8 Constraints

Generic viability uses a declarative `where1` predicate:

```huc
fn1 print(auto value) -> void
where1 (!type_of(value).has_field("not_print")) {
    // ...
}
```

A `where1` predicate:

- executes at translation time;
- must produce `bool`;
- may inspect immutable metadata and its arguments;
- must not mutate ambient compile-time state or emit code.

A false predicate removes the declaration from the candidate set. A failed
predicate is not a body error. This replaces mutation-based mechanisms such as
setting `this.dontSelect`, which make selection order-dependent and difficult
to cache.

### 9.9 Variadic packs

Only `fn1` and `fn2` may declare open-ended generic packs in 0.1:

```huc
fn1 print_all(auto... arguments) -> void {
    for1 (auto argument : arguments) {
        gen {
            io::println($(argument));
        }
    }
}
```

During specialization, a pack is a compile-time `List[Expr]` whose elements
retain concrete types and symbol bindings. The generated runtime function has
one concrete parameter per element.

An ordinary `fn` has no type-generic variadic parameters. C variadics may be
declared only in an `extern "C"` declaration.

### 9.10 Specialization identity

A concrete specialization is identified by:

- the staged declaration's stable identity and definition hash;
- serialized explicit compile-time arguments;
- concrete runtime argument types;
- target ABI and enabled language features;
- hashes of reflected/imported declarations on which evaluation depended.

Runtime argument values are not part of the key unless the call is forced with
`@` or explicitly passed in the compile-time list.

Equivalent requests within one program produce one specialization. Recursive
specialization is permitted, but an identical specialization already being
constructed denotes a recursive reference rather than a new instance.

### 9.11 Compile-time determinism

The core compile-time environment is deterministic. It has no implicit access
to wall-clock time, random devices, process environment, network, or arbitrary
filesystem state.

A build API may expose operations such as `build.read_file(path)`. Such an
operation must register its input and content hash in the compilation
dependency graph. Untracked external effects are outside the 0.1 language.

The implementation may impose configurable instruction, recursion, memory, and
generated-code limits. Exceeding a limit is a compile-time diagnostic.

## 10. Reflection

Reflection is available only at translation time. It is read-only; code changes
are performed through generation rather than mutation of compiler symbol
tables.

Core metadata types include:

```text
Type
Field
Function
Parameter
Module
SourceLocation
Expr[T]
Syntax[T]
List[T]
```

Representative operations are:

```huc
Type type_of(value);
Type type_info[SomeType];
List[Field] Type.fields();
List[Function] Type.methods();
usize Type.size();
usize Type.align();
bool Type.is_copy();
bool Type.has_field(const c8* name);
```

Metadata lists have stable source-declaration order. Handles are immutable and
remain valid for the translation in which they were obtained.

There are no ambient mutable globals named `types`, `functions`, or
`variables`. A module may be queried explicitly:

```huc
for1 (Type item : module_info["app.model"].types()) {
    // ...
}
```

This makes dependencies and cache invalidation observable to the compiler.

## 11. Structural code generation

Code generation never concatenates source text.

### 11.1 Syntax values

`quote` constructs a typed syntax tree. `emit` inserts a syntax value into the
current residual function or declaration context.

```huc
let1 Syntax[Stmt] statement = quote {
    total += $(argument);
};
emit statement;
```

`gen` is shorthand for quoting and immediately emitting:

```huc
gen {
    total += $(argument);
}
```

`gen_global` emits declarations into the current generated module. It does not
permit an arbitrary mutation of another already-checked module.

### 11.2 Splicing

`$(compile_time_expression)` splices a syntax value, symbolic `Expr[T]`,
identifier handle, type, or materializable constant into quoted code. The
compiler verifies that the splice kind is valid at that position.

### 11.3 Hygiene

Generated syntax is hygienic:

- names referenced from the quote definition retain their resolved symbol
  identity;
- names introduced by a quote receive fresh identities;
- spliced identifiers preserve the identity carried by the splice;
- textual coincidence does not capture a local or module name.

An explicit unhygienic identifier-construction API may be added later. It is not
part of 0.1.

### 11.4 Validation

Generated syntax retains origin information for both the generator and the
quote. It passes through ordinary name resolution, type checking, phase
checking, ownership lowering, and backend lowering. A generator cannot bypass
language rules.

Implicitly guessing that an arbitrary statement in a `for1` is a `gen` is not
part of the normative language. Implementations may offer a warning-assisted
shorthand later, but 0.1 generation is explicit.

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
extern "C" fn fwrite(const u8* data, usize size, usize count, File* file)
    -> usize;
```

An exported/imported C function may use:

- fixed-width arithmetic types with documented mappings;
- `void`;
- raw pointers;
- explicitly C-layout structures;
- C variadics where the platform ABI supports them.

It must not expose `T*&`, HUC Move values with `drop`, phase-1/2 types, or
compiler metadata. Ownership across C boundaries is expressed by documented
functions returning or accepting raw pointers and explicit `adopt`/`release`
at the HUC side.

The bootstrap C++ backend is an implementation technique, not a promise that
arbitrary C++ headers or ABIs are directly consumable.

## 14. Undefined behavior summary

The following list is representative rather than exhaustive:

- invalid raw-pointer dereference or arithmetic;
- dereferencing a null or moved-from owner;
- use of inactive Move storage;
- double adoption or adoption of an incompatible allocation;
- violation of an external function's contract;
- out-of-range unchecked indexing;
- signed overflow, invalid shifts, and integer division by zero;
- data races;
- accessing a value through an incompatible pointer type;
- reading uninitialized storage;
- invalid representation produced by `bit_as`;
- returning or storing an observer and later using it after its pointee dies.

Because undefined behavior is part of HUC's low-level contract, a compiler may
optimize on the assumption that it does not occur.

## 15. Deliberate omissions from 0.1

The following require separate proposals:

- checked references or lifetime analysis;
- inheritance and implicit subtype ownership conversion;
- exceptions and stack unwinding;
- coroutines and async suspension;
- shared ownership as a primitive type;
- user-defined move constructors or assignment operators;
- arbitrary implicit conversions;
- user-defined operator overloading;
- dynamic runtime reflection;
- stateful deleters in `T*&`;
- a stable HUC-to-HUC binary ABI;
- compile-time network access or untracked host execution.

Shared ownership, when needed, should initially be ordinary library types such
as `Rc[T]`, `Arc[T]`, and `Weak[T]`. Moving those handles follows normal Move
semantics; cloning them explicitly exposes reference-count changes.

## 16. C++ comparison

The closest C++ equivalents are:

| HUC | Approximate C++ |
|---|---|
| `T*` | `T*` |
| `T*&` | `std::unique_ptr<T>` |
| `T*& b = a` | `auto b = std::move(a)` |
| `inspect(a)` where parameter is `T*` | `inspect(a.get())` |
| `consume(a)` where parameter is `T*&` | `consume(std::move(a))` |
| `copy a` | clone/copy construction, explicitly selected |
| `fn1` | a subset spanning templates and `constexpr` |
| `fn2` | roughly `consteval` plus meta facilities |
| `if1` | roughly `if constexpr` |
| `struct1` | type generation/monomorphization |
| `quote`/`emit` | structural macros/code generation |

This comparison is explanatory, not definitional. HUC does not inherit C++
value categories, overload-selected moves, reference collapsing, preprocessor
macros, template substitution rules, or unspecified operand order.

## 17. Conformance and evolution

The 0.1 compiler should expose:

```text
huc check <root.huc>
huc build <root.huc> -o <program>
huc emit-cpp <root.huc> --out-dir <directory>
huc fmt <files...>
```

Every implementation must report its language revision and target triple.
Feature experiments must be opt-in and must not silently change 0.1 semantics.

The language specification, not emitted C++ behavior, is authoritative. If the
backend language has a different evaluation order, destruction rule, or name
lookup rule, the transpiler must generate code that preserves HUC semantics.

## Appendix A: Consolidated example

```huc
module demo.main;

import std.io as io;

struct Widget {
    i32 value;

    fn clone() const -> Widget {
        return Widget{this->value};
    }
}

fn inspect(const Widget* widget) -> void {
    io::println(widget->value);
}

fn consume(Widget*& widget) -> void {
    inspect(widget);
}

fn1 choose(bool greater = true)(auto left, auto right) -> auto {
    if1 (greater) {
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

export fn main() -> i32 {
    Widget*& first = new Widget{7};
    inspect(first);

    Widget*& second = first;
    Widget*& independent = copy second;

    i32 a = 3;
    i32 b = 5;
    io::println(choose[true](a, b));
    io::println(@choose[false](3, 5));

    consume(second);
    inspect(independent);
    return 0;
}
```

At the end of `main`, `independent` destroys its cloned `Widget`. `second` was
moved into `consume` and destroyed there. `first` is null and its destruction
does nothing.
