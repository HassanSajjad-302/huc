# HUC

**Hassan's Update on C**

HUC is an experimental, unchecked systems programming language aimed at being a
smaller alternative to C++, not a memory-safe alternative to Rust.

Its two central ideas are:

- A set of numbered keywords for runtime code, specialization, compile-time
  execution, generics, reflection, and code generation: `fn`, `fn1`, `fn2`,
  `if1`, `for1`, `struct1`, and `struct2`.
- A small ownership model in which `T*` is an unchecked non-owning pointer and
  `T&` is a pointer-sized unique owner whose ownership transfers automatically
  and whose owned object is destroyed automatically.

HUC `T&` is an owning handle, not a C++ reference. HUC has no general reference
type and no `T&&`. `T*` and `T&` are the only pointer-like forms and cannot be
combined, so `T**`, `T*&`, and `T&*` are invalid. HUC also has no unary `&`.
Use the built-in `addressof(value)` operation to get a non-owning pointer to
an inline value. It cannot be overloaded. The unchecked `std::slot_of(place)`
intrinsic instead returns the address of the storage slot as a `usize` integer.
It also accepts raw-pointer and owner slots.

**Basic values are copied; Advanced values are transferred.** Numbers, raw
pointers, and structures containing only Basic fields with neither `clone()`
nor `drop()` are Basic. Owners and structures with an Advanced field,
`clone()`, or `drop()` are Advanced. Use `copy value` for an explicit duplicate;
Advanced structures support this by defining `clone()`.

An inline Advanced transfer is bitwise relocation, not C++ move construction.
It makes the entire source inactive: the source can no longer be used as a
value until reinitialized. The transfer does not run the source's `drop`,
clean up its fields, or require its owner fields to be set to null. A directly
transferred `T&` source instead remains usable as a null owner.

```huc
module readme.example;

import std.io as io;

struct Widget {
    let i32 value;

    fn init(i32 value) : value(value) {
    }
}

fn inspect(Widget* widget) -> void {
    io::println(widget->value);
}

fn consume(Widget& widget) -> void {
    inspect(widget);
} // destroys widget unless its ownership was relocated elsewhere

fn main() -> i32 {
    let Widget& mod first = new Widget(42);
    inspect(first);                       // observes; first still owns

    let Widget& mod second = first;      // relocates ownership; first is null
    let Widget& third = copy second;     // explicit pointee duplication
    consume(second);                      // relocates; second becomes null
    return 0;
}
```

HUC intentionally permits dangling raw pointers, null dereferences, unchecked
pointer arithmetic, use of inactive storage after relocation, data races, and
other forms of undefined behavior. Its goal is lower language complexity and
low runtime cost, not static memory safety.

## Why HUC

> Keep the metal. Lose the maze.

HUC is for programmers who want the C/C++ cost model but do not want several
overlapping languages hiding inside one compiler. Runtime code, specialization,
and compile-time execution use one set of numbered keywords. `T*` observes;
`T&` owns. Mutation is explicit. Transferring an Advanced value means one thing:
destructive relocation. Expensive duplication happens only when the source
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
- [Ownership example](examples/ownership.huc0)
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
representation (IR). The HUC compiler implements ownership; it does not rely on
the output language to provide it.

## Short positioning statement

> HUC combines C-class unchecked control with language-integrated unique
> ownership and a uniform numbered staging model. It aims for the cost of raw
> pointers and `std::unique_ptr`, without C++'s overlapping ownership,
> value-category, template, `constexpr`, and macro mechanisms.

This is a source-language simplification claim, not a claim that equivalent HUC
programs are inherently faster than optimized C++.

## License

HUC is available under the [MIT License](LICENSE).
