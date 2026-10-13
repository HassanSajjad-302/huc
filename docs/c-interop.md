# HUC C Interoperability Design

Status: proposed design, not yet part of the HUC 0.1 specification

This note describes how HUC code uses C libraries and how C code calls HUC.
It assumes the LLVM-first backend direction and a Clang library embedded in the
HUC compiler. Until it is adopted, the 0.1 rules in
[language specification section 13](language-specification.md#13-foreign-interfaces)
remain in force; section 10 lists the documents that adoption would change.
C++ interoperability is out of scope.

The guiding division is:

- **HUC implements HUC semantics.**
- **Clang implements C semantics.**
- **An explicit integration layer connects them.**

HUC never translates C function bodies, macros, or layouts into ordinary HUC
constructs. It asks Clang what the C code means and connects to the result.

Key terms:

- **C block**: an `extern "C"` block in HUC1 source whose body is C text.
- **C unit**: the readable C file produced from a module's C blocks, including
  any generated wrappers.
- **Foreign declaration**: a HUC-syntax declaration of a C entity, recorded in
  HUC0 and checked against Clang's view.
- **Integration layer**: the compiler component that turns foreign
  declarations and their uses into LLVM IR, relying on Clang for C types,
  layouts, calling conventions, and C code.
- **Direct call**: a call from HUC-generated IR to a C function, lowered using
  Clang's description of the C calling convention for that signature.
- **Wrapper**: a small generated C function, used where a direct call is not
  possible or not practical, such as for a function-like macro.
- **Dependency record**: everything that can change the meaning of a C unit.

## 1. Principles

1. HUC0 stays pure HUC and independently compilable. It contains foreign
   declarations in HUC syntax and a reference to its C unit, never C code.
2. No hidden state crosses from the HUC1 translator to the HUC0 translator. A
   HUC0 module, its C unit, and its dependency record are enough to compile it.
3. Direct calls are used where practical. Wrappers are generated only where
   they simplify or enable a particular foreign operation.
4. Only C-representable values cross the boundary by value: scalars, pointers,
   and C types. HUC ownership never crosses implicitly.
5. HUC's internal calling convention stays private and unfrozen. Only foreign
   calls and explicit exports follow the C calling convention.
6. Correctness never depends on inlining.

## 2. Source form

HUC has one form for foreign declarations: a block. In HUC1, its body is C
text:

```huc
module hello;

extern "C" as c {
    #include <stdio.h>
}

fn main() -> i32 {
    c::puts("Hello, world!"); // relies on the string-literal rule in section 11
    return 0;
}
```

The `as` name gives the block a namespace, just as `import ... as` does for
HUC modules. A single foreign function is simply a block with one declaration.
The spelling is provisional.

A C block may contain:

- `#include` directives and preprocessor definitions;
- C declarations;
- small C helper definitions, for C constructs that HUC cannot use directly
  (section 5.3).

All C blocks in a HUC module form one C translation unit, in source order.
Their `as` names only scope lookup on the HUC side; macros defined in an
earlier block are visible in later ones, as in a C file with several includes.

Clang parses that translation unit for the build's target, include paths, and
preprocessor definitions. Declarations that HUC code uses become foreign
declarations in the block's namespace. HUC1 marks them as foreign, so future
reflection can inspect them.

Declarations that HUC code never uses cost nothing. A C construct that HUC
cannot yet represent is diagnosed only when HUC code uses it, so including a
large header such as `<stdio.h>` never fails merely because some declaration in
it is unsupported.

C constants are compiler values in HUC1. Object-like macros that expand to
constant expressions, and enumeration constants, are evaluated by Clang and can
be used in phase control:

```huc
if1 (c::FEATURE_LEVEL >= 2) {
    fn use_new_path() -> bool {
        return true;
    }
}
```

## 3. Pipeline

```text
HUC1 module with C blocks
    |  HUC1 translator; embedded Clang parses the C text
    v
HUC0 module (foreign declarations) + C unit (.c) + dependency record
    |  HUC0 translator; embedded Clang compiles the C unit
    v
one shared LLVM module per HUC module -> optimization -> object file
    |
    v
linker, with the C libraries and object files named by the build
```

### 3.1 HUC0 form

The generated HUC0 names its C unit and records the foreign declarations that
its HUC code uses:

```huc
module hello;

extern "C" as c unit "hello.c-unit.c" {
    fn puts(text: c8*) -> i32;
}

fn main() -> i32 {
    c::puts("Hello, world!");
    return 0;
}
```

The C unit is ordinary, readable C:

```c
/* Generated from module hello. Do not edit. */
#include <stdio.h>
```

A HUC0 author can write the same form by hand, pointing at a hand-written C
unit. The HUC0 translator runs Clang on the unit with the recorded settings and
checks that each recorded foreign declaration matches Clang's declaration. A
mismatch is a compile-time diagnostic, never a silent calling-convention error.

### 3.2 One shared LLVM module

Clang generates code into the same LLVM module as the HUC0 translator. This is
a requirement, not an optimization: a header's `static inline` function has
internal linkage, so HUC code can call it directly only from the same module.
Swift emits imported C inline functions in the same way.

Clang supplies:

- declarations, types, and record layouts, including bit-field positions;
- the lowering of each foreign signature to LLVM parameters, results, and
  attributes;
- LLVM IR for `static inline` functions, wrappers, and helper definitions.

The combined module is optimized as one unit, so C helpers can be inlined into
HUC code and the reverse. That is an opportunity; correctness does not rely on
it.

With a C17 backend, the same interface is simpler to implement: the generated
C includes the C unit and uses C entities by name.

## 4. How C declarations appear in HUC

| C construct | HUC view | Lowering |
|---|---|---|
| External function | Foreign function | Direct call using Clang's lowering of the signature |
| `static inline` function | Foreign function | Clang emits the body into the shared module; direct call |
| Variadic function | Foreign variadic function | Direct call with C's default argument promotions (section 5.1) |
| Constant object-like macro, enumeration constant | Foreign constant | Evaluated by Clang; recorded as a constant in HUC0 |
| Expression-like macro | Foreign function | Generated wrapper where supported (section 5.2) |
| Structure | Foreign Basic type | Layout from Clang; fields accessible by name |
| Bit-field | Foreign field | Access generated from Clang's layout; `&` is rejected (section 5.4) |
| Union | Foreign Basic type | Members accessible at Clang's offsets |
| Incomplete type | Opaque foreign type | Usable only behind pointers |
| Global variable | Foreign place | Accessed through its address |
| Function-pointer type | Foreign function-pointer type | Can be stored, passed, compared, and called (section 6.3) |

Type mapping rules:

- C integer types map to HUC's exact-width types for the target. For example,
  `int` becomes `i32`; `long` becomes `i64` on LP64 targets and `i32` on
  Windows; `size_t` becomes `usize`; and `char` becomes `c8`.
- A pointer to `const T` becomes `T*`. A pointer to non-`const T` becomes
  `mut T*`.
- Accesses through `volatile`-qualified C types are emitted as volatile
  accesses. `_Atomic` types are unsupported at first; use a helper
  (section 5.3).
- C types are Basic in HUC. They have no `drop`, and copying one is a bitwise
  copy, exactly as in C.

HUC never recomputes a C layout. Sizes, alignments, field offsets, and
bit-field positions always come from Clang.

## 5. Direct calls and wrappers

### 5.1 Direct calls

Ordinary external functions, `static inline` functions, and variadic functions
are called directly. The integration layer asks Clang how the signature is
lowered for the target, including how structures are passed and returned, and
generates a matching call.

A variadic call has two additional rules:

- Only Basic scalars and pointers may be passed as variadic arguments. C's
  default argument promotions apply: `float` is passed as `double`, and
  integers narrower than `int` are passed as `int`.
- The call records how many of its arguments are fixed. Some targets pass
  variadic arguments differently from fixed ones; on Apple's arm64 targets,
  variadic arguments always go on the stack.

### 5.2 Generated wrappers

Wrappers are generated only for forms that cannot be called directly:

- **Expression-like macros with value parameters.** HUC evaluates each
  argument once, left to right, before calling the wrapper. This deliberately
  differs from textual macro expansion.
- **Macros that operate through pointers**, such as `FD_SET(fd, &set)`. These
  become `void` wrappers with pointer parameters.
- **Macros that name an assignable value**, such as `errno`. These become an
  address-returning wrapper when the value is addressable, or a getter and
  setter pair otherwise. HUC uses the result as an ordinary place.

```c
/* Generated wrappers in the C unit (illustrative names). */
static inline int* huc_c_errno_place(void) { return &errno; }
static inline void huc_c_FD_SET(int fd, fd_set* set) { FD_SET(fd, set); }
```

Wrappers are `static inline` and marked for inlining. Because they live in the
shared module, their internal linkage causes no problem. A wrapper that is not
inlined is still correct.

A macro is not wrapped automatically when a wrapper would change its meaning:
when it evaluates an argument only conditionally, stringifies tokens, takes a
type name, or declares something. Section 5.3 covers those.

### 5.3 Hand-written helpers

Any C construct that HUC cannot use directly can be reached through a small
helper written in the C block itself. Clang compiles the helper as real C, so
it always keeps C's meaning:

```huc
extern "C" as c {
    #include "config.h" // declares PICK, fast_timeout, and slow_timeout

    // HUC cannot pass unevaluated arguments; this helper keeps C's meaning.
    static inline int pick_timeout(int fast) {
        return PICK(fast, fast_timeout(), slow_timeout());
    }
}
```

HUC code then calls `c::pick_timeout(1)` like any other foreign function.

### 5.4 Bit-fields

Bit-fields have no addresses, so pointer-returning wrappers cannot expose them.
The integration layer instead generates loads, masks, shifts, and stores
directly from Clang's layout information: each field's storage unit, bit
offset, width, and signedness. Applying `&` to a bit-field is a compile-time
error in HUC as in C.

Two C rules carry over:

- Writing a bit-field reads and rewrites its whole storage unit.
- In C's memory model, adjacent bit-fields form a single memory location. Two
  threads writing neighboring bit-fields therefore race, even though the fields
  are distinct.

## 6. Ownership and values at the boundary

### 6.1 Values crossing into C

C has no destructors and no destructive moves. Only Basic values and pointers
cross the boundary by value. An Advanced HUC value can reach C only by pointer,
or through an export designed for it (section 6.2).

Resources returned by C arrive as raw pointers or handles. Wrap them in HUC
types to get automatic cleanup:

```huc
// Illustrative library family: its drop calls the given release function.
type File = Owned<c::FILE*, c::fclose>;
```

### 6.2 Exports

HUC functions are callable from C only when explicitly exported. An export has
a stable C symbol name and a C-representable signature. Advanced types cross as
opaque pointers, with exported functions for destroying and copying them where
needed.

Each export documents who owns each value. A C caller that passes a value to a
HUC function that consumes it must treat its own copy as gone, because C does
not mark it inactive. The export syntax is still to be decided.

### 6.3 Callbacks and function pointers

A HUC function can be passed by name to a C parameter of function-pointer
type. The integration layer generates an entry point with the C signature and
passes its address. The HUC function's signature must be C-representable, and
a function family must be named with its specialization.

Function-pointer values that come from C keep their C type. HUC code may store,
pass, compare, and call them; a call is lowered using Clang's description of
the signature. General HUC function types are a separate design.

HUC has no exceptions, so nothing unwinds across the boundary. A `panic` inside
a callback terminates the process.

## 7. Dependency record and determinism

A C unit's meaning depends on more than its own text. Its dependency record
contains:

- the C unit's content hash;
- the path and content hash of every header it includes, transitively;
- the preprocessor definitions and include paths;
- the target triple, calling-convention options, and C language standard;
- the identity and version of the embedded Clang and LLVM.

Rebuild these units from the recorded inputs on each compilation; HUC 0.1 has
no compiler or build-artifact caching. The toolchain version is pinned, because
Clang's C++ interfaces are not stable across releases.

## 8. Linking and toolchain

Headers describe interfaces; they do not supply implementations. The build
names the libraries and object files to link.

The same embedded Clang can compile ordinary `.c` files into IR alongside HUC
modules. A complete cross-compilation toolchain needs more than headers: target
C libraries, startup code, compiler runtime libraries, and a linker. That is
later work.

## 9. Validation plan

Each case is built, run, and its output checked, at both `-O0` and `-O2`, on
each target below.

| Case | What it checks |
|---|---|
| Structure passed and returned by value | Small (at most 16 bytes) and large structures, in both directions |
| `static inline` function from a header | Clang-emitted body and direct call in the shared module |
| Variadic call | Promotion of `float`, `char`, and `short`; the fixed-argument count |
| C callback into HUC | `qsort` with a HUC comparison function; calling a C function pointer from HUC |
| `errno` | Reading and writing through the place form |
| `FD_SET` | A `void` wrapper with a pointer parameter |
| Bit-fields and unions | C writes and HUC reads; HUC writes and C reads; updating one bit-field preserves its neighbors; signed bit-fields sign-extend; union members at the right offsets |
| Export | C calls a HUC function by its C name, with documented ownership |

Targets, chosen because their calling conventions differ where this design is
most likely to break:

- x86-64 Linux, the baseline, where structures of up to 16 bytes can be passed
  in registers;
- Windows x64, where structures whose size is not 1, 2, 4, or 8 bytes are
  passed by reference;
- Apple arm64, where variadic arguments are passed on the stack.

Passing these tests is strong evidence, not proof. Sanitizers should run where
the target supports them.

## 10. Adoption

Adopting this design would change:

- [language specification section 13](language-specification.md#13-foreign-interfaces),
  which currently allows only arithmetic types and pointers, and defers C
  structure layouts, opaque handles, and variadic functions;
- the `extern` function declarations in the [grammar](huc.ebnf), which would
  become the block form;
- the milestone order in the [implementation plan](implementation-plan.md),
  which currently builds the C17 backend first;
- the [compiler architecture](compiler-architecture.md), which would gain
  the embedded Clang and the shared LLVM module.

## 11. Open questions

- **String literals.** How a HUC string literal becomes a C string: its type,
  and whether a terminating zero byte is guaranteed.
- **Untyped pointers.** Whether HUC spells C's `void*` as `void*` or as an
  opaque type.
- **Syntax.** The spelling of C blocks, the HUC0 unit reference, exports, and
  bit-field declarations in HUC0.
- **Function types.** Whether HUC gains general function types beyond foreign
  function pointers.
- **Macro coverage.** Which macro forms are wrapped automatically in the first
  release.
- **Reflection.** How HUC1 exposes foreign declarations to compile-time code.
- **Visibility.** Under the adopted [compilation model](compilation-model.md),
  declarations are private to their file unless marked `pub`. Decide whether a
  C block's foreign declarations can be made `pub`, so that one wrapper module
  can share a C library with other modules, and reconcile this note's
  per-module LLVM wording with that model's one program-level module.
- **C++.** Interoperability with C++ is out of scope for this note.
