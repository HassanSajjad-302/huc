# HUC

**Hassan's Update on C**

HUC is an experimental, unchecked systems programming language aimed at being a
smaller alternative to C++, not a memory-safe alternative to Rust.

Its two central ideas are:

- A set of numbered keywords for runtime code, specialization, compile-time
  execution, generics, reflection, and code generation: `fn`, `fn1`, `fn2`,
  `if1`, `for1`, `struct1`, and `struct2`.
- A small ownership model in which `T*` is an unchecked non-owning pointer and
  `T#` is a pointer-sized unique owner whose ownership transfers automatically
  and whose owned object is destroyed automatically.

HUC `T#` is an owning handle, not a C++ reference. HUC has no general reference
type. `T*` and `T#` are the only pointer-like forms and cannot be combined,
so `T**`, `T*#`, `T#*`, and `T##` are invalid. Use unary `&value` to get a
non-owning pointer to an inline value. It cannot be overloaded. The unchecked
`slot_off(place)` intrinsic instead returns the address of the storage slot as
a `usize` integer. It also accepts raw-pointer and owner slots.

Binding a compatible raw pointer to `T#` takes ownership directly:
`let owner: Widget# = raw;`. The raw pointer is left unchanged; only an
owner-to-owner transfer clears its source. The programmer must ensure the
allocation can be owned and is not already owned. HUC intrinsics use
unqualified names, not `std::` names; ordinary library APIs remain separate.

**Basic values are copied; Advanced values are transferred.** Numbers, raw
pointers, and structures containing only Basic fields with neither `clone()`
nor `drop()` are Basic. Owners and structures with an Advanced field,
`clone()`, or `drop()` are Advanced. Use `copy value` for an explicit duplicate;
Advanced structures support this by defining `clone()`.

An inline Advanced transfer is bitwise relocation, not C++ move construction.
It makes the entire source inactive: the source can no longer be used as a
value until reinitialized. The transfer does not run the source's `drop`,
clean up its fields, or require its owner fields to be set to null. A directly
transferred `T#` source instead remains usable as a null owner.

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
}

fn inspect(widget: Widget*) -> void {
    io::println(widget->value);
}

fn consume(widget: Widget#) -> void {
    inspect(widget);
} // destroys widget unless its ownership was relocated elsewhere

fn main() -> i32 {
    let mod first: Widget# = new Widget(42);
    inspect(first); // observes; first still owns

    let mod second: Widget# = first; // relocates ownership; first is null
    let third: Widget# = copy second; // explicit pointee duplication
    consume(second); // relocates; second becomes null
    return 0;
}
```

HUC intentionally permits dangling raw pointers, null dereferences, unchecked
pointer arithmetic, use of inactive storage after relocation, data races, and
other forms of undefined behavior. Its goal is lower language complexity and
low runtime cost, not static memory safety.

## Value, pointer, and owner assignment

The table uses **destination type = source type**. `T` means an inline,
non-pointer value; `T*` observes a `T`; `T#` owns an allocated `T`. These are
ordinary bindings, without an explicit `copy`. The same conversions apply to
initialization, assignment, arguments, and returns.

Assume the underlying `T` matches, mutation permissions are compatible, and
source and destination are distinct slots:

| Combination | Meaning | Source afterward |
|---|---|---|
| `T = T` | Basic `T`: copy the value. Advanced `T`: relocate the value from an eligible source. | Basic: unchanged and usable. Advanced: inactive until reinitialized; no source cleanup. |
| `T = T*` | Rejected: a pointer is not its pointed-to value. Dereference explicitly; see below. | No operation. |
| `T = T#` | Rejected: an owner is not its owned value. Dereference explicitly; see below. | No operation. |
| `T* = T` | Rejected: an inline value does not implicitly become a pointer. Use `&value` to observe addressable inline storage. | Taking its address leaves the value unchanged. |
| `T* = T*` | Copy the pointer address; no ownership is created or transferred. | Unchanged observer. |
| `T* = T#` | Observe the owned object by copying its address. | Owner unchanged; no ownership transfer. |
| `T# = T` | Rejected: an inline value does not implicitly become a heap allocation or an owner. Use `new T(arguments)` to construct an owned object. | No operation. |
| `T# = T*` | Take ownership of an existing compatible allocation. No allocation or pointee copy occurs. | Raw pointer unchanged; it remains an observer. |
| `T# = T#` | Transfer ownership; no allocation or pointee copy occurs. | Active, null owner. |

### Explicit access and copying

`&value` returns a raw pointer, not an owner, and does not extend the value's
lifetime. To read a pointee into an inline `T`, use `*pointer` or `*owner` when
`T` is Basic. For an Advanced `T`, ordinary move-out through dereferencing is
rejected; use `copy *pointer` or `copy *owner` if `T` supports cloning. The
pointee must be live and accessible. Explicit copying leaves it unchanged.

`copy owner` instead produces another `T#` with a separately allocated logical
copy of the pointee. The source owner must be non-null, and its pointee must
support copying.

### Permissions, ownership, and cleanup

- Relocating a named Advanced value or owner requires a writable source slot
  (`mod` before its name). Inline Advanced sources must be whole locals,
  parameters, or fresh results, not individual fields, elements, or pointees.
  Individual owner fields and elements can transfer because they become null.
- Copying or observing a raw pointer does not clear it. Binding it to an owner
  does not clear it either, so even a fixed raw-pointer source is allowed.
  Pointee `mod` may be dropped but never gained by these conversions.
- For `T# = T*`, a non-null pointer must designate a live complete object in
  a compatible allocation that no other owner owns. Taking ownership of a
  local, a subobject, or an already-owned allocation is undefined behavior,
  not a checked ownership conversion. A null raw pointer gives an empty owner.
- Replacing an existing value requires a writable destination. An Advanced
  inline destination is cleaned up before replacement; an owner destination
  destroys and deallocates its old pointee if non-null. Replacing a raw pointer
  never destroys its old pointee. Initialization has no old value to clean up.
- Exact self-relocation of an Advanced value or owner is a no-op. Assigning an
  owner's raw observer back to that owner is not self-relocation and violates
  the raw-to-owner ownership precondition.
- Raw observers do not keep objects alive. They become dangling when the
  object is destroyed and may be invalidated when inline storage relocates.

See [value semantics](docs/value-semantics.md) for the full lifetime rules and
[the ownership example](examples/ownership.huc0) for working through transfers.

## Why HUC

> Keep the metal. Lose the maze.

HUC is for programmers who want the C/C++ cost model but do not want several
overlapping languages hiding inside one compiler. Runtime code, specialization,
and compile-time execution use one set of numbered keywords. `T*` observes;
`T#` owns. Mutation is explicit. Transferring an Advanced value means one thing:
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
- [Default formatting guide](docs/formatting.md)
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
