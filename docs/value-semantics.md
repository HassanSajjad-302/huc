# HUC Value Semantics and Special Operations

Status: current design decision for HUC0

This document defines HUC's construction, copying, destructive relocation,
assignment, and destruction model. It also compares those operations with
their C++20 counterparts.

The comparison is about observable value and resource behavior. HUC is not
source-compatible with C++, and it deliberately exposes fewer customization
points.

## 1. The three basic storage relationships

HUC uses three related type forms:

| HUC type | Meaning | Copy behavior |
|---|---|---|
| `T` | An inline `T` value | Determined by `T` |
| `T*` | A nullable, unchecked, non-owning pointer | Copies one address |
| `T&` | A nullable, unique owner of one allocated `T` | Relocates by default; `copy` duplicates the pointee |

`T&` is not a C++ reference. HUC has no general aliasing-reference type and no
`T&&` type. A non-null `T&` owns an allocation, automatically destroys its
pointee, and then deallocates that storage. Relocating it clears the source
owner to null.

HUC has no unary `&` expression. Address-taking uses the core,
non-overloadable `addressof` intrinsic:

```huc
let Widget value = Widget(7);
let Widget* observer = addressof(value);
let Widget& owner = new Widget(9);
```

The pointer-like forms do not compose:
`T**`, `T*&`, `T&*`, and `T&&` are not HUC0 types, including through aliases.
`addressof` accepts only an inline non-pointer place: it returns `T*` for a
fixed place and `mod T*` for a writable place. Applying it to a raw-pointer or
owner slot is a diagnostic. Using `owner` in a `T*` context observes the
pointee; it does not alias the owner word. Direct owner-slot reseating uses
consume-and-return. Binary `&` remains bitwise AND.

Mutability is independent at each layer:

```huc
let T* observer;          // fixed raw field; constructor obligation
let mod T* observer;      // fixed field; writable T; constructor obligation
let T* mod observer;      // reseatable raw field; defaults to null
let mod T* mod observer;  // reseatable field; writable T; defaults to null

let T& owner;             // fixed owner field; constructor obligation
let mod T& owner;         // fixed field; writable T; constructor obligation
let T& mod owner;         // movable/reseatable owner field; defaults to null
let mod T& mod owner;     // reseatable field; writable T; defaults to null
```

These declarations are field forms. A fixed local requires an explicit
initializer. Every constructor must initialize each fixed field that lacks a
declaration initializer; mutable pointer and owner fields omitted by a
constructor default to null. HUC0 globals are separately restricted to
drop-free Copy scalars and raw pointers, so owners are never global in 0.1.

A fixed containing structure prevents assignment to its field slots and
writable access to inline field subobjects. It does not remove an explicit
leading `mod` carried through a pointer or owner field to a separate pointee.
For example, a read-only method may mutate the pointee of a `mod T*` field but
may not reseat that field. This preserves the layer distinctions above.

A trailing `mod` is required when source code will relocate from, reset,
release, or reseat a **named** pointer, owner, or other Move slot.
Compiler-inserted destruction at the end of the slot's lifetime does not
require source-level `mod`.

A fresh unnamed result place is intrinsically consumable. Function results,
`new T(...)` owner results, evaluated `copy` results, and other temporary Move
values may relocate directly into their next binding without a source-level
`mod` that could not be written:

```huc
let File file = open_file("input.dat");
consume(new Widget(7));
```

Only relocation from named storage tests its trailing `mod`. Direct
constructor initialization, described in section 4, creates no intermediate
source place at all.

## 2. Copy and Move types

Every complete runtime type is classified as either **Copy** or **Move**.

Copy types are:

- scalar arithmetic and boolean types;
- raw pointers;
- arrays whose element type is Copy;
- structures whose fields are all Copy and which declare neither `clone` nor
  `drop`.

Move types are:

- unique owners;
- arrays whose element type is Move;
- structures containing at least one Move field;
- structures declaring `clone`;
- structures declaring `drop`.

Declaring `clone` intentionally makes an otherwise fieldwise-Copy structure
Move. Otherwise ordinary binding could silently bypass the custom copy
behavior:

```huc
struct Ticket {
    let u64 id;

    fn init(u64 id) : id(id) {
    }

    fn clone() -> Ticket {
        return Ticket(probable_issue_related_ticket(this->id));
    }
}

let Ticket mod original = Ticket(100);
let Ticket moved = original;       // destructive relocation; clone is not called
let Ticket copied = copy moved;    // calls Ticket.clone
```

There is no `copy struct` or `move struct` declaration. Classification follows
the structural rules above.

An intentionally non-copyable scalar-only token may use an empty `drop` to opt
into Move semantics in HUC0:

```huc
struct UniqueTicket {
    let u64 value;

    fn init(u64 value) : value(value) {
    }

    fn drop() mod -> void {
        // No external cleanup; this lifecycle declaration makes the type Move.
    }
}
```

With no `clone`, `UniqueTicket` can be relocated but cannot be copied. A future
explicit non-copyable marker may replace this idiom, but the first compiler
does not need another type-category keyword.

## 3. The special-operation surface

HUC exposes only three lifecycle declarations:

```huc
fn init(...) : field(initializer), ... {
    // construction body
}

fn clone() -> T {
    // return an independent logical duplicate
}

fn drop() mod -> void {
    // release non-field resources
}
```

Their roles are:

| Operation | User customization | Invocation |
|---|---|---|
| Construction | `fn init(...)` | `T(...)` or `new T(...)` |
| Fieldwise copy of a Copy type | None | Ordinary binding or `copy` |
| Logical copy of a Move structure | `fn clone() -> T` | Only requested by `copy` |
| Logical copy of a Move array | None | `copy` applies logical copy to each element |
| Destructive relocation initialization | None | Ordinary binding of a Move value |
| Destructive relocation assignment | None | Ordinary assignment of a Move value |
| Destruction | `fn drop() mod -> void` | Inserted by the compiler |
| Field destruction | None | Inserted after `drop`, in reverse field order |

This is deliberately smaller than C++'s special-member system. In particular,
there is no user-defined move constructor, relocation constructor, or
move-assignment operator.

## 4. Construction

A constructor is a runtime method named `init`:

```huc
struct Point {
    let i32 x;
    let i32 y;

    fn init(i32 x, i32 y)
        : x(x),
          y(y) {
    }
}

let Point origin = Point(0, 0);
let Point& heap_point = new Point(10, 20);
```

These are direct-construction contexts. `Point(0, 0)` constructs `origin` in
its own storage, and `new Point(10, 20)` constructs in the final allocation.
No temporary `Point` is created and no relocation occurs. The same guarantee
applies when a constructor expression directly initializes:

- a field through a constructor initializer entry;
- a by-value function parameter;
- a function result in `return T(arguments)`.

This is a semantic guarantee, not optional backend copy elision. It lets `init`
observe the final `this` address and lets a fixed Move value be constructed
without first requiring a writable source temporary. More complex expressions
that produce an existing Move value still follow the normal destructive
relocation rules.

Constructor initializer entries:

1. must follow field declaration order;
2. are evaluated left-to-right;
3. initialize each field before the constructor body executes.

Every fixed field requires a declaration initializer or a constructor entry.
An omitted mutable scalar becomes zero, an omitted mutable pointer or owner
becomes null, and an omitted mutable structure is default-constructed.

`init` does not double as a copy or relocation hook. An overload accepting
another `T` is just an explicitly selected constructor:

```huc
struct ScaledPoint {
    let i32 x;
    let i32 y;

    fn init(Point source, i32 scale)
        : x(source.x * scale),
          y(source.y * scale) {
    }
}
```

## 5. Ordinary Copy value semantics

A structure made entirely from Copy fields, with no `clone` and no `drop`, is
copied field by field:

```huc
struct Point {
    let i32 x;
    let i32 y;

    fn init(i32 x, i32 y)
        : x(x),
          y(y) {
    }
}

fn translate(Point point, i32 dx, i32 dy) -> Point {
    return Point(point.x + dx, point.y + dy);
}

fn copy_values() -> void {
    let Point first = Point(10, 20);
    let Point second = first;                 // implicit fieldwise copy
    let Point third = copy first;             // same fieldwise copy
    let Point fourth = translate(first, 3, 4); // first copied into parameter
}
```

Structure fields are copied in declaration order. This is semantic field
copying, not a promise that padding bytes are duplicated. Because a Copy type
cannot contain a lifecycle hook or Move field, ordinary copying does not invoke
user copy code.

The direct C++20 value-semantic equivalent is:

```cpp
struct Point {
    std::int32_t x;
    std::int32_t y;
};

Point translate(Point point, std::int32_t dx, std::int32_t dy) {
    return Point{point.x + dx, point.y + dy};
}

void copy_values() {
    const Point first{10, 20};
    const Point second = first;
    const Point third = first;
    const Point fourth = translate(first, 3, 4);
}
```

The C++ example uses `const` because it is C++ source. HUC source has no
`const` keyword; HUC declarations are fixed unless a relevant layer has
`mod`.

Writable Copy assignment is also direct:

```huc
let Point mod current = Point(1, 2);
let Point replacement = Point(8, 9);

current = replacement;       // fieldwise Copy assignment
current = copy replacement;  // equivalent for a Copy type
```

```cpp
Point current{1, 2};
const Point replacement{8, 9};

current = replacement;
current = replacement;
```

## 6. Custom logical copying with `clone`

A Move structure is not implicitly copied. It may support explicit logical
copying by declaring exactly one method with this signature:

```huc
fn clone() -> T;
```

Rules:

- it is an ordinary runtime `fn`, not `fn1` or `fn2`;
- it has no explicit parameters;
- its receiver is read-only because it has no trailing `mod`;
- its return type is the enclosing nominal type;
- it must not modify the source object's own storage or invalidate its
  invariants;
- it cannot be called directly or have its address taken;
- ordinary binding, assignment, argument passing, and return never call it;
- `copy source` calls it when `source` is a Move structure;
- declaring it makes the structure Move even when every field is Copy;
- omit `clone` when the type must not be copyable;
- it is infallible in the HUC type system, although allocation failure or panic
  may terminate the process.

“Independent logical duplicate” is the precise meaning of deep cloning in
HUC. A `clone` must duplicate owned resources, or establish another valid
independent ownership share such as a library reference count, so the two
active values can later be destroyed independently. It does **not** recursively
clone every reachable object: a `T*` is an observer, so cloning a raw-pointer
field normally copies only that address. Shared external state, interned data,
immutable tables, and other observed objects may deliberately remain shared.
The type author defines this logical boundary and writes the necessary `copy`
operations inside `clone`; the compiler cannot prove that a custom clone
honored the contract. Allocation and external bookkeeping are permitted. For
example, the clone protocol selected by `copy rc` for a probable HUC1
`Rc<T>`
may increment a shared control block through an explicitly writable observer
while leaving the source handle's own storage and resource identity unchanged.

A type needing recoverable duplication should expose a normally named function
such as `try_copy` returning a probable HUC1 `Result<T, E>` specialization.
That function is not selected by the `copy` expression.

The initial language has one logical copy operation rather than separate
custom copy-construction and copy-assignment hooks. Copy assignment is composed
from `clone`, destruction of the old destination, and destructive relocation:

```huc
let Text mod destination = Text("old");
let Text source = Text("new");

destination = copy source;
```

Conceptually:

```text
temporary = Clone(source)
destroy(destination)
relocate(destination, temporary)
```

Here `Clone` and `relocate` name abstract compiler operations, not callable HUC
functions. The temporary is constructed before the old destination is
destroyed. Thus `value = copy value` is well-defined.

### 6.1 Owning text example

The memory APIs below are probable standard-library APIs:

```huc
module examples.heap_text;

import std.memory as memory;
import std.text as text;

struct HeapText {
    let mod u8* mod data;
    let usize size;

    fn init(text::View source)
        : data(memory::allocate_raw_bytes(source.size())),
          size(source.size()) {
        memory::copy_bytes(this->data, source.data(), this->size);
    }

    fn view() -> text::View {
        return text::View(this->data, this->size);
    }

    fn clone() -> HeapText {
        return HeapText(this->view());
    }

    fn drop() mod -> void {
        if (this->data) {
            memory::free_raw_bytes(this->data, this->size);
            this->data = null;
        }
    }
}

fn demonstrate() -> void {
    let HeapText mod first = HeapText("alpha");
    let HeapText second = copy first; // independent allocation
    let HeapText third = first;       // relocate; first becomes inactive
}
```

A conventional C++20 counterpart is:

```cpp
class HeapText {
public:
    explicit HeapText(std::string_view source)
        : data_(new std::uint8_t[source.size()]),
          size_(source.size()) {
        std::memcpy(data_, source.data(), size_);
    }

    HeapText(const HeapText& source)
        : HeapText(source.view()) {}

    HeapText& operator=(const HeapText& source) {
        if (this != &source) {
            HeapText temporary(source);
            *this = std::move(temporary);
        }
        return *this;
    }

    HeapText(HeapText&& source) noexcept
        : data_(std::exchange(source.data_, nullptr)),
          size_(source.size_) {}

    HeapText& operator=(HeapText&& source) noexcept {
        if (this != &source) {
            delete[] data_;
            data_ = std::exchange(source.data_, nullptr);
            size_ = source.size_;
        }
        return *this;
    }

    ~HeapText() {
        delete[] data_;
    }

    std::string_view view() const {
        return {
            reinterpret_cast<const char*>(data_),
            size_
        };
    }

private:
    std::uint8_t* data_;
    std::size_t size_;
};
```

The user writes five C++ special members here. The HUC user writes `clone` and
`drop`; HUC supplies fixed destructive relocation. This comparison does not
imply that the emitted C++ will have exactly this spelling.

### 6.2 Unique-owner copying

`copy` on an owner performs an explicit deep copy of the owned object:

```huc
let HeapText& mod first = new HeapText("hello");
let HeapText& second = copy first;
```

If `first` is null, `second` is null. Otherwise the operation:

1. evaluates `first`;
2. allocates uninitialized storage for one `HeapText`;
3. logically copies `*first` into that storage, using fieldwise Copy or
   `clone`;
4. returns a new independent owner.

The pointee type must support logical copying at compile time even when the
particular source owner happens to be null.

Copying a raw `HeapText*` never copies the pointee:

```huc
let HeapText* observer = first;
let HeapText* another_observer = copy observer; // copies only the address
```

### 6.3 Specialized copy assignment

HUC's automatic `destination = copy source` always uses clone-then-replace.
When an application needs buffer reuse or another specialized algorithm, it
uses a clearly named writable method:

```huc
struct ReusableBuffer {
    // fields, init, clone, and drop omitted

    fn assign_from(ReusableBuffer* source) mod -> void {
        probable_resize_in_place(this, source->size());
        probable_copy_payload(this, source);
    }
}

let ReusableBuffer mod buffer = ReusableBuffer(...);
let ReusableBuffer other = ReusableBuffer(...);
buffer.assign_from(addressof(other));
```

`assign_from` is not an implicit copy-assignment hook. The call site explicitly
selects the specialized operation.

## 7. Move-category transfer is destructive relocation

`Move` is the type category; **destructive relocation** is the transfer
operation. In ordinary conversation it is reasonable to say that a HUC value
“moves,” but its lifetime behavior is intentionally different from C++ move
construction. “Destructive move” is an acceptable informal name; the
specification uses “relocation” when the lifetime distinction matters.

```huc
let Packet mod source = Packet(...);
let Packet destination = source; // destructive relocation
```

Before this binding, `source` contains an active `Packet` and `destination`
does not. Relocation continues the value in `destination` and ends the active
`Packet` lifetime in `source`. It does not create a second live object and it
does not leave a moved-from `Packet`.

Structural relocation is fixed:

1. Visit each field in declaration order.
2. Copy it when its type is Copy; recursively relocate it when its type is
   Move.
3. After the final field, make the aggregate source place inactive.

Arrays use the same rule in increasing index order. No source `drop` body or
source-field cleanup runs: those resources now belong to the destination. The
operation invokes no user code and cannot fail in the HUC type system.

Field fixedness controls ordinary field assignment; it does not block the
compiler from relocating that field as part of its containing value. The
containing source place must be writable. For example, a fixed `T&` field is
still transferred when its writable enclosing structure is relocated.

The bytes of an inactive source need not be cleared and need not form a valid
representation of `T`. The compiler tracks whether the *storage place* is
active. Assigning a new value to an inactive place begins a new lifetime there.
Using it as a value before reinitialization is undefined behavior.

Primitive `T&` is the deliberate exception to the inactive-source rule.
Relocating an owner transfers its address and writes null to the source. The
source is then an active, usable null owner:

```huc
let Widget& mod first = new Widget(7);
let Widget& second = first;

if (!first) {
    // defined: first is a null owner, not inactive Widget& storage
}
```

This exception does not recursively make a source aggregate usable. After
relocating a structure containing an owner, the whole source structure is
inactive even though its owner field was cleared while implementing the
relocation.

C++ move construction has a different lifetime model: it constructs a
destination object while the source object remains alive and is destroyed
later. A C++ move constructor must therefore establish a destructible source
state. HUC destructive relocation ends the non-owner source lifetime
immediately, so it needs neither a moved-from-state contract nor a source
destructor call.

The backend may replace recursive structural relocation with a bytewise or
bulk representation transfer only when it proves the observable result
equivalent. That is an optimization, not a separate HUC operation and not a
promise that padding bytes are semantically significant.

### 7.1 Why relocation is not customizable in HUC0

There is no recognized declaration such as:

```huc
// These do not override relocation.
fn move(...) -> T;
fn move_init(...) -> T;
fn relocate(...) -> T;
fn operator=(...) -> T;
```

An ordinarily named method may implement a visible domain operation, but
binding, argument passing, returning, assignment, and container growth never
select it implicitly.

Fixed destructive relocation provides:

- no hidden allocation or I/O during ordinary transfer;
- no overload resolution based on value categories;
- no user code running merely because a value crosses a scope boundary;
- predictable recursive lowering;
- one rule for parameters, returns, assignments, and containers;
- a guarantee that relocation itself cannot fail;
- no requirement that resource wrappers reserve an “empty” sentinel merely
  so a source destructor can run.

This is the best default for HUC0. A future proposal may add an explicit
relocation customization point if real programs demonstrate that the
address-dependent cases below are common enough to justify the additional
type-system, generic-code, and failure-semantics complexity. Such a facility
must not be smuggled in as another spelling of `init`.

### 7.2 Address-dependent inline values

Fixed relocation has one important boundary: it cannot repair an invariant
that depends on the inline object's address. For example:

```huc
struct SelfIndexed {
    let i32 mod payload;
    let SelfIndexed* mod self;

    fn init(i32 payload)
        : payload(payload),
          self(null) {
        this->self = this;
    }

    fn has_expected_address() -> bool {
        return this->self == this;
    }

    fn drop() mod -> void {
        // Empty: this opts the scalar-only structure into Move.
    }
}

let SelfIndexed mod first = SelfIndexed(10);
let SelfIndexed second = first; // structural relocation

// second.self still contains the old inline address. Relying on the broken
// invariant is undefined behavior.
```

HUC0 deliberately has no automatic self-pointer discovery or repair. A value
with this invariant must instead:

- live behind `T&`, so relocation transfers only the owner word and the pointee
  address stays stable;
- be redesigned to compute internal addresses when needed rather than storing
  them;
- remain in directly constructed, fixed read-only storage and never be passed,
  returned, assigned, or stored in a relocating container by value; or
- use a purpose-built stable-address library container or low-level storage
  API whose contract avoids inline relocation.

The usual solution is ownership:

```huc
let mod SelfIndexed& mod stable = new SelfIndexed(10);
let mod SelfIndexed& elsewhere = stable;

// Only the owner word relocated. The SelfIndexed allocation did not move.
if (elsewhere->has_expected_address()) {
    probable_use(elsewhere);
}
```

`Vector<SelfIndexed>` is invalid for this invariant when growth relocates
elements. `Vector<SelfIndexed&>` or a stable node container is appropriate.
The same caveat applies to by-value parameters, returns, aggregate fields, and
assignment. HUC is unchecked: obvious cases may be warned about, but the
compiler is not required to prove address independence.

### 7.3 Eligible source places

Destructive relocation of a non-owner Move value is allowed only from an
entire compiler-tracked root place:

- a named local or parameter with the required trailing `mod`;
- a fresh temporary or function-result place;
- the source of compiler-generated recursive relocation of a whole aggregate.

HUC0 rejects explicit non-owner move-out from a pointee or subobject:

```huc
let File value = *file_owner;          // error: indirect Move source
let Lease lease = session.lease;       // error: Move subobject source
let Packet packet = packets[index];    // error: Move element source
```

Otherwise the enclosing object, owner allocation, or array could remain active
while one of the places its later cleanup assumes active no longer contains a
live value. In the first example, `file_owner` would still be non-null and
would later try to drop an inactive `File`, violating the one-word-owner
contract.

This restriction does not affect:

- Copy values, which are copied rather than consumed;
- `copy *file_owner`, which clones while leaving the pointee active;
- relocation of the whole `file_owner`, which leaves its source owner null;
- relocation of a primitive owner field, because that source subobject remains
  an active null owner;
- replacement of a valid indirect destination from an otherwise eligible
  source.

Exact self-assignment remains a no-op. When an eligible root source and an
indirect destination can alias, the backend compares their storage addresses
before destroying the destination. Container and manual-storage
implementations that need to relocate non-owner elements through addresses
will require an explicit compiler-backed storage intrinsic; that low-level API
is a separate standard-library design, not an implicit user relocation hook in
HUC0.

### 7.4 Native handle example

```huc
module examples.file_value;

struct File {
    let i32 mod handle;

    fn init(i32 handle)
        : handle(handle) {
    }

    fn drop() mod -> void {
        if (this->handle >= 0) {
            probable_os_close(this->handle);
            this->handle = -1;
        }
    }
}

fn transfer() -> void {
    let File mod input = File(probable_os_open("input.dat"));
    let File output = input; // destructive relocation; input is inactive
} // output.drop closes the handle exactly once
```

Declaring `drop` makes `File` a Move type. Structural relocation copies the
integer handle into the destination and deactivates the source. It need not
write `-1` into `input.handle`: the source's `drop` is skipped because the
source no longer contains an active `File`.

The conceptual C++20 counterpart needs user-written move code:

```cpp
class File {
public:
    explicit File(int handle) : handle_(handle) {}

    File(const File&) = delete;
    File& operator=(const File&) = delete;

    File(File&& source) noexcept
        : handle_(std::exchange(source.handle_, -1)) {}

    File& operator=(File&& source) noexcept {
        if (this != &source) {
            close();
            handle_ = std::exchange(source.handle_, -1);
        }
        return *this;
    }

    ~File() {
        close();
    }

private:
    void close() noexcept {
        if (handle_ >= 0) {
            probable_os_close(handle_);
            handle_ = -1;
        }
    }

    int handle_;
};
```

The C++ source must write a sentinel because `source` remains a live `File`
whose destructor later runs. HUC does not have that requirement. Its backend
uses static liveness where possible and a sidecar active flag only at
control-flow joins that need one; it must not add a hidden field to `File`.

## 8. Relocation assignment and exact self-relocation

A writable destination can be replaced by destructively relocating another
value:

```huc
let File mod current = File(probable_os_open("old.dat"));
let File mod replacement = File(probable_os_open("new.dat"));

current = replacement;
```

For distinct active objects, relocation assignment:

1. evaluates the destination location once;
2. evaluates the source location once;
3. checks whether they denote exactly the same storage;
4. destroys the old active destination;
5. relocates the source into the now-inactive destination.

The source is then inactive, except that a primitive owner source is usable
null. Exact self-relocation assignment is a no-op and does not destroy the
value:

```huc
current = current;
```

Non-identical overlapping source and destination storage is undefined behavior.
The compiler should diagnose obvious cases.

Assignment to an already inactive destination skips destination destruction
and begins a new active lifetime:

```huc
let File mod first = File(probable_os_open("a.dat"));
let File mod second = first; // destructive relocation; first inactive

first = File(probable_os_open("b.dat")); // first active again
```

For an owner, the same operation destroys the old pointee before adopting the
new address:

```huc
let Widget& mod destination = new Widget(1);
let Widget& mod source = new Widget(2);

destination = source; // destroys Widget(1), transfers Widget(2), clears source
```

The C++ counterpart is:

```cpp
std::unique_ptr<Widget> destination = std::make_unique<Widget>(1);
std::unique_ptr<Widget> source = std::make_unique<Widget>(2);

destination = std::move(source);
```

## 9. Destruction with `drop`

A structure may declare at most one destructor body:

```huc
fn drop() mod -> void;
```

It must:

- be an unnumbered runtime method;
- have no explicit parameters;
- return exactly `void`;
- have a writable receiver;
- not be overloaded.

`drop` cannot be called directly or have its address taken. The compiler calls
it once when an active value is **destroyed**. Destructive relocation is not
destruction: it ends the source lifetime without calling source `drop`, and the
relocated destination later carries the cleanup obligation. `drop` is intended
to release resources that are not already represented by automatically
destroyed fields.

All HUC0 structure members are public. Compiler-inserted destruction applies
wherever destruction of an active value is required.

After `drop` returns, fields are destroyed in reverse declaration order.

```huc
struct Session {
    let Logger& log;
    let Socket& socket;
    let i64 mod native_transaction;

    fn init(Logger& mod log, Socket& mod socket, i64 native_transaction)
        : log(log),
          socket(socket),
          native_transaction(native_transaction) {
    }

    fn drop() mod -> void {
        if (this->native_transaction != 0) {
            probable_abort_transaction(this->native_transaction);
            this->native_transaction = 0;
        }
    }
}
```

Destruction order is:

1. `Session.drop`;
2. `socket` pointee destruction and deallocation;
3. `log` pointee destruction and deallocation.

`drop` must not manually destroy owner or Move fields that normal field cleanup
will destroy. It may relocate a primitive owner field elsewhere, after which
field cleanup observes a null owner. HUC0's source-place rule forbids
relocating a non-owner Move field out of the still-addressable object, even
from `drop`.

There is no exception unwinding in HUC0. A terminating panic need not execute
pending `drop` operations.

## 10. Parameters and argument order

Parameters are passed by value. Their behavior follows their type:

```huc
fn inspect(Document* document) -> void {
    // raw observation; caller retains ownership
}

fn revise(mod Document* document) -> void {
    // raw mutable observation; caller retains ownership
}

fn summarize(Metadata metadata) -> void {
    // copies when Metadata is Copy
}

fn consume(Document& document) -> void {
    // owns document; destroys it on return unless relocated onward
}

fn relay(Document& mod document) -> void {
    probable_send(document); // relocates the parameter onward
}
```

At a call:

```huc
let mod Document& mod document = new Document(...);

inspect(document); // owner-to-observer conversion; no relocation
revise(document);  // mutable owner-to-mutable-observer conversion
consume(document); // relocation; caller's owner becomes null
```

Argument expressions and their corresponding parameter initializations occur
strictly left-to-right.

This guarantee is not a memory-safety mechanism. HUC could leave the order
unspecified and remain unchecked, but then the same ownership expression could
mean different things across backends or optimization choices. The defined
order makes relocations, allocation, I/O, device access, counters, and
temporary construction follow source reading order.

```huc
fn receive(Document* observed, Document& owned) -> void {
    // observed points to the object held by owned
}

let Document& mod document = new Document(...);
receive(document, document);
```

The first argument captures the address. The second argument then relocates
the owner. Reversing that order would clear `document` before observation and
would produce a different result.

Another important example is:

```huc
fn compare_versions(Document left, Document right) -> i32;

let Document mod document = Document(...);
let i32 ordering = compare_versions(copy document, document);
```

HUC clones `document` for `left`, then relocates the original into `right`.

C++20 does not provide the same left-to-right argument guarantee. The backend
must therefore materialize each argument's **parameter value** before it begins
the next argument. For the owner example, generated code is conceptually:

```cpp
Document* huc_param_0 = document.get();
huc_rt::owner<Document> huc_param_1 = std::move(document);
receive(huc_param_0, std::move(huc_param_1));
```

The first declaration completes the owner-to-observer conversion. The second
then clears `document`. Merely saving `auto&&` aliases and postponing both
parameter conversions until the final C++ call would be incorrect. The final
`std::move` above is backend transport from an already initialized holder, not
another HUC operation. The actual emitter uses concrete typed holders or an
explicit call frame and preserves their cleanup. Optimizers remain free to
reorder pure work when observable behavior is unchanged.

## 11. Return values

Returning a Copy value copies it semantically:

```huc
fn origin() -> Point {
    let Point point = Point(0, 0);
    return point;
}
```

Returning a Move value destructively relocates it:

```huc
fn open_file(text::View path) -> File {
    let File mod result = File(probable_os_open(path));
    return result; // relocation; result becomes inactive
}
```

Returning an owner relocates the owner:

```huc
fn make_widget(i32 id) -> Widget& {
    return new Widget(id);
}
```

The implementation may construct directly in caller-provided result storage.
Copy elision may remove transfers, but it must not introduce or remove
observable `clone` calls. `clone` runs exactly when required by an evaluated
`copy` expression.

Comparable C++20:

```cpp
Point origin() {
    const Point point{0, 0};
    return point;
}

File open_file(std::string_view path) {
    File result(probable_os_open(path));
    return result;
}

std::unique_ptr<Widget> make_widget(std::int32_t id) {
    return std::make_unique<Widget>(id);
}
```

## 12. Containers and element value semantics

The following APIs are a probable standard-library design, not yet a separately
accepted container specification. They demonstrate how a container can use the
same Copy/Move rules without adding another ownership model.

The code in this section is HUC1-facing library source: `Vector<T>` is a
phase-1 family request. Before HUC0 checking, staging replaces every request
with a concrete nominal type name. HUC0 itself never parses `Vector<T>`.

```huc
module examples.vector_values;

import std.collections as collections;

// Point, HeapText, and Widget refer to the example types defined above.

fn point_values() -> void {
    let collections::Vector<Point> mod points =
        collections::Vector<Point>();
    let Point point = Point(3, 4);

    points.push(point);      // Point is Copy: copies into the vector
    points.push(copy point); // equivalent, but redundant for Copy Point
}

fn text_values() -> void {
    let collections::Vector<HeapText> mod texts =
        collections::Vector<HeapText>();
    let HeapText mod text = HeapText("first");

    texts.push(text); // destructive relocation; text becomes inactive
    text = HeapText("second");
    texts.push(copy text); // clone, then relocate the clone into the vector
}

fn owner_values() -> void {
    let collections::Vector<Widget&> mod widgets =
        collections::Vector<Widget&>();
    let Widget& mod widget = new Widget(7);

    widgets.push(widget); // relocates one owner word; widget becomes null
}
```

A reallocation of `Vector<T>`:

- copies elements when `T` is Copy;
- uses fixed destructive relocation when `T` is Move;
- never calls `clone` merely to relocate elements;
- does not drop old element places that relocation made inactive;
- drops every element that remains active exactly once;
- may invalidate raw pointers into its storage without a diagnostic.

Because ordinary HUC0 code cannot relocate a non-owner Move value out through
a raw element pointer, the eventual `Vector` implementation needs a narrowly
scoped compiler-backed raw-storage relocation intrinsic. That library
intrinsic applies the same fixed semantics and is not available as a general
user move hook. Its API belongs to the separate standard-library design.

If `Vector<T>` itself supports logical copying, its library implementation can
declare `clone` and explicitly copy each element. Copying a vector of Move
elements is then available only when those elements support `copy`.

Comparable C++20 calls are:

```cpp
std::vector<Point> points;
const Point point{3, 4};
points.push_back(point); // copy

std::vector<HeapText> texts;
HeapText text("first");
texts.push_back(std::move(text)); // move must be requested at the C++ call

text = HeapText("second");
texts.push_back(text); // C++ copy; HUC spells this `copy text`

std::vector<std::unique_ptr<Widget>> widgets;
auto widget = std::make_unique<Widget>(7);
widgets.push_back(std::move(widget));
```

This is the same container-level value model as C++ RAII, with two policy
changes: HUC makes expensive logical copying explicit and makes relocation of a
Move type automatic and non-overridable.

## 13. A complete owner-valued example

The following HUC program combines construction, observation, mutation,
explicit copying, destructive relocation, owner assignment, parameter
transfer, and destruction:

```huc
module examples.image_asset;

import std.io as io;
import std.memory as memory;

struct Image {
    let mod u8* mod pixels;
    let usize width;
    let usize height;
    let usize stride;

    fn init(usize width, usize height)
        : pixels(memory::allocate_zeroed_bytes(width * height * 4)),
          width(width),
          height(height),
          stride(width * 4) {
    }

    fn byte_count() -> usize {
        return this->stride * this->height;
    }

    fn fill(u8 value) mod -> void {
        memory::fill_bytes(this->pixels, value, this->byte_count());
    }

    fn clone() -> Image {
        let Image mod result = Image(this->width, this->height);
        memory::copy_bytes(
            result.pixels,
            this->pixels,
            this->byte_count()
        );
        return result;
    }

    fn drop() mod -> void {
        if (this->pixels) {
            memory::free_raw_bytes(this->pixels, this->byte_count());
            this->pixels = null;
        }
    }
}

fn print_shape(Image* image) -> void {
    io::println(image->width, "x", image->height);
}

fn invert(mod Image* image) -> void {
    probable_invert_bytes(image->pixels, image->byte_count());
}

fn upload(Image& image) -> void {
    probable_gpu_upload(image->pixels, image->width, image->height);
} // image is destroyed here

fn main() -> i32 {
    let mod Image& mod working = new Image(1920, 1080);
    working->fill(0);

    print_shape(working);           // observe; working still owns
    invert(working);                // mutate pointee; still owns

    let Image& snapshot = copy working; // deep copy through Image.clone

    let mod Image& mod outgoing = working; // relocate; working becomes null
    print_shape(outgoing);
    upload(outgoing);               // relocate; outgoing becomes null

    print_shape(snapshot);
    return 0;
} // snapshot destroys its cloned Image
```

An idiomatic C++20 program with the same ownership behavior would use a
move-aware RAII `Image` type and `std::unique_ptr<Image>`:

```cpp
class Image {
public:
    Image(std::size_t width, std::size_t height)
        : pixels_(std::make_unique<std::uint8_t[]>(width * height * 4)),
          width_(width),
          height_(height),
          stride_(width * 4) {
        std::memset(pixels_.get(), 0, byte_count());
    }

    Image(const Image& source)
        : Image(source.width_, source.height_) {
        std::memcpy(pixels_.get(), source.pixels_.get(), byte_count());
    }

    Image& operator=(const Image& source) {
        if (this != &source) {
            Image temporary(source);
            *this = std::move(temporary);
        }
        return *this;
    }

    Image(Image&&) noexcept = default;
    Image& operator=(Image&&) noexcept = default;
    ~Image() = default;

    std::size_t byte_count() const {
        return stride_ * height_;
    }

    void fill(std::uint8_t value) {
        std::memset(pixels_.get(), value, byte_count());
    }

private:
    std::unique_ptr<std::uint8_t[]> pixels_;
    std::size_t width_;
    std::size_t height_;
    std::size_t stride_;
};

void print_shape(const Image* image);
void invert(Image* image);
void upload(std::unique_ptr<Image> image);

int main() {
    auto working = std::make_unique<Image>(1920, 1080);
    working->fill(0);

    print_shape(working.get());
    invert(working.get());

    const auto snapshot = std::make_unique<Image>(*working);

    auto outgoing = std::move(working);
    print_shape(outgoing.get());
    upload(std::move(outgoing));

    print_shape(snapshot.get());
    return 0;
}
```

Both programs provide:

- inline values with copy or move classification;
- explicit unique heap ownership;
- non-owning observation;
- mutable observation;
- deterministic RAII cleanup;
- deep logical copying;
- low-cost by-value transfer and replacement;
- by-value ownership transfer;
- return-value optimization opportunities.

HUC removes the need to spell `std::move`, delete copy members, or implement
move members. Its transfer operation is destructive relocation rather than a
customizable C++ move, so the source does not remain a live non-owner object.
In exchange, HUC does not offer custom implicit copy assignment, custom
relocation behavior, C++ references, or C++ value-category overloads.

## 14. C++ correspondence table

| HUC operation | Closest C++20 operation | Important difference |
|---|---|---|
| `let T value = T(...)` | `const T value(...)` | HUC is fixed by default |
| `let T mod value = T(...)` | `T value(...)` | HUC spells writable storage explicitly |
| `let T b = a` for Copy `T` | `T b = a` | Both copy |
| `let T b = a` for Move `T` | `T b = std::move(a)` | HUC destructively relocates and ends the non-owner source lifetime |
| `copy a` for Move `T` | Copy construction | HUC makes a potentially expensive copy explicit |
| `destination = copy source` | Copy assignment | HUC composes clone plus replace |
| `destination = source` for Move `T` | Move assignment | HUC drops the destination, then performs non-overridable relocation |
| `T*` | `const T*` by default | HUC pointer is always raw and unchecked |
| `mod T*` | `T*` | Writable pointee |
| `T&` | `std::unique_ptr<T>` | HUC `&` is ownership, not reference |
| `new T(...)` | `std::make_unique<T>(...)` | Produces `T&` |
| Owner-to-`T*` conversion | `.get()` | HUC conversion is implicit in observer contexts |
| Owner parameter `T&` | `std::unique_ptr<T>` by value | HUC call relocates the owner automatically |
| `fn clone() -> T` | Copy constructor, often clone function | Invoked only by explicit logical copying |
| `fn drop() mod -> void` | Destructor body | HUC destroys fields after `drop` |
| Fixed HUC destructive relocation | Move constructor/assignment | HUC ends the source lifetime and has no user hook |
| Inactive source storage after relocation | Valid-but-unspecified moved-from object | HUC source contains no live non-owner object until reinitialized |

“Comparable with C++” therefore means that ordinary C++ RAII and value-oriented
designs have direct HUC representations. It does not mean every C++ special
member, reference category, allocator hook, or overload pattern exists in HUC.

## 15. Required diagnostics

The compiler must reject:

- relocating from a fixed source slot;
- explicit relocation of a non-owner Move value from a pointee, field
  subobject, or array element;
- copying a Move structure that has no valid `clone`;
- a `clone` with parameters, a writable receiver, or the wrong return type;
- more than one `clone` in a structure;
- direct calls to `clone`;
- a `drop` with parameters or a non-`void` return;
- a `drop` without a writable receiver;
- more than one `drop` in a structure;
- direct calls to `drop`;
- attempts to declare a special move, relocation-constructor, or
  move-assignment hook;
- ordinary use of inactive storage when detected;
- any chained pointer/owner form, including one hidden by an alias;
- `addressof` applied to a pointer or owner slot;
- attempts to bind `T&` as though it were a C++ reference alias.

Useful warnings include:

- a `clone` that returns or embeds a raw resource address from the source;
- a `drop` that manually destroys an automatically destroyed field;
- obvious non-identical overlapping relocation assignment;
- copying an owner inside a loop where allocation is likely unintended;
- retaining a raw observer across owner relocation or destruction;
- placing a known self-addressing inline type in a relocating container.

Warnings do not turn HUC into a memory-safe language and may be disabled.

## 16. Backend obligations

The HUC0-to-C++20 transpiler must preserve these source semantics rather than
inheriting C++ defaults:

1. `T&` lowers to a pointer-sized owner representation, never a C++ reference.
2. Copy/Move classification is decided by HUC semantic analysis.
3. `clone` runs only for an evaluated `copy` that requires it.
4. relocation is structural, non-failing, and independent of user overloads;
5. owner relocation clears the source to a usable null owner;
6. inactive structure values are not dropped;
7. exact self-relocation assignment is a no-op;
8. `drop` runs before reverse field cleanup;
9. arguments and parameter initialization are left-to-right;
10. generated C++ special members are implementation machinery, not extra HUC
    customization points;
11. active/inactive state belongs to a storage place and must not add a hidden
    field to nominal `T` or to pointer-sized `T&`;
12. user `drop` must not be placed in an unconditional C++ destructor that
    would also run for an inactive raw-handle source;
13. non-owner relocation sources must satisfy the root-place restriction, and
    a possibly aliasing indirect destination must be checked before replacement.

The middle-end must make construction, copy, relocation, activation,
deactivation, and cleanup explicit before C++ emission. This keeps the
semantics usable by a future C-output or native-code backend.

## 17. Design references

HUC's rule is its own language design, but the terminology is informed by
WG21's relocation work:

- [P1144R6, “Object relocation in terms of move plus destroy”](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p1144r6.html)
  analyzes relocation and representation-transfer optimizations.
- [P2786R12, “Trivial Relocatability For C++26”](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p2786r12.html)
  describes relocation in terms of ending the source lifetime and beginning
  the destination lifetime.
- [P2785R3, “Relocation in terms of a relocation constructor”](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2785r3.html)
  explores a user-defined relocation constructor—the customization point HUC0
  deliberately defers.

These papers do not define HUC semantics. In particular, HUC's recursive
structural rule, null-owner exception, `mod` requirement, and unchecked
address-dependence contract are HUC decisions.
