# HUC

**Hassan's Update on C**

HUC is an experimental, unchecked systems programming language aimed at being a
smaller alternative to C++, not a memory-safe alternative to Rust.

Its two central ideas are:

- One staging model for generics, compile-time execution, reflection, and code
  generation: `<...>` marks deferred definitions and their parameters; suffix `1`
  marks compiler storage, imports, and control flow; `@` evaluates at compile
  time; `#` inserts generated code.
- A small value model: Basic values copy, Advanced values transfer, and raw
  `T*` pointers observe without extending lifetimes.

Compiler helpers use the same `fn` spelling, and
[ordinary functions run in the compiler](docs/language-specification.md#121-ordinary-functions-in-compiler-execution)
without a marker. Structures, methods, pointers, floating point, and
[compile-time allocation](docs/language-specification.md#123-compile-time-allocation)
all work; runtime memory, foreign calls, and untracked external effects are
excluded, and undefined behavior is a compile-time error. Compile-time data can
persist in `let1` storage across calls. Ordinary `fn` definitions still appear
in HUC0; a generic `fn` specialization used only at compile time is checked and
evaluated without requiring a runtime definition. Ordinary `fn` bodies are
prepared when encountered; angle-header `fn` and `struct` bodies stay deferred
until requested, even when their parameter list is empty (`fn helper<>()`).

Declarations use `fn`, `struct`, and `let`, with angle-bracket parameters for
families: `fn choose<T: auto>(left: T, right: T)`. A second list supplies a
specialization pattern: `<T: auto><T*>` is partial; `<><i32>` is complete.
Function calls may omit angles when arguments are deduced, defaulted, or absent:
`choose(a, b)` and `@helper()`.
Generic variables always use a block: `let value<T: auto> { let value: T = T(); }`.
The outer header has no attributes or binding `mut`; the same-name inner
declaration supplies the variable's attributes, mutability, type, and initializer.
`let1` and `import1` retain compiler-only bindings and imports. A module-level
`let1` is typed compiler storage. Each module has its own state, so changing
an imported `let1 mut` does not change the dependency or a sibling module.
Imports merge completed states in source order: the later value wins for the
same object, even if unchanged. Pointers, structures, and allocations follow
the same rule. Declaring the same name twice is still an error. `if1`, `for1`,
and `while1` decide which code is prepared. Fields and working locals use `let`.
Compiler-only types and operations are accepted for compiler use, but cannot
survive in runtime storage, signatures, or executable code.

Compiler calls use the caller's active state; retained runtime definitions and
type layouts use their defining module's state. See the
[module-state walkthrough](docs/compilation-model.md#87-one-walkthrough-values-objects-and-pointers).

Specializations are selected by structural patterns, without declaration-level
conditions. Use `if1` inside a selected body to choose code or report an invalid
request with [`compiler::error(message)`](docs/language-specification.md#116-explicit-compile-time-errors).
A failed body does not cause another specialization to be tried. Patterns cover
C++'s structural forms, including repeated binders such as `<T, T>` and family
binders such as `<C<T>>`, so traits like `is_same` are library code.

Each file is a module with its own namespace. `import` processes source once
as part of the same translation unit, preserving the imported code's lexical
scope. A file may name only the modules it imports, plus modules they
re-export with `pub import`. Declarations are private to their file unless
marked `pub`. A specialization lives in the file that declares its family or
one of its types. Generics can accept caller-defined types without reverse
imports. Staging writes one flattened HUC0 file, using `module name;` sections
to preserve namespaces. The [compilation model](docs/compilation-model.md)
explains these rules.

Generated code is a value. Compile-time functions return `compiler::Code`,
built with `compiler::local`, `compiler::module`, or `compiler::around`, and
`#code;` inserts it where the `#` appears. `@` only computes values; a call such
as `let x: i32 = @compute();` never inserts code into its caller. Inside an `@`
or `#` operand, and in code already running in the compiler, compiler calls
need no further `@`.

Built-in [attributes](docs/language-specification.md#49-attributes) use one
bracket syntax right after the keyword: `struct [align(64)] Slot`,
`struct [pack(1)] Header`, `let [bits(3)] mut mode: u32;`,
`fn [inline] add(...)`, and `if [unlikely] (err)`. A generic `fn` or `struct`
header's list is deferred with its body, so it can use the family's parameters.
Generic `let` puts its attributes only on the inner declaration. Outside a
generic variable's selection body, `#` can generate attributes. HUC has no
user-defined attributes.

Compiler calls use ordinary parameters: the argument decides whether each
receives a **value binding** or an **expression binding**. A `let1` or actual
compiler local supplies a value; a runtime `let` supplies an unevaluated
expression. `compiler::is_expr_param(name)` lets `if1` select the right branch
without reading the argument. `compiler::expr(name)` extracts an expression
binding's HUC0 text. Existing deduction and partial specialization can choose
traversal code for a vector, set, or custom container. Expression bindings do
not expose runtime objects; ordinary runtime calls keep their value semantics.

Extracted expressions remain text, not bound syntax handles. Names resolve
again where `#` inserts the text; generated names can shadow them. The resulting
code receives ordinary checks and lifetime rules, with no capture-specific
ownership or lifetime extension. See the
[typed loop builder](docs/use-cases-and-examples.md#74-a-generated-loop-around-an-original-block).

Text generation has two [generation modes](docs/language-specification.md#111-program-mode-and-evaluation-mode):
**program mode** prepares the HUC0 program; **evaluation mode** prepares a private
body for the compiler to execute. Inserted local code is parsed and checked
in both. Module code is inserted forward in program mode and suppressed in
evaluation mode. Repeated runtime uses publish a function or specialization's
generated declarations only once; compile-time evaluation does not consume
that opportunity. Nested specialization requests share the original requesting
top-level declaration as their module insertion anchor: a request inside `main`
publishes module output after `main`. Compiler evaluation cannot use that output
while it is pending, but later evaluations can use it once published.

[Every compile-time call runs again](docs/language-specification.md#122-repeated-compiler-calls),
even with identical arguments; only an explicit `let1` keeps a result. The
compiler still tracks definition identity and publishes each program definition
once; repeated runtime references do not repeat generation.

HUC has no general reference type. `T*`, `T**`, and longer pointer chains are
non-owning raw-pointer types. Use unary `&value` to get a non-owning pointer
to an inline value or storage slot, including a pointer slot. It cannot be
overloaded. When an integer address is needed, use the explicit cast
`ptr_as<usize>(&value)`. HUC intrinsics use unqualified names, not `std::`
names; ordinary library APIs remain separate.

**Basic values are copied; Advanced values are transferred.** Numbers, raw
pointers, and structures containing only Basic fields with neither `clone()`
nor `drop()` are Basic. Structures with an Advanced field, `clone()`, or
`drop()` are Advanced. Built-in `raw_storage<T, N>` is also Advanced, without
automatic element cleanup. Use `copy value` for an explicit duplicate;
Advanced structures support this by defining `clone()`.

An Advanced transfer is bitwise relocation, not C++ move construction.
It makes the entire source inactive: the source can no longer be used as a
value until reinitialized. The transfer does not run the source's `drop` or
clean up its fields, and it does not require clearing the source bytes.
The destination becomes responsible for cleanup.

Unconsumed temporaries live until their full expression finishes, usually at
the semicolon. A temporary used by `print(make_text().view())` stays alive
through the call; saving only its view does not extend its lifetime. See
[temporary lifetimes](docs/value-semantics.md#91-when-temporary-values-are-destroyed).

Declarations use `name: Type`, with `mut` before a writable name. Conditions
stay parenthesized, control-flow bodies require braces, and returning a value
uses explicit `return`. Digit separators and trailing commas are optional. A small
[formatting guide](docs/formatting.md) gives a default style without making
spacing or indentation part of the grammar.

```huc
module readme.example;

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
} // destroys the transferred value unless it was transferred onward

fn main() -> i32 {
    let mut first: Widget = Widget(42);
    inspect(&first); // observes; first is unchanged

    let mut second: Widget = first; // transfers; first is inactive
    let third: Widget = copy second; // explicit logical duplication
    consume(second); // transfers; second becomes inactive
    inspect(&third);
    return 0;
}
```

HUC intentionally permits dangling raw pointers, null dereferences, unchecked
pointer arithmetic, access to relocated storage through raw pointers, data
races, and other forms of undefined behavior. Its goal is lower language
complexity and low runtime cost, not static memory safety. HUC does require
one compile-time check: a directly tracked local or parameter cannot be used
after it may have been transferred, even if the transfer happens in only one
branch. Assigning it a new value makes it usable again. Pointer extraction
does not update that tracked state; its lifetime and cleanup obligations
remain the programmer's responsibility.

## Value and pointer assignment

The table uses **destination type = source type**. `T` means an inline,
non-pointer value; `T*` observes a `T`. These are ordinary bindings,
without an explicit `copy`. The same rules apply to initialization,
assignment, arguments, and returns.

Assume the underlying `T` matches, mutation permissions are compatible, and
source and destination are distinct slots:

| Combination | Meaning | Source afterward |
|---|---|---|
| `T = T` | Basic `T`: copy the value. Advanced `T`: relocate the value from an eligible source. | Basic: unchanged and usable. Advanced: inactive until reinitialized; no source cleanup. |
| `T = T*` | Rejected: a pointer is not its pointed-to value. Dereference explicitly; see below. | No operation. |
| `T* = T` | Rejected: an inline value does not implicitly become a pointer. Use `&value` to observe addressable inline storage. | Taking its address leaves the value unchanged. |
| `T* = T*` | Copy the pointer address; no cleanup obligation is created or transferred. | Unchanged observer. |

### Explicit access and copying

`&value` returns a raw pointer and does not extend the value's lifetime.
To bind a pointee into an inline `T`, use `*pointer`. This copies a Basic
value or transfers an Advanced value. An Advanced transfer requires a
`mut T*` and leaves the old slot inactive without clearing its bytes.
The programmer must prevent cleanup of that old slot; the compiler does not
cancel automatic cleanup through pointer aliases. For example, a container
can remove the extracted slot from its initialized range.

Use `copy *pointer` to clone an Advanced value that supports cloning while
leaving it active. The pointee must be live and accessible in either case.

### Permissions and cleanup

- Relocating a named Advanced value requires a writable source slot
  (`mut` before its name). The compiler manages source cleanup for whole
  locals, parameters, and fresh results. Extraction through `*pointer` uses
  programmer-managed source cleanup instead. Direct field and array-element
  moves remain rejected; HUC does not track partially inactive objects.
- Copying a raw pointer does not clear it. Pointee `mut` may be dropped but
  never gained by ordinary pointer conversion.
- Replacing an existing value requires a writable destination. An active
  Advanced destination is cleaned up before replacement. Replacing a raw
  pointer never destroys its old pointee. Initialization has no old value to
  clean up.
- Exact self-relocation of an Advanced value is a no-op.
- Raw observers do not keep objects alive. They become dangling when the
  object is destroyed and may be invalidated when inline storage relocates.

See [value semantics](docs/value-semantics.md) for the full lifetime rules and
[the resource-transfer example](examples/ownership.huc0) for working through
transfers.

### Manually managed storage

`construct_at(pointer, value)` initializes an inactive slot and returns its
pointer. Basic values copy, Advanced values transfer, and `T(arguments)`
constructs directly in that slot. `destruct_at(pointer)` destroys a live value
and its fields without freeing the slot's storage.

Both infer the type from `mut T*`. Neither changes automatic cleanup state
through aliases. The programmer supplies valid storage and tracks which
slots are live. See [manual storage lifetimes](docs/value-semantics.md#92-constructing-and-destroying-values-in-manual-storage).

`raw_storage<T, N>` reserves aligned inline slots without constructing `T`.
`storage_ptr(&storage)` obtains their typed address; `size_of<T>()` and
`align_of<T>()` provide target layout constants. A container tracks its live
elements and cleans them up in `drop`. `is_basic<T>()` lets generic code tell
a copied Basic source from an extracted Advanced source when retiring slots.
See [raw storage](docs/value-semantics.md#93-reserving-raw-inline-storage).

Conditional expressions produce values. With Advanced operands,
`(condition ? a : b).size()` transfers the selected value and destroys the
temporary after the call. Use `(condition ? &a : &b)->size()` to observe instead.

## Why HUC

> Keep the metal. Lose the maze.

HUC is for programmers who want the C/C++ cost model but do not want several
overlapping languages hiding inside one compiler. Runtime code, specialization,
and compile-time execution share one staging model. Values manage
resources through `drop`; `T*` observes. Mutation is explicit. Transferring
an Advanced value means one thing: destructive relocation. Expensive duplication happens only when the source
says `copy`.

The staging pipeline produces readable HUC0 that can be inspected,
tested and passed through the same public runtime translator as
handwritten code. The goal is metaprogramming power without making generated
runtime code a private compiler secret.

The planned reflection API starts with the code's immediate context.
Reserved future syntax such as `@this` may expose the narrowest enclosing
compiler object—a function inside a function, a class inside a class body, or
a module at module scope. The API would let code ask for broader context when
needed. Its specification can be developed before the compiler implements it;
it is not a prerequisite for the first implementation slice.

## Documents

- [Controlling implementation plan](docs/implementation-plan.md)
- [Value semantics and special operations](docs/value-semantics.md)
- [Detailed use cases and examples](docs/use-cases-and-examples.md)
- [HUC language specification](docs/language-specification.md)
- [Compilation model: modules, visibility, and packages](docs/compilation-model.md)
- [Compiler architecture](docs/compiler-architecture.md)
- [Language grammar](docs/huc.ebnf)
- [Default formatting guide](docs/formatting.md)
- [Resource-transfer example](examples/ownership.huc0)
- [Value-semantics example](examples/value-semantics.huc0)
- [Staging and generics example](examples/phases.huc1)
- [Expression arguments and text insertion](examples/expression-arguments.huc1)

## Status

HUC is at the design stage. There is not yet a compiler and the syntax is not
stable. The goal is to make the language specification as complete as possible,
then choose a practical implementation subset for release 0.1. A specified
feature is not a claim that 0.1 will implement it. The implementation plan will
track supported features and targets separately; unsupported features must
receive clear diagnostics, not silently different semantics.

Both translators and the command-line driver will be written in C++20:

```text
HUC1 -> printable HUC0 -> LLVM IR -> object files -> executable
```

LLVM IR is the first backend target, replacing the earlier C17-output plan.
HUC implements construction, relocation, cleanup, and evaluation order before
LLVM optimization and machine-code generation. The readable HUC0 boundary stays.

The recommended first milestone is a **thin end-to-end slice** through both
translators and LLVM, testing compiled execution and compile-time evaluation
early. **HUC1 to HUC0 first** remains an alternative, but evaluating ordinary
functions still needs the shared frontend, HIR, and MIR for the selected subset;
it postpones native code generation, not semantic analysis. The exact release
subset is still open. HUC0 remains independently usable for handwritten code.

## Short positioning statement

> HUC combines C-class unchecked control with destructive value transfer,
> deterministic cleanup, and a uniform numbered staging model. It aims for
> low runtime cost without C++'s overlapping value-category, template,
> `constexpr`, and macro mechanisms.

This is a source-language simplification claim, not a claim that equivalent HUC
programs are inherently faster than optimized C++.

## License

HUC is available under the [MIT License](LICENSE).
