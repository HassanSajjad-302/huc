# HUC

**Hassan's Update on C**

HUC is an experimental, unchecked systems programming language aimed at being a
smaller alternative to C++, not a memory-safe alternative to Rust.

Its two central ideas are:

- A numbered phase vocabulary for runtime code, specialization, compile-time
  execution, generics, reflection, and code generation: `fn`, `fn1`, `fn2`,
  `if1`, `for1`, `struct1`, and `struct2`.
- A small ownership model in which `T*` is an unchecked non-owning pointer and
  `T&` is a pointer-sized unique owner whose ownership transfers automatically
  and whose pointee is destroyed automatically.

HUC `T&` is an owning handle, not a C++ reference. HUC has no general reference
type and no `T&&`. `T*` and `T&` are the only pointer-like forms and never
compose, so `T**`, `T*&`, and `T&*` are invalid. HUC also has no unary `&`;
the non-overloadable `addressof(value)` intrinsic obtains a raw observer to an
inline value.

A type whose ordinary binding transfers rather than copies is classified
**Move**, but that transfer operation is destructive relocation rather than
C++ move construction. A relocated non-owner source becomes inactive without
running its `drop` or field cleanup; the special `T&` source remains usable as
a null owner.

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
and compiler execution use one visible numbered vocabulary. Ownership has two
spellings. Mutation is explicit. Moving a Move value means one thing:
destructive relocation. Expensive duplication happens only when the source
says `copy`.

The staging pipeline also leaves a receipt. HUC1 does not disappear directly
into backend machinery; it produces readable HUC0 that can be inspected,
tested, cached, and passed through the same public runtime translator as
handwritten code. The goal is metaprogramming power without making generated
runtime code a private compiler secret.

The longer-term reflection direction is contextual rather than omniscient.
Reserved future syntax such as `@this` may expose the narrowest enclosing
compiler object—a function inside a function, a class inside a class body, or
a module at module scope—with broader context reached deliberately through
that API. That reflection model is not part of the bootstrap milestones.

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

1. HUC0 to C++20.
2. HUC1 to HUC0.

Both transpilers and the command-line driver will be written in C17. C++20 is
the first generated backend language, not the compiler's implementation
language.

## Short positioning statement

> HUC combines C-class unchecked control with language-integrated unique
> ownership and a uniform numbered staging model. It aims for the cost of raw
> pointers and `std::unique_ptr`, without C++'s overlapping ownership,
> value-category, template, `constexpr`, and macro mechanisms.

This is a source-language simplification claim, not a claim that equivalent HUC
programs are inherently faster than optimized C++.

## License

HUC is available under the [MIT License](LICENSE).
