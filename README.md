# HUC

**Hassan's Update on C**

HUC is an experimental, unchecked systems programming language aimed at being a
smaller alternative to C++, not a memory-safe alternative to Rust.

Its two central ideas are:

- A numbered phase vocabulary for runtime code, specialization, compile-time
  execution, generics, reflection, and code generation: `fn`, `fn1`, `fn2`,
  `if1`, `for1`, `struct1`, and `struct2`.
- A small ownership model in which `T*` is an unchecked non-owning pointer and
  `T&` is a pointer-sized unique owner that moves automatically and destroys
  its pointee automatically.

HUC `T&` is an owning handle, not a C++ reference. HUC has no general reference
type and no `T&&`.

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
} // destroys widget unless it was moved elsewhere

fn main() -> i32 {
    let Widget& mod first = new Widget(42);
    inspect(first);                       // observes; first still owns

    let Widget& mod second = first;      // moves; first becomes null
    let Widget& third = copy second;     // explicit pointee duplication
    consume(second);                      // consumes; second becomes null
    return 0;
}
```

HUC intentionally permits dangling raw pointers, null dereferences, unchecked
pointer arithmetic, use-after-move in unchecked paths, data races, and other
forms of undefined behavior. Its goal is lower language complexity and low
runtime cost, not static memory safety.

## Documents

- [Controlling implementation plan](docs/implementation-plan.md)
- [Value semantics and special operations](docs/value-semantics.md)
- [Detailed use cases and examples](docs/use-cases-and-examples.md)
- [HUC 0.1 language specification](docs/language-specification.md)
- [Bootstrap transpiler architecture](docs/transpiler-architecture.md)
- [Initial grammar](docs/huc.ebnf)
- [Ownership example](examples/ownership.huc)
- [Value-semantics example](examples/value-semantics.huc)
- [Numbered-phase example](examples/phases.huc)

## Status

HUC is at the design stage. There is not yet a compiler and the syntax is not
stable. The implementation plan records the latest decisions and takes
precedence for staging and implementation. The value-semantics document records
the current runtime lifecycle rules. Together they take precedence where the
older specification, grammar, or architecture still shows superseded syntax.

The implementation order is:

1. HUC0 to C++20.
2. HUC1 to HUC0.

Both translators and the command-line driver will be written in C++20.

## Short positioning statement

> HUC combines C-class unchecked control with language-integrated unique
> ownership and a uniform numbered staging model. It aims for the cost of raw
> pointers and `std::unique_ptr`, without C++'s overlapping ownership,
> value-category, template, `constexpr`, and macro mechanisms.

This is a source-language simplification claim, not a claim that equivalent HUC
programs are inherently faster than optimized C++.

## License

HUC is available under the [MIT License](LICENSE).
