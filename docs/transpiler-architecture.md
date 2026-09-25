# HUC Bootstrap Transpiler Architecture

Status: implementation design for HUC 0.1

This document gives implementation details for the
[HUC Implementation Plan](implementation-plan.md). If the two documents
disagree, follow the plan.

Key terms:

- **AST**: abstract syntax tree, which records parsed source syntax.
- **HIR**: high-level intermediate representation, with resolved names,
  types, and value operations.
- **MIR**: mid-level intermediate representation, with explicit execution
  order, control flow, transfers, and cleanup.
- **Lowering**: translating an operation into a simpler representation.
- **Residualization**: producing the runtime code left after compile-time work.
- **Materialization**: writing a compiler-known value as valid HUC0 runtime
  syntax.
- **Place**: a storage location. **Liveness** here means whether that place
  contains an active HUC value that may need cleanup.
- **Insertion cursor**: the position where generated HUC0 text will be added.

## 1. Recommendation

The first two HUC translators and their command-line driver are implemented in
portable ISO C++20. Their first output language is ISO C17:

```text
HUC1 source
    |
    | huc1_expand
    v
printable per-module HUC0 source
    |
    | huc0_transpile
    v
C17 headers + one root translation unit
```

The translators are independently usable. Milestone 1 implements
HUC0-to-C17 first. Milestone 2 then implements HUC1-to-HUC0 and invokes the
already public HUC0 translator only through the generated HUC0 module files.
A user may always write, inspect, and compile HUC0 without using HUC1.

This choice is for the first implementation; it does not make HUC's rules
depend on C or C++. HUC has its own parser, type checker, staging evaluator,
specialization engine, lifetime lowering,
and intermediate representation, all implemented in C++20. Generated C17 is a
separate output artifact and never defines HUC's behavior.

The reason for choosing C is that HUC already needs to make its operations
explicit: construction, destructive relocation, active/inactive state,
destruction, and evaluation order. C structures, pointers, and generated free
functions express those operations without adding implicit constructor,
assignment, or destructor behavior.

The first compiler must emit cleanup for every normal scope exit, early
return, loop exit, replacement, and abandoned initialized prefix, with flags
where a value is active on some paths but not others. Building this cleanup
into MIR is required from the start. The same explicit MIR can support future
C++ or native-code output without redoing HUC semantic analysis.

## 2. C versus C++ as the first emitted backend

| Concern | First C17 backend | Possible later C++ backend |
|---|---|---|
| Methods and modules | Mangled free functions and explicit receivers | Free functions or native namespace/member syntax |
| Advanced values | Plain storage plus MIR-controlled relocation and drops | The same model; ordinary C++ moves alone are insufficient |
| Generic specializations | Concrete functions and structures | Concrete functions and structures; templates are unnecessary |
| Lifetime implementation | Cleanup edges and conditional flags | Same HUC analysis; optional RAII only where equivalent |
| Backend constraints | C effective types, aliasing, alignment, and sequencing | C++ object lifetimes, aliasing, alignment, and sequencing |
| Foreign integration | Direct supported C ABI; adapters for C++ later | C ABI plus potential direct C++ bindings later |

The C emitter must not translate HUC source by token substitution. Copying a
C structure does not by itself implement HUC relocation or its cleanup
obligations. Typed HIR and MIR record whether the operation copies or
transfers a value. The compiler also implements construction in final storage
and HUC's fixed-by-default permissions; C declarations do not supply those rules.

Use ISO C17 as the initial output baseline. Compiler-specific extensions,
direct C++ interoperability, and additional backends are not bootstrap
requirements. C17 does not guarantee faster output; correctness and generated
operation counts must be tested.

The table compares output languages. The transpiler executable, frontend,
staging interpreter, middle end, emitters, and driver are C++20 programs.

## 3. Compiler overview

```text
.huc1 module tree
    |
    v
HUC1 source manager + delimiter scanner + staged index
    |
    v
selected-body parser + phase checker + C++20 interpreter
    |
    v
family specialization + residualizer + raw-text insertion
    |
    v
printable per-module .huc0 files + origin maps

-------------------- explicit textual boundary --------------------

.huc0 module tree
    |
    v
HUC0 source manager -> lexer -> parser -> syntax AST
    |
    v
declaration collection -> name/type/lifecycle checking
    |
    v
concrete typed HIR -> lifetime/drop elaboration -> sequenced MIR
    |
    v
C17 emitter + source map + runtime header
    |
    v
clang / gcc (-std=c17) -> object files -> linker
```

`huc0_transpile` accepts source files, not an HUC1 AST or an undocumented
in-memory residual IR. `huc build` must first write the HUC0 artifacts and then
pass those paths to `huc0_transpile`. This boundary is both a correctness test
and the reason generated HUC0 remains a useful inspection and interchange
artifact.

Neither translator asks the downstream C compiler to implement HUC families,
compile-time evaluation, overload selection, or phase semantics. All such work
is complete before C is emitted.

## 4. Proposed repository layout

```text
CMakeLists.txt
include/
  huc/common/
  huc/huc0/
  huc/huc1/
src/
  common/
  huc0/
  huc1/
  driver/
runtime/
  huc_runtime.h
tests/
  unit/
  golden/
  negative/
  execute/
```

The implementation language is C++20. Standard containers, RAII, value types,
and explicit ownership are preferred. Arenas and stable ID-indexed tables
remain useful where they simplify AST/IR lifetime and deterministic output. A
future LLVM integration stays behind a narrow backend interface.

The logical libraries are:

- `huc_common`: sources, diagnostics, target configuration, module graphs,
  arenas, IDs, and shared delimiter/parser support;
- `huc0`: the independently usable HUC0 frontend, HIR/MIR passes, ownership
  lowering, and C17 emitter;
- `huc1`: staged indexing, deferred-body handling, the phase interpreter,
  specialization, and printable HUC0 residualization;
- `huc`: a thin C++20 command-line driver.

The generated `huc_runtime.h` belongs to the C17 output, not to either
translator executable. Compiler sources do not include it.

### 4.1 C++20 implementation conventions

Compiler modules expose typed C++ interfaces:

```cpp
using FileId = StrongId<struct FileTag>;
using ModuleId = StrongId<struct ModuleTag>;

struct ParseResult {
    std::optional<ModuleId> module;
    std::vector<Diagnostic> diagnostics;
};

class Compiler {
public:
    [[nodiscard]] ParseResult parse_module(FileId file);
};
```

Pass state belongs to `Compiler` or a short-lived pass object rather than
process-global variables. RAII owns resources, `std::variant` represents
heterogeneous nodes where appropriate, and pass boundaries exchange stable
IDs rather than pointers into movable storage. User source errors are recorded
as diagnostics.

### 4.2 Public translator interfaces

The two libraries expose value-oriented C++20 APIs with RAII-managed results.
Exact container choices may evolve, but the direction and artifact boundary do
not:

```cpp
namespace huc {

struct Huc0TranspileRequest {
    std::string root_module;
    std::vector<std::filesystem::path> import_roots;
    std::filesystem::path output_directory;
    TargetConfig target;
    bool write_source_maps = true;
};

struct Huc0TranspileResult {
    std::vector<Artifact> artifacts;
    std::vector<Diagnostic> diagnostics;
    DependencyGraph dependencies;
    bool succeeded = false;
};

[[nodiscard]] Huc0TranspileResult
huc0_transpile(const Huc0TranspileRequest& request);

struct Huc1ExpandRequest {
    std::string root_module;
    std::vector<std::filesystem::path> import_roots;
    std::filesystem::path output_directory;
    TargetConfig target;
    PhaseLimits limits;
    bool write_source_maps = true;
};

struct Huc1ExpandResult {
    std::vector<Artifact> modules;
    std::vector<Diagnostic> diagnostics;
    DependencyGraph dependencies;
    bool succeeded = false;
};

[[nodiscard]] Huc1ExpandResult
huc1_expand(const Huc1ExpandRequest& request);

} // namespace huc
```

Ordinary source errors are returned as diagnostics rather than thrown. A
chained build passes the paths from `Huc1ExpandResult::modules` back through a
normal `Huc0TranspileRequest`.

## 5. Frontend

### 5.1 Source manager

`SourceManager` owns immutable source buffers and assigns each one a `FileId`.
Every token and syntax node carries a compact `SourceSpan`. Its range includes
the start position but excludes the end position (a half-open range):

```cpp
typedef struct HucSourceSpan {
    HucFileId file;
    uint32_t begin;
    uint32_t end;
} HucSourceSpan;
```

Line and column numbers are computed lazily from per-file newline indexes.
Generated nodes additionally carry an expansion chain:

```text
generated HUC0 range
  -> selected HUC1 source or raw generation call
  -> phase helper call stack
  -> specialization request chain
  -> original source declaration
```

This is required for useful staging diagnostics.

### 5.2 Lexer

The lexer should:

- recognize numbered keywords as dedicated tokens;
- recognize `in` as a reserved keyword;
- preserve comments and trivia when formatter support is enabled;
- diagnose unsupported phase spellings such as `fn3`;
- use `&` for prefix address-taking and binary bitwise AND, separate from
  `&&` logical AND;
- retain the raw spelling of numeric and string literals;
- validate optional numeric `_` separators between digits of the applicable
  base, then ignore them when computing the value; preserve the spelling for
  diagnostics and source-preserving tools;
- never depend on semantic type information.

Whitespace separates tokens but does not determine operator roles. Formatting
preferences are not lexer errors; a future formatter follows the separate
[formatting guide](formatting.md).

Representative token kinds:

```cpp
typedef enum HucTokenKind {
    HUC_TOKEN_IDENTIFIER,
    HUC_TOKEN_INTEGER_LITERAL,
    HUC_TOKEN_FLOAT_LITERAL,
    HUC_TOKEN_STRING_LITERAL,

    HUC_TOKEN_KW_FN,
    HUC_TOKEN_KW_FN1,
    HUC_TOKEN_KW_FN2,
    HUC_TOKEN_KW_STRUCT,
    HUC_TOKEN_KW_STRUCT1,
    HUC_TOKEN_KW_STRUCT2,
    HUC_TOKEN_KW_IF,
    HUC_TOKEN_KW_IF1,
    HUC_TOKEN_KW_FOR,
    HUC_TOKEN_KW_FOR1,
    HUC_TOKEN_KW_IN,
    HUC_TOKEN_KW_WHILE,
    HUC_TOKEN_KW_WHILE1,
    HUC_TOKEN_KW_LET,
    HUC_TOKEN_KW_LET1,
    HUC_TOKEN_KW_LET2,
    HUC_TOKEN_KW_IMPORT,
    HUC_TOKEN_KW_IMPORT2,

    HUC_TOKEN_STAR,
    HUC_TOKEN_AMPERSAND, /* prefix address-taking or binary bitwise AND */
    HUC_TOKEN_ARROW,
    HUC_TOKEN_LESS,
    HUC_TOKEN_GREATER,
    HUC_TOKEN_AT,
    // ...
} HucTokenKind;
```

`Ampersand` produces an address-of node in prefix expression position and
bitwise AND in infix position; it is not a type suffix. Operator roles do not
depend on whitespace. No C++-reference AST node exists. HUC0 has
one pointer-like form: a terminal `T*` raw-pointer suffix. The type parser
diagnoses `T**` immediately; canonical-type validation catches equivalent
forms exposed through aliases.

### 5.3 Parser

A hand-written recursive-descent parser with Pratt expression parsing is the
best bootstrap choice:

- the grammar is small;
- recovery can be tailored to declaration and statement boundaries;
- numbered constructs map directly to AST variants;
- no generated-parser dependency is needed;
- individual `<` and `>` tokens support phase-family binders, patterns, and
  requests.

The parser builds syntax, not types. It should preserve invalid placeholder
nodes after an error so later diagnostics remain local.

Variable and parameter parsers read an optional binding `mod`, then the name,
colon, and type. A `mod` inside that type is a separate pointee permission.
`let` and `let2` precede variable declarations but not parameters. Phase binders
also use `name: Type` or `name: auto`; they have no binding `mod`. All phase
families, including `let1`, put their name before their binder list. The
`for1` parser reads `let2 [mod] name: Type in expression` inside parentheses.

Use a shared list parser that accepts one optional trailing comma, including
for constructor initializer entries. Do not make acceptance depend on whether
the list spans several lines. Empty items remain errors. Ordinary control
flow parses braced bodies, with an `else if` alternative for chained
conditionals; phase-control bodies still use the deferred-body scanner.
Do not retain the old declaration order or `T&` as compatibility spellings.

`slot_off(place)` uses ordinary call syntax with a compiler-known,
non-overloadable, unqualified name. Resolve it as an intrinsic, not a
`std::` library call. Only `as` and `ptr_as` use the HUC0
type-operand call production. The old named address-taking intrinsic is
replaced by unary `&`.

Important recovery points are:

- `;`, `}`, and the next declaration keyword;
- the close of parenthesized binder/runtime lists and angle patterns or
  requests independently;
- `else` following `if` or `if1`;
- `)` following a parameter or condition.

The parser preserves an ambiguous angle postfix when syntax alone cannot
distinguish `name<arguments>` from comparison operators. Semantic name
resolution later decides whether the name denotes a phase-1 family. This is
not delegated to downstream C parsing.

### 5.4 AST

The syntax AST retains user spelling:

```cpp
typedef uint32_t HucDeclId;
typedef uint32_t HucParamDeclId;
typedef uint32_t HucTypeSyntaxId;
typedef uint32_t HucExprId;
typedef uint32_t HucBlockStmtId;
typedef uint32_t HucDeferredBodyId;
typedef uint32_t HucAnglePatternId;

typedef enum HucPhaseClass {
    HUC_PHASE_RUNTIME,
    HUC_PHASE_STAGED,
    HUC_PHASE_META,
} HucPhaseClass;

typedef enum HucBodySyntaxKind {
    HUC_BODY_PARSED,
    HUC_BODY_DEFERRED,
} HucBodySyntaxKind;

typedef struct HucBodySyntax {
    HucBodySyntaxKind kind;
    union {
        HucBlockStmtId parsed;
        HucDeferredBodyId deferred;
    } as;
} HucBodySyntax;

typedef struct HucFunctionDecl {
    HucDeclId id;
    HucPhaseClass phase;
    HucParamDeclId *family_binders;
    size_t family_binder_count;
    HucAnglePatternId specialization_pattern; /* zero on a primary */
    HucParamDeclId *runtime_params;
    size_t runtime_param_count;
    HucTypeSyntaxId return_type;
    HucExprId specialization_predicate; /* zero means absent */
    HucBodySyntax body;
    HucSourceSpan span;
} HucFunctionDecl;
```

`HUC_PHASE_RUNTIME` denotes an unnumbered declaration that survives in HUC0,
`HUC_PHASE_STAGED` denotes a suffix-1 family or structural phase operation
that produces HUC0, and `HUC_PHASE_META` denotes suffix-2 compiler-only code.
These are availability classes, not repeated compiler passes.

Use stable arena-allocated IDs rather than pointers as persistent cross-pass
identities. C pointers may be used temporarily within a pass, but caches and
diagnostics should refer to IDs.

### 5.5 HUC1 indexing and deferred bodies

The HUC1 frontend first indexes imports, declaration/family headers,
specialization patterns, and opaque body spans. A dedicated delimiter scanner
finds an outer matching `}` while recognizing nested braces, strings,
characters, line comments, and block comments. It does not produce HUC tokens
for the body.

Only a selected `if1` branch, a reached phase-loop body, or a chosen family
body is then lexed and parsed. A skipped body may contain invalid HUC; the
scanner may report only that it cannot find the outer closing delimiter.
Ordinary HUC0 code selected for output is kept by the per-module residualizer.
The HUC0 translator checks it again after it has been written to a file.

`struct1`, `fn1`, and `let1` use distinct indexed declaration nodes even
though they share binder, pattern, predicate, and deferred-body components.
The syntax tree also has a phase-expression item node for standalone
`@expression;` at module, type-body, function, or block scope. Its expression
must evaluate to compiler `void`; expansion erases the node after applying its
effects at the node's insertion cursor.

Ordinary `import` exposes runtime declarations and phase-1 families and remains
as a runtime import in the corresponding HUC0 module. `import2` is available
only to compiler code and is erased. There is no `import1` in 0.1.

## 6. Semantic analysis

The HUC0 translator owns runtime semantic analysis:

1. collect declarations and module imports;
2. resolve names;
3. resolve concrete types and layouts;
4. enforce permissions, constructors, and special-operation rules;
5. build concrete typed HIR;
6. make execution order, relocation, and cleanup explicit in MIR.

The module checker admits runtime globals only for drop-free Basic scalars and
raw pointers. It requires constant initializers for fixed globals, supplies
zero/null only to omitted mutable initializers, and rejects structures,
Advanced values, runtime initialization, and global cleanup in 0.1.

HUC1 performs its own phase checking, family matching, and selected-body
analysis before printing HUC0. The HUC0 translator then reparses and checks
that text exactly as it checks user-written HUC0. No generic HIR or
specialization cache crosses the translator boundary.

A selected deferred body is parsed according to its enclosing module, type,
function, or block. Module/type bodies may mix generated runtime declarations
or members with compiler-only statements. Those statements must use values
available at compile time; they execute during expansion and disappear from
the output. Skipped bodies are only delimiter-scanned, not parsed.

### 6.1 Symbol table

Every symbol records:

```cpp
typedef struct HucSymbol {
    HucSymbolId id;
    HucInternedString name;
    HucSymbolKind kind;
    HucPhaseClass phase;
    HucModuleId module;
    HucSourceSpan definition;
} HucSymbol;
```

Scopes form an explicit tree. HUC1 origin records may point back to stable
symbol IDs, but emitted HUC0 names are ordinary text and are resolved again by
the HUC0 translator.

### 6.2 Canonical types

Types are interned: equivalent types share one canonical compiler record.

```cpp
typedef enum HucTypeKind {
    HUC_TYPE_ERROR,
    HUC_TYPE_VOID,
    HUC_TYPE_NEVER,
    HUC_TYPE_BOOL,
    HUC_TYPE_INTEGER,
    HUC_TYPE_FLOAT,
    HUC_TYPE_STRUCT,
    HUC_TYPE_RAW_POINTER,
    HUC_TYPE_FIXED_ARRAY,
    HUC_TYPE_FUNCTION,
    HUC_TYPE_META_TYPE,
    HUC_TYPE_SYMBOLIC,
} HucTypeKind;
```

A raw pointer has a canonical node containing its pointee type and access
permissions. HUC has no `const` type qualifier: fixedness and `mod` are
represented on the correct value, pointer-slot, and pointee layers, never
inferred from emitted C declarator strings.

Canonicalization rejects a raw-pointer node whose pointee is itself a raw
pointer. Unary `&` accepts an addressable inline non-pointer place,
preserves its mutation permissions, and lowers to an address calculation.
Evaluate the place once; do not materialize a temporary, transfer its value,
extend its lifetime, or update cleanup state. Reject pointer slots and
non-place operands, including function symbols.

`slot_off` accepts any addressable data place, including a raw-pointer
slot, and lowers to its address converted to `usize`. It does not load the
value or invoke relocation or cleanup. Reject non-place operands rather than
materializing temporaries. An explicit `ptr_as` from `usize` reconstructs
a one-level raw pointer; it never synthesizes a composed pointer type.
Permissions and lifetime preconditions remain the caller's responsibility;
the integer carries no permission or liveness metadata. Escape and alias
analysis must account for the exposed address.

Raw-pointer initialization, assignment, arguments, and returns copy the
address without modifying the source or touching the pointee's lifetime.
Check type compatibility and permit dropping pointee `mod`, but never
gaining it. `auto` retains the inferred raw-pointer type.

Member access preserves each layer's permissions. A fixed object prevents
replacement of a pointer field and mutation of an inline field. It does
not remove a field type's `mod` permission to mutate the separate object
that the field points to.

Each concrete type records:

- size and alignment;
- type category (Basic or Advanced);
- whether it requires drop;
- field layout;
- phase class;
- a stable structural fingerprint for caches.

### 6.3 HUC1 phase checking

Every expression receives:

```cpp
typedef enum HucAvailability {
    HUC_AVAILABILITY_RUNTIME_VALUE,
    HUC_AVAILABILITY_COMPILE_TIME_VALUE,
    HUC_AVAILABILITY_SYMBOLIC_RUNTIME_EXPR,
} HucAvailability;
```

This is separate from `PhaseClass`, which classifies declarations.

Examples of rejected edges:

- a runtime value used as an `if1` condition;
- a `struct2` stored in runtime memory;
- an ordinary `fn` called by the compile-time evaluator;
- a `fn2` referenced as a residual runtime symbol.
- reserved future `@this` contextual reflection in bootstrap HUC1.

The diagnostic should name both sides:

```text
error: phase-1 condition requires a compile-time bool
  --> file.huc1:18:10
   |
18 |     if1 (runtime_flag) {
   |          ^^^^^^^^^^^^ runtime value originates here
```

### 6.4 Overload resolution

The resolver implements only the ranking specified by the language:

1. exact match;
2. lossless built-in numeric promotion;
3. permission-dropping pointer conversion.

Values and pointers do not implicitly convert to each other.

For a phase-family specialization, its optional parenthesized predicate is
evaluated only after arity and structural-pattern matching. Predicate failure
silently removes a candidate; evaluator failure is a diagnostic. Predicates
filter candidates but never rank them.

### 6.5 C-linkage validation

An `extern "C"` signature is accepted only when every parameter and result
lowers through the target's documented C ABI mapping. HUC0 permits built-in
arithmetic scalars, `void` results, and one-level raw pointers to built-in
types. It rejects structures, fixed arrays, Advanced/drop types, phase
types, compiler metadata, pointer chains, opaque foreign handles, and
variadics. Aliases are checked after canonicalization so they cannot hide a
forbidden boundary type.

## 7. Typed HIR

HUC0 HIR is expression-oriented, typed, structured, and fully concrete.

`ValueUse` distinguishes:

```cpp
typedef enum HucValueUse {
    HUC_VALUE_USE_READ,
    HUC_VALUE_USE_OBSERVE,
    HUC_VALUE_USE_COPY,
    HUC_VALUE_USE_RELOCATE,
    HUC_VALUE_USE_PLACE,
} HucValueUse;
```

Every expression includes:

```cpp
typedef struct HucHirExprHeader {
    HucHirExprId id;
    HucTypeId type;
    HucValueUse use;
    HucSourceOrigin origin;
} HucHirExprHeader;
```

This distinction prevents the C emitter from rediscovering transfer intent
from plain storage representations.

Basic and Advanced are type categories; `Copy`, `Clone`, and `Relocate` name
compiler operations, not categories. Same-type Basic binding uses `Copy`.
An inline Advanced transfer uses `Relocate`: the destination becomes active
and the source becomes inactive without receiving `drop`.
Explicit duplication through `clone()` uses `Clone`.

Representative HIR operations:

```text
LoadCopy(place)
LoadRelocate(place)
AddressOf(place)
SlotAddress(place)
PointerFromAddress(address, raw_pointer_type)
Clone(value)
ConstructInPlace(destination, constructor, arguments)
Call(function, arguments)
```

`ConstructInPlace` is not shorthand for “construct a temporary, then
relocate.” When `T(arguments)` initializes a local, field, by-value parameter,
or return result, HIR names that final place as the constructor destination.
This preserves the final `this` address for constructors and avoids an observable
relocation. A call that returns a structure uses an explicit result place so
the callee can construct directly there.

HIR records whether a source is a tracked root, a pointer dereference, or a
direct subobject. Transferring from a named root requires `mod` before its
name; extraction through a pointer requires pointee `mod` instead. A fresh
function result, explicit `copy`/clone result, or
unavoidable constructor temporary can be transferred once without a source
declaration marked `mod`. Direct construction in the destination remains
preferred and creates no such temporary.

Relocation accepts a whole tracked local, parameter, temporary, or result, or
an explicit dereference `*pointer` through `mod T*`. Both forms transfer the
whole source value. HIR records the cleanup mode: a tracked root updates
compiler-managed source state, while pointer extraction relies on the
programmer to prevent cleanup of the inactive source slot. Evaluate the source
pointer once without changing its address value.

Relocation remains rejected from `value.field`, `array[i]`, and
`pointer->field`. HUC does not track partially inactive aggregates. Explicit
dereference of a writable pointer to one of those slots uses programmer-managed
cleanup; it does not suppress the enclosing object's field/element cleanup.
Neither pointer provenance nor stack/heap allocation selects the cleanup mode.
In particular, `*(&local)` does not acquire the direct local's cleanup behavior.
Preserve this distinction through inlining and other optimizations.

Indirect destinations may be replaced when their validity and activity
preconditions hold; the restriction above concerns sources. Containers manage
their backing storage and initialized element ranges explicitly. The only
additional raw-storage lifetime intrinsics planned are `construct_at` and
`destruct_at`; their interfaces and lowering will be designed later,
without adding user-overloadable relocation hooks.

## 8. Compile-time evaluator

### 8.1 Initial strategy

Interpret typed HUC1 phase AST in-process. Do not:

- generate and compile a temporary C++ program for every `fn2` call;
- execute arbitrary host C++ plugins;
- represent metadata as source strings;
- use host addresses as stable identities.

An interpreter is slower than native execution, but makes it much easier to
provide repeatable results, inspect execution, cache results, and report errors
at source locations. Bytecode can improve performance later.

### 8.2 Values

```cpp
typedef enum HucCtValueKind {
    HUC_CT_BOOL,
    HUC_CT_INTEGER,
    HUC_CT_FLOAT,
    HUC_CT_STRING,
    HUC_CT_LIST,
    HUC_CT_STRUCT,
    HUC_CT_TYPE,
    HUC_CT_SYMBOLIC_EXPR,
    HUC_CT_SEMANTIC_ID,
} HucCtValueKind;

typedef struct HucCtValue {
    HucCtValueKind kind;
    union {
        bool boolean;
        HucCtInteger integer;
        HucCtFloat floating;
        HucCtStringId string_id;
        HucCtObjectId object_id; /* list or structure in the CT heap */
        HucTypeId type_id;
        HucSymbolicExprId symbolic_expr_id;
        HucSemanticId semantic_id; /* internal stable identity */
    } as;
} HucCtValue;
```

The evaluator owns a compile-time heap with deterministic object IDs. Its
allocation order is not observable except through equality defined by HUC.
The stable semantic-ID case is an implementation facility for family
selection and residualization; it is not yet a public reflection API.

### 8.3 Symbolic execution for `fn1`

A `fn1` is not generally executed with concrete runtime values. Its runtime
parameters enter the evaluator as `SymbolicExprId` values carrying concrete
types.

Operations behave as follows:

- a pure operation on concrete compile-time inputs evaluates immediately;
- an ordinary runtime operation involving a symbolic input records residual
  source that the residualizer can print as HUC0;
- `if1` requires a concrete boolean and selects one branch;
- ordinary `if` with a symbolic condition creates residual branches;
- `for1` iterates a concrete compile-time sequence;
- raw generation calls append text to their defined HUC0 output cursor.

This is a small partial evaluator, not C++ template substitution. It has no
bootstrap syntax-object, quote, splice, or structural emit value.

### 8.4 Forced mode

`@call` supplies concrete values and sets the evaluator mode to
`HUC_EVALUATION_FORCED`. Creation of an unresolved symbolic operation in forced
mode is a diagnostic.

A normal `fn1` call requests or calls a concrete runtime specialization.
Prefixing that call with `@` instead requires complete compile-time evaluation,
and only a result that can be written as a HUC0 value may leave the island
for runtime code.

### 8.5 Resource control

The evaluator tracks:

- executed instruction count;
- call depth;
- allocated compile-time bytes;
- generated-byte count;
- specialization depth.

Limits are configurable and included in diagnostics. The implementation should
print the most recent evaluation frames rather than a C++ stack trace.

### 8.6 Effects and dependencies

The bootstrap phase interpreter has no filesystem, process-environment,
network, clock, random, or host-process APIs. Its only compiler-supplied
effects are the two raw HUC0 generation operations in Section 10, whose bytes,
cursor, and origin are recorded explicitly. The build dependency graph
therefore consists of loaded HUC1/HUC0 modules and configured target inputs.
Any future explicit external-input API must return a dependency record with
its value and is part of the separate compiler-library design.

### 8.7 Staging contexts

`if1`, `for1`, and `while1` execute while HUC1 is expanded and may occur at
module, type-body, function, or nested-block scope. Selected output must be
valid declarations/members at module or type scope and may be declarations or
statements at function/block scope. Inside `fn2`, ordinary `if`, `for`, and
`while` already execute in compiler phase, so no `if2`, `for2`, or `while2`
tokens exist.

All compiler-only variables are spelled `let2`; parameters inherit the phase
of their enclosing function. `fn2`, `struct2`, and `let2` are erased.
`@` opens a compile-time evaluation island and never qualifies a type. A
selected `let1` specialization must leave exactly one ordinary runtime `let`
with the family name and no other residual statement. Raw generation is
forbidden inside a `let1` selection body because it would evade that invariant.
`let1` is permitted at module and function/block scope, but never as an
instance field.

## 9. Specialization engine

HUC1 indexes one primary family per name and scope for `struct1` and `let1`,
and one primary family per name and scope for `fn1` in 0.1. Syntax is retained
in these distinct parts:

```huc
struct1 Buffer(T: auto, N: usize) { ... } // primary binders
struct1 Buffer(T: auto, N: usize)<T*, N>(predicate) { ... }
struct1 Buffer()<Widget, 8> { ... } // full specialization

fn1 convert(T: auto)(mod value: T) -> T { ... }
fn1 convert(T: auto)<T*>(predicate)(value: T*) -> T* { ... }
fn1 convert()<Widget>(mod value: Widget) -> Widget { ... }

let1 selected(T: auto) { let selected: T = T(); }
let1 selected(T: auto)<T*>(predicate) { let mod selected: T* = null; }
let1 selected()<Widget> { let selected: Widget = Widget(); }

let value: Buffer<i32, 16> = Buffer<i32, 16>(); // request
let result: i32 = convert<i32>(input);
use(selected<Widget>);
```

Parentheses after the family name introduce binders. Angle brackets after the
binder list introduce a specialization pattern. An empty binder list denotes
a full specialization. Angle brackets at a use site request a specialization.
The lexer emits individual `<` and `>` tokens; the parser preserves ambiguous
angle postfixes until name resolution.

A `T: auto` binder's entity kind is inferred from use; a binder used in a type
position must hold a type, while typed non-type binders use ordinary types.
Normal `fn1` requests may deduce binders by unifying runtime parameter and
argument types. Non-deducible binders require explicit angle arguments.
Primaries may have trailing defaults; specializations may not add defaults.

Selection first matches argument count and structural angle pattern, then
binds variables and evaluates the optional pure predicate. Exact patterns
outrank partial patterns, structurally more specific partial patterns outrank
broader ones, and the primary is the fallback. Predicates filter but do not
rank, and declaration order never resolves a tie.

The primary must precede its specializations, and 0.1 requires every
specialization in the defining module. Only the selected body is parsed.
Every statically requested concrete specialization is printed at its family
declaration site. A function-local `let1` result therefore initializes on each
runtime execution that reaches that declaration; staging never inserts a lazy
runtime flag.

### 9.1 Keys

```cpp
typedef struct HucSpecializationKey {
    HucDeclFingerprint declaration;
    HucSerializedCtValue *compile_time_arguments;
    size_t compile_time_argument_count;
    HucTypeFingerprint *runtime_types;
    size_t runtime_type_count;
    HucTargetFingerprint target;
    HucFeatureFingerprint features;
} HucSpecializationKey;
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

Generated HUC0 specialization names use a reserved readable prefix plus a
stable hash:

```text
_HU3app4math6select__A7F19C2E
```

The HUC0 name is deterministic for the defining family, canonical arguments,
and target-relevant configuration. The C backend encodes this HUC0 symbol as
a collision-free C identifier without using C implementation-reserved names;
the HUC-only prefix above is not emitted verbatim. Exact spellings remain
implementation details until a native HUC ABI is specified; comments and
origin maps recover the source family and compile-time arguments.

When a request originates in another module, its generated concrete HUC0
declaration remains in the defining module and the requesting module refers to
it through the preserved runtime import. All module declarations are
importable in 0.1. Every requested type must already be nameable through the
defining module's import closure; a requester-local type that would create a
reverse dependency is rejected in 0.1.

Expansion uses a deterministic worklist across the whole module graph.
Initial requests are sorted by stable source identity. Expanding one
specialization may add more requests; processing continues until no new
requests remain (a fixpoint).

A defining module's output and cache dependency include the sorted keys of
all specializations assigned to it, including those found through other
requests. A new request in an importing module therefore invalidates or
extends the defining module's output. The compiler must not reuse a cached
module that lacks the newly requested specialization.

## 10. Bootstrap raw generation

The bootstrap compiler-supplied `import2 compiler` module exposes only:

```huc
compiler::emit_huc(text);
compiler::emit_huc_module(text);
```

A call uses `@` when it enters compiler execution:

```huc
@compiler::emit_huc("let mod generated: i32;");
```

`emit_huc` writes raw HUC0 text at the insertion cursor inherited from its
call site. That cursor can represent a module declaration sequence, a type
member sequence, or a function/block statement sequence. Nested `fn2` helpers
inherit the caller's cursor, so helper location never changes where output is
inserted.

`emit_huc_module` appends raw HUC0 declarations to the call-site module's
generated declaration buffer. It cannot inject a statement into an arbitrary
block. Bootstrap HUC has no operation meaning “one lexical scope above.”

The HUC1 translator deliberately does not parse or semantically validate these
strings. It records their byte ranges and origins, writes them into the
per-module `.huc0` artifacts, and lets the public HUC0 translator diagnose
them. Raw text containing a numbered HUC1 construct is therefore rejected by
the HUC0 translator, not recursively staged. A generated-code diagnostic
composes:

1. the location in generated HUC0;
2. the raw-generation call site;
3. the phase helper call stack;
4. the specialization request chain.

There is no bootstrap `gen`, `gen_global`, `quote`, splice, `parse_huc`,
structural syntax object, or structural `emit` API. A compiler reflection and
structured-generation library may later provide environment, module,
declaration, type, function, statement, and expression handles, but that is a
separate language/library design. The bootstrap architecture must not invent
its API. Such handles will be ordinary qualified phase-2 types, never
`@Type` spellings, and should expose immutable snapshots and stable IDs rather
than mutable internal AST storage.

The inspection mechanism is the ordinary stage output:

```text
huc stage app/main.huc1 --out-dir build/huc0
```

Its printable modules are the exact input consumed by the next translator.

## 11. Concrete HIR and MIR

Successful HUC1 output contains no numbered construct, `@`, `import2`,
compiler-only declaration, family binder, specialization request, or raw
generation call. Local `let name: auto` remains legal HUC0; unresolved inferred
function parameters or results do not.

The concrete HIR is lowered to a control-flow MIR:

```cpp
typedef struct HucBasicBlock {
    HucMirStatement *statements;
    size_t statement_count;
    size_t statement_capacity;
    HucMirTerminator terminator;
} HucBasicBlock;
```

Representative MIR statements:

```text
StorageLive place
StorageDead place
Copy destination, source
Clone destination, source
Relocate destination, source
AddressOf destination_raw, source_inline
SlotAddress destination_usize, source_place
PointerFromAddress destination_raw, source_usize
ConstructInPlace destination, constructor, parameter_places
Drop place
Call result_destination_or_none, callee, parameter_places
```

Representative terminators:

```text
Goto
Branch
Switch
Return
Unreachable
```

MIR semantics are sequenced. The C emitter must introduce temporaries when
necessary to preserve HUC's left-to-right order.

`Relocate` is the complete destructive-relocation operation, not one half of
an operation that requires a later `Deactivate`. It transfers the whole
value's representation and makes the source and all its subobjects
inactive as part of that operation. It never calls source `drop`, performs
source component cleanup, or requires source fields to be cleared.

The source operand retains the HIR cleanup mode. A tracked root updates its
active-state metadata. A pointer source ends the pointee lifetime but does
not search for or alter cleanup metadata of aliased roots or enclosing
objects. This needs no ownership registry, hidden pointer metadata, or
per-element flags. The destination is active in either case. Backend
temporaries for the source address must not create a second owning value.

The two-place MIR operations require different source and destination storage.
Relocation assignment emits a static no-op or a runtime address-equality
branch before reaching `Relocate` when source and destination might alias.

`ConstructInPlace` makes an inactive final destination active only after its
constructor obligations succeed. `Call` consumes already bound parameter
places in declaration order; its result destination is the caller's final
return place, so a structure result is constructed there rather than returned
through an observable temporary. `Drop` makes its place inactive. These state transitions are MIR semantics, not optional annotations
for the backend.

## 12. Lifetime and drop elaboration

### 12.1 Place state

The pass tracks whether each compiler-managed root place of an
Advanced-classified type is:

```text
Inactive
Active
MaybeActive
```

This analysis arranges exactly-once destruction for tracked roots, provided
unchecked pointer operations respect their cleanup contracts. Pointer
extraction does not update root state through aliases; programmer-managed
storage uses its own initialized-range or equivalent bookkeeping. Attempting
cleanup of an inactive extracted slot is undefined behavior, not a request
to discover and suppress that cleanup at runtime.

The compiler must reject a use it knows is `Inactive`. A
`MaybeActive` use remains allowed in unchecked mode, but executing it on a
path where the value is inactive is undefined behavior. The compiler may
warn, or a strict lint mode may reject it.

### 12.2 Drop flags

- No flag is needed when static control flow proves a place active or inactive.
- `MaybeActive` values receive a hidden boolean only when necessary.
- `Relocate` from a tracked root marks that root inactive.
- `Relocate` through a pointer does not update aliased roots' drop flags;
  the programmer prevents cleanup of the now-inactive pointee slot.
- successful construction or reinitialization marks the destination active.

A conditional flag belongs to one MIR storage place. It is stored separately,
as a local or guard state, never as a field of the HUC type. It therefore does
not change `sizeof(T)`, field offsets, or foreign-interface layout.

### 12.3 Cleanup edges

The pass adds cleanup before:

- return;
- break and continue;
- normal scope exit;
- assignment over an active destination.

There are no exception cleanup edges in 0.1.

Explicit MIR cleanup is mandatory for the C backend:

- it implements HUC destruction order independently of C scope exit;
- it supports deterministic MIR dumps in compiler tests and debug builds;
- it handles HUC temporaries, abandoned initialized prefixes, and conditional
  liveness without implicit backend cleanup;
- it provides the same semantic input to future backends.

The emitter may share cleanup tails through labels and branches or emit
equivalent calls directly on each exit. Static analysis removes unnecessary
flags; remaining flags belong to storage places, not nominal value layouts.

### 12.4 Special-operation elaboration

Semantic analysis classifies a structure as Basic only when every field is Basic
and it declares neither `clone` nor `drop`. Declaring either method makes the
structure Advanced.

Elaboration handles the operations as follows:

- same-type Basic binding becomes fieldwise `Copy`;
- `copy` of an Advanced structure with the required protocol becomes `Clone` into
  the copy expression's result place; lacking `clone` is a diagnostic, and
  copy assignment specifically uses a fresh result before destination
  destruction;
- copying `T*` copies only the observer address and permits null;
- binding from an Advanced value becomes one fixed representation-transfer
  `Relocate`, which intrinsically makes the source and all subobjects inactive
  without requiring source-field clearing; tracked roots update their cleanup
  state, while pointer sources leave source cleanup to the programmer;
- a fresh unnamed Advanced result can be transferred directly;
  relocation from named root storage requires `mod` before its name;
- extraction through `*pointer` requires `mod T*`, not a writable pointer
  binding, and does not modify the pointer's address value;
- relocation assignment first proves distinctness or compares source and
  destination storage addresses at runtime when aliasing is possible; exact
  self-relocation is a no-op, while a distinct active destination is dropped
  before the fixed relocation;
- `destination = copy source` completes `Clone` before dropping destination;
- destruction calls user `drop`, then drops fields in reverse order.

`Clone` means an independent logical duplicate whose two active values can be
destroyed independently. A user `clone` must duplicate owned resources or
establish an explicit independent share, but it need not recursively duplicate
objects reached only through raw observers. This is a semantic contract the
unchecked compiler cannot prove.

Field fixedness affects source-level assignment, not this compiler operation.
Once an eligible source has the required writable storage permission,
`Relocate` transfers the whole representation, including its fixed fields.

The source of a successful relocation contains no active HUC value.
Its representation may retain old bits, but they are not a usable value.
Tracked source cleanup is suppressed; pointer extraction instead requires
the programmer to prevent later source cleanup. Reinitializing an inactive
place starts a new lifetime without first dropping it. Ordinary indirect
assignment requires a live destination; it does not construct into an
inactive slot or correct stale automatic cleanup state. This is destructive
relocation, not a C++-style move that leaves a destructible moved-from object.

No HIR or MIR operation performs overload resolution for relocation. A C
relocation helper emitted for backend convenience is compiler machinery, not
a HUC customization point. See
[Value Semantics and Special Operations](value-semantics.md).

The compiler does not repair self-pointers or other address-dependent
invariants during bitwise relocation. Such inline values must remain in
directly constructed fixed storage, use a suitable future stable-address
container, or use other stable-address storage. Violating that contract and
then relying on invalidated addresses is undefined behavior.

Representation transfer is the semantic definition of relocation,
not an optional optimization of recursive field operations. No separate proof
is needed to omit source clearing: every source subobject is inactive,
and reading it as a live value is invalid. The lowering must still preserve
layout, valid C storage access, source deactivation, and exactly-once cleanup.
It may use bulk copies, loads/stores, registers, or elision when observably
equivalent; the contract neither mandates a literal memory-copy instruction
nor gives padding bytes semantic significance.

## 13. C17 lowering

### 13.1 Runtime representation

Raw pointers lower to typed C pointers. Structures lower to concrete C
structures with the specified field layout. The emitted representation does
not determine whether HUC copies or transfers a value: MIR does.

Generate per-type initialization, clone, and drop functions where needed.
Cleanup of a structure invokes its `drop` body and then destroys its fields
in reverse order. Raw-pointer fields receive no pointee cleanup. Explicit
allocation or release of resources in user code remains part of that code's
library or external-call contract.

Generated helper parameters may use C pointer chains internally; this does
not add pointer-chain types to HUC.

### 13.2 Operation mapping

| HUC operation | Generated C17 |
|---|---|
| `T*` / `mod T*` | Typed C pointers with HUC access permissions checked before emission |
| `&value` | Address of the corresponding typed storage place |
| `slot_off(place)` | the slot's data-storage address converted to target `usize` |
| `ptr_as<T*>(address)` for `usize` | target-supported integer-to-data-pointer reconstruction |
| Advanced relocation | Transfer the whole representation without source clearing; update tracked root state, or leave indirect source cleanup to the programmer |
| `copy value` for an Advanced type | Call the resolved clone function into a fresh result place |
| Destruction | Call user `drop`, then reverse field cleanup, only for active values |
| `T(values)` initializing a final place | Generated initializer with an explicit final-destination pointer |

The backend must preserve the specified data-address round trip, using
`uintptr_t` or an equivalent target-supported unsigned representation. Targets
must support the specified pointer-width integer round trip. Taking a slot
address requires no allocation or runtime helper call. Taking a pointer slot's
address addresses that slot, not the pointee. Representation
access must respect HUC's byte-aliasing rules, not depend on an incompatible
C typed dereference. Raw writes do not automatically update HUC drop flags
or perform lifecycle operations. User code must preserve valid values and
cleanup obligations.

Methods, constructors, clone/drop helpers, and relocation helpers are ordinary
generated C functions. HUC overloads are resolved before emission and receive
distinct C names. Advanced-value replacement drops the active destination before
transfer, after checking exact self-relocation where aliasing is possible.

Allocation failure follows the chosen library API's contract. HUC 0.1 has
no exception unwinding. Foreign adapters must not unwind or otherwise bypass
generated cleanup across a HUC frame; a later C++ adapter must catch and translate or
terminate before an exception crosses that boundary.

### 13.3 Sequencing

Although C17 does not choose a left-to-right order for all function
arguments, HUC requires each argument expression *and its corresponding
by-value parameter initialization* to complete before evaluation of the next
argument begins. Retaining pointers to argument source places is insufficient:
it delays the Copy/Relocate operation that binds the actual parameter until
the final C call.

MIR therefore creates a typed parameter place for each by-value HUC parameter
and fills those places sequentially. A backend call frame is typed raw storage
plus MIR-managed active flags; constructing the frame itself has no HUC
effect:

```huc
call(first(), second());
```

becomes conceptually, using pseudocode rather than a required public runtime
API:

```c
call_frame_for_call huc_frame;

// Evaluate first(), then complete Copy/Relocate/direct construction of
// parameter 0 in its final typed holder.
evaluate_and_bind_parameter_0_in_place(&huc_frame.parameter_0);

// Only now evaluate second() and complete parameter 1.
evaluate_and_bind_parameter_1_in_place(&huc_frame.parameter_1);

call_impl(&huc_frame.parameter_0, &huc_frame.parameter_1);
```

Generated functions may accept pointers to these already-initialized
parameter slots instead of C by-value parameters. Passing those pointers
must not call a clone, relocation, constructor, or user hook in an order
chosen by C. The called function still sees ordinary HUC by-value parameters,
and MIR controls their cleanup. Generated wrappers may provide a conventional
ABI where needed, after all HUC parameter initialization has completed.

The holder's concrete type and initialization operation come from typed HIR:
Basic parameters use `Copy`, Advanced parameters use `Relocate`, observing
parameters use the resolved observer value, and direct constructor/function
results use their final destination place. The same call-frame lowering must
preserve temporary and parameter destruction order.

Function results with observable destination identity also use an explicit
result pointer supplied by the caller. Initializers write directly into that final
storage; ordinary C structure returns must not introduce an observable
intermediate address. Scalar results and other cases proven equivalent may
use ordinary C returns. This preserves HUC construction guarantees without
depending on backend copy elision.

### 13.4 Plain storage and explicit lifecycle functions

Each concrete HUC structure becomes a C structure containing its physical
fields. HUC Basic/Advanced classification does not follow the C representation:
an owning HUC value can have a C structure representation that C would freely
copy. Only HUC's resolved operations authorize copying or relocation.

For each HUC type, generate the required concrete functions or equivalent
inline code for:

- initialization in supplied final storage;
- whole-representation relocation of Advanced values without a user
  relocation hook or required source clearing;
- user `drop` followed by reverse field cleanup;
- explicit logical cloning when supported;
- explicit library calls for resource allocation and deallocation.

MIR emits automatic drop calls according to tracked root state. A directly
relocated source may retain old handle bits but receives no HUC drop call.
Pointer extraction does not update cleanup state through aliases: user code
must prevent cleanup of the inactive slot, such as by reducing a container's
initialized length. Automatic cleanup reaching that slot is undefined
behavior. No automatic C cleanup must be neutralized.

When static analysis cannot decide whether cleanup is needed, a separate
boolean tracks whether the value is active. It is never a field of the C
value type.
HUC temporaries follow the same MIR rules. Cleanup calls or shared cleanup
labels cover every normal exit and active destination replacement.

HUC fixed-by-default permissions are enforced before emission. Do not add C
`const` to physical storage when doing so would prevent permitted compiler
relocation or reinitialization. Read-only access remains a HUC semantic
property even when the underlying C storage is physically writable.

C structure assignment or `memcpy` may implement relocation when it
preserves the representation-transfer semantics and activation state. Do not
clear fields merely because an aggregate manages resources.
Observe C effective-type, alignment, and aliasing rules; an arbitrarily cast
byte buffer is not automatically valid typed storage.
Prefer concrete typed storage and explicitly aligned allocation. HUC
inactivity is compiler bookkeeping and need not force the physical C storage
object's lifetime to end at the same point.

### 13.5 Names and modules

Each HUC module maps to a unique generated identifier prefix and normally
produces:

```text
build/c/huc_runtime.h
build/c/app/main.huc.h
build/c/app/main.huc.c
build/c/app/main.huc.map
build/c/dependency/module.huc.h
```

Every HUC module produces a guarded header. Runtime imports become generated
header includes. The root `.c` includes the root header and is the only
translation unit compiled by the first whole-program backend. This avoids
pretending that the initial generated ABI already supports independent object
compilation.

Names are mangled deterministically to distinguish modules, overloads,
methods, and generated helpers. Headers include required forward declarations
and concrete type definitions before function bodies that need complete types.
No C namespace or overload facility is assumed.

The root translation unit also defines the hosted C `int main(void)` adapter.
It calls the configured root module's HUC `fn main() -> i32` and returns that
value through the documented target mapping. Operating-system arguments remain
behind the deferred standard-library argument API.

### 13.6 Diagnostics and source mapping

The HUC frontend should catch all language errors. Backend diagnostics are
treated as compiler bugs or foreign-code diagnostics.

Generated files contain `#line` directives where they improve messages, but a
sidecar `.huc.map` records richer expansion chains. The driver rewrites backend
diagnostics to HUC locations and includes the raw backend message under
`--verbose-backend`.

### 13.7 Backend fence

Generated code must use:

- collision-free generated identifiers outside C's implementation-reserved
  namespace;
- concrete types and resolved function symbols, with no unresolved families or
  overloads;
- explicit lifecycle functions and cleanup edges;
- explicit temporaries for HUC sequencing;
- inert pointer transport for already bound internal parameter and result
  places where their identities matter;
- explicit casts for HUC numeric conversions;
- typed storage and target-validated alignment, aliasing, and integer/pointer
  representations;
- ISO C17 facilities, with extensions considered separately only if required.

These rules stop C from silently becoming the specification. C integer
promotions, floating-point transformations, and expression sequencing must not
change defined HUC behavior.

## 14. Additional backends and C++ interoperability later

A later C++ or native-code emitter consumes the same concrete, sequenced MIR,
including final destinations, relocation, active state, and cleanup. No phase
evaluator or semantic pass should need replacement. If another emitter must
reinterpret HUC ownership, the MIR is too weak.

C++ interoperability is independent of the first output language. A C backend
can call C-linkage adapters implemented in C++; a future C++ backend or binding
layer may support more direct integration. Either path needs explicit rules
for foreign types, allocation/deallocation, exceptions, and object lifetimes.
Neither a C++ output backend nor arbitrary C++ header import is a bootstrap
requirement.

## 15. Driver and build pipeline

### 15.1 Commands

```text
huc lower <root.huc0> --out-dir <c-dir>
huc stage <root.huc1> --out-dir <huc0-dir>
huc build <root.huc1> --out-dir <build-dir>
```

`lower` is the public, independently usable HUC0-to-C17 translation and is
implemented first. `stage` is the public HUC1-to-printable-HUC0 translation
and is implemented second. `build` runs `stage`, closes and writes every
generated HUC0 module, and then invokes `lower` on those files. It must not
pass an HUC1 AST, HIR, or private in-memory module directly into HUC0 semantic
analysis.

HIR/MIR dumps, backend invocation, and run conveniences may be added as debug
or later commands, but the bootstrap public contract is the three commands
above.

### 15.2 Backend discovery

The driver searches, in order:

1. `--cc <path>`;
2. a project configuration;
3. `clang`;
4. `gcc`.

It invokes the generated-code compiler in C17 mode (`-std=c17` for Clang/GCC),
in a deterministic argument order, and records:

- executable identity/version;
- target triple;
- compile flags;
- HUC runtime header hash.

This selects the compiler for generated C, not the C++20 compiler used to
build the HUC transpiler and driver.

### 15.3 Build artifacts

```text
build/
  huc0/          printable staged module tree
  c/             generated C17 headers, root source, runtime header, maps
  cache/         content-addressed frontend/staging cache
  obj/           backend object files
  compile_commands.json
```

Generated HUC0 and C are inspectable build artifacts. Normal builds may
remove them only through an explicit build-system operation.

## 16. Caching and separate compilation

### 16.1 Content addressing

Cache entries are keyed by cryptographic hashes of canonical serialized data,
not timestamps.

Initial cacheable units:

- HUC1 indexed module and selected-body parse;
- pure HUC1 family specialization and pure `fn2` result;
- emitted per-module HUC0 text plus its origin map;
- HUC0 lexed/parsed module and runtime interface;
- concrete HUC0 HIR/MIR;
- generated C17 header, root source, and object.

Even on a cache hit, `build` materializes the HUC0 module files before invoking
the HUC0 translator. A cached private AST is not a substitute for the textual
boundary.

The phase checker classifies transitive compiler execution as pure or
effectful. A normal family specialization is constructed once per key; its
cache entry includes its complete residual declaration and ordered raw
fragments. Local/type fragments enter the declaration; module fragments enter
the defining module buffer once. Repeated normal requests only reference that
one result.

An actual `fn2` call, forced `@fn1` call, or standalone phase expression that
reaches `emit_huc` or `emit_huc_module`, directly or through helpers, runs each
time it is evaluated. It uses the current inherited insertion cursor; a
cached return value alone cannot replace that execution. The compiler may
still cache selected-body parsing and pure computations within the call.

A future cache for these calls would need to record their output and replay
it in order at the current cursor. That feature is outside the bootstrap.

### 16.2 Interface files

A future HUC0 interface format may contain:

- runtime signatures;
- type layouts where ABI-visible;
- definition and dependency fingerprints.

HUC1 family caches and future compiler-reflection metadata have separate
formats and must not be passed to HUC0 semantic analysis. Until interface
serialization is stable, lower whole HUC0 programs without promising a stable
generated ABI.

### 16.3 Reproducibility

Canonical serialization sorts maps by stable IDs, normalizes paths relative to
the project root where possible, and excludes allocator addresses. Generated
symbol hashes must be identical across processes for identical inputs.

## 17. Testing strategy

### 17.1 Unit tests

- tokenization, including `&`, `<`, `>`, `@`, `in`, and numbered keywords;
- valid and invalid numeric separator positions in all supported bases;
- name-first declarations, independent binding/type `mod`, and `let1` headers;
- optional trailing commas in nonempty single/multiline lists, and rejection
  of a comma in an empty list;
- required control-flow braces, `else if`, and `for1 (... in ...)`;
- rejection of old `T&`, type-first declarations, and colon-only `for1` headers;
- Pratt precedence, ambiguous angle postfixes, and recovery;
- prefix `&` versus binary `&` and `&&`, including postfix field/index binding;
- non-composable pointer-suffix and forbidden pointer-slot address diagnostics;
- typed `&` addresses and `slot_off` integer addresses, each evaluating its
  place once without transferring its value;
- slot addresses of fixed, inline, and raw-pointer slots;
- invalid non-place slot operands and integer-to-pointer type checking;
- canonical type interning;
- raw-pointer copying, including null, with no source changes or pointee cleanup;
- raw-pointer inference for `auto` and permission-dropping conversions;
- rejection of incompatible pointer types and added pointee `mod`;

- unqualified intrinsic resolution, with no former-spelling compatibility aliases;
- overload ranking;
- C-linkage signature acceptance and rejection;
- phase availability;
- specialization keys;
- evaluator arithmetic and resource limits;
- deferred-body delimiter scanning;
- raw-generation cursor inheritance;
- tracked-root versus pointer-extraction cleanup modes;
- rejection of direct field/array-element and read-only pointer sources;
- drop-state analysis and definite-inactive diagnostics.

### 17.2 Golden tests

Store input plus expected:

- diagnostics;
- printable per-module HUC0;
- concrete HIR;
- MIR;
- generated C17.

Golden C tests require deterministic temporary names rather than masking
nondeterministic numbering.

Relocation goldens must show whole-value `Relocate`, including for nested
Advanced aggregates, with no recursive source-field transfers or null stores.
Replacement goldens must check exact self-relocation before destination
cleanup. Address-taking goldens distinguish `&inline_place` from
`slot_off(place)`; only the latter returns an integer and accepts pointer slots.
Pointer-extraction goldens must evaluate the source address once, create an
active destination, and emit no source clearing, alias-driven flag updates,
or runtime ownership lookup. Inlining must preserve the source cleanup mode.

### 17.3 Execute tests

Compile and run small programs that record:

- argument evaluation and by-value parameter-binding order;
- direct-construction `this` addresses for locals, fields, parameters, and results;
- destructive-relocation and clone counts;
- copying fixed and writable raw pointers without modifying the source;
- relocation of nested Advanced aggregates with cleanup only through the
  active destination;
- extraction from manually managed slots followed by initialized-range
  updates, including a pop-style return with exactly one destination cleanup;
- extraction through a fixed `mod T*` binding without changing its address;
- Basic pointee copying and explicit pointee cloning leaving sources active;

- exact self-relocation behavior;
- destruction order;
- early-return cleanup;
- conditional relocation;
- `if1` branch removal;
- `for1` unrolling;
- forced `@` evaluation;
- raw generation at local, type-body, and module insertion points;
- HUC0 rejection of numbered syntax emitted as raw text;
- acceptance of every successful staged module by the standalone HUC0
  translator.

Tests verify relocated values through the active destination, not by reading
inactive source fields or requiring their old bytes to remain intact.

### 17.4 ABI and cost assertions

Use generated C17 static assertions to compare sizes, alignments, and field
offsets with HUC's target layout calculations. Active-state flags must not
change a nominal value's layout.

Inspect optimized assembly for representative Basic copies, Advanced
transfers, observations, and cleanup paths. Assembly shape is a regression
aid rather than a language guarantee; a transfer may be optimized away.

### 17.5 Differential backend tests

Initially compile generated C17 with both Clang and GCC, at unoptimized and
optimized settings, and compare defined observable results. Use strict C17
diagnostics to detect accidental reliance on extensions. When another HUC
backend exists, run the same MIR through both emitters as well. No backend
becomes the oracle for undefined HUC behavior.

## 18. Implementation milestones

### Milestone 1: HUC0 to C17

Implement in this order:

1. common C++20 project/source infrastructure, diagnostics, target data, and
   acyclic module discovery;
2. the standalone HUC0 lexer and recursive-descent/Pratt parser;
3. names, concrete types, permissions, constructors, overloads, Basic/Advanced
   classification, type aliases, fixed-signature C linkage, and lifecycle
   checking;
4. typed HIR and sequenced MIR with explicit parameter places,
   direct-construction destinations, relocation, and cleanup;
5. scalar C17 headers plus one root translation unit;
6. explicit bitwise relocation and clone/drop
   functions, cleanup edges, sidecar active-state handling, and lifecycle
   hardening.

Milestone 1 is complete only when user-written `.huc0` programs compile and
run through generated C17, lifecycle and left-to-right tests pass, and no
HUC1 facility is required by `huc0_transpile`.

### Milestone 2: HUC1 to HUC0

Implement in this order:

1. staged module indexing, `import2`, opaque deferred-body spans, phase-free
   pass-through, per-module HUC0 buffers, and origin maps;
2. the deterministic phase-2 interpreter with resource and effect limits;
3. `if1`, finite `for1`, bounded `while1`, selected-body parsing, and
   module/type/function/block insertion contexts;
4. `struct1`, `fn1`, and `let1` primary/partial/full family selection with
   angle patterns, deduction, predicates, and recursion states;
5. `@`, forced evaluation, `compiler::emit_huc`,
   `compiler::emit_huc_module`, cursor inheritance, and composed diagnostics.

The HUC1 frontend loads a versioned, inspectable generated interface for the
compiler module before indexing user modules. This is the implementation
equivalent of an always-included header; `import2 compiler` explicitly binds
it. The bootstrap interface contains only the raw-emission functions and never
exposes pointers to internal compiler objects.

Milestone 2 is complete only when every numbered/compiler-only construct is
removed, output modules are deterministic and readable, and the public
Milestone 1 translator accepts them without a compatibility mode.

Compiler reflection and structured generation are intentionally not a third
bootstrap milestone. They require a separate design after these two
translators exist.

## 19. First implementation subset

The first vertical slice belongs entirely to the independently usable HUC0
translator:

```text
module/import
fn
struct
bool, integer, float, void
local declarations
if/while/return
ordinary calls
concrete HIR/MIR
C17 root translation unit
```

Then complete HUC0 with:

```text
T* and independent mod layers
constructors and direct destination construction
Basic/Advanced classification
destructive relocation, clone, and drop
left-to-right typed parameter binding
```

Only after the HUC0 completion gate should the HUC1 staging skeleton begin.
This prevents staging from concealing unresolved runtime semantics.

## 20. Architectural invariants

The following should be enforced in code review and tests:

1. No HUC phase rule is delegated to the downstream C compiler or preprocessor.
2. No ownership decision is inferred from the generated C representation.
3. HIR records Copy, Relocate, Observe, and Clone explicitly.
4. MIR is fully concrete and contains no phase construct.
5. Compile-time evaluation has no ambient untracked host effects.
6. HUC1 produces written per-module HUC0; HUC0 never consumes a private HUC1
   AST or HIR.
7. Bootstrap raw generation is source concatenation only at defined cursors;
   no structural-generation API is implied.
8. Backend output preserves HUC left-to-right argument evaluation and
   by-value parameter initialization.
9. Source inactivity and cleanup state do not add fields to nominal value types.
10. Diagnostics retain source and expansion chains across both translators.
11. A future C++ or native-code backend can consume MIR without recreating
    semantic analysis.
