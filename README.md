# HUC

**Hassan's Update on C**

HUC is an experimental, unchecked systems programming language aimed at being a
smaller alternative to C++, not a memory-safe alternative to Rust.

Its two central ideas are:

- A set of numbered keywords for runtime code, specialization, compile-time
  execution, generics, reflection, and code generation: `fn`, `fn1`, `fn2`,
  `if1`, `for1`, `struct1`, and `struct2`.
- A small value model: Basic values copy, Advanced values transfer, and raw
  `T*` pointers observe without extending lifetimes.

HUC has no general reference type. `T*` is the only pointer-like form, and
pointer chains such as `T**` are invalid. Use unary `&value` to get a
non-owning pointer to an inline value. It cannot be overloaded. The unchecked
`slot_off(place)` intrinsic instead returns the address of a storage slot as
a `usize` integer and also accepts raw-pointer slots. HUC intrinsics use
unqualified names, not `std::` names; ordinary library APIs remain separate.

**Basic values are copied; Advanced values are transferred.** Numbers, raw
pointers, and structures containing only Basic fields with neither `clone()`
nor `drop()` are Basic. Structures with an Advanced field, `clone()`, or
`drop()` are Advanced. Use `copy value` for an explicit duplicate;
Advanced structures support this by defining `clone()`.

An Advanced transfer is bitwise relocation, not C++ move construction.
It makes the entire source inactive: the source can no longer be used as a
value until reinitialized. The transfer does not run the source's `drop` or
clean up its fields, and it does not require clearing the source bytes.
The destination becomes responsible for cleanup.

Declarations use `name: Type`, with `mod` before a writable name. Conditions
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
    let mod first: Widget = Widget(42);
    inspect(&first); // observes; first is unchanged

    let mod second: Widget = first; // transfers; first is inactive
    let third: Widget = copy second; // explicit logical duplication
    consume(second); // transfers; second becomes inactive
    inspect(&third);
    return 0;
}
```

HUC intentionally permits dangling raw pointers, null dereferences, unchecked
pointer arithmetic, use of inactive storage after relocation, data races, and
other forms of undefined behavior. Its goal is lower language complexity and
low runtime cost, not static memory safety.

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
`mod T*` and leaves the old slot inactive without clearing its bytes.
The programmer must prevent cleanup of that old slot; the compiler does not
cancel automatic cleanup through pointer aliases. For example, a container
can remove the extracted slot from its initialized range.

Use `copy *pointer` to clone an Advanced value that supports cloning while
leaving it active. The pointee must be live and accessible in either case.

### Permissions and cleanup

- Relocating a named Advanced value requires a writable source slot
  (`mod` before its name). The compiler manages source cleanup for whole
  locals, parameters, and fresh results. Extraction through `*pointer` uses
  programmer-managed source cleanup instead. Direct field and array-element
  moves remain rejected; HUC does not track partially inactive objects.
- Copying a raw pointer does not clear it. Pointee `mod` may be dropped but
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

## Why HUC

> Keep the metal. Lose the maze.

HUC is for programmers who want the C/C++ cost model but do not want several
overlapping languages hiding inside one compiler. Runtime code, specialization,
and compile-time execution use one set of numbered keywords. Values manage
resources through `drop`; `T*` observes. Mutation is explicit. Transferring
an Advanced value means one thing: destructive relocation. Expensive duplication happens only when the source
says `copy`.

The staging pipeline produces readable HUC0 that can be inspected,
tested, cached, and passed through the same public runtime translator as
handwritten code. The goal is metaprogramming power without making generated
runtime code a private compiler secret.

The planned reflection API starts with the code's immediate context.
Reserved future syntax such as `@this` may expose the narrowest enclosing
compiler object—a function inside a function, a class inside a class body, or
a module at module scope. The API would let code ask for broader context when
needed. This reflection model is not part of the first two compiler milestones.

## Documents

- [Controlling implementation plan](docs/implementation-plan.md)
- [Value semantics and special operations](docs/value-semantics.md)
- [Detailed use cases and examples](docs/use-cases-and-examples.md)
- [HUC 0.1 language specification](docs/language-specification.md)
- [Bootstrap transpiler architecture](docs/transpiler-architecture.md)
- [Bootstrap grammar](docs/huc.ebnf)
- [Default formatting guide](docs/formatting.md)
- [Resource-transfer example](examples/ownership.huc0)
- [Value-semantics example](examples/value-semantics.huc0)
- [Numbered-phase example](examples/phases.huc1)

## Status

HUC is at the design stage. There is not yet a compiler and the syntax is not
stable. The implementation plan controls milestone scope and sequencing; the
language, value-semantics, grammar, and architecture documents define the
corresponding current design in detail.

The implementation order is:

1. HUC0 to C17.
2. HUC1 to HUC0.

Both transpilers and the command-line driver will be written in C++20. The
first output language is C17. Generated C uses plain storage representations
and explicit construction, relocation, and cleanup from HUC's intermediate
representation (IR). The HUC compiler implements value lifetimes; it does not rely on
the output language to provide it.

## Short positioning statement

> HUC combines C-class unchecked control with destructive value transfer,
> deterministic cleanup, and a uniform numbered staging model. It aims for
> low runtime cost without C++'s overlapping value-category, template,
> `constexpr`, and macro mechanisms.

This is a source-language simplification claim, not a claim that equivalent HUC
programs are inherently faster than optimized C++.

## License

HUC is available under the [MIT License](LICENSE).
