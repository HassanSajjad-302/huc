# HUC Compilation Model

Status: adopted; not implemented

This document describes how HUC builds a program: how its files are loaded,
what each file can name, where specializations may live, and who may change
compile-time values. The [language specification](language-specification.md),
the [grammar](huc.ebnf), the
[compiler architecture](compiler-architecture.md), and the
[implementation plan](implementation-plan.md) follow it. The specification
states the rules; this document explains them with examples and reasons.

The aim is to keep the first implementation simple without closing the door
to faster builds later. Release 0.1 compiles one whole program at a time.
Section 13 sets the rules that later, separately compiled packages must
follow, but it does not design them in full. This is not a new macro system.
Appendix A compares the model with C++ and Rust.

## 1. Recommendation in brief

Use one compilation session for one HUC program:

1. Load each source module once.
2. Give each source module its own name-lookup environment.
3. Allow access to the source module itself, its direct imports, and modules
   deliberately re-exported by those imports with `pub import`.
4. Keep each top-level declaration private to its file unless it is marked
   `pub`.
5. Allow a specialization only in the file that declares its family, or in
   the file that declares a type in its pattern when no binder comes before
   that type.
6. Keep one shared registry of declarations and concrete specializations.
7. Give each module its own compiler state. Merge completed dependency states
   in import order, with later values winning for the same logical object.
   Writes change only the active state, never the saved dependency. A finished
   module gains no new names (sections 8.4 and 10.7).
8. Give each module-level `let` one runtime definition in the program.
9. Produce independently readable HUC0, then lower it to LLVM.
10. Record observed inputs for diagnostics and reproducibility; consider
    finished-build reuse after 0.1.
11. After 0.1, add separately compiled packages under the rules in section 13.

In short:

> Load once. Define once. Share by identity. Keep lookup scopes separate.

Do not start with per-module PCM-like files. They can be added later if their
benefit justifies the additional rules and implementation work (section 13.8).

## 2. Terms and scope

- **Compilation session:** one preparation of a root program and its imports.
  It has its own compiler objects, declaration records, and generated output.
  After 0.1, each package build is its own session (section 13).
- **Root module:** the file a build starts from. A program's root module
  contains `main`.
- **Source module:** a source provider selected by the module resolver. It
  has an identity and one file-wide import environment; importing it is not
  textual copying.
- **Namespace:** a group of declaration names, such as `app::common`. A
  namespace is not a separate object file or a new instance of a module.
- **Dependency closure:** a module's direct imports, their imports, and so on.
  This says what must be loaded, not what the module's users may name.
- **Visible modules:** the source module itself, its direct imports, and the
  modules exposed by following `pub import` edges from those imports.
- **Re-export:** an explicit promise that an imported module is also part of
  the importing module's public source interface.
- **Private and `pub`:** a top-level declaration is private to its file unless
  it is marked `pub`. Other files can use only `pub` declarations
  (section 6.6).
- **Owning file:** the file that declares a family or a type. Only an owning
  file may write a specialization (section 9.5).
- **Compiler state:** a module's values and lifetime records for logical
  compiler objects, including imported objects and supporting allocations.
- **Completed state:** the saved result of processing a module. Importers
  merge it into their states without changing that saved result (section 8.4).
- **Package:** a group of modules that is compiled in one run and saved for
  reuse, like a Rust crate or a C++ library target. In 0.1, a whole program is
  one package (section 13).
- **Definition fingerprints:** hashes saved with each package so that later
  builds can check that all packages agree (section 13.7).
- **Input record:** a report of sources, configuration, and resolution choices
  observed during a build. In 0.1, it supports diagnostics and reproducibility,
  not a decision to skip the next build.
- **Declaration identity:** the compiler's record of which entity a name
  denotes. Two references to one variable share storage within the active
  state; different module states may hold different values for that identity.
- **Build artifact:** a saved result, such as HUC0, an object file, or an
  executable, together with the inputs needed to validate that result.

The model concerns HUC definitions in one configured program build. It
does not promise to detect every conflict inside independently produced C
libraries or shared libraries. Section 13 covers future HUC packages.

Examples are independent unless a section explicitly combines their files.
Paths illustrate a possible module-to-file mapping; they do not require all
modules to live in one directory. Comments marked `Error` describe required
diagnostics. These are design examples, not compiler-verified programs.

## 3. What changes, and what stays

| Area | Earlier design | This model |
|---|---|---|
| Source loading | Root and imports form one translation unit; sources are processed once | Keep |
| Qualified lookup | Loaded namespaces are broadly accessible | Require a direct import or an explicit re-export path |
| Re-exports | None | Add `pub import` for ordinary source imports |
| Declaration visibility | Every top-level declaration is public | Private to its file unless marked `pub`; structure members stay public |
| Mutability keyword | `mod` | `mut`, as in Rust |
| Import placement | Each namespace section can start with imports | One leading import block per source file |
| Import access | Runtime/generic and compiler-only exposure are described separately | Ordinary source imports expose both, under the usual phase rules |
| Import aliases | Lexical aliases, with no wildcard imports | Keep in authored source; generated module paths use canonical names |
| `as _` | Binds an ordinary alias named `_` | Import without binding an alias |
| Repeated direct imports | A repeated import can bind another alias | Reject duplicate direct imports in one file |
| Namespace sections | Can introduce ordinary declarations in other namespaces | Own namespace only; specialize prefixes foreign specializations |
| Specialization placement | Any file may specialize a family in the family's namespace | Only the family's file, or a type's file when no binder comes before that type |
| Compiler calls | Execute freshly under the staging rules | Keep |
| Module-level `let1` | One shared compiler object per declaration | One declaration identity with module-local state; later imports replace earlier values |
| Runtime definitions | Shared identities and once-only program preparation | Keep and make duplicate checks explicit |
| Build reuse | The architecture and plan prohibit build-artifact caches | Keep that policy for 0.1; section 12 is a later option |
| HUC0 boundary | Printable, independently compilable text | Keep |
| HUC0 input kind | Source imports and flattened sections are supported without an explicit selector | Default to source input; add `lower --flattened` for a complete program |
| Separate HUC libraries | Not specified | Packages after 0.1, under the rules in section 13 |
| Build tool | Separate commands only | After 0.1, a package manifest and a package tool built into `huc` |
| Text generation | Plain text, checked where inserted | Keep plain text; check generated module paths without import aliases |

An implementation-only dependency no longer becomes visible automatically.
A re-export makes it a deliberate public dependency. This is an explicit
dependency rule, not a requirement to repeat every re-export as a direct
import. Inside a visible module, other files can use only its `pub`
declarations (section 6.6). Structure members stay public.

The new rules affect authored and generated references. Section 9.5 decides
which files may write specializations, so every request in a program makes
the same specialization choice. Section 8.4 isolates module compiler states,
so one sibling cannot change how another sibling is processed.

### 3.1 New spellings

This model adds or changes these spellings:

- `pub` marks a declaration that other files may use (section 6.6).
- `pub import` re-exports a module (section 6.2). Earlier drafts wrote
  `export import`. The word `export` stays free for exporting HUC functions to
  C ([C interoperability](c-interop.md), section 6.2).
- `mut` replaced `mod` as the mutability keyword.

| Meaning | Earlier HUC | HUC now | Rust |
|---|---|---|---|
| Mutable variable | `let mod count: i32 = 0;` | `let mut count: i32 = 0;` | `let mut count = 0;` |
| Writable pointer | `mod T*` | `mut T*` | `*mut T` |
| Method that changes its receiver | `fn reset() mod -> void` | `fn reset(&mut self) -> void` | `fn reset(&mut self)` |

Rust uses `mod` to declare modules and `mut` for mutability. HUC declares
modules with `module`, so `mod` never clashed in HUC itself. But Rust readers
would read `let mod x` as something about modules.

## 4. One command, two language stages

The normal user command remains:

```sh
huc build app/main.huc1
```

Conceptually, it performs:

```text
root source and source imports
        |
        v
HUC1 preparation and evaluation
        |
        v
one flattened HUC0 program
        |
        v
HUC0 checking and LLVM generation
        |
        v
object output and linking
```

The user can also invoke the staging and lowering commands separately. The
HUC0 text is still written when the driver performs both stages. `build`
passes that file to `lower --flattened` (section 11). An in-memory shortcut
must not replace that public boundary.

Initially, one HUC program can produce one LLVM module and one object file.
That is an implementation starting point, not a promise that backend work
must always use one thread or one output object.

If the separate C-interoperability proposal is adopted, its C units may
remain separate input files while their relevant IR is combined with this
program's HUC IR. Its per-module LLVM wording would need reconciliation;
this model does not silently change the C design.

No hand-maintained declaration header is required. No external build system
is required for a plain HUC program. The compiler driver still performs build
work: selecting inputs, resolving dependencies, invoking tools, and linking.

## 5. Module identity and repeated imports

### 5.1 A diamond import loads the shared source once

Consider these dependencies:

```text
app.main
  -> app.parser   -> app.common
  -> app.renderer -> app.common
```

With depth-first processing, the order is:

1. Start `app.main`.
2. Start `app.parser`.
3. Load and finish `app.common`.
4. Finish `app.parser`.
5. Start `app.renderer`.
6. Its import refers to the existing `app.common` record.
7. Finish `app.renderer`, then continue `app.main`.

Step 6 does not parse `common` again, rerun its initializers, or create new
copies of its variables. It makes the imported module available in the
renderer source's lookup environment, binds an alias if requested, and merges
the completed compiler state into the renderer's state (section 8.7).

### 5.2 A module-loading table is enough

| State | Action when imported |
|---|---|
| Unseen | Create its record and begin processing |
| Loading | Diagnose an import cycle, including the import chain |
| Loaded | Reference the existing record, merge its completed compiler state, and bind an alias unless `as _` was used |
| Failed | Report the dependency failure; do not construct a partial second copy |

This table is not an evaluation-result cache. It establishes the identity of
one source module during one compilation session.

### 5.3 One import block per source file

All `import`, `import1`, and `pub import` statements go directly after the
file's initial `module` declaration, before any other declaration or namespace
section. Comments and blank lines may separate them. Once that block ends,
later imports are errors, even after another `module` line.

These imports, aliases, and visibility permissions apply throughout the file.
A namespace switch changes where declarations belong, not the file's imports.
A re-export belongs to the source module named by the file's initial
declaration; it cannot be added inside a later specialization section.

Earlier drafts allowed imports at the start of each namespace section.

Reject duplicate direct imports by resolved module identity:

```huc
module app.main;

import app.common as first;
import app.common as second; // Error: app.common was already imported here.
```

The same error applies if both statements use the same alias or no explicit
alias. Compare resolved module identities, not just the written spellings.
This is a source-cleanliness rule, not the mechanism that prevents duplicate
objects. Different files may still import the same module under different
aliases and refer to the same declarations.

A direct import remains valid when an earlier dependency already loaded the
module indirectly:

```huc
module app.main;

import app.parser; // Parser imports app.common internally.
import app.common; // Allowed: this is main's first direct import of common.
```

`pub import app.common;` both imports and re-exports the module. Use that
one statement rather than an ordinary import followed by a second,
re-exporting import of the same module. A re-export does not cause a second
load.

The duplicate check is file-wide, across all its import forms. Switching
namespaces does not reset it. Different files still share the same
program-wide module records.

For source modules, ordinary `import` gives access to compiler-only
declarations, including `let1`, as well as runtime declarations
and generic families. It includes the access provided by `import1`, plus runtime access.
Normal phase rules still apply: importing a compiler-only value does not
make it usable as runtime storage. `import1` retains compiler-only access
without creating a runtime dependency. The compiler-supplied API continues
to use `import1 compiler;`.

Thus one `import app.tools;` is enough when both kinds of declaration are
needed. Adding `import1 app.tools;` would be a duplicate, not a required
second permission.

`as _` means "do not bind an alias" for any of the three import forms:

```huc
module app.main;

import app.fast_hash as _;
import app.plugins as _; // Allowed: neither import binds a name called _.
```

Loading, compiler effects, canonical-name visibility, and duplicate-module
checks are unchanged. `pub import app.tools as _;` still re-exports the
module, but introduces no local alias. Two imports of the same module remain
an error even with `as _`.

This is a rule for import aliases, not a global reservation of `_`. It adds
no unused-import warning and does not require the marker for imports kept for
their effects. Reserving this meaning now avoids changing an ordinary `_`
alias into a discarded alias later.

### 5.4 Source identity is not a content comparison

The resolver must use a consistent identity for a resolved source provider:

- Aliases do not change identity.
- Equivalent paths, including resolved symbolic links where supported, must
  not create accidental second loads of one provider.
- One logical module cannot be bound to two different providers in one
  compilation session. Diagnose the conflict instead of choosing whichever
  source happens to be requested last.
- Two distinct source files are not merged merely because their bytes match.

For example, if two dependency branches require different providers or
configurations of `app.common`, reject the program unless a future package
design explicitly gives them distinct identities. Do not hide two versions
behind the same `app::common` name.

Search roots and their order are explicit build inputs. Any resolution
priority policy must be consistent throughout the session; this model
does not introduce a new fixed project/standard-library search order.

### 5.5 An import cycle is not an ordinary repeated import

```text
app.a -> app.b -> app.a
```

The second request for `app.a` reaches a module still being processed, so it
is an error. Do not treat a partially processed module as complete merely to
break the cycle.

Two unrelated branches reaching an already completed `app.common` are fine.

## 6. Separate lookup environments

### 6.1 Stop upward and unrelated-sibling leakage

Suppose the imports are:

```text
app.main -> app.parser -> app.common
app.main -> app.graphics
```

Code in `app.parser` may use its own declarations and those in `app.common`.
It may not name declarations in `app.main` or `app.graphics` merely because
the root loaded them.

```huc
module app.parser;

import app.common;

fn permitted() -> i32 {
    return app::common::hits;
}

fn rejected() -> i32 {
    return app::graphics::width(); // Error: not a dependency of app.parser.
}
```

The rejected reference remains invalid even if `graphics` was loaded first,
and even if its spelling is fully qualified. Changing the root's import
order cannot grant `parser` access to an unrelated module.

If `parser` really uses `graphics`, it can import it, provided that this does
not create an import cycle.

### 6.2 Ordinary imports are not automatically re-exported

```huc
module app.main;

import app.parser;

fn main() -> i32 {
    return app::common::hits; // Error if parser only privately imports common.
}
```

Loading a dependency is different from making its names available to users.
The root can fix this by directly importing `app.common`. Alternatively,
`parser` can deliberately expose it:

```huc
module app.parser;

pub import app.common;

fn run() -> void {
    common::bump();
}
```

`pub import` loads the module, binds its normal local alias unless `as _` is
used, and exposes its canonical module name to importers. It does not copy
declarations, add storage, or execute the dependency a second time.

With that version of `parser`, the root's `app::common::hits` reference is
valid. The short alias `common` still belongs to the source that imported it;
it does not leak into the root:

```huc
module app.main;

import app.parser;

fn main() -> i32 {
    return common::hits; // Error unless this source binds the alias common.
}
```

Use the canonical qualified name for a re-export, or add a direct import to
create a local alias. An `as` alias on a re-exporting import remains local
too.

Only `pub import` edges continue visibility through the graph:

```text
main imports A; A re-exports B; B re-exports C -> main may name B and C
main imports A; A re-exports B; B imports C    -> main may name B, not C
main imports A; A imports B; B re-exports C    -> main may name neither B nor C
```

Removing a private import no longer removes a client's permission to name
that dependency: the private import never granted that permission. Removing
a re-export can break clients because it changes the public source interface.
Other effects of loading a module, such as running its compile-time code and
registering its specializations, remain separate concerns.

`pub import` exists only for ordinary imports. HUC has no `pub import1` and
does not re-export single declarations, and it does not make
compiler-only values runtime-capable. `import1` retains its compiler-only use
restrictions.
Source imports and re-exports are erased before flattened HUC0 output;
generated fragments cannot use them to change their insertion site's imports.

### 6.3 Lookup depends on the definition, not on its caller

An ordinary or generic function retains its source lookup environment when
called from another module. A later caller does not replace its aliases or
add all of the caller's modules to that environment. Outside names must be
declared before the definition; the request supplies arguments, dependent
members, applicable specializations, and the active compiler state. It does
not add later ordinary functions or locals. Local generic variables likewise
retain the names visible at their family's declaration.

The compiler can keep one global declaration registry while applying these
separate lookup environments. Knowing that a declaration exists does not
automatically make it a valid source-level reference from every file.

### 6.4 Own namespace and explicit foreign specializations

A source `module` line names only the file's own module. Later sections may
reopen that same namespace but do not reload it. A foreign specialization uses
`specialize canonical.module` before one fn, struct, or let specialization,
or its same-file impl. The module must be visible and the family public; the
owning-file rules still apply. No ordinary declaration can use this prefix.
See [Declarations and Preparation §7](declarations-and-preparation.md#7-specializing-another-modules-family).

An arbitrary namespace with no visible module owner is not allowed. Owning
`app.common` does not also mean owning `app.common.detail`:

```huc
module app.common;

module app.common.detail; // Error: not this file's module or a visible import.

fn helper() -> i32 { return 1; }
```

Put `helper` in `app.common`, or give `app.common.detail` its own source file
and import it. This also prevents a file added later for `app.common.detail`
from conflicting with a namespace claimed by this file. The same restriction
applies to generated module sections before flattening; section 11 explains
why already-flattened HUC0 does not use source-file ownership rules.

A specialize declaration cannot add new ordinary functions, variables, types,
aliases, or primary families. It also does not expose the other module's
private imports.

For example, this is rejected even if `hook` has no previous definition:

```huc
module app.plugin;

import app.common;

module app.common;

fn hook() -> i32 { // Error: plugin cannot add an ordinary function to common.
    return 1;
}
```

Consequently, `app.common` cannot acquire a missing ordinary `hook()` merely
because an importer loaded `app.plugin`. Put the helper in the owning module
or pass an explicit value/interface to its code.

Ownership follows the source provider, not the current namespace spelling.
It also applies to generated module declarations: a module action cannot add
ordinary declarations to a foreign namespace by emitting a `module` line.
A generation site inside a deferred body belongs to the file that wrote that
body, even when its output anchor is a caller in another file. For a primary,
that is the family's file; for a specialization, it is the owning file that
wrote it. Text returned by a builder and published by a caller's module action
belongs to that action's context instead. Text origins alone grant no
declaration ownership.

An allowed specialization keeps its defining source context: the imports,
aliases, and private declarations of the file that wrote it, not those of
the family's file. Its canonical name keeps the family's namespace, while
its printed position follows the original request anchor. Unqualified names
resolve in that namespace, where another file's private declarations stay hidden:

```huc
module app.point;

import library.hash;

pub struct Point {
    let x: i32;
}

fn mix(value: i32) -> u64 { // Private to app/point.huc1.
    return as<u64>(value) * 31;
}

specialize library.hash
fn hash<><app::point::Point>(value: app::point::Point*) -> u64 {
    return app::point::mix(value->x); // Allowed: the same file wrote mix.
}
```

A specialization may contain the members and locals of its own definition,
but namespace-level helpers belong in the specializing module, as `mix` does.
A `#` in a specialization may insert local code, but a module action there
would land in the family's namespace, which this file does not own, so it is
rejected. Section 9.5 says which files may write specializations.

### 6.5 Forward references do not bypass the dependency boundary

Names require declarations before use. A forward declaration permits recursion:

```huc
module app.math;

fn later() -> i32;

fn first() -> i32 {
    return later();
}

fn later() -> i32 {
    return 42;
}
```

A later declaration in an unrelated module does not become available just
because the module was loaded elsewhere; declaration order and visibility both apply.

Compiler execution still needs an available checked definition. This model
does not make future declarations or pending generated output available early.

There is a real loss compared with unit-wide qualified lookup: two source
modules cannot directly call each other if doing so requires mutual imports.
Import cycles remain errors. Put the mutually dependent functions in one
source module, or separate their interaction behind an explicit interface.
This rule does not prevent runtime recursion within a module.

### 6.6 Declarations are private unless marked `pub`

A top-level declaration is private to its file unless it is marked `pub`:

- A private declaration can be used anywhere in its own file, including the
  file's later namespace sections.
- A `pub` declaration can also be used by any file that can see its module
  (sections 6.1 and 6.2), in the same package or another one.

```huc
module app.counter;

let mut count: i32 = 0;      // Private: only this file can name it.

pub fn add(value: i32) -> void {
    count += value;
}

pub fn total() -> i32 {
    return count;
}
```

```huc
module app.main;

import app.counter;

fn main() -> i32 {
    counter::add(2);
    return counter::count;   // Error: count is private to app.counter.
}
```

`main` needs no `pub`. The driver finds it in the root module without an
import.

In C++ terms, a private declaration is what `static` or an unnamed namespace
gives a declaration in a `.cpp` file. `pub` works like `export` on a
declaration in a C++20 module, and like Rust's `pub`. The default is the
reverse of C++: you opt in to sharing, not to hiding.

The rule covers every top-level declaration: functions, generic families,
structures, type aliases, `let`, and `let1`. It has these consequences:

- **Structure members stay public.** Specification section 4.6 makes all
  members public. Member privacy is a separate decision.
- **Specializations have no `pub` of their own.** They apply wherever their
  family is used. To specialize a family from another file (section 9.5),
  the family must be `pub`.
- **Private types can still be type arguments.** A file can pass its private
  type to a library generic, because the generic never names it
  (section 9.1).
- **Generated code is checked where it is inserted.** Code that a builder
  returns and a caller inserts with `#` can use only the builder module's
  `pub` declarations. So any helper that generated code names must be `pub`,
  even if users are not meant to call it directly. Rust has the same rule for
  helpers used by macros. A generic body, and a `#` in the builder's own file,
  are checked in that file and may use its private declarations.
- **Generated declarations are private by default.** A declaration inserted by
  `#` belongs to the file of that `#`. It is private unless the generated text
  includes `pub`.
- **Flattened HUC0 does not check `pub`**, just as it does not check imports
  or ownership (section 11).
- **`pub` controls HUC names, not link names.** Only `main` and functions
  exported to C need link names that are visible outside the program
  (section 15).

Packages may later add a level for "this package only", like Rust's
`pub(crate)` (section 13.1).

## 7. A complete example: compiler state and shared runtime objects

These four files form one example.

`app/common.huc1`:

```huc
module app.common;

pub let1 mut next: usize = 0;
pub let mut hits: i32 = 0;

pub fn next_id<>(counter: mut usize*) -> usize {
    *counter += 1;
    return *counter;
}

let1 reserved: usize = next_id(&next); // 1 in common's compiler state.

pub fn bump() -> void {
    hits += 1;
}
```

`app/parser.huc1`:

```huc
module app.parser;

import app.common;

pub let1 seen: usize = common::next; // 1 inherited from common.

pub fn run() -> void {
    common::bump();
}
```

`app/renderer.huc1`:

```huc
module app.renderer;

import app.common;

pub let1 seen: usize = common::next;

pub fn run() -> void {
    common::bump();
}
```

`app/main.huc1`:

```huc
module app.main;

import app.parser;
import app.renderer;
import app.common;

let1 mut ids: usize = 0;
let1 parser_id: usize = @common::next_id(&ids);
let1 renderer_id: usize = @common::next_id(&ids);

fn main() -> i32 {
    parser::run();
    renderer::run();
    return app::common::hits;
}
```

Expected behavior:

- `common` is loaded once, so its initializers run once. `next` is `1` when
  `common` finishes, and that completed state is saved.
- The parser and renderer each inherit `next == 1`. Both `seen` values are `1`.
- `main` owns `ids`. Its two calls produce `1` and `2`, in `main`'s source
  order.
- The executable contains one `app::common::hits` global.
- The two runtime calls increment that same global; `main` returns `2`.

The root directly imports `common` because it names that module itself.
This does not rerun `common` or its initializers after the two earlier
imports.

Loading once does not mean calling once. Both `next_id` calls in `main`
execute. Only the repeated attempt to load the shared module is skipped.

Reversing the root's imports of `parser` and `renderer` changes no value.
Neither changes the inherited values, and neither can affect the other's state.
Only the order of their declarations in the flattened output changes.
The parser could increment its inherited counter instead:

```huc
let1 id: usize = @common::next_id(&common::next); // 2 in the parser's state.
```

The renderer doing the same would also get 2, not 3. These are independent
states, not a program-wide compiler counter. For unique IDs across branches,
collect and number entries in the parent, as the root does with `ids`.
Its final direct import of `common` restores the inherited `common::next` to 1.

## 8. Preventing duplicate definitions of `let1` and `let`

### 8.1 Same variable, two definitions: error

```huc
module app.settings;

let1 limit: usize = 16;
let1 limit: usize = 16; // Error: a second definition, even with equal text.
```

The same rule applies to runtime storage:

```huc
module app.settings;

let mut count: i32 = 0;
let mut count: i32 = 0; // Error: a second definition of the same variable.
```

Do not create a separate namespace just to allow a `let1 limit` and a
`let limit` in the same lexical scope. Phase changes where storage lives;
it does not provide a second spelling of the same variable name.

Repeated imports are different: they reference the original definition
instead of introducing another definition.

### 8.2 Different namespaces: different variables

In the parser source:

```huc
module app.parser;

let mut count: i32 = 0;
```

In the renderer source:

```huc
module app.renderer;

let mut count: i32 = 0;
```

These are two different globals, `app::parser::count` and
`app::renderer::count`. No conflict exists merely because their short names
match. The same principle applies to compiler variables in different scopes.

### 8.3 Reopening the owned namespace cannot hide a second definition

```huc
module app.common;

let mut hits: i32 = 0;

module app.common;

let mut hits: i32 = 0; // Error: a second definition in the same owned namespace.
```

The diagnostic should show both definition locations. The declaration
registry is shared even though source lookup environments are separate.
An ordinary declaration added by a different module would already fail the
foreign-namespace rule in section 6.4, even without a name collision.

### 8.4 Each module has its own compiler state

A module starts from its own imports, not from the current state of whoever
imports it. It may change accessible `let1 mut` objects using ordinary HUC
operations. Those changes affect only its state, even when the declaration
came from a dependency.

When the module finishes, save that state. Importers merge the saved result
into their own states. They do not execute the module again or write back into
it. Later direct imports win when two branches provide values for the same
object. All values participate, including unchanged inherited values.

There is still only one declaration of a shared variable. The separate states
are not duplicate declarations, cloned source modules, or new runtime globals.
Two pointers to the same logical object still alias within one state. Direct
writes, pointer writes, method calls, and compiler calls follow the same rule.

This prevents sibling pollution. If the root imports `app.enable_fast` and
then `app.point`, changing a shared `let1` in `enable_fast` cannot affect how
`point` is processed. `point` sees that change only if it depends on
`enable_fast` itself. The root does see the two completed states merged in its
own import order.

A module's result therefore depends on its own source, its ordered dependencies,
and the build settings. Independent modules can be processed in parallel; merge
their results in source import order, never worker completion order.

The rules apply to pointers, structures, and allocations, not just scalars.
Ordinary mutation permissions and lifetime checks still apply. Merging does not
run a `copy` method or `drop`; actual HUC assignment and destruction still
do. At the end of translation, each module state's live values are cleaned
up in a private view of that state. Cleanup never changes another module's
saved result.
[Specification section 9.8.1](language-specification.md#981-isolated-compiler-state-and-ordered-imports)
defines object identity, lifetime records, and cleanup.

A finished module's names are still fixed. A late-generated declaration has
the preparation-local visibility described in section 10.7. State merging does
not add visibility or change the owning-file rule for specializations.

### 8.5 Local objects are not module singletons

```huc
module app.local_example;

fn local_count<>() -> usize {
    let1 mut n: usize = 0;
    n += 1;
    return n;
}

let1 first: usize = @local_count();  // 1
let1 second: usize = @local_count(); // 1

fn runtime_count() -> i32 {
    let mut n: i32 = 0;
    n += 1;
    return n;
}
```

Each compiler evaluation gets the local compiler storage required by the
existing preparation rules. Each runtime call gets its own runtime local.
Neither is a duplicate module-level definition.

### 8.6 Two executable builds have separate compiler state

Building `tests/parser.huc1` and `tests/renderer.huc1` creates two compilation
sessions. A module-level `let1` does not carry its last value from one session
to the other, even if one long-running compiler process builds both targets.

Likewise, separate executables have separate runtime globals. This is not a
promise about sharing state between independent native libraries inside one
process; that requires a future binary-library design.

### 8.7 One walkthrough: values, objects, and pointers

The complete example is kept here in one document. Each labeled block describes
a separate module file, because an actual import diamond needs separate source
modules. It is not one HUC source file with repeated namespace sections.

`a.huc1`:

```huc
module a;

pub let1 mut t: u64 = 3;
pub let1 p: mut u64* = &t;

pub struct Pair {
    let mut x: u64;
    let mut y: u64;

    fn init(&mut self, x: u64, y: u64) : x = x, y = y {}
}

pub let1 mut pair: Pair = Pair(1, 2);

pub fn read_t<>() -> u64 {
    return t;
}
```

`c.huc1`:

```huc
module c;

pub import a;

pub let1 before: u64 = a::t; // 3.
a::t = 5;
a::pair.x = 10;
pub let1 after: u64 = *a::p; // 5: the pointer sees c's version too.
```

`d.huc1`:

```huc
module d;

pub import a;

pub let1 before: u64 = a::t; // 3, not c's 5.
*a::p = 7;
a::pair.y = 20;
pub let1 after: u64 = a::t; // 7: direct access sees the pointer write.
```

`app.huc1`:

```huc
module app;

import c;
import d;

let1 selected: u64 = a::t;    // 7: d is imported after c.
let1 pointed: u64 = *a::p;    // 7: the same logical object.
let1 via_call: u64 = a::read_t(); // 7: executes in app's compiler state.
let1 x: u64 = a::pair.x;      // 1, not 10: d's whole Pair wins.
let1 y: u64 = a::pair.y;      // 20.

fn main() -> i32 {
    return as<i32>(selected); // 7.
}
```

The assignments are module-level assignment items: they run in the compiler,
change only that module's state, and disappear from HUC0. The `pub import`
lines let the app name `a`; ordinary private imports also carry state, but do
not expose their dependencies' names.

The completed states are:

| Module | `a::t` and `*a::p` | `a::pair` |
|---|---|---|
| `a` | 3 | (1, 2) |
| `c` | 5 | (10, 2) |
| `d` | 7 | (1, 20) |
| `app`, importing `c` then `d` | 7 | (1, 20) |

Reversing the app's imports gives it 5 and (10, 2). It changes none of the
saved dependency states. The merge replaces whole objects; it does not combine
`c`'s `x` with `d`'s `y`.

#### An unchanged later value wins too

If `d` only imports `a` and makes no changes, the same app gets 3 and (1, 2).
Likewise, adding `import a;` after `import c;` replaces c's inherited values
with a's original values. It does not rerun a's initializers. Any later import
that depends on `a`, even indirectly, has the same effect, so import the module
that changes `a`'s values after those imports.

If both branches instead increment `a::t` once, each independently gets 4.
The parent also gets 4, not 5: it merges values, not a list of updates to
replay.

#### Why the later import wins

C configures code with preprocessor macros tested by `#if`. A module-level
`let1` plays a similar role in HUC, but a `let1 mut` can also be changed by
the modules that import it. That power needs care: changing an imported value
is a deliberate choice, and its effect depends on import order, just as a C
file's results depend on the order of its `#include` lines.

The rule gives the importer full control without new syntax:

- Order the imports so that the module whose values you want comes last.
- Import a module directly, after the others, to get its unchanged values.
- Assign the value in the importing module to set it regardless of order.

HUC does not warn when a later import replaces a changed value. Such
replacements are normal in this design, so the warning could appear often,
and the only way to silence it would be to reorder imports that are already
correct. The rule is also the simplest to implement: merging replaces each
object with the later import's version and needs no history or conflict
tracking.

#### Calls and runtime definitions

A compiler call such as `a::read_t()` in the app reads the active app state.
Its name lookup remains lexical: `t` still means the declaration `a::t`.
Similarly, an imported helper may modify app's version without changing a.

A retained runtime definition or canonical type is prepared using its defining
module's state, not its first requester's merged values. A runtime call to
`a::read_t()` therefore returns 3. The function can observe 7 when executed
as compiler code in app; those are different contexts. Ordinary `fn`
restrictions on naming `let1 mut` remain unchanged, which is why this helper
has `<>`.

Already prepared constants and type layouts do not change retroactively.
Different runtime versions require explicit generic arguments, not implicit
compiler-state additions to mangled names.

### 8.8 Late preparations start fresh

After a module finishes, each new program-mode preparation starts from a fresh
working view of its saved state. It does not inherit writes from another late
preparation, even if both prepare specializations of the same family. Compiler
calls inside one preparation do share its working view. A nested request for
another retained definition gets a separate preparation and view.

For example, if the saved module counter is 0, two late preparations that each
increment it get 1 and 1, not 1 and 2. This is allowed; the writes stay private.
Such a counter therefore cannot supply globally unique generated helper names.
Use names that distinguish the generated declarations, such as names derived
from explicit specialization arguments. Conflicting definitions still diagnose.
Rejecting all late writes would be a separate restriction, not a clarification
of the current rule.

### 8.9 Teardown has a per-state cost

An inherited Advanced value can be destroyed in many module states. That can
be expensive when many modules import a library with large compiler objects.
It is the cost of the current lifetime rule; physically sharing memory does
not automatically make the destructor calls interchangeable.

A destructor can read other values in its active state, so identical object
bytes alone do not prove identical cleanup behavior. Keep the existing rule
for now. Basic values without cleanup need no destructor work. If cleanup
fails, report the module state involved; the compiler can stop at the first
error or group equivalent reports. It must not hide the failure. Sharing one
cleanup across states would need a separate design decision.

## 9. Generic identity and imported types

### 9.1 A generic can still receive its caller's type

`library/read.huc1`:

```huc
module library.read;

pub fn read_value<T: auto>(item: T*) -> i32 {
    return item->value;
}
```

`app/main.huc1`:

```huc
module app.main;

import library.read;

struct Point {
    let value: i32;

    fn init(&mut self, value: i32) : value = value {}
}

fn main() -> i32 {
    let point: Point = Point(42);
    return read::read_value<Point>(&point);
}
```

The result is `42`. `T` carries the identity of the caller's `Point`; the
library does not need a reverse import or permission to look up every name
in `app.main`. This works even though `Point` is private to `app.main`,
because the library never names it.

Directly spelling `app::main::Point` in the library's source would be a
different operation and would require an allowed dependency. Substitution
is not a general grant of access to the caller's namespace.

#### Compiler objects reached through a specialization

A specialization can refer to its own file's compiler data even when the
generic that calls it is being prepared in a different module's state.
For example, these are three source files:

`library/width.huc1`:

```huc
module library.width;

pub fn width_of<T: auto>() -> usize {
    return 0;
}

pub fn prepared_width<T: auto>() -> usize {
    let1 width: usize = width_of<T>();
    return width;
}
```

`app/point.huc1`:

```huc
module app.point;

import library.width;

pub struct Point {}
let1 point_width: usize = 8;

specialize library.width
fn width_of<><app::point::Point>() -> usize {
    return app::point::point_width;
}
```

`app/main.huc1`:

```huc
module app.main;

import library.width;
import app.point;

fn main() -> i32 {
    return as<i32>(width::prepared_width<point::Point>());
}
```

Preparing `prepared_width<Point>` uses the library's state, which has no entry
for `point_width`. The specialization can name that private value because it
was written in the same file. Its read falls back to the completed state of
`app.point`, producing 8. This does not import app into the library or publish
the borrowed value as library state.

The fallback is read-only, including access through pointers. It cannot supply
a writable counter. An existing active entry always wins, even if inactive or
deallocated; fallback never repairs an invalid lifetime. If app.point has not
finished yet, the missing-object read diagnoses rather than guessing a value.
Pass the needed data explicitly for requests that must run that early.

### 9.2 Multiple requesters share one concrete definition

If both `app.parser` and `app.renderer` request a library's
`identity<i32>`, they refer to one program-mode specialization record.

Its identity uses the existing family identity and canonical concrete
arguments, not the importing file's name. Runtime references do not replay
its program-mode preparation or module-level generation.

Calling the same specialization twice through `@` still performs two fresh
evaluations. Reusing a declaration identity is not reusing a call result.

The same identity rule applies to concrete generic types and generic
variables. A module-scope generic variable specialization has one runtime
object; a function-local specialization follows its normal runtime lifetime.

### 9.3 A later specialization cannot change an earlier selection

```huc
module app.selection;

fn identity<T: auto>(mut value: T) -> T {
    return value;
}

fn first() -> i32 {
    return identity(7);
}

fn identity<><i32>(value: i32) -> i32 { // Error: would change the earlier request.
    return value + 1;
}
```

The specification's ordering and ambiguity rules apply (specification
section 9.3). One shared registry lets the compiler diagnose conflicts within
this program. That handles duplicate definitions, but not all dependency
surprises; the placement rules in section 9.5 remove the rest.

### 9.4 Mutable `let1` state is still a design responsibility

Inside a module's compiler state, an already prepared runtime
specialization keeps the compiler values it observed. Changing the value
later in that module does not create a second runtime definition with the
same identity. A later compiler evaluation in that module may observe the
changed value. Once the module finishes, save its state; later importer writes
do not change that saved result (section 8.4).

For distinct runtime implementations, prefer explicit phase arguments:

```huc
module app.configuration;

fn limit<Capacity: usize>() -> usize {
    return Capacity;
}

fn small_limit() -> usize {
    return limit<16>();
}

fn large_limit() -> usize {
    return limit<32>();
}
```

Here the two definitions intentionally have different identities. Do not add
ambient compiler state to symbol names to conceal conflicting definitions.

### 9.5 Specializations live in an owning file

A specialization may be written only in:

- the file that declares its family, or
- a file that declares a type in its pattern, if no binder comes before that
  type when the pattern is read from left to right.

Types nested inside the pattern count too, such as `Point` in
`hash<><Vector<Point>>`. Built-in types such as `i32` belong to no file, so
only the family's file can specialize for them. Binders and values in a
pattern give no ownership. Type aliases give no ownership either: an alias is
another name for the same type (specification section 4.8). So
`type MyInt = i32;` does not let a file specialize `hash<MyInt>`, which is
just `hash<i32>`.

The left-to-right reading includes the arguments of nested types. So the
file that declares `Point` may write `hash<Pair<Point, T>>`, but not
`hash<Pair<T, Point>>`, because there `T` comes first. A binder is any
placeholder in the pattern: a type binder such as `T`, a value binder such as
`N`, or a family binder. The family's own file has no left-to-right
condition.

The check is local. At each specialization, the compiler confirms that the
current file declares the family, or declares a type in the pattern that
comes before every binder.

`library/hash.huc1`:

```huc
module library.hash;

pub fn hash<T: auto>(value: T*) -> u64 {
    return 0;
}

// Allowed: this file owns the family.
fn hash<><i32>(value: i32*) -> u64 {
    return as<u64>(*value);
}
```

`app/point.huc1`:

```huc
module app.point;

import library.hash;

pub struct Point {
    let x: i32;
}

specialize library.hash
// Allowed: this file owns Point.
fn hash<><app::point::Point>(value: app::point::Point*) -> u64 {
    return as<u64>(value->x);
}
```

`app/fast_hash.huc1`:

```huc
module app.fast_hash;

import library.hash;
import app.point;

specialize library.hash
// Error: this file owns neither the family nor Point.
fn hash<><app::point::Point>(value: app::point::Point*) -> u64 {
    return 1;
}
```

#### Why every request sees the same specializations

This argument covers declarations processed before their owning modules finish.
Late generated specializations and impls have the explicit limits in section
10.7: they can make request/import order observable, and still must pass late-
request checks. State isolation alone does not remove those effects.

To request `hash<Point>`, code must name both the family and `Point`. So its
file must see both owning files, through imports or re-exports, and those
files are processed before the requesting code continues. Every
specialization that could apply is therefore loaded wherever it could be
requested. Every request in the program makes the same choice, so every copy
of `hash<Point>` is the same.

This removes two surprises that free placement would allow, even inside one
program:

- A side file like `app/fast_hash.huc1` cannot change how `hash<Point>`
  behaves elsewhere, depending on whether some unrelated file imports it.
- Without late generated specializations, import order cannot cause a
  late-specialization error. The error in
  section 9.3 can now come only from inside one owning file, when that file
  uses a specialization before declaring it.

State isolation (section 8.4) closes the sibling-dependency gap. An owning file
may declare a specialization inside an `if1`. The condition reads its own state,
formed from its ordered imports and its local operations. Reordering imports
elsewhere cannot change which specializations that file declares.

The same reasoning holds across separately compiled packages (section 13).
Every package that uses `hash<Point>` depends on the package that declares
`Point`.

#### Two files cannot clash unseen

Two specializations clash when both match one request and neither is more
specific. Specification section 9.3 makes such a request an error. Without
the left-to-right condition, two files that do not know about each other
could clash. Suppose `library.pick` declares
`pub fn pick<T: auto, U: auto>() -> i32`, and two other files reopen its
namespace:

```huc
// In app/a.huc1, which declares A:
fn pick<T: auto><app::a::A, T>() -> i32 { return 1; } // Allowed: A comes first.

// In app/b.huc1, which declares B:
fn pick<T: auto><T, app::b::B>() -> i32 { return 2; } // Error: T comes before B.
fn pick<><i32, app::b::B>() -> i32 { return 3; }      // Allowed: no binder before B.
```

If the second one were allowed, a request for `pick<A, B>` would match both
and fail. Neither author would see the error, because neither file imports
the other. The user who combined them could not fix it either: their file
owns neither `pick`, `A`, nor `B`.

The condition prevents this. Take two specializations, written in different
files, that can match one request. Look at the first position, from the left,
where either of them has its own file's type. The other one has no binder
there, so it must have the same type there, and it names that type. Its file
therefore imports the first file, which was processed earlier. The family's
file is always imported by a type's file that specializes the family, so the
same holds there. Either way, when the compiler reaches the second
specialization, it can already see the first.

When a file finishes, the compiler checks each specialization the file wrote
against the other specializations of the same family in the files it depends
on. A clash is an error in this file, unless a specialization for exactly the
overlap exists too. The author can always fix it in their own file: change
the specialization, or write the overlap there.

Rust's orphan rule has a similar condition for the same reason: a crate's own
type must come before any open type parameter.

#### Why a file, not a package

Rust's orphan rule works per crate: any file of the crate that owns the type
may write the implementation, because Rust always compiles every file of a
crate. HUC compiles a file only when something imports it. A specialization
placed in some other file of a package might never be loaded, and the
surprises above would return inside that package. The family's file and the
type's file are the only files that are always loaded wherever the family
and the type are used.

#### The alternative: allow it anywhere, and check

The main alternative keeps C++'s freedom: any file may specialize any family
for any type, and checks catch disagreements. Inside one program, the
late-specialization error of section 9.3 would do the checking. Across
packages, the definition fingerprints of section 13.7 would also have to
record every choice and every declared specialization, and a later build
would repeat each choice to find disagreements.

| | Owning-file rule | Anywhere, plus checks |
|---|---|---|
| Can two parts of a program choose differently? | No | Yes |
| Is that noticed? | It cannot happen | Yes, when the two parts meet |
| Who sees the error | The author of the misplaced or clashing specialization, at once | Whoever first combines the two files or packages |
| Can they fix it? | Yes: move or change the specialization, write the overlap, or wrap the type | Only by changing one of the libraries; with prebuilt libraries, not at all |
| A forgotten import | Cannot happen: the owning file is always loaded | The general version is used silently, and no check can see it |
| Someone else's family for someone else's type | Needs a wrapper type or a policy parameter | Allowed |
| Changing the decision later | Can be relaxed without breaking code | Cannot be tightened without breaking code |

C++ allows specializations anywhere, but every use must see the same one. It
does not check this: a violation is "ill-formed, no diagnostic required", and
the linker silently keeps one of the different copies. C++ also restricts its
own library: a program may specialize a standard template such as `std::hash`
only for a type that the program defines.

Rust's orphan rule is the strict choice: a crate may implement a trait for a
type only if it defines the trait or the type, with an ordering condition
like HUC's. Its reasons match the table: two libraries can never conflict,
adding a dependency never breaks a build, and library owners can add
implementations later without breaking users.

#### The cost, and a possible relaxation

A type's file cannot write a pattern with a binder before its own type, such
as `pick<T, B>` or `hash<Pair<T, Point>>`. The family's file can still write
it, and a family's author can put the arguments that users specialize on
first.

The larger cost is glue code: specializing someone else's family for someone
else's type. C++ programmers do this often, for example by writing
`fmt::formatter<glm::vec3>` in their own program. Two ways remain:

- Wrap the type, for example `pub struct MyPoint { let value: geo::Point; }`,
  and specialize for `MyPoint`. Rust programmers call this the newtype
  pattern.
- Design libraries to take a policy type instead of relying on a global
  specialization. For example, a hash map can take a hasher type, as Rust's
  `HashMap<K, V, S>` does.

If glue turns out to be common, a later relaxation could let the program's
root file specialize any family for any type. That would not bring back the
surprises above:

- Nothing imports the root, so it is always part of its program.
- It is processed after all its imports, so the late-specialization error
  catches any clash with an earlier choice, and the end-of-file check sees
  every other specialization.
- Across packages, definition fingerprints would record the choices that
  libraries already made, so the same check would work there.

Section 18 lists this as an open question.

## 10. Generated code with explicit public dependencies

### 10.1 A builder re-exports dependencies used by its generated code

`library/output.huc1`:

```huc
module library.output;

pub let mut total: i32 = 0;

pub fn record(value: i32) -> void {
    total += value;
}
```

`library/log.huc1`:

```huc
module library.log;

import1 compiler;
pub import library.output;

pub fn record_expr<>(value: i32) -> compiler::Code {
    return compiler::local(
        "library::output::record(" + compiler::expr(value) + ");",
    );
}
```

`app/main.huc1`:

```huc
module app.main;

import library.log;

fn main() -> i32 {
    let amount: i32 = 7;
    #log::record_expr(amount);
    return library::output::total;
}
```

The inserted statement is equivalent to:

```huc
library::output::record((amount));
```

The caller can name `library::output` because `library.log` re-exports it.
The same permission applies to the inserted statement and the caller's
handwritten `return`. Text carries no extra import rights. The inserted call
is checked in `app.main`, so `record` must also be `pub` (section 6.6).

If `library.log` used only `import library.output;`, this client would need
its own import of `library.output`. The builder's private dependency would
not be enough.

The example passes a runtime local, `amount`, so `compiler::expr(value)` is
valid. Passing the literal `7` instead would bind a compiler value, not an
expression parameter. A builder that also accepts compiler values must
handle that case separately under the existing parameter-binding rules.

### 10.2 Generated module paths do not use import aliases

If the builder instead returned this text:

```text
output::record((amount));
```

Generated module references must use canonical module paths. Import aliases
are not consulted for those paths, even aliases declared by the caller.

For example, suppose the caller also has:

```huc
import app.output as output;
```

In the caller's authored code, `output::record(amount)` names
`app::output::record`. In inserted text, the alias-only spelling
`output::record((amount))` fails; it cannot silently select that alias.
The builder should emit `library::output::record((amount))` instead.

There is one literal-spelling detail: a visible module actually named
`output` has the canonical path `output`. Text using that path can name it.
The rule disables alias lookup; it does not guess which module the author
intended from a string's spelling.

For a concrete test, suppose both `library.output` and a distinct module
named `output` provide a compatible `record` function. A caller imports them:

```huc
import library.output as output;
import output as real_output;
```

Authored `output::record(amount)` uses `library::output::record`. The same
spelling in generated text uses the real module `output`, even though it was
imported as `real_output`. That is accepted, not an error. Builders must still
write the intended canonical path; plain text cannot preserve their intent.

This check happens when text is inserted and its names are resolved, not
when the builder constructs a string. Unused text is not checked as HUC.
An error can show both the insertion site and the text's origin, but origin
information never grants a dependency or selects a different declaration.

Only module-path lookup changes. Locals, fields, methods, and unqualified
names still follow the normal insertion-site rules. The original user block
wrapped by `compiler::around` remains authored code and keeps its aliases;
the generated prefix and suffix follow the generated-text rule.

`compiler::expr` already prints resolved module references with concrete,
canonical names. Its local names are still plain text: a generated local can
capture them if the builder chooses a conflicting name. This model avoids
module-alias capture, not every kind of name capture.

### 10.3 Re-exports compose through public wrapper modules

Suppose a facade returns the first builder's code:

```huc
module library.facade;

import1 compiler;
pub import library.log;

pub fn record_expr<>(value: i32) -> compiler::Code {
    return log::record_expr(value);
}
```

A client importing `library.facade` can insert the returned code. The
facade exposes `library.log`, which exposes `library.output`. The expression
parameter is forwarded under the existing rules.

If the facade only privately imports `library.log`, its clients do not gain
access to `library.output`. It can re-export the log module as above, or
directly re-export just `library.output` to expose a smaller public interface.
The required property is a visible path to every module named by the
generated text, not a special exemption for wrapper builders.

### 10.4 Plain text does not carry extra access rights

Returning text from another module does not grant access to any arbitrary
module named by that text. Insertion uses the relevant source environment
and visible-module set, together with the ordinary local scopes. Equal text
inserted into the same environment follows the same lookup rules regardless
of which builder produced it.

Code values stored and later inserted elsewhere must satisfy the later
insertion site's rules. Creating a fragment does not validate its names,
preserve local bindings, or extend the lifetime of a represented object.

A specialize prefix in generated module text changes the specialization's
namespace, not its source dependency permissions or ownership. Section 6.4
also restricts foreign-namespace additions. Function-owned insertion anchors
and program/evaluation modes otherwise retain their existing rules.

### 10.5 A passed type and text spelling its name are different

Section 9.1 passes `Point` to a library as a type argument. The parameter `T`
denotes that type without looking up `app::main::Point` in the library.

Text containing `app::main::Point` is different. If inserted at a `#` in the
library's source context, it requires that module to be visible there. If
returned and inserted at a `#` in `app.main`, it may name the caller's own
type. A function-owned output anchor after the caller does not turn the
library's `#` into the caller's `#`.

HUC has no type-to-text API yet. If one is added later and returns
plain text, that text must follow these same lookup rules. Giving it a
preserved type binding would require an explicit new facility, not a hidden
property of equal strings.

### 10.6 Generated duplicates need the existing narrow sharing rule

Generated HUC0 declarations use the same declaration registry. A generated
function cannot silently replace a differently defined function or variable.

The specification has a narrow rule for sharing identical generated
module declarations when their lexical naming contexts also match. This
model does not remove it, broaden it to authored definitions, or apply it
to `let1` initialization. Generated fragments are HUC0 and cannot declare
`let1` in the first place.

For example, two requests that generate the same helper may share it under
that rule. Two helpers with the same name but different bodies must diagnose.
Both builder calls still execute, even when their output is shared.

The sharing check must also compare dependency permissions and resolved
bindings. Matching characters alone are not enough. In particular,
different prepared constants or different referenced declarations must not
be hidden by merging text. When sharing cannot be established under the
specified rule, diagnose the conflict rather than selecting the first body.

### 10.7 Late names and late specializations

State isolation keeps a completed module's values unchanged. A module's names
need the same protection. A deferred body can be prepared
long after its module finished, and a module action can add declarations to that
module's namespace:

```huc
module library.gen;

import1 compiler;

pub fn install<>() -> void {
    @compiler::module("pub fn generated() -> i32 { return 7; }");
}
```

Suppose an ordinary function in `app.a` calls `gen::install()`. That runtime
request prepares `install<>`, which adds `generated` to `library.gen` after
`app.a`'s anchor. Now another file uses it:

```huc
module app.b;

import library.gen;

let1 seven: i32 = @gen::generated(); // Error: generated was added later.
```

Without a rule, this would work if the root imported `app.a` before `app.b`,
and fail the other way round. No saved value changed; what differed was
which ordinary names existed. Late ordinary names are therefore restricted:

- A declaration that a deferred body generates into its module after the
  module finished is visible only to the preparation that generated it.
- Other files cannot name it, even if it is marked `pub`.
- Another preparation can name it only by generating the identical
  declaration itself. Section 10.6's sharing rule then keeps one copy.

An evaluation rejects module actions, so it cannot use a late-generated
helper either. Otherwise `@read::read_value<T>(&item)` would work or fail
depending on whether another module had requested `read_value<T>` at run
time first. A body that needs such a helper at compile time takes its own
path with `compiler::emitting()`, as the `area` example in specification
section 11.1 does. Helpers generated while their module was being processed
are part of its names, so later evaluations may use them.

The [use cases](use-cases-and-examples.md), section 7.7, follow this rule:
each request for `read_value<T>` generates the same `bonus` helper and calls
it. Code that other files should use belongs in the module's own source, or
in a builder that returns code for the caller's `#`, so that it lands in the
caller's file.

Late specializations and impl blocks are different: they apply to code after
their insertion point. A specialization that would change an earlier request
is rejected, as is a generic impl after an applicable request. A late ordinary
impl can change later overload choices. Request/import order may therefore
affect programs using late module generation; per-module state isolation does
not promise order independence for such generated declarations.

## 11. Flattened HUC0 remains a real source boundary

The driver selects the input kind explicitly; a `.huc0` extension, absence
of imports, or source map does not identify a flattened program.

| Command | Input rules |
|---|---|
| `huc stage app/main.huc1` or `huc build app/main.huc1` | Source files use the import and ownership rules above |
| `huc lower app/main.huc0` | Handwritten HUC0 source uses those same source rules and may import other HUC0 sources |
| `huc lower --flattened build/program.huc0` | One complete HUC0 program; imports are rejected, and source-file ownership and `pub` rules do not apply |

`--flattened` selects the second input kind. Keeping source input as the
default preserves the path for handwritten HUC0 with imports. The two kinds
use the same runtime language; the difference is source loading and
namespace access.
`build` uses the explicit flattened input kind for the HUC0 file it writes.

For example, this handwritten file is rejected by default `lower`, because
it adds an ordinary function to a namespace it does not own. With
`--flattened`, it is a complete program returning `42`:

```huc
module app.main;

fn main() -> i32 { return library::math::answer(); }

module library.math;

fn answer() -> i32 { return 42; }
```

Source-module dependency permissions are checked before flattening. The
printer then writes canonical names and ordinary concrete declarations.
Source imports and re-exports disappear. Checks on aliases in generated
pieces have already happened; the resulting HUC0 uses canonical module names.

For the generic example, the printed concrete function may have a parameter
whose type is spelled `app::main::Point*`. That is the printed form of an
already supplied type identity, not a new lookup by the original library
source.

Already-flattened HUC0 is one explicit runtime compilation unit. Its
namespace sections do not reconstruct the original file import boundaries.
The first section names the root, but does not claim ownership of every
namespace in that file. The printer may open and reopen all namespaces needed
by the program. The source-file ownership rules in section 6.4, the `pub`
rule in section 6.6, and the specialization placement rule in section 9.5 do
not apply to this import-free output; they have already been checked before
flattening.

The standalone HUC0 reader checks runtime syntax, types, lifetimes, duplicate
definitions, and references within that unit. It does not need hidden HUC1
state or an origin sidecar to recover erased source-import permissions.

Anyone may write a flattened program and select `--flattened`; no compiler
stamp or hidden sidecar is required. This is a source-organization boundary,
not a security boundary. Generated fragments inside a source build still
follow their insertion site's source rules; they cannot select a different
input kind by containing a `module` line.

Source maps remain useful for diagnostics and debugging, but they are not
required to give the flattened program its runtime meaning.

## 12. After 0.1: reuse finished builds, not compiler-call results

This section is a possible later optimization, not part of release 0.1. For
0.1, each build invocation starts a fresh session and rebuilds its program.
Recording dependencies does not enable artifact reuse by itself.

The 0.1 input record describes the build that ran; it does not prove that
nothing relevant has changed since. An external tool skipping `huc build`
would also be reusing an artifact. Moving that decision outside HUC does not
remove the validation requirements in section 12.4. Using the 0.1 record
alone to skip a build is not supported.

### 12.1 Two kinds of reuse must not be confused

| Operation | Behavior |
|---|---|
| Import an already loaded source in one session | Reference the existing module |
| Request an existing runtime specialization | Reference the existing definition |
| Repeat a compiler call during compilation | Execute it again under the staging rules |
| Rebuild in release 0.1, even with unchanged inputs | Start fresh and compile again |
| After 0.1: rebuild with unchanged complete inputs | May reuse validated finished artifacts |
| Rebuild after a relevant source or configuration change | Begin a fresh compilation session |

The first artifact-reuse implementation need not reuse results within a
changed program. If one source changes, preparing the whole HUC program again
is acceptable. This is a deliberate limit on incremental performance, not a
hidden promise of per-module reuse. Section 13 describes later reuse at the
level of packages.

### 12.2 A future no-change build

```text
Build 1:
  load sources -> execute compiler work -> write HUC0 -> compile -> link

Build 2, with all relevant inputs unchanged:
  validate inputs and outputs -> reuse the finished build
```

Build 2 does not need a live `let1` heap. It reuses the completed result, not
the compiler objects that produced it. HUC continues to forbid
untracked host effects during compiler execution.

### 12.3 A changed source starts fresh

```text
Build 3, after editing app/parser.huc1:
  new session -> ids starts at 0 -> compiler calls execute -> new output
```

The example in section 7 again assigns IDs `1` and `2`. It does not continue
from the previous build's final counter value.

### 12.4 Reuse needs complete inputs and validation

Reuse must account for at least:

- Root identity and all loaded source contents.
- Module mappings, search paths, and source-resolution decisions.
- Compiler and compiler-API versions.
- Target, language options, code-generation options, and relevant limits.
- Toolchain inputs needed by the particular stage being reused.
- Required generated inputs and any future tracked external-input API.
- The presence and integrity of the expected outputs.

If the C-interoperability proposal is adopted, this also includes each C
unit's contents, included headers, macro definitions, include paths, sysroot,
C options, target, and Clang/LLVM version. Link-stage reuse must account for
the selected object files, libraries, linker, and linker options too.

A root source timestamp alone is insufficient. Changes in the module search
environment can select a different provider even when the previously loaded
files have not changed. The build driver must revalidate resolution or use
a complete explicit input mapping. If it cannot prove an output is current,
it rebuilds.

The same issue applies to C headers. A new header earlier in an include path
can replace a previously selected header without changing that old file.
Hashing only files that were read last time misses this change. Resolution
checks must cover those earlier search locations as well.

A basic first implementation may use one conservative key for the entire
build. More selective stage keys can follow later. For example, a changed
linked library need not ultimately force HUC1 to run again, but correctness
comes before that optimization.

### 12.5 Identical HUC0 does not always mean identical backend inputs

Suppose a source edit produces unchanged HUC0 runtime text. Backend reuse
may be possible, but it must also check target/options and any debug or
source-origin inputs that affect the object file. Do not key a debug build
only by the HUC0 text and accidentally keep stale source locations.

The mandatory HUC0 artifact must still be available to users. An intact
validated artifact may be retained rather than rewritten solely to update
its timestamp.

Enabling any of this later would require changing the architecture's and
implementation plan's no-artifact-caching policy, and the equivalent rule in
the C-interoperability proposal if adopted. Those rules stay in force for
0.1. This model does not add compiler-call caching or a cacheability
analysis.

## 13. Packages after 0.1

This section sets the rules that separately compiled packages must follow.
It is not their full design: the manifest format, versions, and the way saved
compile-time values are stored still need a separate proposal. Release 0.1
keeps one package per program.

### 13.1 What a package is

A package is a group of modules that is compiled in one run and saved, so
that later builds can reuse it. It matches a Rust crate, or a C++ library or
executable target. A program is a root package plus the packages it depends
on. In 0.1, the whole program is one package.

The files inside a package are its modules, as today. The file that a build
starts from is the root module (section 2).

This document says "package" rather than Rust's "crate". In Rust, one Cargo
package can hold several crates. In HUC, one manifest describes exactly one
package, so one word is enough.

`pub` works across packages just as it works across files (section 6.6). A
later level for "this package only", like Rust's `pub(crate)`, would let a
package's files share helpers without making them part of its public
interface.

### 13.2 Manifest and build tool

Package management is built into `huc` rather than shipped as a second tool,
as in Go and Zig. For example, `huc build` builds a package and its
dependencies, and `huc test` runs its tests. A manifest file, for example
`huc.toml`, lists the package's name, version, and dependencies. It replaces
hand-set module search paths for dependencies. The tool should not be called
`cargo`, because Rust's `cargo` command is already installed on many machines.

Later, a package may also describe its build in HUC itself, as Zig does with
`build.zig`. HUC's compile-time code already runs inside the compiler, so this
needs no second language. Unlike Zig's build code, it cannot read files or run
programs (specification section 9.11). That keeps builds repeatable, but it
limits what a build description can do. The dependency list should stay plain
data, as in Zig's `build.zig.zon`, so that other tools can read it without
running code.

### 13.3 Two levels of reuse

Rust reuses work at two levels:

- **Between crates:** a crate whose inputs did not change is not compiled
  again. Its saved output is reused, like a prebuilt library.
- **Inside a crate:** when a crate must be rebuilt, the compiler reads all its
  files again, but it reuses saved results for the parts that did not change.
  This is Rust's incremental compilation.

HUC plans only the first level for now. State isolation (section 8.4) makes
reuse inside a package possible later, because each module's compile-time
results depend only on its source, its dependencies, and the build settings.
It still needs the saved-module design that section 13.8 describes. Until then,
large projects can split into several packages to get faster rebuilds, as Rust
projects do.

For example, suppose `app` depends on `net`, which depends on `json` and
`tls`:

- First build: all four packages are compiled.
- Edit only `app`: only `app` is compiled again.
- Edit a file in `net`: `net` and `app` are compiled again, because `app` uses
  `net`. `json` and `tls` are reused.

Within each package, sections 5–11 apply unchanged. The three rules below make
separately compiled packages behave like one program.

### 13.4 Rule 1: preserve each module's completed state

A saved package retains each module's completed compiler state. A consuming
module merges that result exactly as it would after processing the source.
It may then change its own values without writing back into the saved package.

Preserve logical declaration and allocation identities across diamond imports,
including aliases and lifetime records. Matching names or equal bytes are not
enough. A package is not one shared mutable state for all of its modules.

For example, two packages that each import a counter initialized to 3 and
increment it independently both produce 4. Their parent merges 4, not 5.
That is also the result in one whole-program build. For globally unique IDs,
the parent must collect and number the entries itself, rather than expecting
independent dependency branches to advance one shared compiler counter.

Imports of saved modules are merged in source order. Later values win even
when unchanged. This makes a source build and a build using saved packages
agree without replaying initializers or making compiler-call results persistent.

### 13.5 Rule 2: saved packages keep generic bodies

When package `lib` is built, it cannot know every caller-defined type that
will later be passed to a generic. A consumer therefore needs the generic
bodies, their source lookup context, and the completed state of each defining
module, as well as declarations and type identities.

Program-mode preparation uses the defining module's state. Evaluation-mode
calls use the active consumer state. These are the same rules as within one
whole-program build; loading an artifact does not select a different behavior.

Saving object graphs with pointers and allocations is part of artifact
serialization, not permission to change a dependency's saved state. Section
13.8 lists the remaining implementation work.

### 13.6 Rule 3: use an existing copy

If a dependency already contains a prepared runtime definition, such as
`largest<Circle>`, use that definition rather than preparing a new one.
Its constants may have been computed earlier while its own module was still
being processed. The completed module state can contain later values.

This extends the once-only runtime-definition rule within one program.
Fresh compiler calls still run and can read the active caller's values; they
do not replace an existing runtime definition.

When independent packages each need a runtime definition that no dependency
already provides, both prepare it from the same defining module's completed
state, not their own merged states. The definition and dependency fingerprints
in section 13.7 check that their results agree. The final build retains one
compatible definition and rejects incompatible ones.

### 13.7 Definition fingerprints

The rules keep correct builds consistent. Definition fingerprints catch builds
that are not correct:

- a package built against an older version of a library;
- packages built with different options, target, or code-generation
  settings.

Each saved package stores:

- For every copy it contains, its name and a hash of its checked HUC0 body.
  For a type, the hash covers its layout and members.
- Its content hash: one hash of everything it saves. This covers signatures,
  type layouts, generic and ordinary function bodies, completed module states
  with object identities and lifetime records, ordered imports, and
  specializations.
- For every dependency, direct or indirect, the content hash of the version
  it was built against.
- Its build settings.

The dependency hashes cover whole contents, not only signatures and layouts,
because a package keeps whatever it computed from a dependency at compile
time. For example, if package `app` runs `@library::answer()` while it is
built, the answer is stored in `app`. If `library` later changes `answer`,
`app` is stale, even though no signature or layout changed. The same holds
for copies that `app` made from `library`'s generic bodies, or that read
`library`'s saved values. Rust does the same: rustc stores a hash of each
crate and refuses to mix crates built against different versions of a
dependency (error E0460).

So any change to a library means rebuilding the packages that use it. A later
version could record only what each package actually read. Then, for
example, changing the body of an ordinary function that no dependent ran at
compile time would need only relinking.

The copy hashes use the checked HUC0 form, not machine code, because
optimizer and inlining choices change machine code even when the source is
the same. Source locations are left out, so builds in different folders
match. Architecture section 9.3 already gives each copy a deterministic name
that depends on its family and arguments, not on its body. So two copies with
the same name must have the same hash.

Each package build loads the fingerprints of everything it depends on and
checks three things: every package was built against the same content hash
for each shared dependency, the build settings fit together, and copies with
the same name have the same hash. The final program build is the last check,
so no separate step is needed before linking. Fingerprints are stored inside the
package output and the final binary, as Rust stores metadata inside `.rlib`
files, so they cannot get lost or go stale. Only a non-HUC tool doing the
final step, such as a plain C linker or `dlopen`, skips the check.

Because of the owning-file rule (section 9.5), a fingerprint error always
means a build mistake. Rebuilding everything with the same versions and
settings fixes it. The fingerprints also act as an index: they tell a package
which copies its dependencies already contain (rule 3).

The name follows architecture section 16.2, which already plans "definition
and dependency fingerprints" for a future interface file. Cargo uses
"fingerprint" for its change-detection hashes, and Clang calls its similar
hash an "ODR hash".

### 13.8 Why not save each module separately?

Packages are saved whole. Saving each module separately, like a C++ PCM file,
would be a further step.

A PCM-like artifact saves a compiler representation for later imports. It
does not, by itself, decide variable ownership or prevent conflicting
definitions. Clang also distinguishes header-unit PCMs from named-module
artifacts: a header-unit PCM cannot simply be compiled into its own object
file. See [Clang's header-unit documentation](https://clang.llvm.org/docs/StandardCPlusPlusModules.html#header-units).

For HUC, a fully prepared module artifact would need rules for:

- Saving and restoring compiler objects, including pointers and allocations.
- Preserving shared logical object identities and isolated values across
  dependency branches, with ordered whole-object merging.
- Deferred generic bodies and ordinary bodies callable during evaluation.
- Caller-triggered specialization and generated declarations.
- Source contexts, dependency identity, and artifact version compatibility.

Those questions are larger than saving a parsed file. Merely assigning
numbers to compiler allocation blocks does not define how shared pointers,
type identities, or compiler handles should be restored.

State isolation (section 8.4) means a finished module's results cannot be
changed by importers or siblings. Rules 1–3 and definition fingerprints answer
more of these questions for whole packages. The rest still apply to packages
too.

Keep the first module records in memory. After 0.1, whole-build reuse is a
smaller next step than serializing each module's live compiler heap.

This does not rule out a future syntax/preparse cache or prepared module
interface. Those provide different amounts of reuse and should not all be
described as the same feature.

## 14. HMake is optional coordination

The compiler driver must be able to build an ordinary program without HMake.
After 0.1, the package tool built into `huc` (section 13.2) handles
dependencies, so HMake stays optional. HMake can use its stage commands and
inspect its input records. Those records are for diagnostics and
reproducibility in 0.1, not rebuild-skipping decisions.

For example:

```text
HMake builds a generated HUC input.
HUC compiles each requested program afresh and reports the inputs it read.
HMake schedules independent executable builds in parallel.
Each HUC compilation owns its own let1 state.
```

A later IPC protocol could let a compiler request an input while other
independent work proceeds. HMake would not need to understand specialization
selection or mutate the compiler's declaration registry.

The protocol, if added, should not change program meaning compared with
one-process compilation. Required compiler-state ordering remains the
compiler's responsibility. Detect dependency cycles rather than waiting
forever on an input whose producer waits on the requesting compilation.

## 15. Runtime emission and future libraries

One checked runtime entity must have one definition owner. Initially that
owner is the program's one HUC output object.

Only `main` and functions exported to C ([C interoperability](c-interop.md),
section 6.2) need link names that are visible outside the program. Every other
HUC symbol can stay internal to it, which is what C++ `static` gives one file.

If the backend later splits output across several objects, choose one owner
for every function and global and emit references from other objects. Do not
create a private global in each object merely to avoid a linker collision.

For example, splitting parser and renderer backend work must still leave
one `app::common::hits`, not a private counter for each output object.

Section 13 sets the rules for separately compiled packages. Their full
design, including interfaces and ABI agreement, still needs a separate
proposal. This model does not promise that two separately compiled copies
of a HUC source library can be linked together as if they were one shared
instance.

The existing supported C boundary remains available. The separate
[C interoperability proposal](c-interop.md) is not adopted by this document;
neither its header-import pipeline nor its choice of C compilation units is
changed here. If that proposal is adopted, reconcile the LLVM output unit
as noted in section 4. One program-level LLVM module does not require one
combined C source unit or a shared C preprocessor environment.

## 16. Implementation sequence and limits

Implement the model in small steps:

1. An in-memory source-module loader with canonical identities and cycle
   diagnostics.
2. Separate lookup environments with direct imports, explicit re-exports,
   `pub` checks, and canonical generated module paths, backed by one
   declaration registry.
3. The owning-file check for specializations, with the left-to-right
   condition and the end-of-file clash check (section 9.5).
4. Once-only module initialization and program-definition preparation,
   while preserving fresh compiler-call execution.
5. Isolated compiler states, completed-state records, and ordered import
   merging for all supported compiler values (section 8.4).
6. Flattened HUC0 output and the existing standalone runtime pipeline.
7. An input record for diagnostics and reproducibility, not artifact reuse.

After 0.1, consider conservative whole-build artifact reuse first. Then add
packages under section 13: the manifest and built-in tool, rules 1–3, and
definition fingerprints. Build-system coordination and independent backend
work can be separate improvements. Independent executable builds can already
run as separate jobs, each with its own compiler state.

A small edit inside one large package may still require a full frontend run.
This model does not promise reuse inside a package or measured build-time
gains. State isolation makes reuse inside a package, and parallel processing of
modules that do not depend on each other, possible later.

A general generated-name hygiene system remains a separate decision. Section
13 sets the rules for packages; their full design is still separate.

## 17. Acceptance examples and checks

The implementation must pass these tests. None has run yet.

| Case | Expected result |
|---|---|
| Parser and renderer both import common | One declaration identity per global; independent compiler states and one shared runtime global |
| The same module is directly imported twice in one file | Diagnostic, even with different aliases or import forms |
| An import follows a declaration or a later module line | Diagnostic; the file's import block has ended |
| A specialization section uses the file's initial import aliases | Accepted in authored code; the import environment is unchanged |
| Different files import a module under different aliases | All aliases refer to the same declarations |
| A direct import follows an indirect load | Accepted; no repeated initialization |
| Ordinary import is used to name a module's let1 and runtime declarations | Accepted under each declaration's phase rules |
| Distinct modules are imported with as _ | No alias conflict; each module loads normally |
| The same module is imported twice with as _ | Duplicate-import diagnostic |
| pub import uses as _ | Canonical module access is re-exported; no local alias is bound |
| One dependency is requested through equivalent paths | One provider identity, not duplicate initialization |
| Two providers claim the same logical module | Clear conflict diagnostic |
| A imports B and B imports A | Import-cycle diagnostic |
| Parser names a root-only graphics module | Diagnostic regardless of the root's import order |
| Root names a dependency reached only through a private import | Diagnostic |
| Root names a module reached through explicit re-exports | Accepted |
| A private edge interrupts a chain of re-exports | Visibility does not cross that edge |
| Root uses an alias declared only inside parser | Diagnostic |
| Another file names a declaration that is not pub | Diagnostic |
| The root module's main has no pub | Accepted; the driver finds main without an import |
| A private type is passed to a library generic | Accepted; the generic never names it |
| Generated code names a builder's private helper | Diagnostic at the insertion site |
| A module repeats its own variable definition | Diagnostic with both definition locations |
| Another source adds an ordinary declaration to common | Diagnostic even if the name is new |
| A source opens an unowned child namespace such as app.common.detail | Diagnostic; a name prefix does not grant ownership |
| Equal variable names in different namespaces | Distinct permitted variables |
| Two calls in one module update that module's let1 counter | Both calls run and update the same object |
| An importer assigns to an imported let1 mut | Changes only the importer's state |
| A module-level assignment item such as a::t = 5; | Runs in the compiler in source order and is erased |
| A module-level assignment item targets a runtime let global or uses pub | Diagnostic |
| A struct body assigns a let1 mut | Runs while the type is prepared, in that preparation's state |
| A struct body assigns one of its own runtime fields as a compiler item | Diagnostic |
| A generic struct requested late from another module assigns a module counter | The write stays in that preparation's private view |
| An importer changes an imported let1 through a pointer, a method, or a compiler call | Same isolation as direct assignment |
| A library function changes a let1 that its caller owns, through a pointer | Accepted |
| Two dependencies change a shared imported value to 5 and 7 | Each keeps its own state; their parent gets the later value |
| Imports with conflicting values are reversed | Only the parent's merged state changes; dependencies remain unchanged |
| A later dependency leaves the original 3 unchanged | Its 3 replaces an earlier branch's 5 |
| Two dependencies independently increment an inherited 3 | Both produce 4; their parent gets 4, not 5 |
| Branches change different fields of one structure | Later whole structure wins; fields are not combined |
| Direct and pointer references identify one imported object | They resolve to the same version in the active state |
| Branches create separate allocations with equal contents | Distinct logical objects; no content-based merging |
| The winning version has an inactive slot or deallocated block | That lifetime state wins; subsequent invalid access diagnoses |
| A noncopyable Advanced object is imported | State sharing needs no `copy` method and does not deactivate the dependency's value |
| A compiler helper reads or writes an imported let1 mut | It uses the active caller state, with lexical name lookup |
| A retained generic definition or canonical type is first requested by an importer | Uses its defining module's state, not caller overrides |
| A selected specialization reads its own module's object absent from the active state | Read-only fallback to that module's completed state |
| The missing object's module has not completed | Diagnostic; no implicit finishing or lookup in the requester's state |
| An active entry exists but is inactive, uninitialized, or deallocated | Normal lifetime diagnostic; no fallback |
| A fallback object is modified, relocated, destroyed, or freed directly or through aliases | Diagnostic; borrowed entries are read-only and get no local teardown |
| Two late preparations increment a saved counter starting at 0 | Each gets 1 in a fresh view; saved state remains 0 |
| A new retained definition is requested inside a late preparation | It gets its own preparation view; compiler calls stay in the current view |
| Saved package imports replace source imports | Same object identities and ordered state merges |
| The root reverses its imports of two modules that read one shared module | Same compile-time values; only the order of declarations in the output may change |
| A module-level let1 value's drop clears fields, destroys elements, and frees memory at teardown | Accepted in that state's private cleanup view |
| A drop changes an accessible mutable imported object during teardown | Changes only that cleanup view; other saved states remain intact |
| A module changes a runtime let global of an imported module at run time | Accepted; runtime globals are shared, not isolated compiler state |
| Two calls use a function-local let1 counter | Separate local lifetimes under the existing staging rules |
| Two callers request one runtime specialization | One retained definition |
| A later specialization changes an earlier selection | Diagnostic |
| A file that owns neither the family nor a type in the pattern writes a specialization | Diagnostic at that specialization |
| A type's own file specializes a visible family for that type | Accepted; every request for it sees the specialization |
| A type's file writes a pattern with a binder before its own type | Diagnostic at that specialization |
| A type's file writes a pattern with its own type nested before every binder | Accepted |
| Two files' specializations of one family can match one request, and neither is more specific | Diagnostic in the later file when it finishes |
| That later file also writes the specialization for exactly the overlap | Accepted |
| A specialization written in a type's file calls that file's private helper | Accepted; the body is checked in the file that wrote it |
| A module action in such a specialization | Diagnostic; the family's namespace belongs to another file |
| A deferred body adds a pub declaration to its finished module, and another file names it | Diagnostic, whatever the import order |
| Two requests of one generic generate the same helper after its module finished | One shared declaration; each preparation may name it |
| An evaluation calls a deferred body that uses a helper generated after its module finished | Diagnostic, whatever the import order |
| That body takes another path in evaluation mode with compiler::emitting() | Accepted |
| A specialization pattern uses an alias of a built-in type | Diagnostic; an alias gives no ownership |
| Generic receives a caller-defined type | Accepted through the type parameter |
| Builder emits a canonical name of a re-exported module | Accepted at a compatible insertion site |
| Builder emits an alias-only module path | Diagnostic at insertion; caller aliases cannot take it over |
| Generated output:: names a real visible module output while source alias output names library.output | Accepted; generated text names output, authored alias use names library.output |
| An original block wrapped by compiler::around uses its source alias | Authored-code alias lookup is preserved |
| Plain generated text names an unavailable caller module | Diagnostic, even if the builder received one of its types |
| Generated declarations differ in meaning | No silent conflict suppression |
| A release 0.1 build has unchanged relevant inputs | Fresh compilation |
| A tool uses the 0.1 input record alone to skip compilation | Not supported; the record is not a freshness check |
| After 0.1: completed build has validated unchanged inputs | Finished artifacts may be reused if that feature is adopted |
| Source or module resolution changes | Fresh compilation; no old let1 state |
| After 0.1: debug origins change while runtime text does not | Backend reuse checks the debug inputs too |
| After 0.1: a new earlier C header changes include resolution | Previous artifact is not reused as if inputs were unchanged |
| After 0.1: a package changes an imported let1 mut value | Changes only the active module state; the dependency artifact stays unchanged |
| After 0.1: a package needs a copy that a dependency already contains | The dependency's copy is used |
| After 0.1: two packages that don't depend on each other both make `largest<i32>` | One copy is kept; their fingerprints match |
| After 0.1: two packages were built against different versions of a library | Fingerprint diagnostic naming both packages |
| After 0.1: a package used a dependency's compile-time result, and the dependency changed with no signature or layout change | Fingerprint diagnostic; the package must be rebuilt |
| Two roots are compiled by one long-running process | Separate compilation state and globals |
| Default lower receives the two-namespace source example from section 11 | Source ownership diagnostic, even with no imports |
| lower --flattened receives the same example | Accepted without source ownership checks or hidden HUC1 state |
| lower --flattened receives an import | Diagnostic; the input must already be one complete program |

## 18. Open questions

The model is adopted, but these edges still need a focused review during
implementation or later design work:

1. **Source identity details.** Choose the exact module-resolution conflict
   rules, including provider aliases, symbolic links, and case sensitivity.
2. **Insertion contexts.** Check that each `#` uses the imports of the file
   containing it, including in a generic body whose output lands after a
   caller elsewhere. Section 10.5 settles passed types versus plain text.
3. **Generated sharing.** Audit the identical-declaration sharing rule
   against resolved names, dependency permissions, and prepared constants.
   Keep it separate from ordinary variable redefinition.
4. **Root-file specializations.** Decide later whether the program's root file
   may specialize any family for any type (section 9.5).
5. **C output units.** Reconcile the per-module LLVM wording if the separate
   C-interoperability proposal is adopted.
6. **Package design.** Choose the manifest format, versioning, a
   package-only visibility level, and how completed compiler states are
   saved (section 13). Also consider a small prelude: a few standard names
   that every file can use without an import, like Rust's `Vec` and `Option`
   (appendix A.1).

Unused-import warnings are a separate future question. An import can be
intentional even when no names are used: the module may declare
specializations that other code relies on. Section 5.3 defines `as _` as an
import without an alias. If an unused-import warning is added later, that
form should be exempt. Ordinary imports kept for their effects remain valid
too.

Artifact validation is a separate post-0.1 task. Define its complete input
record and revise the no-caching policies before enabling reuse. A rebuild
is preferable to an unverifiable hit.

## 19. Final position

This model deliberately keeps one global view of program identity while
restricting what individual source modules can look up. Repeated imports
share declaration identities while merging module-local compiler state.
Conflicting definitions are checked by the compiler, not left for a linker
to choose between.

The first step is reliable load-once behavior, explicit public dependencies,
private-by-default declarations, specializations that live in an owning
file, and isolated compiler states saved when each module finishes, with fixed
module names and no persistent compiler state between builds. Finished-build
reuse and separately compiled packages come after 0.1, under the rules in
section 13. Per-module artifacts remain a possible later step.

Each module's compile-time results depend only on its own source, the
modules it imports in source order, and the build settings. Reordering sibling
imports may change the parent's merged values, but not those dependencies'
states, names, or specializations. When two files' specializations clash, the
author of the second one sees the error at once.

## Appendix A: Comparison with C++ and Rust

### A.1 Global names

HUC has no global namespace for your own code. Every file starts with
`module name;`, and everything it declares lives in that module's namespace.
To use another file's declaration, import its module and write
`alias::name`. There is no `using namespace`.

A few names really are global:

- Built-in types and intrinsics, such as `i32`, `size_of<T>()`, and
  `construct_at`. They work like C++ keywords and `sizeof`.
- Link names. `main` and functions exported to C keep plain names in the
  object file (section 15).

Module-level variables are global variables with namespaced names, like
`namespace app::log { int count; }` in C++. Today they are limited to plain
scalars and pointers with constant starting values (specification
section 2.2), so C++'s static-initialization-order problem cannot happen.

Rust works the same way: each crate's items live under the crate's name. Rust
adds a prelude, a few standard names such as `Vec` and `Option` that every
file can use without importing them. HUC could add a small prelude later
(section 18).

### A.2 `static` and `inline`

| C++ | HUC |
|---|---|
| `static` function or variable, for internal linkage | Not needed. Only `main` and functions exported to C need link names outside the program (section 15). |
| `static` or an unnamed namespace, to hide a name from other files | The default. A declaration without `pub` is private to its file (section 6.6). |
| `static` local variable | A private module-level `let mut` variable. |
| `static` class member | A module-level function or variable next to the structure. |
| `inline` function in a header, so the linker keeps one copy | Not needed. There are no headers, each function is defined once, and importing does not copy it. Generic copies are covered in section 13.6. |
| `inline` as a speed hint | Not needed. The whole program is one LLVM module, so LLVM can inline across modules, as with C++ link-time optimization. |

### A.3 Closer to Rust or to C++?

The build model is closer to Rust. The language is closer to C++.

| Rust | HUC | C++ |
|---|---|---|
| Module: a file and a namespace | Module | A source file and a namespace |
| Crate: compiled in one run | Package; in 0.1, the whole program | A library or executable target |
| `use` | `import` | `#include` plus a namespace alias |
| `pub` | `pub` | `export` on a declaration in a C++20 module |
| `pub use` | `pub import` | `export import` |
| `Cargo.toml` and `cargo build` | A manifest and `huc build` (section 13.2) | A separate build system |
| Orphan rule, per crate | Owning-file rule, per file (section 9.5) | Allowed anywhere, and not checked |

Differences from Rust:

- Two files of one Rust crate may use each other. HUC forbids import cycles,
  as Go does (section 6.5).
- Rust privacy covers a module and its child modules. HUC privacy covers one
  file.
- Rust compiles every file of a crate. HUC compiles the files that imports
  reach.
- Rust has traits instead of template specialization. In HUC,
  specializations play that role.

The language itself is C++-like: templates with specialization, and raw
pointers without a borrow checker. Moves work as in Rust: a moved-from value
is gone, not "valid but unspecified".

### A.4 Precompiled headers

C++ needs precompiled headers because every `.cpp` file parses the same large
headers again.

Within one build, HUC already avoids this. Each module is parsed once per
program, however many files import it, much like a C++ unity build. C headers
follow the same rule if each C library is wrapped in one HUC module. Under the
C interoperability proposal, all C blocks of one module form one C file. So a
header that twenty modules include is parsed twenty times, like twenty `.cpp`
files, but a header in one wrapper module that the others import is parsed
once.

Across builds, release 0.1 starts fresh every time. Later steps, from easiest
to hardest, are whole-build reuse (section 12), package reuse (section 13),
and perhaps a cache of parsed files. Caching parsed files is safe because
parsing does not depend on other modules. Embedded Clang already supports
precompiled headers, so C headers could get them sooner.

Rust has no precompiled headers either. It reuses compiled crates, and it has
incremental compilation inside a crate.
