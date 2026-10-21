# Declarations and Preparation

Status: normative companion to the [language specification](language-specification.md).
These rules apply to the complete language design; release 0.1 may implement
a documented subset. This document defines completion, declaration order,
receivers, implementation blocks, generated declarations, and residual HUC0
placement. It is not a pending proposal.

## 1. Summary

- **Incomplete and complete types.** As in C++, a structure is incomplete
  until its body has been processed and every type it holds inline is
  complete. Compile-time work that needs its layout or value properties
  before then is an error. No answer is guessed (section 2).
- **Declare before use.** As in C++, a name can be used only after its
  declaration. A generic body sees only the names declared above it. Its
  point of call adds only its arguments, their members, and specializations.
  Forward declarations of structures, functions, methods, and generic
  families let code use something defined later. Printed specializations
  follow the placement rules below. Printed HUC0 keeps every call target
  that HUC1 resolved (section 3).
- **Generated code.** `#` inserts only local code. Module code is
  generated directly: `compiler::module` and the new
  `compiler::module_before` are actions that add it after or before the
  current item. They are allowed only in program mode, and an error in
  evaluation mode, so a builder returns text for its caller to pass to
  them. Generated text may contain any HUC1 code that would be valid
  written there, including generic families and compile-time constructs
  (section 3.4).
- **`impl` blocks.** Method bodies may follow the structure in Rust-style
  `impl` blocks, for generic structures too. `init`, `copy`, and `drop` stay
  declared in the structure, so the structure alone decides the type's
  properties. Like a structure body, an `impl` is read in order and may use
  `if1`, `let1`, `@`, and `#`. An `impl` may itself be selected by `if1` or
  generated as module code (section 4).
- **Receivers and deduction.** As in Rust, a method declares its receiver
  as `&self`, `&mut self`, `self`, or `mut self`, and reaches members only
  through it, as in `self.limit`. `this` and the trailing `mut` are
  removed. A function in a structure without a receiver is a static
  function, called through its type, as in `Counter::make(10)`
  (section 4.5). Deduction may drop `mut` from a pointer argument but
  never adds it (section 3.1).
- **Call depth.** Compile-time execution allows at most 512 active calls.
  The error shows the inner and outer call frames, skipping the middle
  (section 6).
- **`specialize`.** A file specializes another module's family by writing
  `specialize` and that module's name before the one declaration, instead
  of switching namespace sections. A `module` line then names only the
  file's own module. Generated module code may use `specialize` too
  (section 7).

A generic container that queries an incomplete element type still fails.
Section 5 explains this limit and a pattern that works. Parameter-to-field
forwarding in constructors is a separate topic.

## 2. Incomplete and complete types

A structure has one identity from its first declaration on. As in C++, it is
first incomplete and then complete:

- An **incomplete type** is a structure that is declared but not complete:
  after a forward declaration (section 3.2) and before its definition, or
  inside its own body. A structure that is never defined stays incomplete.
- A **complete type** is a structure whose body has been processed and every
  type it holds inline is complete. Pointer fields do not need a complete
  pointee. Completion establishes the structure's fields, layout, member
  declarations, and value properties. Method bodies, including those in
  later `impl` blocks, are not part of it.

A structure's name cannot be used before its first declaration
(section 3.1).

```huc
struct Node {
    let mut next: mut Node*; // Allowed: the pointer's size is known.
    let value: i32;
}
```

A structure that holds itself inline never completes:

```huc
struct Node {
    let child: Node; // Error: Node would contain itself.
}
```

No declaration can make this valid: every `Node` would contain another whole
`Node`. An array such as `Node[2]` has the same problem. This is the existing
recursive-layout error.

### 2.1 What each stage allows

| Use of `Node` | Rule |
|---|---|
| `Node*`, `mut Node*`, pointer chains, pointer fields, type aliases, and function declarations taking or returning `Node` | Always allowed |
| A type query about `Node*` | Always allowed: the pointer type is complete |
| An inline `Node` field, or an array of `Node` | Allowed; the containing type completes only after `Node` |
| Naming a field or method of `Node` | Only after that member's declaration (section 3.1) |
| Runtime code that constructs, copies, relocates, or destroys a `Node`, or does arithmetic on `Node*` | Checked after preparation; an error if `Node` never becomes complete |
| Compile-time evaluation of those operations | Only once `Node` is complete |
| A type query about `Node` during preparation | Only once `Node` is complete |

Naming `Node` in a generic argument is not an error. Preparing the generic
body is an error if it needs facts about `Node` that are not yet known.

### 2.2 Queries need a complete type

The six type queries, `size_of`, `align_of`, `is_basic`, `is_relocatable`,
`is_copyable`, and `is_default_initializable`, need a complete operand when
they are evaluated during preparation. There is no provisional answer:

```huc
struct Node {
    let value: i32;

    if1 (is_copyable<Node>()) { // Error: Node is not complete yet.
        // ...
    }
}
```

This holds even if `Node` has already declared `copy`. Some answers may be
known early, but the rule makes no exceptions for them.

Runtime code is checked after preparation for facts that need a complete
type. This extends spec section 3.4's rule for queries left in runtime code.
So an inline method may construct, copy, or destroy its own type, as the
`copy` method of `SelfLinked` does in spec section 6.13. What fails is
preparation-time work in the method that needs facts about its incomplete
type:

- an `if1` or `let1` query about it;
- a request for a generic helper whose body runs such a query, such as a
  `swap<T>` that branches with `if1 (is_basic<T>())`.

Write such a method in an `impl` block instead (section 4).

### 2.3 Future reflection

Reflection will be available through `compiler::`, but its API is not
decided (spec section 10.2). A reflection query returns a snapshot, and
iterating over it is not affected by declarations made during the
iteration. For example, declaring fields based on a structure's field list
does not change the list being iterated. Fetching the list again returns the
current members. A snapshot of an incomplete type can describe only what is
known so far. Facts that need a complete type remain errors (section 2.2).

## 3. Declaration order

### 3.1 Declare before use

A name can be used only after its declaration (spec section 2.2). The
rule covers every name: structures, functions, globals, and members. To use
something defined later, declare it first (sections 3.2 and 3.3, and
section 4.1 for methods), or move its definition up.

```huc
fn report() -> i32 {
    return compute(); // Error: compute is declared later.
}

fn compute() -> i32 {
    return 42;
}
```

The error should suggest a forward declaration. Runtime and compile-time
code use the same declaration order.

- **Members.** A field, method, or static function can be named only after
  its declaration. An inline method sees the members declared above it.
  Code after the structure body, including its `impl` blocks, sees every
  member that the body declares. A method that only an `impl` defines is
  declared at its own header, so only code below it can name it
  (section 4.4). Inside a method, members are reached only through the
  receiver `self`, as in `self.value`. A bare name never finds a member
  (section 4.5).
- **Generic bodies.** Like an ordinary body, a generic body sees only the
  outside names declared above its definition, in its own lookup
  environment (compilation model, section 6.3). Its definition is the
  primary or specialization that contains it, or, for a method, the method
  itself. Its own parameters and locals become visible in order, as in any
  body. Code that a `#` inserts follows the same rules (spec section 11.2).
  So does module code that its own compile-time code places before the
  current item (section 3.4). The body is still prepared at its point of
  call (section 3.5), which supplies only:
  - the type and value arguments;
  - the choice among the overloads declared above, made with the
    arguments' real types;
  - the members of the type arguments and of types built from them, such
    as `value->hash()` or the static function `T::make()`;
  - the specialization selected for each family request, which may be
    declared below the definition but above the point of call (spec
    section 9.3);
  - compiler state, such as an earlier `let1 mut`, as it is then.
- **Local generic variables.** Their runtime declarations stay where the
  family is declared (section 3.5). So, besides their generic parameters
  and their own declarations, such as a `let1`, their bodies see only the
  names visible there. They are still prepared at their point of call, so
  they read compiler state, such as an earlier `let1 mut`, as it is then.
- **Deduction and `mut`.** Deduction may drop `mut` from a pointer
  argument where an ordinary call could (spec section 7.5). It never adds
  `mut`. So for `fn fingerprint<T: auto>(value: T*)`, a call with a
  `mut Robot*` deduces `T = Robot`, and `value` is read-only in the body.
  A generic that writes declares `value: mut T*`, and then a read-only pointer
  is rejected. A generic that must keep the caller's permission binds the
  whole pointer type, as in
  `fn first<P: auto>(items: P) -> P`. This applies only to value
  arguments. A captured expression in a compiler call still needs an exact
  match (spec section 9.7.1), so no conversion wraps it.
- **Type facts.** Facts that need a complete type, such as its layout or
  whether it has `drop`, are still checked after preparation (section 2.2).
- **Compile-time calls.** A compile-time call needs the callee's
  definition, not just a forward declaration (section 3.3).
- **HUC0.** Handwritten and printed HUC0 follow the same rule, as in C++.
  The printer adds the forward declarations that printed code needs
  (section 3.5).

For example, `describe` sees only the `g` declared above it, even when it is
called after a better one:

```huc
fn g(value: i64) -> i32 {
    return 64;
}

fn describe<T: auto>(value: T) -> i32 {
    return g(value); // Sees only g(i64), declared above.
}

fn g(value: i32) -> i32 {
    return 32;
}

fn main() -> i32 {
    return describe(5); // Returns 64, not 32.
}
```

Without the first `g`, the call in `describe` would be an error, even though
`g(i32)` is declared above `main`. The error should point to the later
declaration and suggest a forward declaration. Generic functions that call
each other need a forward declaration (section 3.3).

### 3.2 Structure forward declarations

A module-scope forward declaration lets code name a non-generic structure
before its definition. Until the structure completes, it is an incomplete
type:

```huc
struct Node;

struct Links {
    let mut first: mut Node*;
    let mut last: mut Node*;
}

struct Node {
    let value: i32;
    let links: Links;
}
```

`struct Node;` and the later definition denote one type. A forward
declaration has two uses:

- **Using a structure before its definition.** After `struct Node;`, code
  can use `Node` wherever an incomplete type is enough (section 2.1), for
  example in `mut Node*`, `size_of<Node*>()`, or a pointer-only generic
  such as `Links<Node>` (section 5.1). Structures that point to each other
  need this.
- **Opaque types.** A structure that is declared but never defined is an
  opaque type, as in C. It stays incomplete, so it is usable only behind
  pointers. Whether such types may appear in `extern "C"` signatures is left
  to the deferred design for opaque foreign handles (spec section 13).

A definition, if any, is in the same file as the forward declaration. A
forward declaration may be repeated. It takes no attributes, and all
declarations of one type have the same visibility. A structure family can be
forward-declared too, such as `struct Forest<T: auto>;` (section 3.3).

### 3.3 Function and family forward declarations

A body-less module-scope `fn` is a forward declaration. It lets code call a
function defined later, as mutual recursion needs. Its definition follows in
the same file, with the same signature:

```huc
fn compute() -> i32;

fn report() -> i32 {
    return compute(); // Valid: compute is declared above.
}

fn compute() -> i32 {
    return 42;
}
```

- A body-less `fn` with no definition in the file is an error.
  `extern "C"` declarations are unchanged.
- A forward declaration makes the signature available, not the body. A
  compile-time call still needs the definition first.
- A function may be forward-declared more than once, with the same
  signature. Function and parameter attributes go only on the definition.
  The exception is `callconv`, which is part of the signature (spec
  section 4.9.3), so every declaration repeats it. Callers above the
  definition need it, because a `gpu_kernel` function cannot be called
  directly (spec section 4.9.10). All declarations of one function have
  the same visibility.
- Methods are declared in their structure instead (section 4.1).

A generic family's primary can be forward-declared the same way: its header,
binders included, without a body. A generic body sees only the names
declared above it (section 3.1), so generic functions that call each other
need this:

```huc
fn odd<T: auto>(value: T) -> bool;

fn even<T: auto>(value: T) -> bool {
    if (value == 0) {
        return true;
    }
    return odd(value - 1); // Valid: odd is declared above.
}

fn odd<T: auto>(value: T) -> bool {
    if (value == 0) {
        return false;
    }
    return even(value - 1);
}
```

Without the forward declaration, `odd` would be an error in `even`'s body.
Moving `odd` up would not help, because `odd` calls `even`. A structure
family is forward-declared the same way, such as `struct Forest<T: auto>;`,
so that generic structures can point to each other.

- The definition follows in the same file. The first forward declaration
  carries the binder defaults. Later forward declarations and the
  definition repeat the header without them, as an `impl` header does
  (section 4.2). Attributes go on the definition too, apart from a
  function's `callconv`, which every declaration repeats.
- A family may be forward-declared more than once. All its declarations
  have the same visibility. A forward declaration with no definition in the
  file is an error.
- Only a primary is forward-declared. Specializations are selected at the
  point of call, so they need none, and they may follow the forward
  declaration.
- A request needs the definition that it selects, as a compile-time call
  does. Preparing `even<i32>` requests `odd<i32>`, so `odd`'s definition must
  come before that point of call. A call from a `main` below both is valid.
- Generic variable families have no forward declarations. Declare them
  above their uses.

### 3.4 Generated code

**Module code is generated directly.** `#` inserts only local code:
code from `compiler::local` and `compiler::around`, and attributes (spec
sections 4.9.6 and 11.2). Module code needs no `#`. The two module
operations are compiler actions. They add module code when they run,
and return nothing:

```huc
compiler::module(text)        // module code after the anchor
compiler::module_before(text) // module code before the anchor
```

So a `compiler::Code` value holds only local code. Module actions follow
these rules:

- **Program mode only.** A module action is allowed only in program mode,
  where `compiler::emitting()` is true. Calling one in evaluation mode is
  an error (spec sections 11.1 and 11.2). The error covers every compiler
  call: a builder runs in evaluation mode, so it cannot call a module
  action. It returns the text instead, and its caller passes the text to
  the action. A body that is also called at compile time guards its
  actions with `if1 (compiler::emitting())`, as spec section 11.1's `area`
  does.
- **Generation site.** So an action is called directly in compile-time
  code that runs in program mode, usually an `@` item or statement. Spec
  section 11.2 calls a `#`'s position its generation site, and the
  action's position is the same idea. The code goes where a `#` there
  would have put it, with the same anchor, file, and namespace.
- **Timing.** The code is processed when the compile-time code at its site
  finishes. `compiler::module` output then waits for its anchor, as
  before.

Module actions are the exception to the rule that `@` does not insert
code. A builder cannot call them: the action must occur in program-mode
preparation at an explicit generation site.

**What generated text may contain.** Generated text may contain any HUC1
code that would be valid written where it is inserted. It may contain generic
families, `specialize`, `impl` blocks, and compile-time constructs such as
`if1`, `let1`, `@`, and `#`. The compiler processes the text as it would
written source at that point:

- Local code is processed in place of its `#`, before the source after it.
  `compiler::module` output is processed when it is published after its
  module insertion anchor (spec section 11.2). `compiler::module_before`
  output is processed when the code that generated it finishes (below).
- Compile-time items in the text run then, in order. Code that the text
  generates belongs to the same file as the text (spec section 11.2).
- A family body in the text is prepared when it is selected, like a
  written one, and its compile-time code runs then.
- Generation that never ends, such as text that generates itself again, is
  stopped by the limits of spec section 9.11, as a `while1` that never
  ends is.

**Declaration order.** `compiler::module` output is published after its
module insertion anchor, so code above that point cannot name it. A
function that generates a helper and calls it is an error:

```huc
fn health_check_text<>() -> compiler::Text {
    return "fn health_check() -> i32 { return 200; }";
}

fn run() -> i32 {
    @compiler::module(health_check_text());
    return health_check(); // Error: health_check is published after run.
}
```

The compiler adds no declaration for generated code. Compile-time code
already cannot use pending module code, and runtime code now follows the
same rule. As for any later function, a forward declaration of
`health_check` above `run` fixes the error, because the generated
definition is in the same file. Three patterns avoid the error altogether:

- Place the code above the function with `compiler::module_before`
  (below).
- Generate the code at module scope, above the code that uses it. Module
  code generated by a module-scope item is published right after that
  item.
- Write an on-demand helper as a generic family. Its point of call names a
  family declared earlier, and its body can be generated with
  `compiler::local`.

**Module code before an item.** `compiler::module_before` places module
code right before the current item, instead of after it. With it, the
helper example above is valid:

```huc
fn run() -> i32 {
    @compiler::module_before(health_check_text());
    return health_check(); // Valid: health_check is placed above run.
}
```

- **Placement.** The code goes right before the module insertion anchor
  that `compiler::module` output would follow. Several pieces go in the
  order they were generated.
- **Printing.** Printed HUC0 puts the placed declarations before the item
  too, but every function body goes right after the item, as for a
  specialization (section 3.5). A placed structure keeps its fields before
  the item, and its method bodies go in an `impl` right after it. So a
  placed body may use what the item declared before the call, as in the
  example below.
- **Visibility.** The code is processed as soon as the compile-time code
  at its site finishes. It sees the module-scope names declared so far,
  including the current item and the members that the item declares above
  the site. Code processed after that can name what it declares: the rest
  of the item and the code below it. Code above the item cannot, and
  neither can the part of the item before the site. A generic body may
  name what its own compile-time code placed, although it is not above the
  body's definition.
- **No more module code.** Processing the placed code must generate no
  module code, either directly or through an instantiation it causes.
  Otherwise it is an error, reported where that code was generated, with
  the chain back to the placed code. Local code is fine, and so are
  compile-time calls, which never generate module code. So placed code may
  contain full structure and function definitions, as long as preparing
  them generates no module code. An instantiation runs only at its first
  request, so whether placed code causes one can depend on order.
- **Other rules.** Placed code follows the rules of other module code. The
  action is an error in evaluation mode, and in a finished module the
  placed names are late names (spec section 11.2, and late module output
  below).

For example, a method can place a helper that reads a field of its own
structure:

```huc
struct Item {
    let value: i32;

    fn read(&self) -> i32 {
        @compiler::module_before(
            "fn read_item(p: Item*) -> i32 { return p->value; }");
        return read_item(&self);
    }
}
```

HUC1 resolves `p->value`, because `value` is declared above the method.
Printed whole before `Item`, the helper would be invalid HUC0: a forward
declaration of `Item` declares no fields. So the printer splits it:

```huc
struct Item;

fn read_item(p: Item*) -> i32;

struct Item {
    let value: i32;

    fn read(&self) -> i32 {
        return read_item(&self);
    }
}

fn read_item(p: Item*) -> i32 {
    return p->value;
}
```

**Late module output.** A deferred body may be prepared after its module
finished, when another file first requests it. Its compile-time code may
still generate module code then, including specializations and `impl`
blocks (spec section 11.2). A late name can be used only by the code that
generated it. A late specialization or `impl`, by contrast, applies to all
code after its insertion point, and code before that point does not see
it.

A specialization that would change an earlier request is still an error
(spec section 9.3). So is a generic `impl` that comes after a point of call
it applies to (section 4.2). Which requests come first depends on which
file requests the body first, and so on the root file's import order. One
program may therefore compile in one import order and fail in another. With
a late `impl` that adds a better overload, both orders may compile but call
different methods.

Generating module code is like using macros: it needs great care, both in
designing a builder and in using one. This is most true in a deferred body
that other files may request.

### 3.5 Point of call and point of instantiation

HUC distinguishes where a specialization is requested from where its
definition is printed:

- The **point of call** is where code requests a specialization: a call
  such as `twice(x)`, or a type such as `Box<i32>`.
- The **point of instantiation** is where printed HUC0 places the
  specialization's definition.

A generic body is prepared at its point of call, as specified. Its point of
instantiation may see more declarations than its point of call did, so one
rule governs printing:

> Printed HUC0 keeps every call target that HUC1 resolved. Moving a
> definition never runs overload selection again against a different set
> of declarations.

HUC1 records each call's target when it resolves the call, under the
declaration order of section 3.1. It never resolves a name again at
printing time. The printer then makes each target explicit:

- **Overloads.** HUC0 has no function or method overloading, in printed
  and handwritten code alike. The printer gives each overload its own
  name, as it does for specializations, and each call names the overload
  that HUC1 selected. So every HUC0 call names exactly one function, and
  moving a definition cannot change which one. HUC0 still checks the
  arguments against that function, so their conversions follow from it.
- **Constructors.** Inits are the one exception. A structure may still
  have several inits in HUC0, such as its generated memberwise and default
  inits (spec section 4.6). A construction such as `Item(argument)` names
  no function, so there is nothing to rename. It is printed as it is, so
  construction directly into the destination and argument order do not
  change. Choosing the init in HUC0 is safe. A structure declares all its
  inits in its body (section 4.1), and init selection waits until the
  structure is complete, in HUC1 and HUC0 alike. So every position sees
  the same inits. The backend gives each init its own symbol, for example
  from its position in the structure. No other symbols need to tell apart
  functions with the same name.
- **Comments.** The printer adds a one-line comment before each printed
  declaration of a specialization or a renamed overload. This includes
  forward declarations and the `impl` blocks of structure specializations.
  The comment is the source declaration's header with its generic
  arguments filled in, such as `// fn twice<i32>(value: i32) -> i32` or
  `// struct Box<i32>`. It uses source type names and leaves out `pub` and
  attributes. The printer builds it from the resolved declaration, not
  from text. So generated declarations get one too, and diagnostics can use
  the same form. The comments have no meaning for HUC0.
- **Generated code.** The printer works from resolved code, not from text.
  Generated text is resolved when it is inserted, and its calls are printed
  like any other.

For example, `invoke<Item>` below is prepared inside `Item`, where only
`pick(i64)` is declared. So it calls `pick(i64)`, as an inline method at
that point would:

```huc
fn invoke<T: auto>(p: T*) -> i32 {
    return p->pick(1);
}

struct Item {
    fn pick(&self, value: i64) -> i32 { return 64; }

    fn run(&self) -> i32 {
        return invoke(&self); // Calls pick(i64): pick(i32) is declared later.
    }

    fn pick(&self, value: i32) -> i32 { return 32; }
}
```

Its definition is printed after `Item`, where `pick(i32)` is also visible
and spec section 7.5 would prefer it for the literal `1`. The printed call
names the selected overload, so the result stays 64:

```huc
struct Item;

// fn invoke<Item>(p: Item*) -> i32
fn __huc_invoke_item(p: Item*) -> i32;

struct Item {
    // fn pick(&self, value: i64) -> i32
    fn __huc_pick_i64(&self, value: i64) -> i32 { return 64; }

    fn run(&self) -> i32 {
        return __huc_invoke_item(&self);
    }

    // fn pick(&self, value: i32) -> i32
    fn __huc_pick_i32(&self, value: i32) -> i32 { return 32; }
}

// fn invoke<Item>(p: Item*) -> i32
fn __huc_invoke_item(p: Item*) -> i32 {
    return p->__huc_pick_i64(1);
}
```

Placement then only has to make each declaration available before its use.
Printed HUC0 places definitions as follows (spec sections 9.3 and 9.4):

- **Functions.** A function specialization goes right after the top-level
  declaration that contains the point of call, before the module code
  queued there. The printer declares it before that top-level declaration.
- **Structures.** The requesting code may need a structure's layout, so a
  structure specialization goes right before that top-level declaration.
  Its methods are declared there without bodies. Their bodies, including
  those from a generic `impl`, go in an `impl` right after the top-level
  declaration.
- **Variables.** A module-scope variable specialization goes right before
  that top-level declaration. A local one stays at its family's
  declaration, so its initialization point does not move (section 3.1).
- **Deferred method families.** A method specialization is declared in its
  structure, at the family's position. Its body goes in an `impl` right
  after the top-level declaration, like a function specialization. A
  family declared only in an `impl` has no position in the structure
  (section 4.3). Its requested specializations are then declared at the
  end of the printed structure, in the order of their first requests, and
  their bodies go after the requesting top-level declaration under these rules.
  The printer forward-declares any later type that their signatures name.
  Deferred static functions are printed the same way (section 4.5).
- **Nested requests.** A request made while preparing another
  specialization follows the same rules, using the top-level declaration
  that contains the original point of call. A structure that another
  structure's definition needs goes right before that definition.
- **Placed code.** Code from `compiler::module_before` is printed the same
  way: its declarations right before the top-level declaration, and its
  function and method bodies right after it (section 3.4).

Specialization bodies are printed after the top-level declaration that
contains their point of call, where every name they use is already
declared. What is printed before it uses only names declared before it,
plus later structures that the printer forward-declares, such as `Node` in
`Links<Node>` (section 5.1). A specialization is printed once, for the
first point of call that needs it in HUC0.

```huc
struct Box<T: auto> {
    let value: T;

    fn get(&self) -> T {
        return self.value;
    }
}

fn twice<T: auto>(value: T) -> T {
    return value + value;
}

fn main() -> i32 {
    let item: Box<i32> = Box<i32>(21);
    return twice(item.get());
}
```

Printed HUC0, with shortened names:

```huc
// fn twice<i32>(value: i32) -> i32
fn __huc_twice_i32(value: i32) -> i32;

// struct Box<i32>
struct __huc_box_i32 {
    let value: i32;

    fn get(&self) -> i32;
}

fn main() -> i32 {
    let item: __huc_box_i32 = __huc_box_i32(21);
    return __huc_twice_i32(item.get());
}

// struct Box<i32>
impl __huc_box_i32 {
    fn get(&self) -> i32 {
        return self.value;
    }
}

// fn twice<i32>(value: i32) -> i32
fn __huc_twice_i32(value: i32) -> i32 {
    return value + value;
}
```

## 4. Method bodies in `impl` blocks

```huc
struct Node {
    let value: i32;
}

impl Node {
    fn value_bytes(&self) -> usize {
        let1 bytes: usize = size_of<Node>();
        return bytes;
    }

    fn doubled(&self) -> i32 {
        return self.value * 2;
    }
}
```

An `impl` method behaves exactly like an inline one. Only its position
differs. An ordinary method is defined once, inline or in an `impl`, and the
structure may also declare it (section 4.1). `Node` is complete before
`value_bytes` is prepared, so its `let1` query is valid. The same query in
an inline method of `Node` would be an error. Inline methods remain allowed.

### 4.1 Method declarations in the structure

Together, `init`, `copy`, and `drop` decide a type's category, copy support,
generated inits, and default initialization. So the structure body must
declare them. Their bodies may be inline or in an `impl`:

```huc
extern "C" fn release_ticket(value: u32) -> void;

struct Ticket {
    let value: u32;

    fn drop(&mut self) -> void;
}

let1 can_copy: bool = is_copyable<Ticket>(); // false: Ticket declares drop

impl Ticket {
    fn drop(&mut self) -> void {
        release_ticket(self.value);
    }
}
```

Ordinary methods may be declared in the structure too. Under declaration
order (section 3.1), a method that calls a method defined below it needs
this, whether it is inline or in an `impl` (section 4.4). So does code
between the structure and its `impl`:

```huc
struct Parser {
    let mut depth: u32;

    fn parse_term(&mut self) -> i32; // Declared: parse_expression calls it.

    fn parse_expression(&mut self) -> i32 {
        return self.parse_term();
    }
}

impl Parser {
    fn parse_term(&mut self) -> i32 {
        if (self.depth == 0) {
            return 0;
        }
        self.depth -= 1;
        return self.parse_expression();
    }
}
```

- A structure `T` may declare any of its methods without a body:
  `fn init(&mut self, ...);`, `fn copy(&self) -> T;`,
  `fn drop(&mut self) -> void;`, or an ordinary method. Static functions
  may be declared the same way.
- A body-less declaration needs exactly one definition, in an `impl` in the
  same file, with the same signature. An `impl` may also define ordinary
  methods and static functions that the structure does not declare, but
  not lifecycle members.
- A method declared inside a type-level `if1` may be defined inside an
  `if1` in the `impl` (section 4.3). Each selected declaration still needs
  exactly one definition.
- Constructor entries and function attributes go on the definition, as
  for forward-declared functions (section 3.3). The declaration in the
  structure repeats only `callconv`.
- A deferred method family may be declared without a body too, as a
  generic function may be (section 3.3). Only its primary is declared this
  way. The definition repeats the header without binder defaults, and a
  request needs the definition that it selects.

The structure body alone therefore decides every rule that depends on
declaring `init`, `copy`, or `drop`. An `impl` only supplies bodies and adds
ordinary methods, which affect no type property, so the compiler never scans
ahead for `impl` blocks.

If constructor parameter forwarding is adopted later, a forwarding entry
would affect callers, so it would have to appear in the declaration.

### 4.2 Generic structures

An `impl` for a generic structure repeats the target's angle lists, so that
it names exactly one primary or specialization. It omits attributes and
binder defaults:

```huc
struct Cell<T: auto> {
    let mut element: T;
}

struct Cell<T: auto><T*> {
    let mut element: T*;
}

impl Cell<T: auto> {
    fn get(&self) -> T* {
        return &self.element;
    }
}

impl Cell<T: auto><T*> {
    fn get(&self) -> T* {
        return self.element;
    }
}
```

- The header must match an existing primary or specialization, with the
  same binders and pattern. It never declares a specialization.
- Its methods belong to every specialization selected from that
  declaration. Like a family body, the block stays unparsed until such a
  specialization is selected.
- An `impl` must come before every point of call for a specialization it
  applies to (section 3.5). Otherwise it is an error, like a late
  specialization (spec section 9.3).
- The block's methods are prepared with the specialization, under the same
  rules as its inline methods. Section 4.1's checks apply to each selected
  specialization.

### 4.3 Initial scope

- An `impl` is a module-scope item in the same file and namespace as its
  target's definition, after that definition. It takes no `pub`, since its
  methods are public like all members. For a structure specialization of another
  module's family, the `impl` takes `specialize` too (section 7).
- The target is a HUC structure named directly: a non-generic structure, or
  a generic primary or specialization (section 4.2). Foreign types and
  aliases are not targets.
- Several blocks may target one structure. Their methods share one member
  namespace with inline methods, under the normal overload and duplicate
  rules.
- A block contains method definitions: bodies for methods the structure
  declares, other ordinary methods, static functions, and deferred method
  families. It has no body-less declarations; those go in the structure
  (section 4.1).
- A block cannot add fields or attributes. Like a structure body, it may
  contain `if1`, `for1`, `while1`, `let1`, compiler assignments, standalone
  `@` items, and `#` (spec sections 9.6, 9.9, and 11.2). What they select or
  insert must be methods or static functions that the block could contain
  directly.
- For a generic target, these items run once for each prepared
  specialization, and each specialization gets its own `let1` values, as in
  a generic structure body.
- An `impl` may itself be selected by phase control at module scope, or
  generated as module code (spec section 9.6, and section 3.4). It then
  follows the rules of a written one. A generated `impl` belongs to the
  file of its generation site, so that site must be in its target's file.
  It goes into the namespace of that site, unless it takes `specialize`
  (section 7). Late module output may contain an `impl` too (section 3.4).

A `let1` in an `impl` block, outside its methods, lives until the block's
closing brace, so its lifetime is its scope. Only the block's compile-time
items can use it. A method in the block cannot name it, because the method
may be prepared after the brace: a deferred method at its first call, and
any method again at each compiler call (spec section 12.2).

```huc
struct Item {
    let value: i32;
}

impl Item {
    let1 answer: i32 = 42;

    if1 (answer > 40) { // Valid: runs inside the block.
        fn big(&self) -> bool { return true; }
    }

    @compiler::module_before("let1 placed_answer: i32 = 42;");

    fn read<>(&self) -> i32 {
        return answer; // Error: a method cannot name the block's let1.
    }

    fn read_placed<>(&self) -> i32 {
        return placed_answer; // Valid: a module-level let1 above the impl.
    }
}
```

Put a value that methods need in a module-level `let1`. Compile-time code
in the block can place one above the block with `compiler::module_before`,
as `read_placed` shows (section 3.4). A block's `let1` is destroyed at the
brace, so a pointer to it dangles after that (spec section 12.1).

Placing a `let1` works while the module is being processed. A generic
`impl` first prepared after its module finished is different. It runs in
a late private view of the module's saved state (spec section 9.8). A
`let1` that it places lives only in that private view, which is discarded
afterwards, and its name is a late name (section 3.4). A method prepared
later, such as a deferred method at its first call, starts again from the
saved state, which has no such `let1`. The fallback for missing objects
reads only that saved state too (spec section 9.8.1). So naming the
placed `let1` there is an error.

The same rule covers a `let1` in a structure body, and the header `let1`
of a `for1` at module scope (spec section 9.6). In general, a function can
name only a module-level `let1`, or one in its own body. So a deferred
function in a module-scope `for1` cannot name the loop's counter, because
it may be prepared after the loop ends. Compile-time code in the loop can
instead generate the function with the counter's value written into its
text, or generate a module-level `let1` above the function.

Design status: these capture and later-access restrictions are provisional,
pending a separate capture/lifetime design. Capturing an enclosing block's
`let1`, or later naming one created only in a discarded private view, remains
an error for now. This does not settle whether a future design may support it.

### 4.4 Preparation order

An `impl` block is processed in order, like a structure body or module
scope. Each item is handled when it is reached:

- An ordinary method is declared at its header, and its attributes are
  evaluated there. This is their one program-mode evaluation (spec
  section 4.9.7). For a non-generic target, its body is then prepared,
  before the next item.
- A deferred method is registered like any family (spec section 9.5) and
  prepared at its first point of call, as specified. Its attribute list stays
  unparsed until a specialization is prepared, because the list may use
  the method's own binders (spec section 4.9.5).
- Compile-time items run in source order (section 4.3).

So a method sees the members that the structure declares and the methods
defined above it. To call a method defined below, declare it in the
structure (section 4.1). A generic `impl` is processed the same way when its
specialization is prepared (section 4.2).

A compiler call to an `impl` method evaluates its attributes again, as for
any function (spec section 4.9.7). Compile-time code in the block can run
the methods defined above it.

Code before an `impl` can call only the methods that the structure declares
(section 4.1). Compile-time execution needs the definition to be available,
and the call then prepares the body as usual. This includes implicit calls:
constructing, copying, or destroying a value at compile time runs its
lifecycle bodies.

The `impl` block is the insertion anchor for module code generated by its
ordinary methods and by compile-time code in the block. `compiler::module`
output is published after the whole block, and `compiler::module_before`
output right before it. Local output stays where its `#` is, and deferred
requests keep their existing anchor.

HUC0 supports `impl` blocks, structure and function forward declarations, and
body-less method declarations. HUC1 prints each block at its position,
which for a generated block is where its generation put it. Each block has
the methods that its compile-time items selected or inserted. HUC1 prints
specializations at their points of instantiation (section 3.5), and checks
the same-file rule before flattening.

### 4.5 The receiver `self` and static functions

A method declares an explicit receiver. It accesses its receiver
through `self`; a bare name never implicitly names a member. A member
function without a receiver is a static function.

```huc
let limit: i32 = 10;

struct Counter {
    let limit: i32;
    let mut hits: i32;

    fn over(&self, n: i32) -> bool { // Reads the object.
        return n > self.limit; // The field.
    }

    fn over_global(&self, n: i32) -> bool {
        return n > limit; // The module-level limit, not the field.
    }

    fn record(&mut self) -> void { // Changes the object.
        self.hits += 1;
    }

    fn finish(self) -> i32 { // Takes the object.
        return self.hits;
    }

    fn make(limit: i32) -> Counter { // No receiver: a static function.
        return Counter(limit, 0);
    }
}

fn main() -> i32 {
    let mut counter: Counter = Counter::make(10);
    counter.record();
    return counter.finish(); // Counter is Basic, so this copies it.
}
```

| Receiver | The method | Like the parameter (spec section 6.9) |
|---|---|---|
| `&self` | reads the object | `value: T*` |
| `&mut self` | changes the object | `value: mut T*` |
| `self` | takes the object | `value: T` |
| `mut self` | takes the object and may change it | `mut value: T` |

- **Receiver.** A method declares its receiver as its first parameter, in
  one of these forms. The receiver takes no type: it is always the
  structure that the method belongs to. `&self` and `&mut self` replace
  implicit receivers and trailing `mut`, so `fn drop() mut -> void` becomes
  `fn drop(&mut self) -> void`. Calls do not change: `counter.over(7)` and
  `pointer->record()` pass the receiver under these rules.
- **What `self` means.** In every form, `self` names the object, so
  `self.limit` is a field and `self.get()` calls a method. `&self` is its
  address: `mut T*` with `&mut self` or `mut self`, and `T*` otherwise. `copy self` copies it. In
  Rust, `self` in an `&self` method is a reference instead, but HUC has no
  references (spec section 4.3).
- **Borrowed receivers.** With `&self` or `&mut self`, `self` is the
  caller's object, with the existing unchecked pointer-extraction rules:
  - Binding `self` copies or moves nothing, and returning destroys
    nothing.
  - `self` is not a tracked local (spec section 6.12). Moving a field out
    of `&mut self`, as in `let name: Text = self.name;`, is pointer
    extraction (spec section 6.4). The method must refill the slot before
    the caller uses the object.
  - `drop self;` and `forget self;` are errors (spec section 7.6).
  - A method executed in the compiler still needs an actual receiver
    object, never an expression binding (spec section 9.7.1).
  - Read-only covers the object's own fields, not what a pointer field
    points to. With a field `target: mut i32*`, an `&self` method may
    write `*self.target = 5;`, but not `self.target = null;`.
- **Owned receivers.** With `self` or `mut self`, the method takes the
  object, as a by-value parameter does (spec section 6.9). The call moves
  an Advanced object in, so the caller's local must be `mut` (spec
  section 6.1), and the caller cannot use it afterwards (spec
  section 6.12). A Basic object is copied. Inside the method, `self` is an
  owned parameter, tracked like any by-value parameter. Scope-exit cleanup
  runs for both forms unless ownership was transferred. Explicit `drop self;`,
  `forget self;`, and relocating an Advanced `self` onward require `mut self`;
  fixed `self` cannot be consumed again. `mut self` also permits mutation.
  The receiver is bound before ordinary arguments. Thus moving a named
  Advanced `x` into `x.consume(x.field)` makes the later argument a checked
  use of an inactive root. A direct-result receiver constructs in the owned
  parameter's final slot, including for `[norelocate]`; an existing
  non-relocatable object cannot transfer into it.
  In `x = x.transform()`, an owned receiver transfers `x` during assignment
  preparation, so no old value remains to destroy (spec section 6.3).
- **Pointer and subobject calls.** Both `p->finish()` and
  `(*p).finish()` may bind an owned receiver. Basic objects copy. Advanced
  objects require writable storage (`mut T*` for a pointer), relocate into
  the parameter, and leave the source inactive. This is unchecked extraction,
  not a tracked transfer of the caller's root. The caller must restore the
  slot or prevent its cleanup; field and element receivers follow the same
  rules (spec sections 6.4 and 6.12). No runtime check is added. All compiler
  method calls, borrowed or owned, require a real receiver object, not an
  expression binding.
- **No `this`.** `this` and `this->` are removed, and `&self` replaces
  `this`, as in `read_item(&self)`. `this` stays reserved: using it is an
  error that suggests `self` or `&self`, and the reserved `@this` of spec
  section 10.3 keeps its meaning. `self` is a reserved word too.
- **Members only through `self`.** A bare name never finds a member, so
  in `over_global`, `limit` is the module-level one.
- **Static functions.** A function in a structure or `impl` without a
  receiver is a static function. It has no object, so it cannot name
  `self`. It is called through its type: `Counter::make(10)`,
  `geo::Point::origin()`, `Box<i32>::make()`, or `T::make()` in generic
  code. When the part before `::` is a type, the name after it is one of
  that type's static functions. A static function cannot be called
  through an object, and a method cannot be called through its type.
  Otherwise, static functions follow every method rule in this document:
  declaration, preparation, specialization, attributes, generation
  anchors, and printing. `if1` and `#` in an `impl` may produce them too.
- **One kind per name.** In one structure, a name is either methods or
  static functions, never both. So overloads of one name are all methods
  or all static functions.
- **No receiver-only overloads.** Two declarations with the same name and
  ordinary parameter types and arity cannot differ only in receiver form.
  `self` versus `mut self` is binding mutability, not a distinct type. For
  overloads with different ordinary parameters, use the ordinary ranking
  rules, then check the selected receiver's permissions and transfer rules.
  Ownership, liveness, and receiver mutability are not tie-breakers or a
  reason to retry a different overload. HUC0 has no ordinary overloads.
- **Lifecycle members.** `init`, `copy`, and `drop` take borrowed
  receivers: `fn init(&mut self, ...)`, `fn copy(&self) -> T`, and
  `fn drop(&mut self) -> void`. None of them can be static or take an
  owned receiver.
- **HUC0.** HUC0 has the same rules, since both stages share syntax.

## 5. Recursive containers

### 5.1 Pointer-only containers work

```huc
struct Links<T: auto> {
    let mut first: mut T*;
    let mut last: mut T*;
}

struct Node {
    let links: Links<Node>; // Valid: Links only stores pointers.
}
```

An inline container that holds `T[N]` instead needs a complete element type,
so it cannot hold its incomplete owner.

### 5.2 Where the strict rule fails

A generic structure's whole selected body, including its inline and `impl`
methods, is prepared at its point of call. Any preparation-time query there
about its element type needs that type complete:

- the `Vector<T>` in section 12 of the
  [value semantics guide](value-semantics.md) chooses its `copy` and
  `resize` members with `if1 (is_copyable<T>())` and
  `if1 (is_default_initializable<T>())`;
- a method body may branch with `if1 (is_basic<T>())`, the idiom of spec
  section 6.7.4, or call a helper that queries, such as `relocate_one` in
  spec section 6.13.

So `Vector<Node>` as a field of `Node` is an error, even though the vector's
pointer-and-count layout would fit. Moving either type's methods into an
`impl`, or adding a forward declaration, does not help: the queries are in
the vector's selected body.

Order matters as well, even without recursion. If
`struct A { let items: Vector<B>; }` comes before `struct B`, it is an error,
even with a forward declaration of `B`, because `B` is not complete when
`Vector<B>` is prepared. The error should suggest moving `B` above `A`.

### 5.3 Unconditional operations break the cycle

A concrete container can declare its operations unconditionally, so that no
query about its incomplete element type is needed:

```huc
import std.memory as memory;

struct Node;

struct NodeList {
    let mut data: mut Node*;
    let mut length: usize;
    let mut capacity: usize;

    fn copy(&self) -> NodeList;
    fn drop(&mut self) -> void;
}

struct Node {
    let value: i32;
    let mut children: NodeList;
}

// Assumes both memory helpers accept a zero count.
impl NodeList {
    fn copy(&self) -> NodeList {
        let mut result: NodeList = NodeList(
            memory::allocate_raw_items<Node>(self.length), 0, self.length);
        while (result.length < self.length) {
            construct_at(result.data + result.length,
                copy *(self.data + result.length));
            result.length += 1;
        }
        return result;
    }

    fn drop(&mut self) -> void {
        while (self.length > 0) {
            self.length -= 1;
            destruct_at(self.data + self.length);
        }
        memory::free_raw_items<Node>(self.data, self.capacity);
    }
}
```

1. `NodeList` declares `copy` and `drop`, so it is Advanced and supports
   copying. It holds only a pointer and integers, so it completes without
   `Node`.
2. `Node` completes. Its generated `copy` uses `NodeList`'s declared `copy`.
3. The `impl NodeList` bodies are prepared after `Node` is complete.

No query is answered by assuming an incomplete type is copyable. The bodies
request `memory::allocate_raw_items<Node>` and `memory::free_raw_items<Node>`.
If those helpers query `Node` while they are prepared, as a generic allocator
may, the bodies must come after `Node` is complete. The `impl` puts them
there.

### 5.4 Reporting a cycle

A query about an incomplete type is an error even when the types depend on
each other in a cycle. No answer is assumed. The error shows the query and
the chain that reached it, innermost first:

```text
error: cannot evaluate is_copyable<Node>() while Node is incomplete
  note: this condition decides whether Vector<Node> declares copy
  while preparing Vector<Node>
  requested by the children field of Node
```

## 6. At most 512 active compile-time calls

Compile-time execution allows at most 512 active HUC function calls. The
first call in an evaluation has depth 1, and entering depth 513 is an error.
This limit is fixed, not configurable (spec section 9.11). The instruction,
allocation, specialization-depth, and generated-byte limits stay as they are.

Every active call counts, recursive or not:

- ordinary and deferred functions, methods, and code-builder functions;
- written `init`, `copy`, and `drop` bodies, and generated `init` and `copy`
  methods, including implicit calls;
- HUC code that an intrinsic runs, such as a `drop` run by `destruct_at`.

Automatic field and element cleanup adds no frame of its own, and neither
does an intrinsic itself. `@` and `#` neither add a frame nor reset the
depth. A call's frame starts before its evaluation-mode body is prepared and
lasts through execution and cleanup, so compiler calls made while preparing
or cleaning up are nested. Calls that compute arguments run in the caller,
before the callee's frame starts. Preparing a runtime definition is not a
call, but compiler calls it makes count. Optimizations never change the
count.

```huc
fn descend<>(remaining: usize) -> usize {
    if (remaining == 0) {
        return 0;
    }
    return descend(remaining - 1);
}

let1 allowed: usize = @descend(511); // 512 active calls at the deepest point.
let1 rejected: usize = @descend(512); // Attempts call 513: error.
```

A runtime call to `descend(1000)` is not limited. Recursion over
compile-time data counts the same way: destroying a compile-time linked list
whose `drop` destroys the next node uses one frame per node, and copying the
section 5.3 tree uses two per level.

The error names the limit and the attempted call. It then lists the active
frames from the innermost out, each with its function or specialization and
call site, and ends with the evaluation or insertion that started the chain.
The default diagnostic shows the 5 innermost and 5 outermost frames and
replaces the rest with one line saying how many it skipped. An
implementation option may show every frame. Type-completion and
specialization cycles are not calls. Their errors show the dependency chain
instead (section 5.4).

## 7. Specializing another module's family

A file specializes another module's family with a `specialize` prefix on
one declaration:

```huc
module app.point;

import library.hash;

pub struct Point {
    let x: i32;
}

// Allowed: this file declares Point.
specialize library.hash
fn hash<><app::point::Point>(value: app::point::Point*) -> u64 {
    return as<u64>(value->x);
}

pub fn x_of(point: Point*) -> i32 {
    return point->x;
}
```

`specialize library.hash` puts only the next declaration in that module's
namespace, and the code after it is in `app.point` again. The keyword also
states the rule: another module accepts only specializations.

The declaration after the prefix is one specialization of a module-scope
`fn`, `struct`, or `let` family. Each specialization takes its own prefix,
and attributes stay after the declaration keyword, as in `fn [inline]`. An
`impl` for a structure specialization written this way takes the prefix
too, since an `impl` must be in its target's namespace (section 4.3):

```huc
specialize library.box
struct Box<><app::point::Point> { ... }

specialize library.box
impl Box<><app::point::Point> { ... }
```

- **Module name.** It is written as in a `module` line: the canonical name,
  not an import alias. It must name a visible module other than the file's
  own (spec section 2.1).
- **Names inside.** Unqualified names resolve in the family's namespace, where
  other files' private declarations stay hidden (spec section 9.3;
  compilation model section 6.4). The file's own imports, aliases, and private
  declarations still apply. So the example names its own type
  `app::point::Point`.
- **Unchanged rules.** Only an owning file may write the specialization, the
  family must be `pub`, and the end-of-file clash check and the
  late-specialization error apply under these rules (spec section 9.3; compilation
  model section 9.5).
- **Code generated inside.** Local code from a `#` in such a
  specialization is allowed. Module code generated there would go into the
  family's namespace, so it is still an error (spec section 11.2). Put such
  helpers in the file's own namespace and call them with a qualified name,
  as compilation model section 6.4 calls `app::point::mix`.
- **Generated specializations.** Generated module code may contain
  `specialize` declarations, which go into the named module's namespace as
  written ones do. Such a declaration belongs to the file of its
  generation site (section 3.4), so that file must be an owning file. Late
  module output may contain them too.
- **`module` lines.** Outside flattened HUC0, a `module` line names only the
  file's own module. `specialize` is the only way to add to another module.
- **HUC0.** HUC0 has no families, so it has no `specialize`. A printed
  specialization still keeps its family's namespace (spec section 9.3), so
  printed HUC0 opens a section of that namespace for it. A flattened file
  skips the ownership checks (spec section 2.1).

For example, these lines in `app.point`, below `Point`, generate the `hash`
specialization from the start of this section. The file also needs
`import1 compiler;`.

```huc
fn hash_text<>() -> compiler::Text {
    let point: compiler::Text = "app::point::Point";
    return "specialize library.hash fn hash<><" + point + ">(value: "
        + point + "*) -> u64 { return as<u64>(value->x); }";
}

@compiler::module(hash_text()); // Allowed: this file owns Point.
```

