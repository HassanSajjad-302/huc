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
- **Module**: one source file, with its own namespace, name lookup, and
  private declarations; not a separate compilation unit.
- **Visible module**: a module that a file may name: itself, its direct
  imports, and modules re-exported along `pub import` edges.
- **Compiler state**: the active module's values and lifetime records for
  logical compiler objects; imports merge completed states without changing them.
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
    | huc0_compile
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

`huc0_compile` accepts source files, not an HUC1 AST or an undocumented
in-memory residual IR. `huc build` must first write the HUC0 artifacts and then
pass those paths to `huc0_compile`. This boundary is both a correctness test
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

struct Huc0CompileRequest {
    std::string root_module;
    bool flattened = false; // a complete flattened program
    std::vector<std::filesystem::path> import_roots;
    std::filesystem::path output_directory;
    TargetConfig target;
    bool write_source_maps = true;
};

struct Huc0CompileResult {
    std::vector<Artifact> artifacts;
    std::vector<Diagnostic> diagnostics;
    DependencyGraph dependencies;
    bool succeeded = false;
};

[[nodiscard]] Huc0CompileResult
huc0_compile(const Huc0CompileRequest& request);

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
`Huc0CompileRequest`. `ModuleId` identifies a canonical namespace, not an
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
    HUC_TOKEN_KW_PUB,
    HUC_TOKEN_KW_MUT,

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

Variable and parameter parsers read an optional binding `mut`, then the name,
colon, and type. A `mut` inside that type is a separate pointee permission.
`let` and `let1` precede variable declarations but not parameters. There is no
expression-parameter marker. Compiler calls classify their arguments into
value or expression bindings; record this on each call, not the declaration's
AST/signature and not by guessing from body uses of `compiler::expr`. Phase binders
also use `name: Type` or `name: auto`; they have no binding `mut`. All phase
families, including variable families, put their name before an angle binder list. Both
`for` and `for1` read three optional clauses separated by two semicolons.
The initializer is one binding or expression; reuse the local-binding parser
without its statement semicolon. Ordinary `for` accepts `let`, including
compiler-executed ordinary loops; `for1` accepts `let1`. The condition is one
expression and the update clause is an ordered list of expressions. Keep that
list as a loop AST field, not a comma-expression node. A header binding has a
loop scope with a nested body scope. `for1` requires compiler-only clauses and
retains a deferred body. There is no range-based loop production.

Parse an optional attribute list directly after `struct`, `fn` (including
`init` and foreign declarations), ordinary `let`, `if`, `else`, `while`, and
`for`, and before a parameter. Keep the attributes as an AST field of the
declaration or statement; attribute names are not keywords and are resolved
against the built-in list, not the symbol table. In HUC0, an argument is an integer
literal or an option name; `align` also accepts the layout-expression grammar.
In HUC1, it is an expression, and an entry may be
`#expression`, ended by the first top-level `,` or `]` after a complete
expression. After `fn` or `struct`, skip a balanced list to find the name and
check for `<`. Such a family keeps its list as an opaque span next to its
deferred body. A generic `let` has only its name and angle header before the
required block. Reject attributes and binding `mut` on that outer header;
they belong to the inner ordinary declaration. Do not evaluate an invalid
outer attribute list or defer its rejection until the family is requested.

Use a shared list parser that accepts one optional trailing comma, including
for constructor initializer entries and loop updates. Do not make acceptance
depend on whether the list spans several lines. Empty items remain errors. Ordinary control
flow parses braced bodies, with an `else if` alternative for chained
conditionals; phase-control bodies still use the deferred-body scanner.
Do not retain the old declaration order or `T&` as compatibility spellings.

`construct_at(pointer, value)` and `destruct_at(pointer)`
use ordinary call syntax with compiler-known, non-overloadable, unqualified
names. Resolve them as intrinsics, not `std::` library calls. `as`, `ptr_as`,
`size_of`, `align_of`, `is_basic`, `is_relocatable`, `is_copyable`, and
`is_default_initializable` use HUC0 type-operand calls.
Parse interleaved `*` and `[N]` suffixes left to right. HUC0 counts accept
literals and layout queries combined with `+`, `*`, and parentheses;
HUC1 counts accept compile-time expressions. Array literals are nonempty
comma-separated lists in brackets. Attributes follow declaration/statement
keywords, so an expression beginning with `[` is unambiguous. Constructor
entries use `field = expression`. There is no array constructor call syntax.
Address-taking uses unary `&`.

`take` is a reserved unary keyword in both parsers, with `copy`'s precedence.
`drop` and `forget` are reserved statement keywords that take one name. Because
`copy` and `drop` are reserved, the lifecycle method declarations `fn copy()`
and `fn drop()` use their own grammar form, like `fn init`.

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
though they share binder, pattern, and deferred-body components. A family
declaration's attribute list is also an opaque span. Parse it only with the
selected body, after the binders are bound, so its arguments may name them.
Prepare an attribute list with its declaration or statement: evaluate integer
expressions and run `#` entries left to right in the preparation's state,
merge the results, check names, placement, conflicts, and ranges, and print
literal arguments in HUC0. Omit a list that ends up empty.
The syntax tree also has a phase-expression item node for standalone
`@expression;` and an insertion item node for `#expression;`, both at module,
type-body, function, or block scope. A phase-expression item must evaluate to
compiler `void` or `never` and inserts nothing. A compiler assignment item
node holds a module-level or type-body assignment to compiler storage, such as
`defaults::checks = false;`; the compiler executes it in the active state,
in source order or while the type body is prepared, and erases it. An
insertion item's expression must produce `compiler::Code`; preparation erases
the node, inserts the code's local part at that position, queues its module
part, and processes inserted text before the following source. When the code
has a hole, the following braced source block fills it. Do not check that
block's loop context prematurely. Reject numbered syntax, `@`, and `#` in
inserted HUC0; insertion does not start another staging pass.

Ordinary `import` exposes a module's `pub` declarations of every kind, under
their phase rules; `import1` is compiler-only; `pub import` also re-exports
the module. Keep one module-loading table for the translation unit:

| State | Action when imported |
|---|---|
| Unseen | Create its record and begin processing |
| Loading | Diagnose an import cycle, including the import chain |
| Loaded | Reference the existing record and bind an alias unless `as _` was used |
| Failed | Report the dependency failure; do not construct a partial second copy |

Resolve and process a source at its first import, depth-first, then restore
the importer's lexical context. A later import from another file references
the existing record and replays no source or compiler effects. Within one
file, accept imports only in the leading block after the first `module`
declaration, and diagnose a second import of the same resolved module by any
form or alias. Record each file's direct imports and re-export edges; they
determine its visible modules (section 6). Flattened output resolves away
imports and uses canonical references and `module name;` namespace sections.
Keep imported buffers and import chains rather than copying bytes or losing
their original locations.

## 6. Semantic analysis

HUC0 performs the following runtime semantic analysis. HUC1 reuses these rules
for prepared source and parsed generated text:

1. collect declarations and module imports;
2. resolve names, checking module visibility, `pub`, and namespace ownership;
3. resolve concrete types and layouts;
4. enforce permissions, constructors, and special-operation rules;
5. build concrete typed HIR;
6. make execution order, relocation, and cleanup explicit in MIR.

Name resolution uses one global declaration registry but a separate lookup
environment per source file. A qualified name must start from a visible
module of the file that contains it, and a declaration from another file must
be `pub`. A later namespace section may reopen the file's own namespace, or a
visible module's namespace only to add specializations. Default `lower`
applies these source rules to handwritten HUC0 too; `lower --flattened`
skips them for one complete flattened program.

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
    HucSourceFileId owner_file;
    bool is_pub;
    HucSourceSpan definition;
} HucSymbol;
```

`owner_file` is the source file whose processing declared the symbol. It
decides privacy and which files may specialize a family or type. A symbol
inserted by `#` belongs to the file of that `#`.

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
permissions. HUC has no `const` type qualifier: fixedness and `mut` are
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
Check type compatibility and permit dropping pointee `mut`, but never
gaining it. `auto` retains the inferred raw-pointer type.

Member access preserves each layer's permissions. A fixed object prevents
replacement of a pointer field and mutation of an inline field. It does
not remove a field type's `mut` permission to mutate the separate object
that the field points to.

`take` requires a place. For a Basic type, type-check an ordinary read only:
no writable access, address, or default constructor is needed. Bit-fields and
under-aligned Basic fields retain their existing load rules. For an Advanced
type, require writable addressable storage and a valid `T()`; apply existing
named-root state checks. The actual indirect slot lifetime remains unchecked.

Each concrete type records:

- size and alignment, including the effect of `align`, `pack`, and `bits`;
- its explicit alignment from `align`, including through inline fields, so
  `pack` can reject a cap below it;
- type category (Basic or Advanced);
- whether it requires drop;
- field layout;
- phase class;
- a canonical identity for type comparison and specialization arguments.

`construct_at` requires exactly two operands and `destruct_at` one. Infer a
complete runtime object type `T` from `mut T*`; reject read-only/non-pointer
operands, invalid pointee types, and first-class or overload declarations for
these intrinsics. Pointer-slot pointees are allowed. Neither operation
has a HUC0 angle-delimited type operand.

Type-check the second operand of `construct_at` as an initializer for `T`,
not as an ordinary call parameter. Apply normal built-in conversions,
Basic/Advanced classification, source permissions, and relocation-state
checks. Both intrinsics return `void`. Capacity, alignment, and source
lifetime preconditions are unchecked. `construct_at` ends an existing
destination lifetime without cleanup; `destruct_at` requires a live value.

Intern `T[N]` by canonical element type and count. Require complete runtime
elements, positive counts, and representable target layouts. Size is
`N * size_of<T>()`; alignment and Basic/Advanced category follow `T`.
Array literals use an expected array type or infer their default type from
the elements. Validate counts and per-element initialization. There is no
implicit conversion between existing array values of different element types.

Keep unsuffixed numeric literals untyped until context or their default fixes
the type. Typed operands decide before destination context. Fold maximal
literal-only integer arithmetic exactly, then check the result's fit; bit
operations use inferred widths. Find lossless common numeric types, handle
mixed-sign integer comparisons without lossy conversion, and type shift
amounts independently. Literal overload candidates must pass fit and count
checks before ranking. No inference crosses statement boundaries.

Validate `[nozero]` only for scalars and nested scalar arrays. Suppress
omitted writes, not explicit initializers or fixed-field obligations. Generate
memberwise and eligible zero-argument inits only for structures with no
explicit `init`; preserve field order, declaration initializers, and mutable
Advanced parameter bindings. See specification section 4.6.

Byte arrays are Basic storage regions, not opaque Advanced wrappers. Typed
pointers may traverse their whole region without payloads already being live.
The programmer manages payload construction and destruction; no runtime
payload map is needed. Enclosing Advanced relocation transfers live payloads
at matching offsets; byte copying alone does not.

Compute a structure's layout from declaration order, natural alignments, and
its `align` and `pack` attributes (specification section 4.9.8). For a
structure with a `bits` field, build an equivalent C record in Clang, with
each bit-field typed as its C integer type. Use corresponding target C types
for other built-ins and layout-only record/array descriptions for concrete
HUC structures and fixed arrays. Do not translate their lifecycle methods.
Preserve natural alignment and explicit alignment separately, including
through nested types; size and alignment alone are not a sufficient layout
description. An aligned byte-array replacement can wrongly introduce
packing-resistant alignment on the Microsoft ABI. Apply maximum field
alignment for `pack`, and explicit alignment only where HUC has it.

Read Clang's record layout for offsets, bit positions, size, and alignment.
Validate that each layout description also preserves embedding behavior under
packing. Diagnose an unsupported representation rather than approximate it.
Test against independently declared C records on each supported ABI, including
packed records nested in other records. Mark fields that `pack` leaves
under-aligned, and reject `&`, method calls, Advanced types, and `init` on
them. Reject `&` on bit-fields.

Fold `size_of<T>()` and `align_of<T>()` to target `usize`, and `is_basic<T>()`,
`is_relocatable<T>()`, `is_copyable<T>()`, and `is_default_initializable<T>()`
to `bool`, with no runtime operands. HUC1 evaluation and HUC0 lowering must
use the same target layout and classification rules; retain the target layout
inputs in the translation context. A query evaluated during preparation
about a structure whose body is still being processed is a diagnostic. HUC1
resolves an array count and prints an integer literal in residual HUC0.
Runtime counts are not valid type operands.

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
arithmetic scalars, `void` or `never` results, and raw pointers, including
pointer chains, to built-in types. A `never` result maps to a C `void` result
with a non-returning contract; `never` parameters remain invalid. It rejects
structures, fixed arrays, Advanced/drop types, phase types, compiler metadata,
opaque foreign handles, and variadics. Aliases are checked after
canonicalization so they cannot hide a forbidden boundary type.

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

Basic and Advanced are type categories; `Copy`, `LogicalCopy`, and
`Relocate` name compiler operations, not categories. Same-type Basic binding
uses `Copy`. An inline Advanced transfer uses `Relocate`: the destination
becomes active and the source becomes inactive without receiving `drop`.
Explicit duplication through a type's `copy` method, written or generated,
uses `LogicalCopy`.

Representative HIR operations:

```text
LoadCopy(place)
LoadRelocate(place)
Take(place)
AddressOf(place)
SlotAddress(place)
PointerFromAddress(address, raw_pointer_type)
LogicalCopy(value)
ConstructInPlace(destination, constructor, arguments)
InitializeAt(pointer, initializer)
DestructAt(pointer)
DropLocal(root)
ForgetLocal(root)
Call(function, arguments)
Replace(destination, rhs_preparation_plan)
```

`ConstructInPlace` is not shorthand for “construct a temporary, then
relocate.” When `T(arguments)` initializes a local, field, by-value parameter,
or return result, HIR names that final place as the constructor destination.
This preserves the final `this` address for constructors and avoids an observable
relocation. Direct calls and Advanced copies likewise use the final result
place, forwarding it through nested direct returns even without inlining.
Array literals build in the final element slots. A conditional forwards the
place recursively to its selected arm, including its own temporary outside
direct-result positions. An explicit named local still has a
separate place and is relocated; no named-return optimization is guaranteed.

`Replace` retains the RHS form rather than first materializing a complete
replacement. Bind callee/receiver and arguments, or retain the copy/relocation
source, before dropping the destination. Only then run the outer call,
constructor, or copy into its slot, or relocate the retained source.
Conditionals evaluate the condition once and retain only the selected arm's
preparation plan, recursively. Array literals prepare all element inputs
first, then drop old elements in reverse and construct new ones forward.
Other expressions are fully evaluated first. Preserve
this distinction through inlining. Track ordinary source uses during
preparation; the later operation uses retained operands with unchecked lifetime
preconditions, not a new required alias/dependency check.

HIR records whether a source is a tracked root, a pointer dereference, or a
direct subobject. Transferring from a named root requires `mut` before its
name; extraction through a pointer requires pointee `mut` instead. A fresh
function result, explicit `copy` result, or
unavoidable constructor temporary can be transferred once without a source
declaration marked `mut`. Direct construction in the destination remains
preferred and creates no such temporary.

Relocation accepts a whole tracked local, parameter, temporary, or result, or
an explicit dereference `*pointer` through `mut T*`, or a writable field or
element. These forms transfer the whole source value. HIR records the cleanup
mode: a tracked root updates compiler-managed source state, while pointer,
field, and element extraction rely on the programmer to prevent cleanup of
the inactive source slot. Evaluate the source place once without changing
any pointer's address value.

Direct `value.field`, `items[i]`, and `pointer->field` sources lower with
the same permissions and cleanup mode as `*(&place)`. Evaluate the place
once. HUC does not track partially inactive aggregates; extraction does not
suppress the enclosing object's field/element cleanup. Do not reject a typed
runtime program solely because extraction leaves an inactive slot for later
use or cleanup. Optional warnings are separate; existing whole-root and
permission errors remain.
Neither pointer provenance nor stack/heap allocation selects the cleanup mode.
In particular, `*(&local)` does not acquire the direct local's cleanup behavior.
Preserve this distinction through inlining and other optimizations.

Indirect destinations may be replaced when their validity and activity
preconditions hold. Containers manage their backing storage and initialized
element ranges explicitly.

`Take` of a Basic type lowers to `LoadCopy`, including ordinary bit-field or
under-aligned loads. It leaves the source unchanged. For Advanced `T`, retain
the place address, relocate the old value into a separate result, then
`ConstructInPlace` with `T()` in the old slot before yielding that result.
Do not drop the old value, clear its bytes, or change automatic source cleanup
state. The constructor sees the source slot's actual address. The result has
normal temporary cleanup if it is not consumed.

`Replace` treats Advanced `Take` as a fully prepared RHS, not a delayed
constructor or call. Do not relocate directly into an assignment destination
before its old value is destroyed: it may be the very slot being taken from.
`x = take x` reconstructs `x`, drops that replacement, then transfers the
prepared result back. These operations must keep their order without inlining.

`InitializeAt` represents `construct_at`: evaluate the pointer once, then
initialize its slot from the second operand using `Copy`, `LogicalCopy`,
`Relocate`, or destination construction. The result is `void`. Do not create
a by-value initializer parameter or drop an old destination. `DestructAt`
performs complete destruction at a retained pointer, without deallocation.
Both leave automatic cleanup state of aliased roots unchanged. A directly
tracked source consumed by `Relocate` still updates its own state normally.

`DropLocal` and `ForgetLocal` represent the `drop` and `forget` statements on
a tracked root. `DropLocal` performs the same complete destruction as scope
exit; `ForgetLocal` performs none. Both make the root Inactive, like a
transfer, so scope exit skips it and later named uses are rejected.

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
string literal, or a `compiler::allocate` allocation. A pointer is a logical
block ID plus a path to a slot or subobject, resolved in the active compiler
state, never a host address. Each block version records
its type, written bytes, and live objects, including payloads constructed in
byte regions. Every dereference, `construct_at`, `destruct_at`, extraction,
and deallocation is checked against specification section 12.1. Constructing
over a live value ends it without cleanup after input preparation; this is
not an evaluator error. Relocating an enclosing object that carries a live
`[norelocate]` payload, and `drop` or `forget` of a root whose lifetime
already ended, are evaluator errors. Whole-aggregate transfers carry
unwritten bytes without typed reads. Pointer arithmetic follows
array/allocation slots or the whole byte-storage region, including one-past
addresses. A pointer to
any other object behaves like a pointer into a one-element sequence. Ordering pointers into different blocks
and converting a pointer to an integer are diagnostics. Actual `u8` and `c8`
objects are ordinary values; viewing an object of another type through `u8*`
or `c8*` remains a diagnostic until representation inspection is supported.
This does not prohibit constructing and using typed objects in byte-array
backing storage; that only needs target offsets and typed payload records. Integers use
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
`destruct_at`, `take`, and pointer/field/element extraction, and compile-time
allocation. Execute them over the checked block model of section 8.2.
Deferred compiler calls additionally
keep their compiler-only value kinds, compiler intrinsics, and `let1 mut`
assignment; their runtime requests must still reject compiler-only signatures
and residual operations.

Support local mutation, control flow, and nested calls with shared instruction
and recursion budgets. Apply the same rules in callees, including concrete
generic `fn` bodies. Check executed operations, not merely the outer
signature. An untaken ordinary branch may contain an unsupported operation,
but must still pass ordinary syntax and type checking. Never invoke arbitrary
native or foreign code. Reject accesses to runtime storage, an ordinary
function naming `let1 mut` storage, and untracked external inputs. An ordinary
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
even if its declaration includes binding `mut`.
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

Keep a function-local `let1` alive until its program-mode preparation completes
or its evaluation-mode call finishes executing. Module-level bindings belong
to the active module's compiler state, indexed by stable declaration identity.
Allocation IDs identify logical blocks, not host addresses. The current state
maps each ID to a complete block version: values, alias targets, initialization
and lifetime status, and cleanup information.

Start a module with only its own dependencies. On each direct import, merge
the dependency's completed state into the importing state in source order.
Replace conflicting block entries with the later version, even if unchanged.
Keep nonconflicting entries. Treat a structure or allocation as a whole block,
not a collection of independently merged fields. Distinct allocations remain
distinct; imported paths to one shared allocation retain one logical ID.

An import does not run language-level assignment, relocation, a `copy`
method, or `drop`. The implementation may overwrite a parent map entry,
share immutable block versions, and detach on a write. It must never
overwrite the saved dependency version in place. Preserve aliases, private
supporting objects, inactive slots, and deallocation records. Prune
unselected versions only when no surviving entry or pointer needs them;
discarding compiler metadata is not a user-language leak or cleanup event.

All evaluator reads and writes resolve through the active state. Calls inherit
that state even when the callee is imported; lexical lookup still resolves names
in the callee's source. No effect or caching analysis is needed. Check mutation
permissions and lifetime validity exactly as for local compiler objects.
Executing HUC assignment or destruction performs the usual cleanup in that
state; it cannot change another module's completed state.

A program-mode definition or canonical type is prepared from its defining
module's active state, or a private working view of its completed state after
the module finishes. Each distinct late preparation gets a fresh view, never
a view changed by a previous preparation. Compiler calls share the enclosing
view; a nested request for another retained definition gets its own. Recursive
references to in-progress definitions do not create new preparations. Retain
views through dependent work and queued insertion, then clean them privately.
It does not inherit the requester's merged values.
Evaluation-mode preparation and execution inherit the active caller state.
Retain already prepared runtime definitions without adding state to mangled
names. Preserve these context distinctions through nested requests and delayed
generation; insertion records retain their actual insertion context.

For a resolved module-level object missing from the active state, consult its
declaring module's completed state through a read-only fallback. Supporting
allocation records can be borrowed from the same provider. Never bypass an
active entry, including an inactive or deallocated one, and never borrow from
an unfinished module or from a sibling's modified state. Missing or invalid
saved entries diagnose with the specialization/evaluation chain.

Track borrowed provider entries separately from owned working-state entries.
Every mutating operation, including relocation, lifetime changes, and writes
through aliases, checks the read-only flag. Retain provider references for
borrowed pointers. Borrowing creates no import, visibility, merge entry, or
cleanup obligation; record it as a dependency for future artifact validation.

At the end of translation, visit completed module states in reverse processing
order. In a private working view of each, destroy all remaining live module-level
values, including inherited values, in reverse recorded initialization order,
then check for leaks. Separate states have separate lifetimes. Cleanup may
write accessible mutable objects within that view but never publishes changes
back to other states. Saved results used for imports are the pre-teardown
states. Call-local and otherwise unreachable leaked allocations keep the
existing diagnostic rules.

Per-state teardown can repeat inherited cleanup many times. Sharing backing
blocks is not permission to skip user destruction or its diagnostics. There
is no new cleanup-sharing analysis. Avoid destructor work for Basic values;
report the affected state on a cleanup failure. Stop at the first failure or
group equivalent diagnostics without accepting a failed translation.

Fix each module's names when it finishes. For declarations generated later,
record the preparations that generated them (including identical-declaration
sharing). Such names are visible only within those preparations; state merging
does not broaden their visibility. Evaluation-mode preparation suppresses module
code and does not join that set.

Future package serialization preserves per-module state, shared logical object
IDs, aliases, lifetime records, and existing prepared definitions. Its content
fingerprints include those values and ordered dependencies. Source and saved
imports must merge identically. Ordinary compiler-call caching remains disabled.

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

Every generic `let` uses a selection block, including empty-binder primaries
and partial/full specializations. Reject direct-declaration shorthand and
require the family's name on the inner `let`. The outer header has no
attributes, binding `mut`, type, or initializer. Prepare the inner ordinary
`let` and its attributes when reached, using the bound parameters and compiler
locals available there. Apply the normal attribute checks to that one list;
no header/inner merging is needed. The ban on `#` in the selection body also
covers attribute entries there.

## 9. Specialization engine

HUC1 indexes one primary family per name and scope for generic `struct` and generic `let`,
and one primary family per name and scope for generic `fn`. Syntax is retained
in these distinct parts:

```huc
struct Buffer<T: auto, N: usize> { ... } // primary binders
struct Buffer<T: auto, N: usize><T*, N> { ... }
struct Buffer<><Widget, 8> { ... } // full specialization

fn convert<T: auto>(mut value: T) -> T { ... }
fn convert<T: auto><T*>(value: T*) -> T* { ... }
fn convert<><Widget>(mut value: Widget) -> Widget { ... }

let selected<T: auto> { let selected: T = T(); }
let selected<T: auto><T*> { let mut selected: T* = null; }
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

Patterns may contain concrete types and values, binders, pointers whose `mut`
must agree, family applications, array suffixes `T[N]`, repeated binders, and
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
specialization in the primary's namespace. Check placement locally at each
specialization: its file must declare the family, or declare a type in the
pattern that comes before every binder in a left-to-right walk of the
pattern, including nested type arguments. Use canonical types, so an alias
gives no ownership, and give built-in types no owner. When a file finishes,
compare each specialization it declared with the same family's
specializations from the files it depends on. Two that unify, with neither
matching a strict subset of the other and no specialization for exactly
their overlap, are a diagnostic in the finishing file. Only the selected body
is parsed.
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
Substitution carries canonical type identity. Body names resolve in the
source context of the file that wrote the selected definition: the family's
file for the primary, or the owning file for a specialization. That file's
import environment, aliases, and private declarations apply, and its `#`
statements belong to it. Unqualified names resolve in the family's
namespace. The concrete function keeps the family's namespace and
declaration position in the shared translation unit.
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
#compiler::local("let mut generated: i32;");
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
function that inserted it, unless it was generated after its module
finished (section 8.6). If completing an active anchor requires evaluating
its pending code, diagnose the dependency cycle rather than draining the queue
early.

Share HUC0 lexing/parsing and semantic rules with HUC1's inserted-code path.
Reject numbered constructs, `import1`, `@`, and `#` in generated HUC0 during
HUC1 preparation. Resolve generated and original source together, allowing
module forward references but rejecting changes to finalized compiler
decisions. A `#` belongs to the file that wrote its body; reject module code
from a `#` in a specialization written in another file's namespace. Mark each
generated span so that its module paths are resolved
as canonical names, without consulting import aliases, and only through
modules visible to the file that owns the `#` site. Generated code may name
only `pub` declarations of other files. Declarations it inserts belong to the
`#` site's file and are private unless the text says `pub`. A block that
fills a hole is authored code and keeps its file's aliases. Print only
checked program-mode output into `.huc0` artifacts; the standalone HUC0
translator independently validates those files.

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
LogicalCopy destination, source
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
active-state metadata. A pointer, field, or element source ends the slot's
lifetime but does not search for or alter cleanup metadata of aliased roots
or enclosing objects. This needs no ownership registry, hidden pointer
metadata, or per-field/element flags. The destination is active in either case.
Backend temporaries for the source address must not create a second owning value.

`Relocate` and Advanced `LogicalCopy` require different, non-overlapping
source and destination storage. Basic `Copy` retains read-then-store
self-alias behavior.
Advanced assignment adds no identity comparison or no-op
branch. Even direct self-assignment is accepted under the normal static rules,
but has undefined execution. Optional warnings or sanitizer instrumentation
are separate from ordinary lowering.

`ConstructInPlace` makes an inactive final destination active only after its
constructor obligations succeed. `Call` consumes already bound parameter
places in declaration order; its result destination is the caller's final
return place, so a structure result is constructed there rather than returned
through an observable temporary. `Drop` makes its place inactive. These state
transitions are MIR semantics, not optional annotations for the backend.

Lower the retained assignment plan in order: prepare inputs, drop the old
destination according to its post-preparation state, then `ConstructInPlace`,
`Call`, `LogicalCopy`, or `Relocate` into that destination. A source
temporary for `copy make_source()` survives through the copy. A nested copy
argument
completes before drop, while a selected conditional arm follows its own
preparation plan and forwards the slot. Source deactivation is recorded
before destination activation, so a self-relocation root ends Active in the
flow model without giving its undefined execution a meaning.

Lower `InitializeAt` by preparing inputs, ending any old lifetime without
cleanup, then initializing the retained indirect destination. Use the existing
value operations, but return `void` and register no automatic cleanup for that
slot. Do not end enclosing structure or byte-array lifetimes. Lower `DestructAt` to `Drop` of an indirect manual place with
its required-live precondition. Do not consult or update aliased root flags,
even for `&local`; preserve that cleanup mode through inlining. Basic
destruction needs no runtime call but ends the pointee lifetime in HUC.
Neither operation adds allocation/deallocation or source/pointer-clearing
code. Invalid runtime preconditions are not required to be checked.

Array initialization builds each element in increasing index order. Array
copying does likewise; destruction goes in reverse. `[nozero]` omits the
default writes of a scalar or scalar-array declaration. Byte storage has no
payload cleanup. Enclosing Advanced relocation transfers payload lifetimes
and unwritten bytes without scanning them or reading inactive values. The
evaluator tracks these lifetimes for host safety; runtime MIR needs no
per-payload flags. Library length or tag state controls explicit cleanup.

## 12. Lifetime and drop elaboration

### 12.1 Place state

The pass tracks whether each compiler-managed root place of an
Advanced-classified type, and each Basic root named by `drop` or `forget`, is:

```text
Inactive
Active
MaybeActive
```

This analysis arranges exactly-once destruction for tracked roots, provided
unchecked pointer, field, and element operations respect their cleanup
contracts. Extraction does not update root state; programmer-managed
storage uses its own initialized-range or equivalent bookkeeping. Attempting
cleanup of an inactive extracted slot is undefined behavior, not a request
to discover and suppress that cleanup at runtime.

The same state drives the required relocation-state check (language
specification section 6.12). Any source-level use of a tracked root that is
`Inactive` or `MaybeActive` at that point is a diagnostic in every compilation
mode. HIR carries those uses into MIR: copying, relocating, taking an address,
field projection, method-receiver formation, or storing into a field of the
root. A field's root is checked when its destination is formed and after RHS
preparation, before replacement. Storing a whole new value, scope-exit
`Drop`, and `SlotAddress` are not uses of the old value. Computing their
operands may still read values and must be checked. Field projections used
only to form `SlotAddress` through inline fields do not read the root.
MIR is already sequenced in HUC evaluation order, so later arguments are
checked against the state left by earlier parameter bindings.

Compiler-inserted destruction during replacement does not add a second
source-use check on already retained operands. Otherwise `x = copy x` would
become a required error instead of an unchecked precondition violation.
On a normal completion edge, record a tracked source's transfer before
marking the destination Active. This leaves `x = x` and `x = copy x` Active
in the flow model without validating their executions. Relocation into an
indirect destination makes the tracked source Inactive; possible aliasing
does not add a no-op edge that would produce MaybeActive.

The pass also rejects a `Relocate` from a root that happens during a call's
argument evaluation, including in a nested expression, in two cases. The
first is when the root is the root of the call's receiver place. The second
is when an earlier argument of that call is an `AddressOf` of the root or of
one of its fields. An address formed inside an earlier argument that does
not itself become an argument does not count. Constructor expressions and
intrinsic calls count as calls, including inside `field = expression`
initializers. Receiver roots, destination roots, and earlier `AddressOf`
arguments are recognized as specification section 6.12 describes.
HIR records the source spellings needed for this: parentheses, field
projections, `(&r)->f`, `*&r`, `ptr_as`, and either arm of `?:`. That
recognition does not change the cleanup mode of `*&r` (section 7).

These checks concern relocations already tracked during argument binding.
Following dependencies through later assignment destruction would require
new analysis, which is not required. Keep those existing checks without
turning `x = make_replacement(&x)` or `x = x.rebuilt()` into new required errors.

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

Semantic analysis classifies a structure as Basic only when every field is
Basic, it declares neither a `copy` method nor `drop`, and it is marked neither
`[nocopy]` nor `[norelocate]`. Each of these makes the structure Advanced.

It also computes three type properties. A type supports copying when it is
Basic, has a written or generated `copy` method, or is an array of such
elements. A type is non-relocatable when it is marked `[norelocate]`, has an
inline non-relocatable field, or is an array of such elements. A type can be
default-initialized when it is a scalar or raw pointer, a structure with a
written or generated zero-argument init, or an array of such elements. The
`is_copyable`, `is_relocatable`, and `is_default_initializable` queries
report these properties. Generate a structure's `copy` method only under
specification section 6.6: it is Advanced on its own, declares no `copy`
method or `drop`, is neither `[nocopy]` nor explicitly `[norelocate]`, and has
only copyable fields. Its body copies the named fields in declaration order
directly into the destination fields. Skip a generated memberwise init that
would relocate a non-relocatable field. The properties need no runtime
metadata and must survive HUC0 printing.

Elaboration handles the operations as follows:

- same-type Basic binding becomes fieldwise `Copy`;
- `copy` of an Advanced structure with a written or generated `copy` method
  becomes `LogicalCopy` into the copy expression's result place; a type that
  does not support copying, including a `[nocopy]` type, is a diagnostic.
  Direct copy assignment uses the destination slot after old-value
  destruction;
- copying `T*` copies only the observer address and permits null;
- any relocation of a non-relocatable type is a diagnostic, whatever its
  source; direct results still construct in their final place;
- binding from an Advanced value becomes one fixed representation-transfer
  `Relocate`, which intrinsically makes the source and all subobjects inactive
  without requiring source-field clearing; tracked roots update their cleanup
  state, while pointer, field, and element sources leave source cleanup to
  the programmer;
- a fresh unnamed Advanced result can be transferred directly;
  relocation from named root storage requires `mut` before its name;
- extraction through `*pointer` requires `mut T*`, not a writable pointer
  binding, and does not modify the pointer's address value;
- direct field and element extraction has the same meaning as `*(&place)`;
- `take` becomes a Basic copy or Advanced extraction into a separate result
  followed by default construction in the source slot;
- Advanced assignment retains a preparation plan, drops the old destination
  if it remains active, then performs the outer operation in its slot;
- `destination = copy source` retains source, drops destination, then runs
  `LogicalCopy` there. `destination = make(args)` binds args, drops
  destination, then runs the factory into that result slot;
- `drop name;` becomes `DropLocal` and `forget name;` becomes `ForgetLocal`.
  Both require a mutable, Active root; a Basic operand emits no code but still
  becomes Inactive;
- Basic assignment keeps its ordinary copy behavior, including self-aliasing;
- destruction calls user `drop`, then drops fields in reverse order.

`LogicalCopy` means an independent logical duplicate whose two active values
can be destroyed independently. A user `copy` method must duplicate owned
resources or
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
aggregate layouts. HUC pointer types and their `mut` permissions remain in
HIR/MIR even when LLVM represents their addresses with opaque `ptr` values.
Storage is allocated with the target-required alignment; the emitter must not
infer source ownership or permissions from an LLVM pointer.

Represent `T[N]` with the target layout of `[N x T]`, including nested
arrays. Emit no default writes for `[nozero]`. Byte arrays provide backing
regions for typed objects but add no payload cleanup or runtime metadata.
Aggregate copies and relocations must carry unwritten representation without
turning it into typed scalar reads or poison-dependent behavior. Use memory
transfers or equivalent lowering that preserves this contract. Only enclosing
Advanced relocation transfers payload lifetimes; a Basic byte copy does not.

### 13.2 Operation mapping

| HUC operation | LLVM lowering responsibility |
|---|---|
| `T*` / `mut T*` | Address values; HUC checks permissions before lowering |
| `&value` | Address of the retained storage place, materializing storage when required |
| Pointer arithmetic / field access | Target-scaled address calculation with HUC bounds and layout rules |
| Pointer/integer conversion | Target-supported conversion preserving HUC's specified address round trip |
| Basic binding | Scalar or aggregate copy; source remains active |
| Advanced relocation | Representation transfer without source clearing; update cleanup state only for tracked-root sources |
| `take place` | Basic read, or Advanced extraction followed by in-slot default construction; no source cleanup-state change |
| `copy value` | Resolved `copy` method call when required by the source type |
| `construct_at(pointer, value)` | Prepare inputs, end any old lifetime without cleanup, initialize in place; result is `void` |
| `destruct_at(pointer)` | Emit full drop/field cleanup without freeing storage or updating aliased root flags |
| Destruction | Explicit user `drop` followed by reverse field cleanup for active values |
| `T(arguments)` initializing a final place | Initializer call with the final destination address |
| Array literal / Advanced array copy | Direct element initialization or copying in increasing index order |
| `drop name;` / `forget name;` | Full cleanup at the statement, or none; the root becomes Inactive and scope exit skips it |
| Array destruction | Reverse element cleanup |
| `[nozero]` | Omit default writes; preserve aggregate representation-copy rules |
| Direct-result `?:` | Branch on the condition once and forward the destination to the selected arm |
| Layout and type queries | Constants from the shared target/type model |
| Under-aligned packed field access | Loads and stores with the field's actual alignment |
| Bit-field read or write | Load the storage unit, then shift and mask; a write keeps neighboring bits |
| `inline`, `noinline`, `cold` | `alwaysinline`, `noinline`, `cold`; report a call that cannot be inlined |
| `likely`, `unlikely`, `unroll(N)` | Branch weights or `llvm.expect`; `llvm.loop.unroll.count` loop metadata |
| `noalias` parameter | LLVM `noalias` |
| `callconv(gpu_kernel)`, `launch_bounds` | `ptx_kernel` or `amdgpu_kernel`; `maxntid`, `minctasm`, `maxclusterrank`, or AMDGPU flat work-group size |

Use loads/stores or memory-transfer intrinsics only where their alignment,
overlap, and initialization requirements fit the HUC operation. Advanced
assignment's distinct-source precondition needs no implicit address check.
Do not generalize it into a no-alias promise for all function parameters or
reachable resources. Neither a raw write nor a lifetime intrinsic performs
HUC destruction or updates ownership.

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

Lower an `extern "C"` result of `never` as LLVM `void` using the target's C
ABI, with `noreturn` on the declaration and call. End the call's block with
`unreachable`; do not synthesize a result value or a normal cleanup/return
edge after the call. A foreign implementation that returns violates the
source contract. `noreturn` does not imply `nounwind`; the separate ban on
foreign unwinding still applies.

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
not generated C headers. `pub` controls HUC name lookup, not linkage: only
the entry point and functions exported to C need symbols visible outside the
program, so other HUC definitions may use internal linkage.

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
huc lower --flattened <program.huc0> --out-dir <llvm-dir>
huc stage <root.huc1> --out-dir <huc0-dir>
huc build <root.huc1> --out-dir <build-dir>
```

`stage` produces printable HUC0; `lower` checks HUC0 and emits LLVM IR.
By default `lower` reads HUC0 sources with imports under the source rules.
`--flattened` selects one complete flattened program instead: it rejects
imports and skips the import, `pub`, namespace, and specialization-placement
checks, which were made before flattening. The option, not a file name or the
absence of imports, selects the input kind. `build` writes the staged source,
passes that file through the public HUC0 frontend with `--flattened`, emits
objects through LLVM, and links them. It never hands a private
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

Separately compiled packages come after 0.1. Their required rules, including
isolated completed module states, saved generic bodies, reuse of a dependency's
existing copies, and definition fingerprints, are set in the
[compilation model](compilation-model.md), section 13.
HUC1 package metadata preserves shared logical object IDs, aliases, and lifetime
records for each module's state. Saved and source imports merge identically.

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
- pattern forms: `mut`-sensitive pointers, pointer chains, family
  applications, array suffixes, repeated binders, and family binders, with
  strict-subset ranking and late-specialization errors;
- `let1 mut` assignment order within its module, preparation-time reads fixed
  into prepared bodies versus current-value reads in executing compiler code, a
  separate function-local `let1` per preparation, and `let1` redefinition errors;
- isolated state: imported objects may be changed by direct writes, pointers,
  methods, and compiler calls without changing saved dependencies or siblings;
- ordered imports: later complete values win, including unchanged values;
  unrelated names and independently allocated objects remain distinct;
- aliases and object merging: direct and pointer reads agree, entire structures
  and allocation blocks are selected, inactive/deallocated entries win too,
  and merging neither replays initializers nor calls copy methods or drop;
- staging contexts: compiler calls inherit the active caller state; runtime
  definitions and canonical types use the defining module's state; delayed
  insertions preserve the appropriate context without changing mangling;
- read-only fallback: a type-file specialization can read its completed
  module's missing object and supporting memory; active lifetime records take
  priority, unavailable states diagnose, and direct/aliased writes diagnose;
- late views: each preparation starts fresh, compiler calls share its writes,
  nested retained definitions do not, and saved counters cannot allocate
  globally unique names across preparations;
- teardown per state: cleanup runs once per live value in a private view,
  including inherited objects; it cannot alter any other saved state;
- fixed module names: a declaration that a deferred body generates into its
  finished module is usable by that preparation and by preparations that
  share it, but naming it from another file, or from a later evaluation,
  diagnoses in either import order, and a `compiler::emitting()` path avoids
  it in evaluation mode;
- specialization placement: family-file and type-file specializations, the
  left-to-right condition including nested arguments, alias and built-in
  types giving no ownership, and the end-of-file clash check with and without
  a specialization for the overlap;
- specialization source context: a specialization written in a type's file
  uses that file's imports and private helpers, and module code from its `#`
  is rejected;
- rejection of parenthesized generic binders, removed suffix-2 spellings,
  declaration-level `if1` conditions, and mixed generic-family/ordinary-overload names;
- `in` accepted as an ordinary identifier;
- valid and invalid numeric separator positions in all supported bases;
- name-first declarations, independent binding/type `mut`, and generic `let` headers;
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
- array type interning, mixed pointer suffixes, literal/layout-expression
  counts, layout overflow, recursive layout diagnostics, and target queries;
- array literals in contextual and inferred positions, nested literals,
  deduction and overload ranking, no decay, and copying uncopyable elements;
- contextual numeric literals, exact literal-only arithmetic, typed overflow,
  independent shift amounts, mixed-sign comparisons, and fit diagnostics;
- generated memberwise/default inits, suppression by explicit init,
  `field = expression` entries, and `[nozero]` validity and unwritten reads;
- raw-pointer copying, including null, with no source changes or pointee cleanup;
- raw-pointer inference for `auto` and permission-dropping conversions;
- rejection of incompatible pointer types and added pointee `mut`;

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
- tracked-root versus pointer/field/element extraction cleanup modes;
- acceptance of writable direct field/element sources, with rejection of
  fixed fields and read-only Advanced pointer sources;
- `take` source-place checks and Advanced default-constructor selection;
- drop-state analysis and relocation-state diagnostics for both `Inactive`
  and `MaybeActive` uses, covering branches, short-circuit operators, `?:`,
  loops, later arguments of the same call, and receiver or `&` arguments
  followed by relocation of the same root.

Compile-only assignment tests must accept otherwise well-typed `x = x`,
`x = copy x`, overlapping source paths, and destination-dependent pointer or
wrapper arguments without required errors; optional warnings are allowed.
Check that self-assignment leaves the named root Active in the flow model,
and that a tracked source relocated into an indirect destination becomes
Inactive. Preserve errors for already Inactive/MaybeActive sources,
incompatible types, missing `copy` support, fixed sources, and containing-root
relocations such as `packet.field = make_field(packet)`.

Do not run undefined-behavior cases expecting a particular crash or output.

Compile-only extraction cases must accept otherwise well-typed direct
field/element moves followed by inactive-slot access, ordinary assignment, or
enclosing cleanup. Optional warnings are allowed, but these are not required
errors. Do not execute them as runtime conformance tests.

Ordinary lowering tests should use possibly aliasing parameters to check the
absence of an implicit comparison or fallback; optimizers need not preserve
a particular instruction sequence after establishing undefined behavior.
The evaluator keeps its general invalid-access checks to protect the compiler
host, without adding special assignment alias analysis or runtime guarantees.

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
Replacement goldens must show input preparation, destination cleanup, and
the delayed outer operation in that order, with no implicit alias comparison.
Check nested copies that finish before cleanup, selected conditional arms
that forward their result slot, and array-literal prepare/drop/build passes,
and post-preparation drop flags for `x = wrap(x)`. Address-taking goldens include
`&place` for pointer slots and `ptr_as<usize>(&place)` for integer conversion.
The latter must reuse the ordinary address calculation and its permission
and activation checks.
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
Byte-storage goldens must show aligned LLVM backing arrays, omitted writes
for `[nozero]`, no implicit payload destruction, and no per-element flags.
Test payload transfer inside a relocated container, typed pointer arithmetic
through unused capacity, and address casts without uninitialized byte reads.
Check live `construct_at`: inputs are prepared before old lifetime end,
there is no old drop, and no pointer result is produced.

### 17.3 Execute tests

Use [the standalone assignment example](../examples/assignment-order.huc0)
for initial sequencing and placement checks; it returns a nonzero code on
failure. Run it without optimization and with optimization, retaining its
`noinline` boundaries. It contains no executed self-aliasing violations.

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
- destructive-relocation and copy-method counts;
- copying fixed and writable raw pointers without modifying the source;
- relocation of nested Advanced aggregates with cleanup only through the
  active destination;
- extraction from manually managed slots followed by initialized-range
  updates, including a pop-style return with exactly one destination cleanup;
- extraction through a fixed `mut T*` binding without changing its address;
- Basic pointee copying and explicit pointee copying leaving sources active;
- direct local-field, pointer-field, nested-field, and indexed extraction
  followed by reconstruction or removal from a container's live range;
- Basic `take` from fixed scalars, pointers, structures without `T()`,
  bit-fields, and under-aligned fields, with no source changes;
- Advanced `take` through locals, fields, pointers, and indexing, with one
  source evaluation, in-slot `T()`, and unchanged automatic source cleanup;
- `destination = take source` order, including `x = take x`, and destruction
  of discarded `take` results at the full-expression boundary;
- `construct_at` from Basic, Advanced, explicit-copy, constructor, and function
  result initializers, with each operand evaluated once and no hidden copy;
- construct/extract/reconstruct/destruct cycles, final constructor addresses,
  manual reverse field cleanup, and reuse of the allocation afterward;
- unchanged root cleanup state when either intrinsic operates through an alias;

- Basic self-assignment and explicit library guards for equal slots;
- destroy-first constructor, factory, and copy order, including final result
  addresses through nested factories and copies at `-O0` and without inlining;
- explicit temporary preparation for intentional self-copying;
- generated `copy` methods: field order, direct field copies, suppression by
  a written `drop`, `[nocopy]`, explicit `[norelocate]`, or a noncopyable
  field, and Basic structures staying Basic;
- `[norelocate]`: rejection of every relocation source, propagation through
  inline fields and arrays but not pointers, and final addresses for
  constructors, factories, `copy`, array literals, by-value arguments, and
  conditional temporaries;
- `drop` and `forget`: cleanup at the statement or none, Inactive and
  MaybeActive results, drop flags at joins, reinitialization, Basic operands,
  and rejection of field, pointer, fixed, and inactive operands;
- `is_relocatable`, `is_copyable`, and `is_default_initializable` answers for
  scalars, pointers, arrays, written and generated methods and inits,
  `[nocopy]`, and `[norelocate]`; `if1`-guarded members in generic types; and
  rejection of a query about a structure whose body is still being processed;
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
  conversion, ordering of unrelated pointers, and `let1 mut` names in ordinary
  functions, while `u8` and `c8` objects remain ordinary values;
- a function-local `let1 mut` assigned while its compiler call executes, alive
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
  in a `let1 mut`, with leaks diagnosed at the end of a top-level evaluation
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
- one leading import block per file, duplicate-import diagnostics across
  forms and aliases, `as _`, `pub import` chains, invisible loaded modules,
  private declarations, `main` without `pub`, and private types passed as
  generic arguments;
- generated text: canonical module paths without alias lookup, access only to
  `pub` helpers, private generated declarations, and authored hole blocks that
  keep their aliases;
- `lower` source input versus `lower --flattened`, selected only by the option;
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
6. explicit bitwise relocation and copy/drop
   functions, cleanup edges, sidecar active-state handling, and lifecycle
   hardening.

This workstream's chosen subset is complete when user-written `.huc0` programs
compile and run through LLVM, tests cover its supported lifecycle and evaluation
rules, and no HUC1 facility is required by `huc0_compile`. It need not be
complete before work on staging begins.

### HUC1 to HUC0

This workstream reuses the HUC0 source/parser, semantic analysis, HIR, and MIR
above for the features it evaluates. The ordinary-function interpreter runs
that MIR, extended with compiler-only values; it does not require LLVM emission.
Staged indexing can start earlier, but full evaluation is not an independent
frontend. Native output remains a separate integration gate on a staging-first
route.

For a staging-first milestone or the staging side of an end-to-end slice:

1. staged module indexing, file import blocks, `import1`, `pub import`,
   visible-module and `pub` checks, opaque deferred-body spans, phase-free
   pass-through, flattened HUC0 namespace sections, and origin maps;
2. the deterministic compile-time interpreter with resource and effect limits,
   including isolated module states and ordered merging;
3. `if1`, three-part `for1`, bounded phase loops, selected-body parsing, and
   module/type/function/block insertion contexts;
4. generic `struct`, generic `fn`, and generic `let` primary/partial/full family selection with
   angle patterns, deduction, recursion states, owning-file placement, and
   the end-of-file clash check;
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
T* and independent mut layers
constructors and direct destination construction
Basic/Advanced classification
destructive relocation, copy methods, and drop
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
3. HIR records Copy, Relocate, Observe, and LogicalCopy explicitly.
4. MIR is fully concrete and contains no phase construct.
5. Compile-time evaluation has no ambient untracked host effects. Writes change
   only the active state; a saved dependency state remains unchanged.
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
