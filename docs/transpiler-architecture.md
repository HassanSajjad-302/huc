# HUC Compiler Architecture

Status: implementation design; thin end-to-end start recommended;
release 0.1 subset not yet fixed

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
- **Program mode**: prepare code for the HUC0 program.
- **Evaluation mode**: prepare a private function body for the compiler to run.
- **Generation site**: the position of a `#` statement in the code being prepared.
- **Insertion point**: the forward-only destination selected for generated code.
- **Module insertion anchor**: the original requesting top-level declaration;
  its completion releases pending module output, including nested requests.
- **Module**: a namespace, not a separate compilation unit.
- **Translation unit**: a root source and its once-processed import closure.
- **Origin**: the `#` site and the call chain that built a generated span.

## 1. Recommendation

Both HUC translators and their command-line driver will be implemented in
C++20. HUC0 lowers directly to LLVM IR:

```text
HUC1 source
    |
    | huc1_expand
    v
printable flattened HUC0 source
    |
    | huc0_transpile
    v
LLVM IR -> LLVM optimization/code generation -> object files -> linker
```

The translators remain independently usable. The recommended first milestone
is a thin end-to-end slice through both translators and LLVM. Staging first
remains an alternative, but ordinary-function evaluation still needs the shared
frontend, HIR, and MIR for its supported features. It postpones LLVM emission,
not that semantic work. Handwritten HUC0 does not require HUC1, and staged HUC0
is always passed through the public runtime frontend rather than a private AST
handoff.

The specification aims to describe the language as completely as possible;
release 0.1 may implement only a documented subset. The feature and test lists
below describe the target architecture, not an all-or-nothing release gate.
Unsupported features must be diagnosed explicitly. Full reflection and other
unfinished designs can be specified independently of implementation order.

LLVM-first replaces the earlier C17 emitter plan. HUC still owns parsing,
type checking, staging, specialization, lifetime lowering, and its HIR/MIR.
LLVM handles optimization and machine-code generation after those decisions.

The first compiler must emit cleanup for every normal scope exit, early
return, loop exit, replacement, and abandoned initialized prefix, with flags
where a value is active on some paths but not others. Building this cleanup
into MIR is required whenever the implemented subset supports those operations.
The LLVM backend must not reconstruct ownership from raw IR storage.

## 2. Responsibilities at the LLVM boundary

| Concern | HUC responsibility |
|---|---|
| Source rules | Name/type checking, permissions, specialization, and phase execution |
| Value semantics | Final construction destinations, relocation, active state, and explicit cleanup |
| Runtime representation | Concrete layouts, storage, symbols, and target-correct calls |
| LLVM lowering | IR construction, verification, optimization settings, and object emission |
| Foreign integration | Explicit C ABI lowering and ownership contracts; broader bindings separately |
| Toolchain | Pinned LLVM version, target configuration, runtime support, and linking |

Using LLVM is not a substitute for HUC's semantic passes. The C calling
convention also requires target-specific signature lowering; selecting an
LLVM calling-convention name alone does not classify foreign aggregate
arguments or establish all scalar parameter/result attributes. The currently
supported scalar/pointer C boundary stays explicit.
Clang/header integration and C++ interoperation remain separate designs.

No C or C++ source backend is required. Alternative emitters can later consume
the same MIR, but neither their implementation nor full LLVM target coverage
is a prerequisite for release 0.1.

## 3. Compiler overview

```text
.huc1 root + once-processed imports (one translation unit)
    |
    v
HUC1 source manager + delimiter scanner + staged index
    |
    v
selected-body parser + phase checker + C++20 interpreter
    |
    v
family specialization + parsed text insertion + residualizer
    |
    v
one flattened .huc0 file with namespace sections + origin map

-------------------- explicit textual boundary --------------------

.huc0 translation unit (handwritten imports or flattened sections)
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
LLVM IR generation + verification + origin/debug information
    |
    v
LLVM optimization + target code generation -> object files -> linker
```

`huc0_transpile` accepts source files, not an HUC1 AST or an undocumented
in-memory residual IR. `huc build` must first write the HUC0 artifacts and then
pass those paths to `huc0_transpile`. This boundary is both a correctness test
and the reason generated HUC0 remains a useful inspection and interchange
artifact.

Neither translator asks LLVM to implement HUC families,
compile-time evaluation, overload selection, or phase semantics. All such work
is complete before LLVM IR is emitted.

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
  support/
tests/
  unit/
  golden/
  negative/
  execute/
```

The implementation language is C++20. Standard containers, RAII, value types,
and explicit ownership are preferred. Arenas and stable ID-indexed tables
remain useful where they simplify AST/IR lifetime and deterministic output. A
LLVM integration stays behind a narrow backend interface, with a pinned
supported LLVM version and explicit target configuration.

The logical libraries are:

- `huc_common`: sources, diagnostics, target configuration, module graphs,
  arenas, IDs, and shared delimiter/parser support;
- `huc0`: the independently usable HUC0 frontend, HIR/MIR passes, ownership
  lowering, and LLVM IR generation, verification, and object emission;
- `huc1`: staged indexing, deferred-body handling, the phase interpreter,
  specialization, and printable HUC0 residualization;
- `huc`: a thin C++20 command-line driver.

Runtime support is linked into user programs only when needed. It is separate
from the compiler executables and from LLVM's implementation libraries; no
generated C runtime header is required by the backend.

### 4.1 C++20 implementation conventions

Compiler modules expose typed C++ interfaces:

```cpp
using FileId = StrongId<struct FileTag>;
using ModuleId = StrongId<struct ModuleTag>;
using TranslationUnitId = StrongId<struct TranslationUnitTag>;

struct ParseResult {
    std::optional<TranslationUnitId> translation_unit;
    std::vector<Diagnostic> diagnostics;
};

class Compiler {
public:
    [[nodiscard]] ParseResult parse_translation_unit(FileId root_file);
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
    std::optional<Artifact> source;
    std::vector<Artifact> sidecars;
    std::vector<Diagnostic> diagnostics;
    DependencyGraph dependencies;
    bool succeeded = false;
};

[[nodiscard]] Huc1ExpandResult
huc1_expand(const Huc1ExpandRequest& request);

} // namespace huc
```

Ordinary source errors are returned as diagnostics rather than thrown. A
chained build passes the written `Huc1ExpandResult::source` through a normal
`Huc0TranspileRequest`. `ModuleId` identifies a canonical namespace, not an
independently emitted file. Sidecars hold origins and tracked dependencies.

## 5. Frontend

### 5.1 Source manager

`SourceManager` owns immutable source buffers and assigns each one a `FileId`.
Every token carries a compact `SourceSpan`. Its range includes
the start position but excludes the end position (a half-open range):

```cpp
typedef struct HucSourceSpan {
    HucFileId file;
    uint32_t begin;
    uint32_t end;
} HucSourceSpan;
```

Syntax nodes retain a primary span plus additional spans when needed. A loop
with a generated header and an original body cannot be described as one
contiguous range in one buffer. Parsing uses logical input order, not a numeric
comparison of offsets from different files. Generated text gets a new immutable
buffer; no insertion changes earlier source offsets.

Line and column numbers are computed lazily from per-file newline indexes.
Generated nodes additionally carry an origin chain:

```text
generated HUC0 range
  -> original/generated source spans and generation site
  -> text-emission operation
  -> phase helper call stack
  -> specialization request chain
  -> original source declaration
```

This is required for useful staging diagnostics.

### 5.2 Lexer

The lexer should:

- recognize numbered keywords as dedicated tokens;
- treat `in` as an ordinary identifier;
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
    HUC_TOKEN_KW_STRUCT,
    HUC_TOKEN_KW_STRUCT1,
    HUC_TOKEN_KW_IF,
    HUC_TOKEN_KW_IF1,
    HUC_TOKEN_KW_FOR,
    HUC_TOKEN_KW_FOR1,
    HUC_TOKEN_KW_WHILE,
    HUC_TOKEN_KW_WHILE1,
    HUC_TOKEN_KW_LET,
    HUC_TOKEN_KW_LET1,
    HUC_TOKEN_KW_IMPORT,
    HUC_TOKEN_KW_IMPORT1,

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
one pointer-like form: a repeatable `T*` raw-pointer suffix. The type parser
accepts pointer chains directly; canonical-type validation preserves each
level of indirection through aliases.

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
`let` and `let1` precede variable declarations but not parameters. There is no
expression-parameter marker. Compiler calls classify their arguments into
value or expression bindings; record this on each call, not the declaration's
AST/signature and not by guessing from body uses of `compiler::expr`. Phase binders
also use `name: Type` or `name: auto`; they have no binding `mod`. All phase
families, including variable families, put their name before an angle binder list. Both
`for` and `for1` read three optional clauses separated by two semicolons.
The initializer is one binding or expression; reuse the local-binding parser
without its statement semicolon. Ordinary `for` accepts `let`, including
compiler-executed ordinary loops; `for1` accepts `let1`. The condition is one
expression and the update clause is an ordered list of expressions. Keep that
list as a loop AST field, not a comma-expression node. A header binding has a
loop scope with a nested body scope. `for1` requires compiler-only clauses and
retains a deferred body. There is no range-based loop production.

Use a shared list parser that accepts one optional trailing comma, including
for constructor initializer entries and loop updates. Do not make acceptance
depend on whether the list spans several lines. Empty items remain errors. Ordinary control
flow parses braced bodies, with an `else if` alternative for chained
conditionals; phase-control bodies still use the deferred-body scanner.
Do not retain the old declaration order or `T&` as compatibility spellings.

`construct_at(pointer, value)` and `destruct_at(pointer)`
use ordinary call syntax with compiler-known, non-overloadable, unqualified
names. Resolve them as intrinsics, not `std::` library calls. `storage_ptr`
is another such ordinary call. `as`, `ptr_as`, `size_of`, `align_of`, and
`is_basic` use the HUC0 type-operand call production. `raw_storage<T, N>` is a
built-in type primary and may be followed by `()` for empty-storage creation;
its HUC0 count must be an integer literal. Address-taking uses unary `&`.

Important recovery points are:

- `;`, `}`, and the next declaration keyword;
- the close of angle binder/pattern/request lists and parenthesized runtime
  parameter lists independently;
- `else` following `if` or `if1`;
- `)` following a parameter or condition.

The parser preserves an ambiguous angle postfix when syntax alone cannot
distinguish `name<arguments>` from comparison operators. Semantic name
resolution later decides whether the name denotes a generic family. This is
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
    bool has_generic_header; /* distinguishes fn f<>() from fn f() */
    HucParamDeclId *family_binders;
    size_t family_binder_count;
    HucAnglePatternId specialization_pattern; /* zero on a primary */
    HucParamDeclId *runtime_params;
    size_t runtime_param_count;
    HucTypeSyntaxId return_type;
    HucBodySyntax body;
    HucSourceSpan span;
} HucFunctionDecl;
```

`HUC_PHASE_RUNTIME` denotes a concrete runtime declaration.
`HUC_PHASE_STAGED` denotes a generic family or structural phase operation
that is prepared on request. `HUC_PHASE_META` denotes compiler-only storage and
compiler API entities, not a separate user function/type keyword. Determine
runtime eligibility from the requested signature, fields, and retained body.
These internal AST categories are not keyword suffix numbers. Record generic-header presence independently of binder count
so a zero-binder `fn f<>()` remains distinguishable from ordinary `fn f()`.

Use stable arena-allocated IDs rather than pointers as persistent cross-pass
identities. C pointers may be used temporarily within a pass, but definition records and
diagnostics should refer to IDs.

### 5.5 HUC1 indexing and deferred bodies

The HUC1 frontend indexes imports, declaration/family headers,
specialization patterns, and opaque body spans. A dedicated delimiter scanner
finds an outer matching `}` while recognizing nested braces, strings,
characters, line comments, and block comments. It does not produce HUC tokens
for the body.

Only a selected `if1` branch, a reached phase-loop body, a chosen family
body is then lexed and parsed. Register angle-header `fn` and `struct`
headers without preparing their bodies, including zero-binder primaries. Ordinary `fn` bodies are parsed and
prepared in program mode when encountered, including uncalled functions. A
skipped body may contain invalid HUC; the scanner may report only that it
cannot find the outer closing delimiter.
Ordinary HUC0 code selected for output is kept by the translation-unit residualizer.
The HUC0 translator checks it again after it has been written to a file.

Generic structures, functions, and variables use distinct indexed declaration nodes even
though they share binder, pattern, and deferred-body components.
The syntax tree also has a phase-expression item node for standalone
`@expression;` and an insertion item node for `#expression;`, both at module,
type-body, function, or block scope. A phase-expression item must evaluate to
compiler `void` or `never` and inserts nothing. An insertion item's expression must
produce `compiler::Code`; preparation erases the node, inserts the code's local
part at that position, queues its module part, and processes inserted text
before the following source. When the code has a hole, the following braced
source block fills it. Do not check that block's loop context prematurely.
Reject numbered syntax, `@`, and `#` in inserted HUC0; insertion does not start
another staging pass.

Ordinary `import` exposes runtime declarations and generic families. Resolve
and process a source at its first import, depth-first, then restore the
importer's lexical context. Repeated imports bind aliases but do not replay
source or compiler effects; active import cycles diagnose. `import1` is
compiler-only. These are the two import forms. Flattened output resolves away
imports and uses canonical references and `module name;` namespace sections.
Keep imported buffers and import chains rather than copying bytes or losing
their original locations.

## 6. Semantic analysis

HUC0 performs the following runtime semantic analysis. HUC1 reuses these rules
for prepared source and parsed generated text:

1. collect declarations and module imports;
2. resolve names;
3. resolve concrete types and layouts;
4. enforce permissions, constructors, and special-operation rules;
5. build concrete typed HIR;
6. make execution order, relocation, and cleanup explicit in MIR.

The module checker admits runtime globals only for drop-free Basic scalars and
raw pointers. It requires constant initializers for fixed globals, supplies
zero/null only to omitted mutable initializers, and rejects structures,
Advanced values, runtime initialization, and global cleanup in the current design.

HUC1 additionally performs phase checking, family matching, and selected-body
preparation before printing HUC0. Queue forward module output after its original
requesting top-level declaration and collect relevant declarations before final
body resolution. Defer unresolved forward references until collection is complete;
compiler evaluation needs an available checked definition. Diagnose a missing
definition or a dependency cycle rather than inventing one from suppressed
evaluation-mode output. An active anchor's pending declarations remain
unavailable through every nested compiler call, regardless of which function
queued them. A completed nested preparation does not publish them early.
Do not silently revise a finalized lookup or compiler
decision after an insertion; diagnose conflicting insertion.

The HUC0 translator independently reparses and checks the emitted text exactly
as it checks handwritten HUC0. No generic HIR or specialization record crosses
the translator boundary.

A selected deferred body is parsed according to its enclosing module, type,
function, or block. Module/type bodies may mix generated runtime declarations
or members with compiler-only statements. Those statements must use values
available at compile time; they execute during preparation and disappear from
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
inferred from backend type spellings or LLVM pointer types.

Canonicalization preserves raw-pointer nodes at every indirection level.
Unary `&` accepts an addressable place, preserves its mutation permissions,
and lowers to an address calculation. Evaluate the place once; do not
materialize a temporary, transfer its value, extend its lifetime, or update
cleanup state. Reject non-place operands, including function symbols.

`ptr_as<Target>` accepts exactly one value operand and permits raw-pointer to
raw-pointer, raw-pointer to `usize`, and `usize` to raw-pointer conversions.
Reject other pairs. Integer conversions preserve the target's unchanged-address
round trip. Evaluate the operand once and never invoke pointee relocation or
cleanup. The integer carries no permission or liveness metadata; the caller
must preserve access rights and lifetime preconditions. Escape and alias
analysis must account for exposed and reconstructed addresses.

Address-taking is still subject to the named-root activation check, including
inside `ptr_as<usize>(&place)`. Do not let a cast bypass that check. A pointer
retained before a direct transfer may address storage for manual reconstruction,
but using it does not reactivate the original name or register cleanup.

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
- a canonical identity for type comparison and specialization arguments.

`construct_at` requires exactly two operands and `destruct_at` one. Infer a
complete runtime object type `T` from `mod T*`; reject read-only/non-pointer
operands, invalid pointee types, and first-class or overload declarations for
these intrinsics. Pointer-slot pointees are allowed. Neither operation
has a HUC0 angle-delimited type operand.

Type-check the second operand of `construct_at` as an initializer for `T`,
not as an ordinary call parameter. Apply normal built-in conversions,
Basic/Advanced classification, source permissions, and relocation-state
checks. Its result is `mod T*`; `destruct_at` returns `void`. Storage lifetime,
capacity, alignment, and active/inactive preconditions are unchecked.

Intern `raw_storage<T, N>` by canonical element type and count. Require a
complete runtime object type, positive count, and representable target layout.
Its size is `N * size_of<T>()` and alignment is `align_of<T>()`. It is always
Advanced, without `clone` or implicit element drop. Empty-storage creation
establishes the wrapper without initializing its payload. `storage_ptr` accepts
one pointer to that wrapper and produces the element pointer with matching
access permissions; it does not load an element or register cleanup.

Fold `size_of<T>()` and `align_of<T>()` to target `usize`, and `is_basic<T>()`
to `bool`, with no runtime operands. HUC1 evaluation and HUC0 lowering must
use the same target layout and classification rules; retain the target layout
inputs in the translation context. HUC1 resolves a storage count and prints
an integer literal in residual HUC0. Runtime counts are not valid type operands.

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
- a type with compiler-only fields stored in runtime memory;
- an expression-bound argument read or converted to a value, or
  `compiler::expr` used on a value-bound parameter;
- a reached operation that cannot run in the compiler (specification section
  12), such as a foreign call or runtime-storage access. This is checked on the
  executed path, not from the callee's signature;
- a compiler-only signature or executable operation surviving a runtime request;
- an `@` in a function body that reads the body's parameters or `let` locals;
- a standalone `@` statement whose type is `compiler::Code` (use `#`);
- reserved future `@this` contextual reflection in bootstrap HUC1.

Expression bindings allow runtime syntax inside compiler calls: resolve the
argument type without loading its value. This is a per-call binding category,
not a conversion from runtime memory to compiler memory. Preserve the source
context for initial typing and diagnostics;
later text insertion performs ordinary lookup again.

The diagnostic should name both sides:

```text
error: compile-time condition requires a compile-time bool
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

Generic specialization uses argument count and structural-pattern matching,
not executable conditions. A body-level `if1` runs only after selection.
Errors in the chosen body are diagnostics, never a reason to try another
candidate.

### 6.5 C-linkage validation

An `extern "C"` signature is accepted only when every parameter and result
lowers through the target's documented C ABI mapping. HUC0 permits built-in
arithmetic scalars, `void` results, and raw pointers, including pointer chains,
to built-in types. It rejects structures, fixed arrays, Advanced/drop types, phase
types, compiler metadata, opaque foreign handles, and
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
InitializeAt(pointer, initializer)
DestructAt(pointer)
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
their backing storage and initialized element ranges explicitly.
`InitializeAt` represents `construct_at`: evaluate the pointer once, then
initialize its slot from the second operand using `Copy`, `Clone`, `Relocate`,
or destination construction. Return the retained pointer. Do not create a
by-value initializer parameter or drop an old destination. `DestructAt`
performs complete destruction at a retained pointer, without deallocation.
Both leave automatic cleanup state of aliased roots unchanged. A directly
tracked source consumed by `Relocate` still updates its own state normally.

## 8. Compile-time evaluator

### 8.1 Initial strategy

Interpret typed HUC1 phase code and ordinary-function bodies in-process. Lower
ordinary bodies to the same sequenced MIR the LLVM backend uses (section 11),
extended with compiler-only values, so evaluation order, relocation, and
cleanup cannot drift from generated code. Do not:

- generate and compile a temporary C++ program for every compiler call;
- execute arbitrary host C++ plugins;
- represent metadata as source strings;
- use host addresses as stable identities.

An interpreter is slower than native execution, but makes it much easier to
provide repeatable results, inspect execution, and report errors
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
    HUC_CT_POINTER,
    HUC_CT_TYPE,
    HUC_CT_SYMBOLIC_EXPR,
    HUC_CT_EXPRESSION_ARGUMENT,
    HUC_CT_SEMANTIC_ID,
} HucCtValueKind;

typedef struct HucCtPointer {
    HucCtBlockId block; /* zero for null */
    HucCtPathId path;   /* slot index and subobject path within the block */
} HucCtPointer;

typedef struct HucCtValue {
    HucCtValueKind kind;
    union {
        bool boolean;
        HucCtInteger integer;
        HucCtFloat floating;
        HucCtStringId string_id;
        HucCtObjectId object_id; /* list or structure in the CT heap */
        HucCtPointer pointer;
        HucTypeId type_id;
        HucSymbolicExprId symbolic_expr_id;
        HucExpressionArgumentId expression_argument_id;
        HucSemanticId semantic_id; /* internal stable identity */
    } as;
} HucCtValue;
```

The evaluator owns a compile-time heap with deterministic object IDs. Its
allocation order is not observable except through equality defined by HUC.
The stable semantic-ID case is an implementation facility for family
selection and residualization; it is not yet a public reflection API.

An expression-argument record carries the represented type, source expression,
renderable HUC0 text, and source origin. It is not a `HucCtPointer`, has no
represented object or cleanup obligation, and must not be mistaken for the
internal symbolic values used to residualize ordinary generic code. The only
user operations on an expression parameter are unchanged forwarding,
`compiler::is_expr_param`, and `compiler::expr` text extraction. There is no
public first-class handle type. Record the binding-kind bit separately from
the description so preparation can query it without reading argument contents.

Every compile-time object lives in a block: a frame slot, a `let1` object, a
string literal, or a `compiler::allocate` allocation. A pointer is a block ID
plus a path to a slot or subobject, never a host address. Each block records
its element type, slot count, and which slots and `raw_storage` elements hold
live values, so every dereference, `construct_at`, `destruct_at`, extraction,
and deallocation is checked against specification section 12.1. Pointer
arithmetic moves within one slot sequence, a block's slots or a `raw_storage`
field, or to one past its end; a pointer to any other object behaves like a
pointer into a one-element sequence. Ordering pointers into different blocks
and converting a pointer to an integer are diagnostics. Actual `u8` and `c8`
objects are ordinary values; viewing an object of another type through `u8*`
or `c8*` is a diagnostic until a byte-level layout model exists. Integers use
target widths; floats use strict IEEE binary32 or binary64 arithmetic,
independent of host floating-point settings.

### 8.3 Symbolic execution for generic `fn`

A generic `fn` is not generally executed with concrete runtime values. Its runtime
parameters enter the evaluator as `SymbolicExprId` values carrying concrete
types.

Operations behave as follows:

- an operation scheduled for compiler execution evaluates with concrete inputs;
- an ordinary runtime operation involving a symbolic input records residual
  source that the residualizer can print as HUC0;
- `if1` requires a concrete boolean and selects one branch;
- ordinary `if` with a symbolic condition creates residual branches;
- `for1` initializes its optional header binding once in a loop-local compiler
  scope, tests the condition, and prepares and inserts its body once per
  iteration; after each completed iteration it evaluates the update list in
  order before testing again. Neither the binding nor its scope is emitted;
- `#` statements insert code values under the forward insertion rules in
  section 10, in the mode of the body being prepared.

This is a small partial evaluator, not C++ template substitution. It has no
bootstrap syntax-object, quote, splice, or structural emit value. Expression
parameters expose typed text capture only, not this internal symbolic evaluator
or automatic symbolic method execution.

### 8.4 Forced mode

`@call` requires a concrete compiler result and sets the execution requirement
to `HUC_EVALUATION_FORCED`. Every compiler call prepares its callee's body in
evaluation mode and then executes it; it never inserts into the caller.
Compiler intrinsics retain their specified behavior in the active preparation;
for example, `compiler::emitting()` does not open a private body merely to
read the mode. Preparation never sees ordinary argument values: they are
symbolic after a runtime request and concrete only while a compiler call
executes. The separate binding-kind map is available to `is_expr_param` during
preparation. Concrete evaluator inputs do not become family binders or affect
mangling. An unresolved value needed during actual evaluation is a diagnostic.

Expression parameters deliberately carry unevaluated arguments, not unresolved
values to execute. Binding them follows section 8.4.2; `@` still requires a
concrete compiler result, such as the `Text` or `Code` built from that description.

A generic `fn` call retained in runtime code requests or calls a concrete runtime specialization.
Prefixing that call with `@` instead requires complete compile-time evaluation,
and only a result that can be written as a HUC0 value may leave the island
for runtime code.

Completing a generic `fn` body for evaluation is separate from emitting it. A
compile-time-only request does not mark the specialization for HUC0 emission.
A retained runtime reference does, even when that specialization was evaluated
earlier. Program definition/publication records are separate from each call's
fresh evaluation-mode preparation. Ordinary `fn` declarations in retained modules are still
emitted without demand filtering, independently of any evaluation-mode use.

#### 8.4.1 Ordinary-function calls

Code executing in the compiler may call an ordinary `fn` without another `@`.
An explicit `@` can also enter this evaluation from runtime source. Resolve the
call normally, require its HUC definition, and evaluate its prepared, checked
body under specification section 12.1. Local variables keep
their ordinary `let` spelling; their storage is evaluator-owned for this call.

Accept the runtime language: every scalar type including `c8` and floats,
Basic and Advanced structures with constructors, methods, `copy`, relocation,
and `drop`, pointers to compile-time objects, raw storage with `construct_at`,
`destruct_at`, and extraction, and compile-time allocation. Execute them over
the checked block model of section 8.2. Deferred compiler calls additionally
keep their compiler-only value kinds, compiler intrinsics, and `let1 mod`
assignment; their runtime requests must still reject compiler-only signatures
and residual operations.

Support local mutation, control flow, and nested calls with shared instruction
and recursion budgets. Apply the same rules in callees, including concrete
generic `fn` bodies. Check executed operations, not merely the outer
signature. An untaken ordinary branch may contain an unsupported operation,
but must still pass ordinary syntax and type checking. Never invoke arbitrary
native or foreign code. Reject accesses to runtime storage, an ordinary
function naming `let1 mod` storage, and untracked external inputs. An ordinary
function may change `let1` objects through pointers it receives, including
`this`. Immutable compiler-known constants are readable and become
dependencies.

Use target-width integer operations: unsigned overflow wraps, while signed
overflow, division by zero, invalid shifts, and every memory error from
section 8.2 produce diagnostics with the evaluation call stack. Unsupported
operations and exhausted budgets also diagnose; no forced call falls back to
runtime execution.

Prepare the function's phase work and `#` insertions in evaluation mode before
executing it; never insert into its caller. Parse inserted local code into that
private body and apply the same rules to its executed operations.
Compile-time operations may use compiler text while preparing it. Suppress
inserted module code; it supplies no declarations. Later program-mode
preparation must still produce and publish its module output. None of this
permits execution of arbitrary native code.

#### 8.4.2 Expression-argument binding

Classify each argument using specification section 9.7.1 before evaluating it.
Build a per-call binding-kind map, not a declaration/family map. Literals,
`let1` objects, and actual evaluator-owned locals are values; runtime-dependent
arguments are expressions, including references to runtime module globals.
Forwarded parameters keep their kind. Classify
compound arguments structurally, without constant folding or scanning a callee's
body for effects. A call with no runtime inputs is a value computation; an
unsupported execution diagnoses rather than falling back to capture. Explicit
inner `@` and compiler-only result types request compiler computation of that
subexpression, whose own calls use the same argument-binding rules.

Deduce using represented expression types alongside value-argument types, then
apply existing structural selection. Kinds never add an overload-ranking rule,
a body-failure fallback, or a mangling component. Different compiler calls of
the same specialization can have different kinds. Runtime preparation uses an
all-value map and never emits binding tags into HUC0.

Process argument positions left to right. Evaluate value arguments normally.
For an expression binding, resolve and type-check the source expression in the
surrounding prepared body's context, without emitting an executed call, load,
construction, relocation, or cleanup. Match the substituted type exactly; do
not insert numeric or pointer-permission conversions. Initial capture can
describe code in a retained body or a private evaluation-mode body. Substitute
materializable compiler-known components as HUC0 constants; reject compiler
pointers or other components that cannot be rendered. A compiler-only result
is computed, never captured. Being constant-foldable does not change a runtime
binding's phase. Constructor calls, receivers, and intrinsics retain their
specified operand contracts; expression binding adds no symbolic execution.

During compiler execution, an existing parameter can be forwarded unchanged
with its current kind. For an expression binding, preserve its original
description and represented type, not the forwarding parameter's spelling.
Diagnose attempts to read, assign,
dereference, address, store, or return it as a value, or execute its object's
methods. It is not a real parameter place and receives no drop or active flag,
even if its declaration includes binding `mod`.
The expression's ordinary checked-use effects are checked on inserted code,
not registered as transfers or observations at the capture site.
Typing a captured call may need a concrete signature, but capture alone does
not enqueue its runtime definition. Retained inserted calls create that demand
when the generated code is checked.

`compiler::expr(parameter)` accepts one such parameter (optionally parenthesized)
and returns compiler text. Render the corresponding HUC0 expression with
parentheses, staged constants, and concrete names as required by residualization.
Do not preserve raw HUC1 type arguments or phase syntax in that text. Preserve
origin records for diagnostics, not hidden name-binding metadata. Its text can
be copied, edited, stored in `let1`, and passed to the existing code constructors.
At insertion, reparse it and perform ordinary lookup and checking, including
accepting valid rebinding by shadowing. This is not structural AST splicing.
For a family reference, use the same canonical concrete name as the HUC0
printer, and resolve it through the specialization-name table in section 9.3.
Do not leave a family request in generated text or require hidden links on
individual strings. Source identifiers that remain unqualified can still rebind.

Binding descriptions are available during the compiler call's execution, not
its preceding preparation. A nested `@compiler::expr(parameter)` in that body
must not inspect the incoming argument early. Existing type binders still work
in `if1` and other preparation operations.

`compiler::is_expr_param(parameter)` reads only the call's binding-kind map.
Recognize its operand as the current function's parameter identity, optionally
parenthesized, without evaluating it. Reject locals, literals, member access,
and other operand shapes; it is neither overloadable nor first-class. Expose
the bool during preparation and compiler execution. `if1` can then select a
branch before checking its body, keeping value-only operations out of an
expression-bound call and vice versa. No parameter values or text become
available early. False in runtime preparation does not make the parameter a
compiler-readable value. Fold every retained query to its bool result; reject
any compiler-only operation still present in retained runtime code. Preparing
the same function for a compiler call uses that call's fresh map, not an earlier
runtime body's selected branch. This adds no preparation cache or runtime IR
value kind.

There is no capture-specific runtime lifetime analysis. Retaining text does not
retain an object. Generated bindings may own Advanced temporaries, normal
cleanup still applies, and duplicate inserted occurrences may execute repeatedly.
The text helper does not validate whole generated fragments; that remains the
insertion check in section 10.

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
network, clock, random, or host-process APIs. It executes permitted changes to
compiler storage in source order. `#` insertion records its text, insertion
point, and origin explicitly; `compiler::error` reports a diagnostic and fails
translation. Building `compiler::Code` values inserts nothing, and
`compiler::emitting()` only reads the current generation mode. These operation
contracts do not require whole-function caching analysis. The build dependency
graph consists of loaded HUC1/HUC0 modules and configured target inputs.
Any future explicit external-input API must return a dependency record with
its value and is part of the separate compiler-library design.

`compiler::allocate<T>(count)` creates a block of `count` inactive slots;
`compiler::deallocate<T>(slots, count)` requires the same block start and
count and only inactive slots. Reject a retained runtime call to either. When
a top-level evaluation finishes, mark blocks reachable from live `let1`
storage and report every unreachable allocation as a leak.

Keep a function-local `let1` alive while the body it was prepared for can run.
In program mode, release it when that preparation completes; the runtime body
retains only the values it read. In evaluation mode, the executing body may
read and assign it, so tie it to the compiler call's execution frame and
release it when the call finishes. Release module-level `let1` values at the
end of the translation unit. Destroy values in reverse processing order,
running `drop` in the evaluator. After the module-level values are destroyed,
no allocation may remain.

### 8.7 Staging contexts

`if1`, `for1`, and `while1` execute while HUC1 prepares code and may occur at
module, type-body, function, or nested-block scope. Selected output must be
valid declarations/members at module or type scope and may be declarations or
statements at function/block scope. During compiler execution, ordinary `if`, `for`, and
`while` already execute in compiler phase, so no `if2`, `for2`, or `while2`
tokens exist.

`let1` marks explicit compiler storage. Fields and working locals use plain `let`
and belong to the evaluator or runtime according to the call/type request.
Compiler-only requests do not demand HUC0 definitions. Angle headers do not
automatically erase types or functions without compiler-only parts if runtime
code needs them.
`@` opens a compile-time evaluation island and never qualifies a type. A
selected generic `let` specialization must leave exactly one ordinary runtime `let`
with the family name and no other residual statement. Text generation remains
forbidden inside a generic `let` selection body as a separate language restriction.
Generic `let` is permitted at module and function/block scope, but never as an
instance field.

## 9. Specialization engine

HUC1 indexes one primary family per name and scope for generic `struct` and generic `let`,
and one primary family per name and scope for generic `fn`. Syntax is retained
in these distinct parts:

```huc
struct Buffer<T: auto, N: usize> { ... } // primary binders
struct Buffer<T: auto, N: usize><T*, N> { ... }
struct Buffer<><Widget, 8> { ... } // full specialization

fn convert<T: auto>(mod value: T) -> T { ... }
fn convert<T: auto><T*>(value: T*) -> T* { ... }
fn convert<><Widget>(mod value: Widget) -> Widget { ... }

let selected<T: auto> { let selected: T = T(); }
let selected<T: auto><T*> { let mod selected: T* = null; }
let selected<><Widget> { let selected: Widget = Widget(); }

let value: Buffer<i32, 16> = Buffer<i32, 16>(); // request
let result: i32 = convert<i32>(input);
use(selected<Widget>);
```

After a declaration name, `<` starts a deferred header; `(` starts an ordinary
function's parameters. The first list always declares binders. `<>` alone is a
zero-binder primary. A second list supplies a specialization pattern: empty
binders mean a full specialization; nonempty binders mean a partial one.
Diagnose `<T>` in a declaration with a hint for `<T: auto>` or `<><T>`.
The completed header is followed by its body. Reject declaration-level `if1`
conditions on primaries and specializations; `if1` remains valid inside bodies.
Parenthesize angle comparisons in value arguments and defaults. These header
choices do not need name lookup. At expression use sites, retain ambiguous
angle postfixes until resolution as before; the lexer emits separate angle tokens.

Patterns may contain concrete types and values, binders, pointers whose `mod`
must agree, family applications, `raw_storage<T, N>`, repeated binders, and
family binders such as `C<T>`. Match repeated binders by canonical identity,
and bind a family binder to the family of the matched specialization. Ranking
remains "matches a strict subset". Record each resolved request so that a later
specialization that would have been selected for it, or made it ambiguous, is
diagnosed; a later specialization that would still lose is accepted.

A function name in one scope is either an ordinary overload set or one generic
family. Diagnose mixtures rather than adding a combined overload-resolution
system. `fn1` and `struct1` are no longer declaration spellings. Compiler-only
types can appear in deferred signatures for compiler calls, not runtime requests.

A `T: auto` binder's entity kind is inferred from use; a binder used in a type
position must hold a type, while typed non-type binders use ordinary types.
Normal generic `fn` requests may deduce binders by unifying runtime parameter and
argument types. Compiler requests also use the represented types of expression
arguments, with per-call argument-driven binding metadata from section 8.4.2.
Non-deducible, non-defaulted binders require explicit angle arguments.
Resolve `f()` and `f<>()` to the same zero-binder family. Type and variable
requests retain angles, including `S<>` and `v<>`. Call syntax never includes
a second list; family selection chooses the specialization.
Primaries may have trailing defaults; specializations may not add defaults.

Selection first matches argument count and structural angle pattern, then
binds variables. No separate condition filters candidates. Exact patterns
outrank partial patterns, structurally more specific partial patterns outrank
broader ones, and the primary is the fallback. Declaration order never resolves
a tie. A condition or diagnostic inside the selected body does not retry selection.

The primary must precede its specializations, and HUC requires every
specialization in the primary's namespace. Only the selected body is parsed.
Concrete generic `struct` and generic `let` specializations are printed as required by their
retained declarations; a generic `fn` specialization is printed only when retained
runtime code requires its definition. Print concrete definitions at their family
declaration sites, including methods within their containing type. This is
distinct from module code inserted with `#` in those functions, which follows
the original program-mode request's anchor. A function-local generic `let` result therefore
initializes on each runtime execution that reaches that declaration; staging
never inserts a lazy runtime
flag. Merely evaluating a generic `fn` at compile time does not require its runtime
definition.

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

This key identifies a definition within the current translation, not a stored
evaluation result. Record dependency inputs separately for reproducibility.
The key must never contain pointer addresses or non-canonical map iteration order.

### 9.2 State machine

Each program definition and canonical type has a construction record:

```text
Absent -> InProgress -> Complete
                    \-> Failed
```

Requesting the same `InProgress` runtime function specialization from its own body
creates a recursive call edge. Compiler execution uses the evaluator call stack
and budgets instead: an in-progress evaluation is not a result. Requesting an
`InProgress` type layout before indirection makes its size finite is an
infinite-size diagnostic.

Failed records support error recovery within this translation; they are not
stored compiler-call failures. Each compiler call has a fresh private
evaluation-mode preparation and execution frame, not a reusable record indexed
by arguments. Resolve it using declarations already published and checked at
that call. Do not reuse earlier preparations or results.

Track `runtime_needed` and `module_output_published` independently of evaluation
results. An evaluation request can complete without setting either. A retained
runtime reference requires the program-mode preparation record and adds the
specialization's runtime dependencies to the emission worklist.
Requests found only while executing compiler code do not become runtime roots. Ordinary `fn`
declarations remain runtime roots in retained modules, even if uncalled; their
retained bodies can therefore require generic `fn` definitions. Do not treat an
unexecuted ordinary runtime branch as an erased phase branch. Publish ordered
module output once per translation unit and owning function/specialization;
do not skip different emission calls or phase-loop iterations within it.
Distinct `#` statements remain distinct. A generated module declaration
with the same tokens and lexical naming context as one already inserted in that
namespace adds its origin to the existing declaration instead of being inserted
again. Report any other
duplicate with both origins; never silently choose the first conflicting
generated definition.

### 9.3 Naming

Generated HUC0 specialization names use a reserved readable prefix plus a
stable hash:

```text
_HU3app4math6select__A7F19C2E
```

The HUC0 name is deterministic for the defining family, canonical arguments,
and target-relevant ABI identity. Generation mode, body/dependency fingerprints,
and ambient HUC1 state do not enter the mangled name. The LLVM backend maps this
identity to a deterministic, target-linker-compatible symbol in a reserved
HUC mangling namespace. It need not be a C identifier. Distinguish HUC internal
names from explicit foreign symbols and platform C/C++ mangling conventions
(including the Itanium `_Z` prefix), accounting for target symbol decoration.
Detect conflicting emitted definitions and foreign symbol declarations rather
than assuming a prefix alone rules out every user-chosen foreign name. Preserve
readable source names through debug information and origin maps. Exact symbol
spellings remain implementation details until a native HUC ABI is specified.

Assign and record the canonical qualified HUC0 name when a concrete signature
is resolved, so `compiler::expr` can render it before runtime emission. Maintain
the reverse lookup from that ordinary name to its specialization record. The
inserted-text resolver uses this table just as it uses other named declaration
records; reusing unchanged text in another namespace still names the selected
specialization, not a new family lookup there. Reserving a name or extracting
text does not set `runtime_needed` or prepare/publish a runtime body. A retained
program reference does so through the normal worklist and insertion anchor;
a private evaluation reference follows normal compiler-call availability rules.
This table must not make pending generated module output visible early.

Emit all required concrete definitions into final HUC0 using those same names.
The independent HUC0 frontend resolves actual declarations in that file, not
the HUC1 table. Do not expose a public name-construction algorithm or attach
opaque bindings to editable text; builders treat rendered names as opaque.

An imported family may receive a caller-defined type without a reverse import.
Substitution carries canonical type identity; unqualified body names still
resolve in the family's lexical context. The concrete function keeps the
family's namespace and declaration position in the shared translation unit.
Its module output follows the first runtime request's anchor instead. Reopen
namespace sections for that output and print canonical references; do not
invent caller-side lookup for the function body.

Preparation follows deterministic source and depth-first import order. A
worklist must preserve that order and inherited anchors.
The flattened output depends on the complete runtime-needed specialization
closure and its publication order. Construct that closure afresh for each
compilation, including requests from importing sources.

## 10. Text generation and forward insertion

The bootstrap compiler-supplied `import1 compiler` module provides the
compiler-only `compiler::Code` type, three constructors, and `+`:

```huc
compiler::local(text)
compiler::module(text)
compiler::around(before, after)
```

`compiler::expr(parameter)` separately extracts parenthesized HUC0 expression
text from an expression parameter (section 8.4.2). The constructors accept that
text exactly like any other text; they do not validate or bind an expression
fragment. No typed placeholder or first-class expression handle is added.

A `#` statement inserts a code value:

```huc
#compiler::local("let mod generated: i32;");
```

Represent a code value as an ordered list of local fragments, module fragments,
and at most one hole marker, each with its origin. Building or joining values
has no effect; only an insertion item inserts. Evaluate the `#` operand like an
`@` value, then insert its local fragments at the item and queue its module
fragments.

Maintain a preparation context with mode, generation site, lexical namespace,
body owner, original module insertion anchor, and ordered local/module queues.
A `#` uses the context of the body that contains it. A compiler call pushes a
separate evaluation-mode context for its callee's body, then restores the
caller's context. The place where a code value was built never becomes its
destination.

Type preparation owns its local member insertion scope, even when its module
anchor is inherited from the requester. Do not inject fields into a caller's
block. Reject incompatible field definitions for the same nominal specialization
across compiler and runtime requests.

| Generation mode | Local code | Module code |
|---|---|---|
| Program mode | Parsed into retained code | Published after the original request's anchor |
| Evaluation mode | Parsed into a private evaluated body | Suppressed |

The bootstrap query `compiler::emitting()` returns
`true` in program mode and `false` in evaluation mode, read from the active
preparation. `if1` can therefore select a mode-specific branch before either
branch is parsed. Structural selection does not evaluate this query or any
other user condition.

The bootstrap diagnostic operation `compiler::error(message)` accepts one
compile-time text argument and has result type `never`. When reached,
it produces an evaluation failure with the supplied text, operation location,
request site, and compiler call/specialization chain. It does not return,
construct a code value, or emit a runtime call. Execute it in either generation
mode; only unreachable compiler execution or skipped phase bodies avoid it.
Do not turn this failure into candidate rejection or fallback. See specification
section 11.6 for the source-level contract.

Insert local code at the `#` item, then process it before advancing to
subsequent source. Add immutable generated buffers to the parser's logical
input stream; do not rewrite earlier source buffers or parse the next block
prematurely. If the code has a hole, require the following source block and
parse the text before the hole, the block, and the text after the hole as
complete items. Reject a second hole, a hole inside an expression, and other
partial fragments or unmatched braces.

In program mode, queue module code after the original module insertion anchor.
For an ordinary `fn`, use its top-level declaration (the whole containing type
for a method), prepared when encountered even if uncalled. A first runtime
generic `fn` request inherits the active anchor; all nested specialization
requests retain it. Thus `main -> A -> B` queues module code inserted in either
function after `main`. Registering deferred bodies runs no `#`. A method
encountered while preparing a requested generic `struct` keeps the requesting
anchor too. A `#` at module scope anchors its own output.

Finish the anchor's compiler-stage work, retain the anchor declaration, then
parse and publish module code in insertion order before the next original item.
Inserted code uses the namespace and lexical context of its `#` site,
independently of the anchor's namespace. Use namespace sections when these
differ. Do not publish a nested function's output just because that nested
preparation completed. No queue may move fragments ahead of its anchor or
append them all at end of file.

In evaluation mode, do not insert, parse, or register module code and do not
mark it published. Neither placement supports backward insertion or modifying
previously prepared source. Missing declarations from suppressed code diagnose
when needed. Resolve evaluated bodies using declarations already published and
checked. Pending code is unavailable even through another function;
already published code is usable even by the evaluated body of the
function that inserted it. If completing an active anchor requires evaluating
its pending code, diagnose the dependency cycle rather than draining the queue
early.

Share HUC0 lexing/parsing and semantic rules with HUC1's inserted-code path.
Reject numbered constructs, `import1`, `@`, and `#` in generated HUC0 during
HUC1 preparation. Resolve generated and original source together, allowing
module forward references but rejecting changes to finalized compiler
decisions. Print only checked program-mode output into `.huc0` artifacts; the
standalone HUC0 translator independently validates those files.

Record generated-buffer spans, `#` site, module insertion anchor, namespace,
import chain, the call chain that built the code, specialization chain, and
mode. Syntax nodes can have multiple spans. Diagnostics in original blocks point
to original source; generated errors show their text and origin chain. Output
maps add final HUC0 ranges when files are printed. Source locations never
require rewriting earlier buffer offsets.

There is no bootstrap `gen`, `gen_global`, `quote`, splice, `parse_huc`,
structural syntax object, first-class expression-handle API, automatic fresh-name service,
or structural `emit` API. Parsing inserted text is internal compiler behavior,
not a public `parse_huc` function. A compiler reflection and
structured-generation library may later provide environment, module,
declaration, type, function, statement, and expression handles, but that is a
separate language/library design. The bootstrap architecture must not invent
its API. Such handles will be ordinary qualified compiler-only types, never
`@Type` spellings, and should expose immutable snapshots and stable IDs rather
than mutable internal AST storage.

The specified argument-driven bindings, `compiler::is_expr_param`, and
`compiler::expr` text extraction are the narrow exception to the future
expression-library boundary. They do not expose
an AST, preserve lexical bindings after insertion, or alter runtime lifetimes.

The inspection mechanism is the ordinary stage output:

```text
huc stage app/main.huc1 --out-dir build/huc0
```

Its printable translation unit is the exact input consumed by the next translator.

## 11. Concrete HIR and MIR

Successful HUC1 output contains no numbered construct, `@`, `import1`,
compiler-only declaration, family binder, specialization request, or
`#` insertion. Local `let name: auto` remains legal HUC0; unresolved
inferred function parameters or results do not.

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
HIR and MIR retain the full-expression cleanup boundary of each materialized
temporary (language specification section 6.5.1). An emitter-created C
temporary or statement does not create a new HUC lifetime boundary.

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

Lower `InitializeAt` to initialization in the retained indirect destination,
using the existing value operations but without registering automatic cleanup
for that slot. Lower `DestructAt` to `Drop` of an indirect manual place with
its required-live precondition. Do not consult or update aliased root flags,
even for `&local`; preserve that cleanup mode through inlining. Basic
destruction needs no runtime call but ends the pointee lifetime in HUC.
Neither operation adds allocation/deallocation or source/pointer-clearing
code. Invalid runtime preconditions are not required to be checked.

Raw-storage initialization makes its wrapper Active while leaving its element
region inactive. MIR need not track individual element states. Whole-wrapper
relocation transfers live payloads and dormant bytes together, without reading
inactive slots as typed values. Dropping the wrapper emits no element cleanup;
the library's length or tag controls explicit `DestructAt` operations. Basic
copy-and-retire paths may contain a `DestructAt` with no runtime instructions;
do not erase its lifetime meaning before validation.

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

The same state drives the required relocation-state check (language
specification section 6.12). Any MIR use of a tracked root that is `Inactive`
or `MaybeActive` at that point is a diagnostic in every compilation mode. A
use is any `Copy`, `Clone`, `Relocate`, `AddressOf`, field projection,
method-receiver formation, or store into a field of the root. A field store
is checked both when its destination is formed and when the store occurs,
after evaluation of its source operand. Storing a whole new value, scope-exit
`Drop`, and `SlotAddress` are not uses of the old value. Computing their
operands may still read values and must be checked. Field projections used
only to form `SlotAddress` through inline fields do not read the root.
MIR is already sequenced in HUC evaluation order, so later arguments are
checked against the state left by earlier parameter bindings.

The pass also rejects a `Relocate` from a root that happens during a call's
argument evaluation, including in a nested expression, in two cases. The
first is when the root is the root of the call's receiver place. The second
is when an earlier argument of that call is an `AddressOf` of the root or of
one of its fields. An address formed inside an earlier argument that does
not itself become an argument does not count. Constructor expressions,
intrinsic calls, and initializer entries count as calls. Receiver roots, destination roots, and
earlier `AddressOf` arguments are recognized as specification section 6.12
describes.
HIR records the source spellings needed for this: parentheses, field
projections, `(&r)->f`, `*&r`, `ptr_as`, and either arm of `?:`. That
recognition does not change the cleanup mode of `*&r` (section 7).

The pass reports each diagnostic at the HIR source origin of the offending
use, together with the relocation that made the root inactive.

The checker processes operations in conditions but does not fold their
results or infer relationships between them. Every `Branch` and `Switch`
successor is considered possible. Loop back edges join with loop entry;
iterate until the states stop changing before reporting use diagnostics.
Each `break` edge joins the loop exit. A call whose result type is `never`
has no successor. Unreachable code is exempt from this state check, not from
ordinary parsing, name resolution, or type checking.

Lower `for` with a loop scope, initialization, condition block, nested body
scope, ordered update blocks, and exit. Both normal body completion and
`continue` target the first update after body cleanup, or the condition if
there are no updates. `break` and `return` bypass all updates. Track relocation
through each update and condition before joining the back edge. Do not lower
to a naive `while` whose `continue` skips updates or destroys the header local.

Run the check on each concrete HUC0 function, including every residual
specialization produced by HUC1, before runtime constant folding removes
branches. Drop flags remain a cleanup mechanism: no runtime test of a flag
is added to permit a use of an Inactive or MaybeActive root. This guarantee
covers tracked state, not lifetime changes through unchecked pointer aliases.

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

It also adds temporary cleanup at every completed full-expression boundary,
including condition and individual loop-update boundaries (section 12.5).
For a `for`, body cleanup precedes updates; header storage survives `continue`
and is cleaned up only when leaving the loop, if active. The condition-false
edge must clean up that storage even when the body never ran.

There are no exception cleanup edges; HUC has no exception unwinding.

Explicit MIR cleanup is mandatory for the LLVM backend:

- it implements HUC destruction order independently of backend storage placement;
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

No HIR or MIR operation performs overload resolution for relocation. A
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
layout, valid LLVM memory operations, source deactivation, and exactly-once cleanup.
It may use bulk copies, loads/stores, registers, or elision when observably
equivalent; the contract neither mandates a literal memory-copy instruction
nor gives padding bytes semantic significance.

### 12.5 Temporary cleanup boundaries

Track the active temporaries belonging to each full expression. At its end,
destroy the ones still active in reverse order of completed initialization.
Temporaries consumed by `Relocate` are inactive and receive no cleanup there.
Directly constructed locals, fields, parameters, and return results belong
to their destination lifetimes, not the temporary list. Basic temporaries
need no drop call, but any materialized storage must remain valid until the
specified lifetime boundary if its address can be observed.

Preserve these boundaries during lowering:

- expression statements: after evaluating the entire expression;
- local initializers: after the destination is initialized;
- conditions: save the result, clean up temporaries, then branch on the saved
  result; this applies to both outcomes and every loop-condition evaluation;
- `for` initialization: clean up helper temporaries before the first condition,
  without destroying the header binding;
- each `for` update expression: clean up before the next update or condition;
- field initializers: after the field is initialized, before the next field
  or constructor body;
- returns: bind the result, clean up return-expression temporaries, then
  clean up remaining locals and parameters before returning to the caller.

Nested expressions, call arguments, and short-circuit operands share their
caller's enclosing full-expression boundary. Do not drop a receiver
temporary immediately after its method returns, or an argument's helper
temporary when parameter binding finishes. Calls still perform their own
internal cleanup, including cleanup of active by-value parameters before
returning; a consumed argument is not a caller-owned temporary anymore.

For conditional evaluation, register cleanup only on paths that initialized
the temporary. Branch-local cleanup paths or separate active flags may be
used; no flag belongs inside the nominal type. In particular, lowering `?:`
must not clean an arm's unconsumed temporaries before the enclosing expression
ends, and unevaluated operands receive no initialization or cleanup.

For `print(make_text().view())`, keep the caller's `Text` temporary active
through the `print` call, then drop it. For
`let view: View = make_text().view();`, drop that temporary after binding
`view`. Its observer does not extend the lifetime. LLVM lowering must preserve
these HUC boundaries independently of instruction scheduling and stack allocation.

As elsewhere, a terminating `panic` need not run pending cleanup.

## 13. LLVM lowering

The initial backend constructs LLVM IR directly from concrete, sequenced HUC
MIR. No generated C or C++ source sits between HUC0 and LLVM. Keep LLVM's
in-memory module behind the backend API, with textual `.ll` output for inspection
and optional `.bc` serialization. IR validity is checked with LLVM's verifier;
HUC correctness still needs semantic and execution tests.
See the [LLVM language reference](https://llvm.org/docs/LangRef.html#well-formedness)
for the IR contract and verification rules.

### 13.1 Runtime representation

Choose the target triple, CPU/features, and data layout before resolving layout
queries or emitting code. HUC layout calculations and the emitted module must
agree on sizes, alignment, field offsets, and pointer widths. LLVM's
[object-code tutorial](https://llvm.org/docs/tutorial/MyFirstLanguageFrontend/LangImpl08.html)
describes the target-machine and data-layout setup used for object emission.

Lower scalars to matching LLVM value types and HUC structures to concrete
aggregate layouts. HUC pointer types and their `mod` permissions remain in
HIR/MIR even when LLVM represents their addresses with opaque `ptr` values.
Storage is allocated with the target-required alignment; the emitter must not
infer source ownership or permissions from an LLVM pointer.

Represent `raw_storage<T, N>` as aligned space for a contiguous sequence of
`N` concrete `T` layouts. Reserve the region without constructing payloads or
registering element cleanup. `storage_ptr` addresses its first slot. Transfers
must carry the representation, including inactive regions, without turning
uninitialized payload into typed value reads. Cleanup remains the containing
library's responsibility, not an LLVM array operation.

### 13.2 Operation mapping

| HUC operation | LLVM lowering responsibility |
|---|---|
| `T*` / `mod T*` | Address values; HUC checks permissions before lowering |
| `&value` | Address of the retained storage place, materializing storage when required |
| Pointer arithmetic / field access | Target-scaled address calculation with HUC bounds and layout rules |
| Pointer/integer conversion | Target-supported conversion preserving HUC's specified address round trip |
| Basic binding | Scalar or aggregate copy; source remains active |
| Advanced relocation | Representation transfer and tracked-state update, without source clearing |
| `copy value` | Resolved clone call when required by the source type |
| `construct_at(pointer, value)` | Initialize the supplied inactive place directly and yield its address |
| `destruct_at(pointer)` | Emit full drop/field cleanup without freeing storage or updating aliased root flags |
| Destruction | Explicit user `drop` followed by reverse field cleanup for active values |
| `T(arguments)` initializing a final place | Initializer call with the final destination address |
| Layout and category queries | Constants from the shared target/type model |

Use loads/stores or memory-transfer intrinsics only where their alignment,
overlap, and initialization requirements fit the HUC operation. Check exact
self-relocation before dropping a possibly aliased destination. Neither a raw
write nor a lifetime intrinsic performs HUC destruction or updates ownership.

Methods and lifecycle helpers become concrete functions with explicit receiver
and destination parameters where needed. Allocation and release still follow
the chosen library contract. Pointer extraction does not alter automatic
cleanup through aliases; this remains the programmer's responsibility.

### 13.3 Sequencing and calling conventions

Emit each argument expression and its parameter initialization before starting
the next argument. A final LLVM call instruction does not perform this HUC
ordering for the frontend. Inert pointer transport to already initialized
parameter places is a suitable initial internal convention:

```text
evaluate first(); initialize parameter place 0
evaluate second(); initialize parameter place 1
call implementation using those initialized places
```

By-value parameters belong to the callee and are cleaned up before it returns,
unless transferred onward. Caller-owned helper temporaries stay alive until
the enclosing full-expression boundary. Results with observable destination
identity use an explicit final-result address; do not introduce a visible
intermediate object through aggregate return lowering. Scalar returns and
other proven-equivalent cases may use direct SSA results.

HUC's internal calling convention is private. C calls and entry points require
separate, target-correct ABI lowering, including extension and parameter/result
attributes when required. The current scalar/pointer C boundary is the initial
scope; choosing LLVM does not adopt the separate broader Clang/header proposal.
Foreign code must not unwind across HUC frames while exceptions are unsupported.

Narrow signed/unsigned integers and booleans may require `signext` or `zeroext`
on parameters and results. The details are target ABI rules, not a universal
mapping from HUC integer width. Match both declarations and call sites, including
HUC functions entered from C. Apple arm64 and x86-64 System V must not be assumed
to share one rule merely because both targets have 64-bit pointers.

If Clang integration is adopted later, its
[CodeGenABITypes interface](https://clang.llvm.org/doxygen/CodeGenABITypes_8h_source.html)
can provide foreign signature lowering. It takes Clang AST types and a
code-generation context; reusing it requires that integration, not just linking
LLVM or converting HUC types to LLVM types.

### 13.4 Explicit lifetimes and cleanup

MIR determines the cleanup edges for scope exit, return, loop control,
temporary boundaries, and assignment replacement. Emit calls and branches for
those decisions. A moved-from tracked root receives no second cleanup; an
indirect extraction does not cancel an aliased root's cleanup automatically.

Conditional active flags are separate compiler storage, never fields of a
nominal HUC type. Stack allocation and SSA promotion are representation choices;
neither changes the source lifetime. LLVM lifetime markers are optional
optimization information, not constructors or destructors, and may be added
only where the MIR lifetime proves them valid.

Fixed bindings do not imply LLVM constant memory, and raw observers do not
imply exclusive access. Do not attach `noalias`, `readonly`, `nonnull`,
`inbounds`, or related promises just because a HUC type looks suitable.
Establish the exact guarantee from the program and HUC's rules first.

### 13.5 Names, translation units, and artifacts

One HUC translation unit normally produces one LLVM module:

```text
build/llvm/app/main.ll
build/llvm/app/main.bc       # optional
build/llvm/app/main.huc.map
build/obj/app/main.o         # platform object extension may differ
```

Source imports are already flattened or processed; namespaces do not create
separate object files. Deterministic symbols distinguish modules, overloads,
methods, and generated helpers. Forward declarations are LLVM declarations,
not generated C headers.

Emit the hosted entry-point adapter with the selected target's C ABI. It calls
the configured root module's HUC `fn main() -> i32`. Runtime libraries and
startup objects are selected and linked explicitly for the target.

### 13.6 Diagnostics and source mapping

The HUC frontend diagnoses source-language errors before backend emission.
Verifier failures in generated IR are compiler bugs, not user type errors.
Keep HUC1/HUC0 origin sidecars alongside LLVM instruction origins and debug
locations. Linker and foreign-symbol diagnostics should name the corresponding
HUC declaration where possible. Preserve original and generated source buffers;
LLVM output must not renumber earlier HUC locations.

### 13.7 Optimization boundary

No LLVM optimization may change defined HUC behavior. In particular:

- preserve unsigned wrapping; add overflow assertions only when justified;
- preserve pointer/address semantics for supported targets;
- use strict floating-point lowering by default, with no contraction or
  reassociation and the fixed default environment described below;
- do not read inactive raw storage as initialized values;
- do not invent alias, memory-effect, or lifetime guarantees;
- retain every observable cleanup and evaluation-order effect.

HUC's existing rules justify useful LLVM optimization information:

| Source guarantee | Lowering rule |
|---|---|
| Signed addition, subtraction, or multiplication overflows only on undefined executions | Use `nsw` on the matching operation at its HUC width |
| Unsigned arithmetic wraps | Do not use `nuw` unless a separate proof excludes overflow |
| A pointer calculation obeys the required object bounds and offset rules | Use `inbounds` only when LLVM's full requirements, including intermediate offsets, follow |
| Incompatible-type access is undefined, with byte-access exceptions | Type-based alias metadata is a later optimization; preserve compatible accesses and `u8`/`c8` aliasing |

These are optimizer promises, not runtime checks. Do not copy flags onto
compiler-introduced arithmetic unless its own semantics justify them. A cast
or an arbitrary integer-derived address does not prove an `inbounds` promise.
Resource ownership alone does not prove `noalias`. The exact requirements are
defined by the pinned [LLVM language reference](https://llvm.org/docs/LangRef.html);
the source-to-IR mapping must preserve all defined HUC executions.

For the fixed default floating-point environment, use ordinary `fadd`, `fmul`,
and related operations without fast-math flags. Explicitly disable contraction
and reassociation in backend configuration. Constrained intrinsics are not
required by this contract; reconsider them if HUC later supports dynamic
rounding or observable floating-point exception behavior. Foreign calls must
restore HUC's required environment before returning to HUC arithmetic.

Initially support only targets that meet the strict contract. For an initial
32-bit x86 floating-point backend, require SSE2 or reject the target; do not
silently use x87 excess precision or switch to relaxed math. Test each supported
target's rounding, subnormal handling, and non-contraction behavior.

Optimization information can help LLVM prove loop and aliasing properties, but
does not guarantee vectorization or a particular speedup. Measure representative
kernels and inspect generated code; do not substitute flags for performance
tests. See [LLVM's frontend guidance](https://llvm.org/docs/Frontend/PerformanceTips.html).

Verify IR before optimization and verify optimized output in tests. Run the
same defined programs at unoptimized and optimized settings. LLVM verification
alone cannot establish these source-level guarantees.

## 14. Other backends and C++ interoperability

Alternative emitters can later consume the same HIR/MIR without rebuilding
HUC's phase or ownership analysis. They are not required for the LLVM-first plan.
C ABI adapters can connect to C++ code; direct C++ bindings need explicit rules
for foreign types, lifetimes, allocation, and exceptions. LLVM IR is not itself
a C++ interoperability solution. Full C header integration is also a separate
design from changing the HUC backend.

## 15. Driver and build pipeline

### 15.1 Commands

```text
huc lower <root.huc0> --out-dir <llvm-dir>
huc stage <root.huc1> --out-dir <huc0-dir>
huc build <root.huc1> --out-dir <build-dir>
```

`stage` produces printable HUC0; `lower` checks HUC0 and emits LLVM IR.
`build` writes the staged source, passes that file through the public HUC0
frontend, emits objects through LLVM, and links them. It never hands a private
HUC1 AST directly to the runtime backend. `stage` can be the first delivered
command, or all three can grow together for an end-to-end feature subset.

### 15.2 LLVM and target toolchain

Link the compiler against a pinned, supported LLVM build. Record its version,
enabled targets, selected target triple, CPU/features, data layout, optimization
settings, and runtime/linker configuration. Use LLVM's target code generator
for object emission. Discover or configure a target-compatible linker and
sysroot/startup libraries separately.

An optional bootstrap alternative is to write textual `.ll` and invoke a pinned
LLVM tool such as `llc` for object emission. This still lowers directly to LLVM;
it does not restore the C-source backend or delegate C ABI classification. It
requires correct IR syntax, types, SSA names, escaping, and verification. Keep
the LLVM C++ API as the planned interface: it already supports printing `.ll`
for golden tests. Use the external-tool alternative only if it simplifies the
initial dependency setup, behind the same backend boundary.

There is no `--cc` selection for compiling generated HUC C source, because no
such source is emitted. A Clang or platform compiler driver may still be used
to invoke the linker or build foreign sources; that is a separate toolchain
role. Unsupported target/ABI combinations must produce explicit diagnostics.

### 15.3 Build artifacts

```text
build/
  huc0/          flattened staged source and origin/dependency sidecars
  llvm/          textual IR, optional bitcode, maps, and debug information
  obj/           target object files
  bin/           linked output
```

Generated HUC0 and LLVM IR remain inspectable artifacts. Compiler-build
metadata such as `compile_commands.json` is separate from these user-program
outputs. Normal builds remove generated artifacts only through an explicit
build-system operation.

## 16. Compilation records and reproducibility

### 16.1 No caching

Do not implement frontend, staging, evaluation-result, preparation, or
build-artifact caches. Do not classify functions by whether their results can
be reused. Each build processes its sources and produces its artifacts afresh,
including the flattened HUC0 file before invoking the public HUC0 translator.
The textual boundary remains mandatory.

Each compiler call prepares a new private evaluation-mode body and executes
it with the call's arguments, including repeated identical calls and recursion.
Its preparation operations and permitted compiler-state changes occur again.
Discard the private execution/preparation record when the call finishes;
retain returned values and diagnostic origins as needed. Compiler intrinsics
perform their operation directly in the active context. Ordinary arguments
still become available only during execution, not preparation.

Keep per-translation declaration tables, canonical type identities, recursion
states, and program definition/publication records. These establish which
entity a reference denotes; they are not compiler-call result caches. Prepare
each ordinary function or runtime-needed specialization in program mode once.
Retain its checked body, origins, and ordered module output, and publish after
its original requesting anchor. Repeated runtime references share that
definition without replaying generation. Evaluation-mode completion cannot
satisfy later program preparation or mark its output published.

Each `#` during preparation inserts into its own body context. A code builder
executes on every call. A source-level `let1` may explicitly retain a returned
code value; each `#` using it still performs insertion. Separate insertion
statements and phase-loop iterations remain distinct requests. Identical
generated module declarations share one declaration with several origins;
other duplicates diagnose. Runtime execution does not replay preparation.

Source/dependency hashes may still support reproducibility, stable identities,
and diagnostic records; they do not enable result reuse or become ambient-state
inputs to name mangling. No record carries evaluation results between builds.

### 16.2 Interface files

A future HUC0 interface format may contain:

- runtime signatures;
- type layouts where ABI-visible;
- definition and dependency fingerprints.

HUC1 family records and future compiler-reflection metadata must not be
passed to HUC0 semantic analysis. Until interface
serialization is stable, lower whole HUC0 programs without promising a stable
generated ABI.

### 16.3 Reproducibility

Canonical serialization sorts maps by stable IDs, normalizes paths relative to
the project root where possible, and excludes allocator addresses. Generated
symbol hashes must be identical across processes for identical inputs.

## 17. Testing strategy

### 17.1 Unit tests

- tokenization, including `&`, `<`, `>`, `@`, and numbered keywords;
- angle-binder declaration lookahead, zero-binder primaries, full and partial
  patterns, and bodies immediately after complete declaration headers;
- pattern forms: `mod`-sensitive pointers, pointer chains, family
  applications, `raw_storage`, repeated binders, and family binders, with
  strict-subset ranking and late-specialization errors;
- `let1 mod` assignment order across imports, preparation-time reads fixed into
  prepared bodies versus current-value reads in executing compiler code, a
  separate function-local `let1` per preparation, and `let1` redefinition errors;
- rejection of parenthesized generic binders, removed suffix-2 spellings,
  declaration-level `if1` conditions, and mixed generic-family/ordinary-overload names;
- `in` accepted as an ordinary identifier;
- valid and invalid numeric separator positions in all supported bases;
- name-first declarations, independent binding/type `mod`, and generic `let` headers;
- optional trailing commas in nonempty single/multiline lists, and rejection
  of a comma in an empty list;
- required control-flow braces, `else if`, and three-part `for`/`for1` headers;
- empty and expression initializers, loop-local `let`/`let1` initializers,
  optional conditions, and ordered update lists with optional trailing commas;
- rejection of multiple initializer declarations, initializer expression lists,
  condition/update declarations, and comma operators outside update lists;
- rejection of old `T&`, type-first declarations, and range-based loop headers;
- Pratt precedence, ambiguous angle postfixes, and recovery;
- prefix `&` versus binary `&` and `&&`, including postfix field/index binding;
- pointer chains, alias-expanded chains, and typed pointer-slot addresses;
- typed `&` addresses and explicit pointer/integer casts, each evaluating its
  operand once without transferring a pointee;
- slot addresses of fixed, inline, and raw-pointer slots;
- invalid non-place address operands and pointer/`usize` cast type checking;
- Inactive/MaybeActive root address rejection even inside integer casts;
- canonical type interning;
- raw-storage type interning, positive literal counts, layout overflow, and
  recursive layout diagnostics; target layout/category query evaluation;
- raw-pointer copying, including null, with no source changes or pointee cleanup;
- raw-pointer inference for `auto` and permission-dropping conversions;
- rejection of incompatible pointer types and added pointee `mod`;

- unqualified intrinsic resolution, with no former-spelling compatibility aliases;
- lifetime-intrinsic arity, writable-pointer inference, initializer checking,
  result types, and rejection of first-class or overloaded uses;
- overload ranking;
- C-linkage signature acceptance and rejection;
- phase availability;
- specialization keys;
- evaluator arithmetic and resource limits;
- deferred-body delimiter scanning;
- generation modes and insertion points for `#` in runtime and evaluated bodies;
- tracked-root versus pointer-extraction cleanup modes;
- rejection of direct field/array-element and read-only pointer sources;
- drop-state analysis and relocation-state diagnostics for both `Inactive`
  and `MaybeActive` uses, covering branches, short-circuit operators, `?:`,
  loops, later arguments of the same call, and receiver or `&` arguments
  followed by relocation of the same root.

### 17.2 Golden tests

Store input plus expected:

- diagnostics;
- printable flattened HUC0 with namespace sections;
- concrete HIR;
- MIR;
- unoptimized LLVM IR and selected optimized IR.

LLVM IR goldens require deterministic symbols and local numbering for the
pinned toolchain; normalize only documented nonsemantic toolchain metadata.

Relocation goldens must show whole-value `Relocate`, including for nested
Advanced aggregates, with no recursive source-field transfers or null stores.
Replacement goldens must check exact self-relocation before destination
cleanup. Address-taking goldens include `&place` for pointer slots and
`ptr_as<usize>(&place)` for integer conversion. The latter must reuse the
ordinary address calculation and its permission and activation checks.
Pointer-extraction goldens must evaluate the source address once, create an
active destination, and emit no source clearing, alias-driven flag updates,
or runtime ownership lookup. Inlining must preserve the source cleanup mode.
Temporary-lifetime goldens must distinguish full-expression cleanup from
parameter and local cleanup. Check that condition results and return values
are retained before cleanup and that generated helper statements do not end
HUC temporary lifetimes early.
Manual-storage goldens must show pointer-before-initializer sequencing, direct
construction at the supplied address, and complete manual destruction without
deallocation. Verify that direct Advanced sources become Inactive but aliased
destination roots keep their existing tracked state.
Raw-storage goldens must show typed contiguous C backing storage, no element
construction on wrapper creation, no implicit payload destruction, and no
per-element flags. Test payload transfer inside a relocated container and
`storage_ptr` on empty storage without reading uninitialized bytes.

### 17.3 Execute tests

Compile and run small programs that record:

- argument evaluation and by-value parameter-binding order;
- direct-construction `this` addresses for locals, fields, parameters, and results;
- temporary receivers staying live through an outer observing call, followed
  by reverse-order temporary destruction;
- cleanup after discarded results, declaration initializers, condition
  evaluations on both outcomes, `for` initializers and each update, and field initializers;
- loop-local binding visibility, left-to-right updates, and body cleanup before
  updates on normal completion and `continue`;
- header cleanup after a false first condition, `break`, and `return`, with
  updates skipped by the latter two and no cleanup of relocated header values;
- short-circuit and conditional expressions creating and destroying only
  the temporaries on evaluated paths, at the enclosing expression boundary;
- return-result binding before temporary, local, and parameter cleanup;
- by-value parameter cleanup before call return, without double cleanup of
  the transferred source at the caller's full-expression boundary;
- destructive-relocation and clone counts;
- copying fixed and writable raw pointers without modifying the source;
- relocation of nested Advanced aggregates with cleanup only through the
  active destination;
- extraction from manually managed slots followed by initialized-range
  updates, including a pop-style return with exactly one destination cleanup;
- extraction through a fixed `mod T*` binding without changing its address;
- Basic pointee copying and explicit pointee cloning leaving sources active;
- `construct_at` from Basic, Advanced, explicit-copy, constructor, and function
  result initializers, with each operand evaluated once and no hidden clone;
- construct/extract/reconstruct/destruct cycles, final constructor addresses,
  manual reverse field cleanup, and reuse of the allocation afterward;
- unchanged root cleanup state when either intrinsic operates through an alias;

- exact self-relocation behavior;
- destruction order;
- early-return cleanup;
- conditional relocation;
- `if1` branch removal;
- `for1` unrolling, initialization/condition/update order, loop-local `let1`,
  omitted clauses, zero-iteration body skipping, and rejection of
  runtime-dependent clauses;
- ordinary compiler-executed `for` loops using plain `let` header storage,
  including inside selected module/type bodies;
- forced `@` evaluation;
- ordinary `fn` calls from compiler code with structures, methods, pointers,
  raw storage, and floats, nested calls, local mutation, bounded recursion, and
  direct `@` entry, producing the same results as at run time;
- target-width integer arithmetic, unsigned wrapping, strict IEEE floating
  point, and diagnostics for signed overflow, division by zero, invalid
  shifts, and every compile-time memory error;
- rejection of reached foreign calls, runtime-storage access, pointer-to-integer
  conversion, ordering of unrelated pointers, and `let1 mod` names in ordinary
  functions, while `u8` and `c8` objects remain ordinary values;
- a function-local `let1 mod` assigned while its compiler call executes, alive
  until the call finishes;
- unsupported operations skipped by ordinary branches versus reached operations,
  while both branches still receive normal syntax/type checks;
- repeated compiler calls executing again, observing permitted compiler-state
  changes, with no automatic result or private-preparation reuse;
- `compiler::error` in either generation mode, computed messages in compiler
  helpers, correct call/request origins, and no error from untaken compiler branches;
- explicit errors in selected bodies not falling back to another specialization;
- compile-time-only generic `fn` calls omitted from HUC0, later runtime demand causing
  emission, transitive runtime dependencies, and uncalled ordinary `fn` retention;
- parsed generation at local, type-body, and forward module insertion points;
- local generation in both modes, with the evaluator's checks on the prepared
  evaluation body; suppressed module output evaluates arguments but introduces
  no names;
- evaluation-first/program-later preparation still publishing module output;
- once-per-owner program publication, distinct emissions/iterations preserved,
  identical generated declarations inserted once, and other duplicates
  diagnosed;
- `compiler::emitting()` in both modes;
- unused angle-header bodies left opaque, requested bodies checked, and uncalled ordinary
  `fn` bodies prepared with module output immediately following their declaration;
- zero-binder helpers called with and without `<>`, deduced/defaulted calls, and
  explicit `<><T>` full specializations versus `<U: auto><U*>` partial ones;
- deferred functions/types without compiler-only parts accepted at runtime
  when needed, while compiler-only signatures/fields are rejected on runtime
  requests;
- a container built through `compiler::allocate` across several compiler calls
  in a `let1 mod`, with leaks diagnosed at the end of a top-level evaluation
  and of the translation unit;
- compiler-only parameters/results accepted in compiler calls; direct compiler
  API calls in executed compiler code, plain `let` working locals and fields, and
  `let1` staging storage;
- `#counter_decl();` inserting returned code at the `#`, versus
  `let x: i32 = @answer();` keeping `answer`'s own inserted local private, and
  `@` inserting nothing in any position;
- rejection of standalone `@` statements whose type is neither `void` nor
  `never` (with a `#` hint for `compiler::Code`), acceptance of
  `@compiler::error(...)`, and rejection of `@` in a body reading its
  parameters or locals;
- fresh evaluation-mode preparation on every compiler call, independent of
  ordinary argument values, with preparation effects executed each time;
- one program definition per runtime specialization, despite repeated runtime
  references, and explicitly stored code values inserted at every `#` use;
- generic code builders selecting partial/full specializations, and `#` in a
  type body inserting members into that type rather than its requester;
- nested `main -> A -> B` requests publishing module output only after `main`;
  compiler evaluation inside `main` rejecting pending output even through `B`,
  and evaluation after the boundary accepting the published declaration;
- imports processed once in depth-first order, alias restoration, canonical
  namespace sections, and imported generics accepting caller-defined types;
- code with a hole filled by the following block, correct loop control and
  scoping, and errors for a missing block, a second hole, or a hole inside an
  expression;
- immutable source locations, mixed-origin syntax nodes, forward module name
  resolution, and rejection of retroactive semantic changes;
- argument-driven bindings with exact pointer permissions, no marker in the
  grammar, no binding-kind overload ranking, and ordinary runtime calls;
- the same parameter binding as a value for `let1`/evaluator locals and as an
  expression for runtime inputs, mixed kinds in one call, and preserved forwarding;
- structural compound-argument classification, explicit inner evaluation,
  no-input calls evaluated as values, and no evaluation-failure fallback;
- `is_expr_param` available to `if1` before execution, unselected branches not
  checked, false in runtime preparation, and no argument-content access early;
- capture and forwarding without runtime execution or cleanup-state changes,
  parenthesized text extraction, and rejection of invalid query operands;
- runtime-global arguments classified as expressions without reading storage;
- captured family references using the printer's concrete names, insertion
  under a different namespace resolving the same concrete declaration, unused
  text creating no runtime demand, and standalone HUC0 containing all needed
  definitions without depending on the HUC1 reverse-name table;
- represented-object methods rejected in compiler execution, while the same
  method calls in generated text are checked and executed normally;
- expression text retained after its builder returns, later insertion using
  ordinary lookup, and successful same-type shadowing without hidden bindings;
- generated one-time bindings, repeated insertion with repeated side effects,
  and normal Advanced temporary ownership and destruction;
- HUC1 rejection of numbered syntax in inserted text and independent HUC0
  rejection if such syntax is supplied directly;
- acceptance of every successful flattened translation unit by the standalone HUC0
  translator.

Tests verify relocated values through the active destination, not by reading
inactive source fields or requiring their old bytes to remain intact.

### 17.4 ABI and cost assertions

Compare LLVM data-layout results with HUC's size, alignment, and field-offset
calculations. Test the supported foreign ABI with independently compiled C
fixtures; those fixtures are tests, not a HUC backend. Active-state flags must
not change a nominal value's layout.

For every supported target, test arguments and results in both directions
(HUC calling C and C calling HUC), including `bool`, signed/unsigned 8- and
16-bit integers, pointers, and `f32`/`f64`. Use negative signed values and values
with their highest bit set to expose extension mistakes. Build caller and
callee separately and test unoptimized and optimized configurations so inlining
cannot hide a mismatched boundary. Cover the hosted entry-point adapter too.

Inspect optimized assembly for representative Basic copies, Advanced
transfers, observations, and cleanup paths. Assembly shape is a regression
aid rather than a language guarantee; a transfer may be optimized away.

### 17.5 Differential backend tests

Verify and run generated LLVM IR at unoptimized and optimized settings, on
each supported target. Compare results with HUC semantic tests and evaluator
results where compiler execution is defined. When another emitter exists,
run the same MIR through both as well. No backend or interpreter becomes an
oracle for undefined HUC behavior.

The evaluator and backend share semantic analysis and MIR, so agreement alone
does not prove correctness. Retain independently specified expected results,
observable cleanup counters, and the C ABI fixtures above. Include strict
floating-point cases that distinguish separate operations from fused or
reassociated ones, and valid boundary cases for flagged integer/pointer IR.

Build the C++20 compiler itself with ASan and UBSan to test its implementation.
Sanitizing generated HUC programs is separate: ASan needs the LLVM pass, IR
annotations, and target runtime; UBSan-style checks need HUC instrumentation
before optimization exploits undefined operations. Clang's
`-fsanitize=undefined` on already-generated IR does not supply its missing
source-level checks.
Record generated-program sanitizer support separately from compiler build flags.

## 18. Implementation workstreams and milestone choices

### HUC0 to LLVM

Within the chosen runtime subset, the main dependencies are:

1. common C++20 project/source infrastructure, diagnostics, target data, and
   acyclic module discovery;
2. the standalone HUC0 lexer and recursive-descent/Pratt parser;
3. names, concrete types, permissions, constructors, overloads, Basic/Advanced
   classification, type aliases, fixed-signature C linkage, and lifecycle
   checking;
4. typed HIR and sequenced MIR with explicit parameter places,
   direct-construction destinations, relocation, and cleanup;
5. scalar LLVM IR generation, verification, and object emission;
6. explicit bitwise relocation and clone/drop
   functions, cleanup edges, sidecar active-state handling, and lifecycle
   hardening.

This workstream's chosen subset is complete when user-written `.huc0` programs
compile and run through LLVM, tests cover its supported lifecycle and evaluation
rules, and no HUC1 facility is required by `huc0_transpile`. It need not be
complete before work on staging begins.

### HUC1 to HUC0

This workstream reuses the HUC0 source/parser, semantic analysis, HIR, and MIR
above for the features it evaluates. The ordinary-function interpreter runs
that MIR, extended with compiler-only values; it does not require LLVM emission.
Staged indexing can start earlier, but full evaluation is not an independent
frontend. Native output remains a separate integration gate on a staging-first
route.

For a staging-first milestone or the staging side of an end-to-end slice:

1. staged module indexing, `import1`, opaque deferred-body spans, phase-free
   pass-through, flattened HUC0 namespace sections, and origin maps;
2. the deterministic compile-time interpreter with resource and effect limits;
3. `if1`, three-part `for1`, bounded phase loops, selected-body parsing, and
   module/type/function/block insertion contexts;
4. generic `struct`, generic `fn`, and generic `let` primary/partial/full family selection with
   angle patterns, deduction, and recursion states;
5. `@`, forced evaluation, `compiler::Code` values, `#` insertion with holes,
   argument-driven bindings, `compiler::is_expr_param`, and `compiler::expr` text extraction,
   `compiler::emitting()`, `compiler::error`, generation modes, and composed diagnostics.

The HUC1 frontend loads a versioned, inspectable generated interface for the
compiler module before indexing user modules. This is the implementation
equivalent of an always-included header; `import1 compiler` explicitly binds
it. Its implemented interface exposes only the supported compiler APIs; code
values, diagnostics, mode queries, and compile-time allocations can be added
as their subsets are implemented. It never exposes raw host pointers to
internal compiler objects.

The staging subset is complete when every accepted numbered/compiler-only
construct is removed and flattened output is deterministic, readable, and
checked with the shared HUC0 frontend. A first staging milestone may stop
there. End-to-end completion additionally requires that output to compile and
run through the public LLVM backend without a compatibility mode.

Reflection and structured generation may be designed before either translator
is complete. They need explicit specifications and support entries; the first
release is not obliged to implement them. No fixed milestone number reserves
or excludes them.

## 19. First implementation subset

The exact release 0.1 subset is not fixed. Prefer a thin end-to-end first slice:

```text
HUC1 indexing, selected phase control, and scalar evaluation
printable HUC0 functions, locals, calls, and basic control flow
shared HUC0 checking and concrete HIR/MIR
LLVM IR, object emission, and linking
```

Include scalar arithmetic, branches, a hosted entry point, and one
fixed-signature C call. Test an ordinary function both through `@` and as
compiled code, also checking an independently specified expected result. This
exercises the common semantic representation and the native boundary early.

A staging-first alternative can produce ordinary scalar HUC0 from selected
phase control and simple specialization, checking it against expected output
before object emission exists. It still needs shared semantic analysis and
HIR/MIR for any ordinary functions it evaluates.

Grow either route with explicitly chosen features, such as:

```text
T* and independent mod layers
constructors and direct destination construction
Basic/Advanced classification
destructive relocation, clone, and drop
left-to-right typed parameter binding
```

These are examples, not a fixed release checklist. Track each feature as
specified, implemented, and tested, including target limitations. Unsupported
features must fail clearly rather than inherit accidental backend behavior.
Both routes use the same written HUC0 boundary and the same source semantics.

## 20. Architectural invariants

The following should be enforced in code review and tests:

1. No HUC phase rule is delegated to LLVM or a foreign compiler.
2. No ownership decision is inferred from LLVM storage or pointer types.
3. HIR records Copy, Relocate, Observe, and Clone explicitly.
4. MIR is fully concrete and contains no phase construct.
5. Compile-time evaluation has no ambient untracked host effects.
6. HUC1 produces written flattened HUC0; HUC0 never consumes a private HUC1
   AST or HIR.
7. Text generation uses parsed, forward-only insertion in program mode or
   evaluation mode; no structural-generation API is implied.
8. Backend output preserves HUC left-to-right argument evaluation and
   by-value parameter initialization.
9. Source inactivity and cleanup state do not add fields to nominal value types.
10. No checked use of a root whose tracked state is `Inactive` or
    `MaybeActive` is accepted; this diagnostic is never optional. Lifetime
    changes through raw-pointer aliases remain outside that guarantee.
11. Diagnostics retain original/generated spans and origin chains across both
    translators.
12. Alternative backends can consume MIR without recreating semantic analysis.
13. A partial implementation reports its subset and does not claim full
    specification conformance merely because its end-to-end pipeline works.
