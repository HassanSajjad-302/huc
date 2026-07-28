# HUC Bootstrap Transpiler Architecture

Status: implementation design for HUC 0.1

## 1. Recommendation

The first HUC compiler should transpile whole programs to C++20 and then invoke
an existing C++ compiler.

This is a bootstrap choice, not a semantic dependency. HUC should have its own
parser, type checker, phase evaluator, specialization engine, ownership
lowering, and intermediate representation. C++ is used only as the first
portable machine-code backend.

The decisive reason is deterministic cleanup. A C++ backend can map HUC
`T&` to a one-word move-only RAII wrapper and can use generated C++ scopes to
handle normal exits. A C backend would require the first compiler to lower
every scope exit, early return, partial initialization, and conditional drop
into explicit cleanup branches before even the smallest useful HUC program
could run.

The middle end must nevertheless make moves and drops explicit. That keeps the
language independent of C++ and makes a later C backend or LLVM backend
straightforward.

## 2. C versus C++ as the first backend

| Concern | C17 backend | C++20 backend |
|---|---|---|
| `T&` cleanup | Compiler emits all cleanup paths | One-word generated RAII wrapper |
| Methods and namespaces | Lower immediately | Direct mapping |
| Move-only values | Manual structs/functions/drop flags | Generated special members |
| Generic specializations | Name-mangled functions and structs | Name-mangled concrete types; no templates required |
| Early bootstrap size | Larger | Smaller |
| Generated code simplicity | Mechanically simple but verbose | More compact |
| Risk of inheriting backend semantics | Moderate | Higher; must be fenced carefully |
| C ABI integration | Native | Easy through `extern "C"` |
| Future backend neutrality | Good if IR exists | Equally good if IR exists |

The C++ emitter must not translate HUC source by token substitution. In
particular, HUC `T&` must never be emitted as C++ `T&`; it is emitted as
`::huc_rt::owner<T>`.

## 3. Compiler overview

```text
.huc source
    |
    v
SourceManager -> Lexer -> Parser -> syntax AST
                                  |
                                  v
                         declaration collection
                                  |
                                  v
                    name + phase + type resolution
                                  |
                                  v
                         typed generic HIR
                                  |
                 +----------------+----------------+
                 |                                 |
                 v                                 v
        compile-time evaluator          specialization/residualizer
                 |                                 |
                 +--------------+------------------+
                                v
                       concrete typed HIR
                                |
                                v
                  ownership and drop elaboration
                                |
                                v
                         control-flow MIR
                                |
                                v
              C++20 emitter + source map + runtime header
                                |
                                v
                    clang++ / g++ -> object files
                                |
                                v
                              linker
```

The frontend never asks the host C++ compiler to implement HUC templates,
reflection, constant evaluation, overload selection, or phase semantics. All
such work is complete before C++ is emitted.

## 4. Proposed repository layout

```text
huc/
  README.md
  docs/
    value-semantics.md
    language-specification.md
    transpiler-architecture.md
    huc.ebnf
  examples/
  compiler/
    CMakeLists.txt
    include/huc/
      source/
      lex/
      parse/
      ast/
      sema/
      phase/
      hir/
      mir/
      codegen/
      diagnostics/
    src/
      source/
      lex/
      parse/
      ast/
      sema/
      phase/
      hir/
      mir/
      codegen/cpp/
      driver/
    runtime/
      huc_runtime.hpp
    tests/
      lexer/
      parser/
      sema/
      phase/
      mir/
      codegen/
      execute/
  std/
  tools/
```

The implementation language should initially be C++20. That permits easy
integration with LLVM later but does not require LLVM for the bootstrap.

## 5. Frontend

### 5.1 Source manager

`SourceManager` owns immutable source buffers and assigns each one a `FileId`.
Every token and syntax node carries a compact half-open `SourceSpan`:

```cpp
struct SourceSpan {
    FileId file;
    uint32_t begin;
    uint32_t end;
};
```

Line and column numbers are computed lazily from per-file newline indexes.
Generated nodes additionally carry an expansion chain:

```text
generated node
  -> quote site
  -> fn1/fn2 invocation site
  -> requesting source declaration
```

This is required for useful staging diagnostics.

### 5.2 Lexer

The lexer should:

- recognize numbered keywords as dedicated tokens;
- preserve comments and trivia when formatter support is enabled;
- diagnose unsupported phase spellings such as `fn3`;
- tokenize `&` as one token;
- retain the raw spelling of numeric and string literals;
- never depend on semantic type information.

Representative token kinds:

```cpp
enum class TokenKind {
    Identifier,
    IntegerLiteral,
    FloatLiteral,
    StringLiteral,

    KwFn, KwFn1, KwFn2,
    KwStruct, KwStruct1, KwStruct2,
    KwIf, KwIf1, KwFor, KwFor1, KwWhile, KwWhile1,
    KwLet1, KwLet2, KwWhere1,

    Star,
    Ampersand, // owner suffix in a type; address-of in an expression
    Arrow,
    DollarLParen,
    // ...
};
```

The parser turns `Ampersand` into an owner-type node only in postfix type
position. In prefix expression position, the same token is raw address-of. No
AST node represents a C++ reference.

### 5.3 Parser

A hand-written recursive-descent parser with Pratt expression parsing is the
best bootstrap choice:

- the grammar is small;
- recovery can be tailored to declaration and statement boundaries;
- numbered constructs map directly to AST variants;
- no generated-parser dependency is needed;
- square-bracket compile-time arguments avoid C++'s `<` ambiguity.

The parser builds syntax, not types. It should preserve invalid placeholder
nodes after an error so later diagnostics remain local.

Important recovery points are:

- `;`, `}`, and the next declaration keyword;
- the close of runtime and compile-time argument lists independently;
- `else` following `if` or `if1`;
- `)` following a parameter or condition.

### 5.4 AST

The syntax AST retains user spelling:

```cpp
enum class PhaseClass : uint8_t {
    Runtime,
    Staged,
    Meta,
};

struct FunctionDecl {
    DeclId id;
    PhaseClass phase;
    std::vector<ParamDecl> compile_time_params;
    std::vector<ParamDecl> runtime_params;
    TypeSyntax* return_type;
    Expr* where_predicate;
    BlockStmt* body;
    SourceSpan span;
};
```

Use stable arena-allocated IDs rather than C++ pointers as persistent
cross-pass identities. Pointers may be used temporarily within a compilation,
but caches and diagnostics should refer to IDs.

## 6. Semantic analysis

Semantic analysis is split deliberately:

1. collect declarations and module exports;
2. resolve names and phase availability;
3. resolve types and layouts that do not require specialization;
4. build typed generic HIR for `fn1`, `fn2`, and `struct1`;
5. instantiate/specialize on demand;
6. type-check concrete residual HIR.

Trying to type-check every generic body as if it were a concrete C++ template
would reproduce substitution complexity. A generic body is instead validated
for syntax and phase consistency, then type-checked with symbolic constraints;
operations depending on `auto` are finalized per specialization.

### 6.1 Symbol table

Every symbol records:

```cpp
struct Symbol {
    SymbolId id;
    InternedString name;
    SymbolKind kind;
    PhaseClass phase;
    ModuleId module;
    Visibility visibility;
    SourceSpan definition;
};
```

Scopes form an explicit tree. Generated syntax uses symbol IDs and hygiene
marks, not textual lookup alone.

### 6.2 Canonical types

Types are interned:

```cpp
enum class TypeKind {
    Error,
    Void,
    Never,
    Bool,
    Integer,
    Float,
    Struct,
    RawPointer,
    Owner,
    FixedArray,
    Function,
    MetaType,
    Symbolic,
};
```

`Owner<T>` and `RawPointer<T>` are distinct canonical nodes. Const qualification
belongs to the pointee/value type and must not be modeled through C++ declarator
strings.

Each concrete type records:

- size and alignment;
- transfer mode (Copy or Move);
- whether it requires drop;
- field layout;
- phase class;
- a stable structural fingerprint for caches.

### 6.3 Phase checking

Every expression receives:

```cpp
enum class Availability {
    RuntimeValue,
    CompileTimeValue,
    SymbolicRuntimeExpr,
    SyntaxValue,
};
```

This is separate from `PhaseClass`, which classifies declarations.

Examples of rejected edges:

- a runtime value used as an `if1` condition;
- a `struct2` stored in runtime memory;
- an ordinary `fn` called by the compile-time evaluator;
- a `Field` handle embedded into runtime code;
- a `fn2` referenced as a residual runtime symbol.

The diagnostic should name both sides:

```text
error: phase-1 condition requires a compile-time bool
  --> file.huc:18:10
   |
18 |     if1 (runtime_flag) {
   |          ^^^^^^^^^^^^ runtime value originates here
```

### 6.4 Overload resolution

The resolver implements only the ranking specified by the language:

1. exact match;
2. lossless built-in numeric promotion;
3. owner-to-observer conversion.

Before ranking, normalize each signature by replacing `T&` with its observing
form. Two otherwise identical normalized signatures form a forbidden
ownership-only overload set.

`where1` is evaluated only after arity and basic type matching. Predicate
failure silently removes a candidate; evaluator failure inside a selected
predicate is a diagnostic.

## 7. Typed HIR

HIR is expression-oriented, typed, and still structured. It represents both
generic and concrete code.

Every expression includes:

```cpp
struct HirExprHeader {
    HirExprId id;
    TypeId type;
    Availability availability;
    ValueUse use;
    SourceOrigin origin;
};
```

`ValueUse` distinguishes:

```cpp
enum class ValueUse {
    Read,
    Observe,
    Copy,
    Move,
    Place,
};
```

This distinction prevents the C++ emitter from rediscovering ownership intent
through emitted overload resolution.

Representative HIR operations:

```text
LoadCopy(place)
LoadMove(place)
OwnerObserve(place)
OwnerMove(place)
Clone(value)
Call(function, arguments)
Specialize(function, ct_arguments, runtime_arguments)
ForceEvaluate(expression)
Quote(syntax)
Emit(syntax)
Reflect(query)
```

## 8. Compile-time evaluator

### 8.1 Initial strategy

Interpret typed HIR in-process. Do not:

- generate and compile a temporary C++ program for every `fn2` call;
- execute arbitrary host C++ plugins;
- represent metadata as source strings;
- use host addresses as stable identities.

An interpreter is slower than native execution but is dramatically simpler to
make deterministic, inspectable, cacheable, and source-mapped. Performance can
be improved later with bytecode.

### 8.2 Values

```cpp
using CtValue = std::variant<
    CtBool,
    CtInteger,
    CtFloat,
    CtString,
    CtList,
    CtStruct,
    TypeId,
    FieldId,
    FunctionId,
    SymbolicExprId,
    SyntaxId
>;
```

The evaluator owns a compile-time heap with deterministic object IDs. Its
allocation order is not observable except through equality defined by HUC.

### 8.3 Symbolic execution for `fn1`

A `fn1` is not generally executed with concrete runtime values. Its runtime
parameters enter the evaluator as `SymbolicExprId` values carrying concrete
types.

Operations behave as follows:

- a pure operation on concrete compile-time inputs evaluates immediately;
- an ordinary runtime operation involving a symbolic input creates residual
  HIR;
- `if1` requires a concrete boolean and selects one branch;
- ordinary `if` with a symbolic condition creates residual branches;
- `for1` iterates a concrete compile-time sequence;
- `gen` appends structural syntax/HIR at the current insertion point.

This is a small partial evaluator, not C++ template substitution.

### 8.4 Forced mode

`@call` supplies concrete values and sets `EvaluationMode::Forced`. Creation of
an unresolved symbolic operation in forced mode is a diagnostic.

### 8.5 Resource control

The evaluator tracks:

- executed instruction count;
- call depth;
- allocated compile-time bytes;
- generated-node count;
- specialization depth.

Limits are configurable and included in diagnostics. The implementation should
print the most recent evaluation frames rather than a C++ stack trace.

### 8.6 Effects and dependencies

Compiler intrinsics return both a value and dependency records:

```cpp
struct EvalResult {
    CtValue value;
    std::vector<BuildDependency> dependencies;
};
```

For example, `build.read_file("schema.json")` records the canonical input path
and content digest. Network and clock access are absent from the core
interpreter.

## 9. Specialization engine

### 9.1 Keys

```cpp
struct SpecializationKey {
    DeclFingerprint declaration;
    std::vector<SerializedCtValue> compile_time_arguments;
    std::vector<TypeFingerprint> runtime_types;
    TargetFingerprint target;
    FeatureFingerprint features;
};
```

After evaluation, dependency hashes are attached to the cache entry. The key
must never contain pointer addresses or non-canonical map iteration order.

### 9.2 State machine

Each specialization cache entry is:

```text
Absent -> InProgress -> Complete
                    \-> Failed
```

Requesting the same `InProgress` function specialization from its own body
creates a recursive call edge. Requesting an `InProgress` type layout before
indirection makes its size finite is an infinite-size diagnostic.

Failed entries retain diagnostics for the duration of the compilation so the
same bad specialization is not repeatedly evaluated.

### 9.3 Naming

Generated backend names use a stable hash plus a readable prefix:

```text
_HU3app4math6select__A7F19C2E
```

The exact mangling is an implementation detail until a native HUC ABI is
specified. Human-readable comments map it back to the source declaration and
compile-time arguments.

## 10. Reflection and generation

Reflection reads immutable compiler tables through stable IDs. It never exposes
mutable `std::vector` objects or backend AST nodes to HUC code.

Quoted syntax has:

- a syntax kind;
- child syntax IDs;
- source origin;
- hygiene context;
- resolved symbol IDs for captured names;
- typed splice slots.

`emit` copies or moves syntax nodes into a generated AST arena, then ordinary
semantic passes run on the result. A generation failure should show:

1. the error in generated code;
2. the quote that created the node;
3. the `fn1`/`fn2` instantiation chain.

The compiler should support an inspection mode:

```text
huc expand app.main --function print_all
```

It prints residual HUC, not C++, after staging and before ownership lowering.
This is essential for making numbered phases understandable.

## 11. Concrete HIR and MIR

After staging, no `auto`, `fn1`, `fn2`, `if1`, `for1`, `struct1`, `struct2`,
reflection handle, quote, or emit operation may remain.

The concrete HIR is lowered to a control-flow MIR:

```cpp
struct BasicBlock {
    std::vector<MirStatement> statements;
    MirTerminator terminator;
};
```

Representative MIR statements:

```text
StorageLive place
StorageDead place
Copy destination, source
Clone destination, source
Move destination, source
OwnerMove destination, source
OwnerObserve destination_raw, source_owner
Activate place
Deactivate place
Drop place
OwnerDrop place
Adopt destination_owner, source_raw
Release destination_raw, source_owner
Call destination, callee, arguments
```

Representative terminators:

```text
Goto
Branch
Switch
Return
Unreachable
```

MIR semantics are sequenced. The C++ emitter must introduce temporaries when
necessary to preserve HUC's left-to-right order.

## 12. Ownership and drop elaboration

### 12.1 Place state

The pass tracks whether each Move place is:

```text
Inactive
Active
MaybeActive
```

This analysis exists to generate exactly-once destruction, not to claim memory
safety. A later read from `Inactive` or `MaybeActive` may receive a warning or
an error in a strict lint mode, but unchecked compilation may preserve it as
undefined behavior.

### 12.2 Drop flags

- No flag is needed when static control flow proves a place active or inactive.
- Primitive owners normally use their null address as the drop condition.
- Other `MaybeActive` values receive a hidden boolean only when necessary.
- Moves update the relevant state or flag.
- Reinitialization marks the place active.

A conditional flag belongs to one MIR storage place. It is emitted as a
sidecar local or guard state and never becomes a field of the nominal HUC type.
Consequently it does not change `sizeof(T)`, field offsets, FFI layout, or the
one-word representation of `T&`.

### 12.3 Cleanup edges

The pass adds cleanup before:

- return;
- break and continue;
- normal scope exit;
- assignment over an active destination.

There are no exception cleanup edges in 0.1.

Even though the C++ backend can perform cleanup through RAII, explicit MIR
cleanup is still valuable:

- it verifies HUC destruction order independently;
- it supports `huc emit-mir`;
- it prepares the C backend;
- it prevents accidental dependence on C++ temporary-lifetime rules.

The first emitter may coalesce explicit cleanup into generated RAII where the
result is provably equivalent.

### 12.4 Special-operation elaboration

Semantic analysis classifies a structure as Copy only when every field is Copy
and it declares neither `clone` nor `drop`. Declaring either method makes the
structure Move.

Elaboration handles the operations as follows:

- ordinary Copy binding becomes fieldwise `Copy`;
- `copy` of a Move structure becomes `Clone` into a fresh temporary;
- Move binding becomes fixed fieldwise `Move` followed by `Deactivate`;
- Move assignment checks exact self-assignment, drops a distinct active
  destination, then performs the fixed move;
- `destination = copy source` completes `Clone` before dropping destination;
- destruction calls user `drop`, then drops fields in reverse order.

No HIR or MIR operation performs overload resolution for move. A C++ move
constructor emitted for backend convenience is compiler machinery, not a HUC
customization point. See [Value Semantics and Special Operations](value-semantics.md).

## 13. C++20 lowering

### 13.1 Runtime owner

The generated runtime support contains conceptually:

```cpp
namespace huc_rt {

namespace generated {

// Specialized/generated for every owned HUC pointee type. It runs HUC
// drop-in-place logic, ends the C++ storage lifetime, and deallocates.
template<class T>
void destroy_owned(T* pointer) noexcept;

}

template<class T>
class owner {
    T* pointer_ = nullptr;

public:
    owner() noexcept = default;
    explicit owner(T* pointer) noexcept : pointer_(pointer) {}

    owner(const owner&) = delete;
    owner& operator=(const owner&) = delete;

    owner(owner&& other) noexcept
        : pointer_(other.release()) {}

    owner& operator=(owner&& other) noexcept {
        if (this != &other) {
            reset();
            pointer_ = other.release();
        }
        return *this;
    }

    ~owner() { reset(); }

    T* get() const noexcept { return pointer_; }

    T* release() noexcept {
        T* result = pointer_;
        pointer_ = nullptr;
        return result;
    }

    void reset(T* replacement = nullptr) noexcept {
        T* old = pointer_;
        pointer_ = replacement;
        if (old != nullptr) {
            generated::destroy_owned(old);
        }
    }
};

}
```

`destroy_owned<T>` is selected statically from `T`; no deleter or function
pointer is stored in `owner<T>`. The real implementation must handle incomplete
generated types, over-aligned allocation, and partially emitted modules
carefully. The wrapper remains one pointer.

### 13.2 Operation mapping

| HUC operation | Generated C++ |
|---|---|
| `T*` | `T*` |
| `T&` | `huc_rt::owner<T>` |
| owner observation | `.get()` |
| owner move | `std::move(owner)` or `.release()` into a wrapper |
| `new T(values)` | `huc_rt::owner<T>{new T(values)}` |
| `copy owner` | generated clone plus allocation |
| `adopt[T](raw)` | explicit owner construction |
| `release(owner)` | `.release()` |
| `reset(owner)` | `.reset()` |

Generated C++ uses `std::move`; HUC source never does. This is backend syntax,
not HUC semantics.

### 13.3 Sequencing

Although C++20 does not choose a left-to-right order for all function arguments,
HUC requires each argument and corresponding parameter initialization to
complete left-to-right. The emitter therefore
materializes argument temporaries:

```huc
call(first(), second());
```

becomes conceptually:

```cpp
auto&& _hu_arg0 = first();
auto&& _hu_arg1 = second();
call(
    huc_rt::forward_huc(_hu_arg0),
    huc_rt::forward_huc(_hu_arg1)
);
```

The exact temporary type depends on Copy/Move use. The generated form must also
preserve HUC temporary destruction order.

### 13.4 Generated special members

HUC destruction cannot be implemented by placing the user `drop` body directly
in an unconditional C++ destructor. Consider a HUC structure that contains one
raw operating-system handle and declares `drop`. Its synthesized HUC move copies
the handle and marks the source storage inactive; it is not required to rewrite
the raw integer to a sentinel. An unconditional C++ destructor on the source
would therefore close the transferred handle.

For each HUC Move structure, the emitter instead generates:

- deleted C++ copy constructor and copy assignment;
- compiler-only fieldwise move construction/assignment machinery that never
  invokes user code;
- a `huc_drop_in_place(T*)` helper that runs user `drop` and then recursively
  cleans fields in reverse order;
- a backend C++ destructor shell that never invokes HUC `drop` merely because
  C++ storage leaves scope;
- an explicit clone helper when the HUC clone protocol exists.

MIR calls `huc_drop_in_place` only for an active HUC place. It then neutralizes
generated owner/Move fields so the later C++ destructor shell is a no-op. A
moved-from raw handle may retain its old bits, but its HUC place is inactive and
never receives the HUC drop call.

When conditional control flow prevents static cleanup selection, the emitter
uses a sidecar `slot_guard<T>` or boolean that refers to the value without
changing its representation. HUC temporaries are handled by the same MIR
liveness rules rather than by C++ full-expression lifetime rules.

Generated source never asks C++ overload resolution to choose these members.
They exist solely to carry already-resolved HUC operations. The backend may
replace special members with generated free functions where that produces more
obviously correct code.

HUC fixed-by-default permission is enforced before emission. The backend must
not add C++ `const` to physical storage when doing so would prevent synthesized
move assignment, cleanup neutralization, or reinitialization of an inactive HUC
place. Read-only access remains a HUC semantic property even when generated C++
storage is physically writable.

For a HUC Copy structure—one whose fields are all Copy and which declares
neither `clone` nor `drop`—generated C++ may use defaulted copy operations
after layout and ordering checks.

### 13.5 Namespaces and modules

Each HUC module maps to a unique generated namespace and normally produces:

```text
build/huc-gen/app.main.hpp
build/huc-gen/app.main.cpp
```

The header contains concrete exported declarations and private forward
declarations required across generated translation units. The `.cpp` contains
bodies. The first whole-program implementation may emit one `.cpp` to simplify
ordering.

### 13.6 Diagnostics and source mapping

The HUC frontend should catch all language errors. Backend diagnostics are
treated as compiler bugs or foreign-code diagnostics.

Generated files contain `#line` directives where they improve messages, but a
sidecar `.hucmap` records richer expansion chains. The driver rewrites backend
diagnostics to HUC locations and includes the raw backend message under
`--verbose-backend`.

### 13.7 Backend fence

Generated code must use:

- fully qualified runtime names;
- reserved mangled identifiers;
- no unqualified argument-dependent lookup;
- no backend templates derived directly from user generics;
- no backend overload resolution to decide HUC ownership;
- explicit temporaries for HUC sequencing;
- explicit casts for HUC numeric conversions.

These rules stop C++ from silently becoming the specification.

## 14. C backend later

A later C17 emitter consumes the same MIR:

- structures become C structs;
- methods become name-mangled free functions with explicit `this`;
- owners become pointer fields or raw pointers plus explicit MIR drops;
- overloads and modules use mangled names;
- generated `clone`, `drop`, and allocation thunks become free functions;
- cleanup edges become labels and branches.

No phase evaluator or semantic pass should need replacement. If adding the C
backend requires interpreting HUC semantics again, the MIR is too weak.

## 15. Driver and build pipeline

### 15.1 Commands

```text
huc check app/main.huc
huc build app/main.huc -o build/app
huc run app/main.huc -- program-arguments
huc emit-hir app/main.huc
huc emit-mir app/main.huc
huc expand app/main.huc
huc emit-cpp app/main.huc --out-dir build/huc-gen
```

`run` is convenience for build and execute. It must separate compiler and
program arguments at `--`.

### 15.2 Backend discovery

The driver searches, in order:

1. `--cxx <path>`;
2. a project configuration;
3. `clang++`;
4. `g++`.

It invokes a backend in a deterministic argument order and records:

- executable identity/version;
- target triple;
- compile flags;
- HUC runtime header hash.

### 15.3 Build artifacts

```text
build/
  huc-gen/       generated C++
  huc-cache/     content-addressed frontend/staging cache
  huc-obj/       backend object files
  compile_commands.json
```

Generated C++ is retained under `emit-cpp` or a verbose/debug build. Normal
builds may clean it only through an explicit build-system operation.

## 16. Caching and separate compilation

### 16.1 Content addressing

Cache entries are keyed by cryptographic hashes of canonical serialized data,
not timestamps.

Initial cacheable units:

- lexed/parsed module;
- exported semantic interface;
- `fn1`/`struct1` specialization;
- `fn2` result;
- concrete HIR;
- generated C++ object.

### 16.2 Interface files

A future `.huci` interface contains:

- exported runtime signatures;
- exported type layouts where ABI-visible;
- typed HIR bodies of exported `fn1`, `fn2`, and `struct1` declarations;
- reflection metadata permitted across modules;
- definition and dependency fingerprints.

Until interface serialization is stable, whole-program compilation is safer
than pretending to offer separate generic compilation.

### 16.3 Reproducibility

Canonical serialization sorts maps by stable IDs, normalizes paths relative to
the project root where possible, and excludes allocator addresses. Generated
symbol hashes must be identical across processes for identical inputs.

## 17. Testing strategy

### 17.1 Unit tests

- tokenization, including `&` and numbered keywords;
- Pratt precedence and recovery;
- canonical type interning;
- overload ranking;
- phase availability;
- specialization keys;
- evaluator arithmetic and resource limits;
- hygiene and splicing;
- drop-state analysis.

### 17.2 Golden tests

Store input plus expected:

- diagnostics;
- expanded residual HUC;
- concrete HIR;
- MIR;
- generated C++.

Golden C++ tests should ignore nondeterministic temporary numbers by making
temporary naming deterministic.

### 17.3 Execute tests

Compile and run small programs that record:

- argument evaluation order;
- move and clone counts;
- owner nulling;
- self-move behavior;
- destruction order;
- early-return cleanup;
- conditional owner movement;
- `if1` branch removal;
- `for1` unrolling;
- forced `@` evaluation;
- reflection-generated fields and calls.

### 17.4 ABI and cost assertions

Generated C++ contains static assertions:

```cpp
static_assert(sizeof(huc_rt::owner<T>) == sizeof(void*));
static_assert(alignof(huc_rt::owner<T>) == alignof(void*));
```

Compiler tests should inspect optimized assembly for representative owner moves
and observations, but assembly shape is a regression aid rather than a language
guarantee.

### 17.5 Differential backend tests

When a C backend exists, run the same MIR through both backends and compare
observable results. The C++ backend must not become the oracle for undefined
HUC behavior.

## 18. Implementation milestones

### Milestone 0: project skeleton

- driver, source manager, diagnostics;
- lexer and parser;
- AST dump;
- module/import discovery;
- grammar and syntax tests.

Exit criterion: all examples parse and malformed examples recover sensibly.

### Milestone 1: runtime scalar core

- scalar types, functions, locals, structures;
- expressions and runtime control flow;
- name and type checking;
- concrete HIR/MIR;
- C++ emission and backend invocation;
- `extern "C"` declarations.

Exit criterion: small C-like HUC programs compile and run with guaranteed
left-to-right evaluation.

### Milestone 2: ownership

- `T*`, `mod T*`, and `T&`;
- one-word runtime owner;
- new/adopt/release/reset/swap;
- observing versus consuming parameter binding;
- Move/Copy classification;
- clone protocol;
- drop elaboration and destruction tests.

Exit criterion: ownership examples demonstrate the same optimized operation
count as the corresponding `std::unique_ptr` patterns.

### Milestone 3: staged generics

- `fn1` and `struct1`;
- `auto` parameters and monomorphization;
- explicit compile-time argument lists;
- `if1`, `for1`, and `where1`;
- specialization cache;
- residual HUC inspection.

Exit criterion: generic max, fixed-capacity structures, and variadic unrolling
produce concrete runtime HIR without backend templates.

### Milestone 4: meta execution

- `fn2`, `struct2`, `let1`, and `let2`;
- `@` forced evaluation;
- deterministic HIR interpreter;
- resource limits and evaluation traces;
- incremental result cache.

Exit criterion: nontrivial compile-time programs evaluate without compiling or
executing temporary C++.

### Milestone 5: reflection and generation

- immutable Type/Field/Function metadata;
- quote, splice, emit, `gen`, and `gen_global`;
- hygiene;
- expansion-chain diagnostics;
- dependency-aware reflection cache.

Exit criterion: a serializer or formatter can be generated structurally from a
runtime structure definition.

### Milestone 6: hardening

- fuzz lexer/parser/evaluator;
- sanitizer builds;
- deterministic builds;
- performance profiles;
- standard library seed;
- formatter and language-server foundations.

## 19. First implementation subset

The fastest credible prototype should not attempt the entire specification at
once. Its accepted source subset should be:

```text
module/import
fn
struct
bool, integer, float, void
T*, mod T*, T&
local declarations
if/while/return
ordinary calls
new/adopt/release/reset
extern "C"
```

Then add one vertical staging slice:

```text
fn1 with auto runtime parameters
if1 over a bool compile-time parameter
one specialization cache
```

This proves the distinctive HUC idea while keeping reflection, variadics, and
general syntax generation out of the initial executable compiler.

## 20. Architectural invariants

The following should be enforced in code review and tests:

1. No HUC phase rule is delegated to C++ templates or `constexpr`.
2. No ownership decision is delegated to C++ overload resolution.
3. HIR records Copy, Move, Observe, and Clone explicitly.
4. MIR is fully concrete and contains no phase construct.
5. Compile-time evaluation has no ambient untracked host effects.
6. Generated code uses syntax nodes and hygiene, never source concatenation.
7. Backend output preserves HUC left-to-right sequencing.
8. `T&` remains one word under the default allocator.
9. Diagnostics retain source and expansion chains across every pass.
10. A future C backend can consume MIR without recreating semantic analysis.
