# HUC Value Semantics and Special Operations

Status: current design decision for HUC0

This document explains how HUC creates, copies, transfers, replaces, and
destroys values. It gives the exact rules and compares them with C++20.

The comparison concerns how values and resources behave, not source
compatibility. HUC deliberately lets users customize fewer operations.

A few terms used throughout:

- **Source** and **destination**: where a value comes from and where it goes.
- **Slot** or **place**: a storage location, such as a variable or field.
- **Inline value**: a `T` stored directly, rather than a `T*` holding
  its address.
- **Pointee**: the object a pointer points to.
- **Binding**: giving a value to a destination, such as a new local or parameter.
- **Active**: the slot contains a live value. An **inactive** slot must be
  reinitialized before it can be used as a value again; it receives no cleanup.

## 1. Values and raw pointers

HUC uses two related type forms:

| HUC type | Meaning | Binding behavior |
|---|---|---|
| `T` | An inline `T` value | Basic values copy; Advanced values transfer |
| `T*` | A nullable, unchecked, non-owning pointer | Copies one address |

HUC has no general reference type that aliases another variable. Raw pointers
never destroy or deallocate their pointees. Resource-managing structures use
the same `clone`, `drop`, and relocation rules as other Advanced values.

Address-taking uses the non-overloadable unary `&` operator:

```huc
let value: Widget = Widget(7);
let observer: Widget* = &value;
```

Pointer chains such as `T**` are not HUC0 types, including through aliases.
Unary `&` accepts an addressable inline non-pointer place: it returns
`T*` for a fixed place and `mod T*` for a writable place. Applying it to a
raw-pointer slot is a compile-time error. Its operand is evaluated once
without transferring a value, creating a temporary, extending storage
lifetime, or changing cleanup state. Binary `&` remains bitwise AND.

The separate non-overloadable intrinsic `slot_off(place) -> usize` exposes
the address of an addressable storage slot, including a raw-pointer slot.
For a pointer, this is the address of its slot, not its pointee.
`usize` is target-pointer-sized. The operation evaluates the place once
without loading or transferring its value. It does not clone, destroy,
allocate, extend a lifetime, or change cleanup state.

An ordinary function may receive this integer and reconstruct a raw pointer
with `ptr_as<T*>(address)`. This gives low-level access to the stored bytes,
not a new reference type or permission to use inactive storage. Byte access
through `u8*` or `c8*` does not itself perform lifetime operations or
establish another valid cleanup obligation for a resource.

Taking a fixed slot's address is allowed. The compiler does not track mutation
permissions through these integers and casts or insert permission checks.
The programmer must respect storage permissions, lifetime, alignment, access
type, valid representation, and cleanup. Writing fixed storage remains
undefined behavior; ordinary typed `mod` rules are unchanged.

Declarations use `name: Type`. `mod` before the name controls the slot;
`mod` inside a pointer type controls the pointee:

```huc
let observer: T*; // fixed field; constructor obligation
let observer: mod T*; // fixed field; writable T; constructor obligation
let mod observer: T*; // reseatable field; defaults to null
let mod observer: mod T*; // reseatable field; writable T; defaults to null
```

These are field forms. A fixed local requires an explicit initializer. Each
constructor must initialize fixed fields that lack declaration initializers.
Omitted mutable pointer fields default to null. HUC0 globals are restricted
to drop-free Basic scalars and raw pointers.

A fixed structure prevents assignment to its inline fields. A `mod T*`
field still allows mutation of the separate pointee, but not replacement of
the pointer stored in that field. Each layer keeps its own permissions.

Relocating a named Advanced value or assigning to a named slot requires
binding `mod`. Compiler-inserted destruction does not. Fresh unnamed results
need no binding `mod`: there is no source declaration on which to write it.

```huc
let file: File = open_file("input.dat");
consume(make_widget(7));
```

Direct constructor initialization, described in section 4, creates no
intermediate source place. Copying a raw pointer leaves it unchanged, even
when null, and never transfers or extends the pointee's lifetime.

`slot_off` is a compiler-known unqualified intrinsic, not a `std::`
library function.

## 2. Basic and Advanced types

Every complete runtime type is either **Basic** or **Advanced**.

The short rule is: **Basic values are copied; Advanced values are transferred.**
Ordinary binding leaves a Basic source usable. Transferring an inline Advanced
value makes its source inactive. Explicit `copy` requests a separate logical value
when the type supports it.

Basic types are:

- scalar arithmetic and boolean types;
- raw pointers;
- arrays whose element type is Basic;
- structures whose fields are all Basic and which declare neither `clone` nor
  `drop`.

Advanced types are:

- arrays whose element type is Advanced;
- structures containing at least one Advanced field;
- structures declaring `clone`;
- structures declaring `drop`.

Having `init()` alone does not make a structure Advanced. A raw `T*` is Basic
even when `T` is Advanced; it copies only the observer address. In contrast,
an inline Advanced field makes its containing structure Advanced too.

Declaring `clone` intentionally makes an otherwise Basic structure Advanced.
Otherwise ordinary binding could silently bypass the custom copy behavior:

```huc
struct Ticket {
    let id: u64;

    fn init(id: u64) : id(id) {
    }

    fn clone() -> Ticket {
        return Ticket(probable_issue_related_ticket(this->id));
    }
}

let mod original: Ticket = Ticket(100);
let moved: Ticket = original; // destructive relocation; clone is not called
let copied: Ticket = copy moved; // calls Ticket.clone
```

Basic and Advanced are category names, not keywords or declaration modifiers.
The compiler determines the category from the rules above.

A token containing only scalars can use an empty `drop` to become Advanced
when its author wants to prevent copying:

```huc
struct UniqueTicket {
    let value: u64;

    fn init(value: u64) : value(value) {
    }

    fn drop() mod -> void {
        // No external cleanup; this lifecycle declaration makes the type Advanced.
    }
}
```

With no `clone`, `UniqueTicket` can be relocated but cannot be copied. A future
explicit non-copyable marker may replace this pattern, but the first compiler
does not need another type-category keyword.

## 3. Special operations

HUC lets a type define three lifecycle methods:

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
| Construction | `fn init(...)` | `T(...)` |
| Fieldwise copy of a Basic type | None | Ordinary binding or `copy` |
| Logical copy of an Advanced structure | `fn clone() -> T` | Only requested by `copy` |
| Logical copy of an Advanced array | None | `copy` applies logical copy to each element |
| Destructive relocation initialization | None | Ordinary binding of an Advanced value |
| Destructive relocation assignment | None | Ordinary assignment of an Advanced value |
| Destruction | `fn drop() mod -> void` | Inserted by the compiler |
| Field destruction | None | Inserted after `drop`, in reverse field order |

This is deliberately smaller than C++'s special-member system. In particular,
there is no user-defined move constructor, relocation constructor, or
move-assignment operator.

## 4. Construction

A constructor is a runtime method named `init`:

```huc
struct Point {
    let x: i32;
    let y: i32;

    fn init(x: i32, y: i32)
        : x(x),
          y(y),
    {
    }
}

let origin: Point = Point(0, 0);
```

The expression `Point(0, 0)` constructs `origin` directly in its own storage.
No temporary `Point` is created and no relocation occurs. The same guarantee
applies when a constructor expression directly initializes:

- a field through a constructor initializer entry;
- a by-value function parameter;
- a function result in `return T(arguments)`.

HUC guarantees this; it is not an optional optimization. `init` sees the final
`this` address, and constructing a fixed Advanced value needs no writable
temporary first. More complex expressions that supply an existing Advanced
value still follow the normal transfer rules.

Constructor initializer entries:

1. must follow field declaration order;
2. are evaluated left-to-right;
3. initialize each field before the constructor body executes.

Every fixed field requires a declaration initializer or a constructor entry.
An omitted mutable scalar becomes zero, an omitted mutable pointer
becomes null, and an omitted mutable structure is default-constructed.

`init` does not double as a copy or relocation hook. An overload accepting
another `T` is just an explicitly selected constructor:

```huc
struct ScaledPoint {
    let x: i32;
    let y: i32;

    fn init(source: Point, scale: i32)
        : x(source.x * scale),
          y(source.y * scale),
    {
    }
}
```

## 5. Copying Basic values

A structure made entirely from Basic fields, with no `clone` and no `drop`, is
copied field by field:

```huc
struct Point {
    let x: i32;
    let y: i32;

    fn init(x: i32, y: i32)
        : x(x),
          y(y),
    {
    }
}

fn translate(point: Point, dx: i32, dy: i32) -> Point {
    return Point(point.x + dx, point.y + dy);
}

fn copy_values() -> void {
    let first: Point = Point(10, 20);
    let second: Point = first; // implicit fieldwise copy
    let third: Point = copy first; // same fieldwise copy
    let fourth: Point = translate(first, 3, 4); // first copied into parameter
}
```

Structure fields are copied in declaration order. The rule copies field values;
it does not promise to copy padding bytes. Because a Basic type
has no `clone` or `drop` method and no Advanced field, ordinary copying does
not invoke user copy code. It may still have constructors.

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

Writable Basic assignment is also direct:

```huc
let mod current: Point = Point(1, 2);
let replacement: Point = Point(8, 9);

current = replacement; // fieldwise copy of a Basic value
current = copy replacement; // equivalent for a Basic type
```

```cpp
Point current{1, 2};
const Point replacement{8, 9};

current = replacement;
current = replacement;
```

## 6. Custom logical copying with `clone`

An Advanced structure is not implicitly copied. It may support explicit logical
copying by declaring exactly one method with this signature:

```huc
fn clone() -> T;
```

Rules:

- it is an ordinary runtime `fn`, not `fn1` or `fn2`;
- it has no explicit parameters;
- its receiver is read-only because it has no trailing `mod`;
- it returns the same structure type in which it is declared;
- it must not modify the source object's own storage or leave the source invalid;
- it cannot be called directly or have its address taken;
- ordinary binding, assignment, argument passing, and return never call it;
- `copy source` calls it when `source` is an Advanced structure;
- declaring it makes the structure Advanced even when every field is Basic;
- omit `clone` when the type must not be copyable;
- it cannot return a recoverable error, although allocation failure or panic
  may terminate the process.

An “independent logical duplicate” means that the original and its copy can
each be destroyed without invalidating the other. To achieve this, `clone`
must duplicate owned resources or create another valid ownership share, for
example by increasing a library-managed reference count.

Deep cloning in HUC does **not** mean copying every reachable object. A `T*`
is an observer, so cloning a raw-pointer field normally copies only its
address. Shared external state, interned data, immutable tables, and other
observed objects may remain shared. The type author decides what belongs to
the value and writes the necessary `copy` operations inside `clone`. The
compiler cannot prove that the method follows these rules.

Allocation and updates to external bookkeeping are allowed. For example, a
future HUC1 `Rc<T>` could implement `copy rc` by increasing a shared reference
count through an explicitly writable observer. The source handle's own
storage must remain unchanged, and it must still refer to the same resource.

A type that needs to report a copying failure should provide an ordinary
function such as `try_copy`, returning a future HUC1 `Result<T, E>`
specialization.
That function is not selected by the `copy` expression.

The initial language has one logical copy operation rather than separate
custom copy-construction and copy-assignment hooks. Copy assignment is built
from `clone`, destruction of the old destination, and destructive relocation:

```huc
let mod destination: Text = Text("old");
let source: Text = Text("new");

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
    let mod data: mod u8*;
    let size: usize;

    fn init(source: text::View)
        : data(memory::allocate_raw_bytes(source.size())),
          size(source.size()),
    {
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
    let mod first: HeapText = HeapText("alpha");
    let second: HeapText = copy first; // independent allocation
    let third: HeapText = first; // relocate; first becomes inactive
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
describe the emitted backend code: the first backend emits C17 with explicit
lifetime operations, not C++ special members.

### 6.2 Copying through a raw pointer

Copying a raw pointer never copies its pointee and is valid even when null:

```huc
let text: HeapText = HeapText("hello");
let observer: HeapText* = &text;
let another_observer: HeapText* = copy observer; // copies only the address
let duplicate: HeapText = copy *observer; // calls HeapText.clone
```

The dereferenced form requires a live, accessible pointee that supports
logical copying. It creates an independent value and leaves the pointee
unchanged; it does not allocate storage for a separate wrapper object.

### 6.3 Specialized copy assignment

HUC's automatic `destination = copy source` always uses clone-then-replace.
When an application needs buffer reuse or another specialized algorithm, it
uses a clearly named writable method:

```huc
struct ReusableBuffer {
    // fields, init, clone, and drop omitted

    fn assign_from(source: ReusableBuffer*) mod -> void {
        probable_resize_in_place(this, source->size());
        probable_copy_payload(this, source);
    }
}

let mod buffer: ReusableBuffer = ReusableBuffer(...);
let other: ReusableBuffer = ReusableBuffer(...);
buffer.assign_from(&other);
```

`assign_from` is not an implicit copy-assignment hook. The call site explicitly
selects the specialized operation.

## 7. Transferring Advanced values

Basic and Advanced are the type categories. **Transfer** is the everyday term
for moving an Advanced value to a destination. The precise term is
**destructive relocation**: the value continues in the destination, and the
inline source becomes inactive. This differs from C++ move construction, which
leaves the source alive. The compiler specification uses “relocation” when
that distinction matters.

Here `Packet` is Advanced because it contains an Advanced `HeapText`, even
though it declares neither `clone` nor `drop`:

```huc
struct Packet {
    let tag: i32;
    let payload: HeapText;

    fn init(tag: i32, mod payload: HeapText)
        : tag(tag),
          payload(payload),
    {
    }
}

let mod source: Packet = Packet(1, HeapText("payload"));
let destination: Packet = source; // destructive relocation
```

Before this binding, `source` contains an active `Packet` and `destination`
does not. Relocation continues the value in `destination` and ends the active
`Packet` lifetime in `source`. It does not create a second live object and it
does not leave a moved-from `Packet`.

For an Advanced value, relocation transfers its stored representation:

1. Transfer the whole value's representation to the destination.
2. Make the entire source, including every field subobject, inactive.
3. Leave the destination responsible for the value's eventual cleanup.

The operation need not clear `source.payload`; that field is inactive,
not a second live value. Neither it nor `source` may be used as a value before
the whole source is reinitialized. No source `drop` body or source-field
cleanup runs. Arrays in the Advanced category use the same whole-value rule.
The operation invokes no user code and cannot fail in the HUC type system.

The rule is the same whether the structure is Advanced because of `clone`, `drop`,
or an Advanced field. It does not change ordinary binding of Basic types, which
duplicates the value and leaves the source active.

A fixed field cannot be assigned to directly, but it can be transferred as
part of a whole value. The containing source must be writable. For example,
transferring a writable structure also transfers its fixed Advanced fields.

The bytes of an inactive source need not be cleared and need not form a valid
representation of `T`. The compiler tracks whether the *storage place* is
active. Assigning a new value to an inactive place begins a new lifetime there.
Using it as a value before reinitialization is undefined behavior.

C++ move construction has a different lifetime model: it constructs a
destination object while the source object remains alive and is destroyed
later. A C++ move constructor must therefore establish a destructible source
state. HUC destructive relocation ends the source lifetime
immediately, so it needs neither a moved-from-state contract nor a source
destructor call.

Bitwise relocation is the language rule, not an optimization of separate field
transfers. It does not require a memory-copy instruction or give padding bytes
a defined meaning. The backend may copy memory, use loads and stores, keep
values in registers, or remove the transfer entirely. The program must still
behave the same: the source becomes inactive, and cleanup stays correct.

For example, `let a: T = b;` for an inline Advanced `T` may keep the value in
the same registers and need no machine instruction for the transfer. `b`
still becomes inactive; physical reuse does not make it usable afterward.

HUC permits transfers to be optimized away; it does not guarantee that every
transfer uses the same physical storage.

### 7.1 Why relocation is not customizable in HUC0

There is no recognized declaration such as:

```huc
// These do not override relocation.
fn move(...) -> T;
fn move_init(...) -> T;
fn relocate(...) -> T;
fn operator=(...) -> T;
```

A normally named method may implement a type-specific operation, but
binding, argument passing, returning, assignment, and container growth never
select it implicitly.

Fixed destructive relocation provides:

- no hidden allocation or I/O during ordinary transfer;
- no overload resolution based on value categories;
- no user code called by the transfer itself;
- predictable compiler implementation of the transfer;
- one rule for parameters, returns, assignments, and containers;
- a guarantee that relocation itself cannot fail;
- no need for a resource wrapper to reserve an “empty” value merely so a
  source destructor can run.

This is HUC0's chosen default. A future proposal may allow custom relocation
if real programs show that the address-dependent cases below are common enough
to justify more complex type rules, generic code, and failure handling. That
would need a separate feature, not another form of `init`.

### 7.2 Address-dependent inline values

Fixed relocation has one important limit: it cannot repair values that depend
on the object's current address. A stored pointer to the object itself is one
example:

```huc
struct SelfIndexed {
    let mod payload: i32;
    let mod self: SelfIndexed*;

    fn init(payload: i32)
        : payload(payload),
          self(null),
    {
        this->self = this;
    }

    fn has_expected_address() -> bool {
        return this->self == this;
    }

    fn drop() mod -> void {
        // Empty: this opts the scalar-only structure into Advanced.
    }
}

let mod first: SelfIndexed = SelfIndexed(10);
let second: SelfIndexed = first; // bitwise relocation; first becomes inactive

// second.self still contains the old inline address. Relying on the broken
// invariant is undefined behavior.
```

HUC0 does not automatically find or repair self-pointers. A value that needs
such a pointer to stay correct must instead:

- be redesigned to compute internal addresses when needed rather than storing
  them;
- remain in directly constructed, fixed read-only storage and never be passed,
  returned, assigned, or stored in a relocating container by value; or
- use a purpose-built stable-address library container or low-level storage
  API whose contract avoids inline relocation.

For a local value, one option is to construct it in fixed storage and pass
only observers:

```huc
let stable: SelfIndexed = SelfIndexed(10);
let elsewhere: SelfIndexed* = &stable;

if (elsewhere->has_expected_address()) {
    probable_use(elsewhere);
}
```

`Vector<SelfIndexed>` breaks this requirement when growth relocates elements.
Stable-address storage or a different representation is necessary.

The same caveat applies to by-value parameters, returns, aggregate fields, and
assignment. HUC is unchecked: obvious cases may be warned about, but the
compiler is not required to prove address independence.

### 7.3 Where a transfer may take its value from

Transferring an Advanced value requires a whole value whose active
state the compiler tracks directly:

- a named local or parameter with the required `mod` before its name;
- a fresh temporary or function-result place.

Transferring a whole structure or array includes all its fields or elements.
It does not remove them one at a time while leaving the containing value active.

HUC0 rejects explicit Advanced move-out from a pointee or subobject:

```huc
let value: File = *file_pointer; // error: indirect Advanced source
let lease: Lease = session.lease; // error: Advanced subobject source
let packet: Packet = packets[index]; // error: Advanced element source
```

Otherwise a containing object or array could remain active even though a
value it must later destroy is already inactive. A raw pointer also carries
no information about the pointee's cleanup scope. HUC0 does not track such
partial lifetimes for ordinary relocation.

This restriction does not affect:

- Basic values, which are copied rather than consumed;
- `copy *file_pointer`, which clones while leaving the pointee active;
- replacement of a valid indirect destination from an otherwise eligible
  source.

Exact self-assignment remains a no-op. When an eligible root source and an
indirect destination can alias, the backend compares their storage addresses
before destroying the destination. Container and manual-storage implementations
manage backing storage and initialized element ranges explicitly; this does
not extend ordinary relocation-source eligibility.

### 7.4 Native handle example

```huc
module examples.file_value;

struct File {
    let mod handle: i32;

    fn init(handle: i32)
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
    let mod input: File = File(probable_os_open("input.dat"));
    let output: File = input; // destructive relocation; input is inactive
} // output.drop closes the handle exactly once
```

Declaring `drop` makes `File` an Advanced type. Bitwise relocation transfers the
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
uses compile-time knowledge of whether the value is active where possible.
Where control-flow paths join and that state is uncertain, it uses a separate
flag if needed; it must not add a hidden field to `File`.

## 8. Assignment by transfer, including self-assignment

A writable destination can be replaced by destructively relocating another
value:

```huc
let mod current: File = File(probable_os_open("old.dat"));
let mod replacement: File = File(probable_os_open("new.dat"));

current = replacement;
```

For distinct active objects, relocation assignment:

1. evaluates the destination location once;
2. evaluates the source location once;
3. checks whether they refer to exactly the same storage;
4. destroys the old active destination;
5. relocates the source into the now-inactive destination.

The source is then inactive. Exact self-relocation assignment is a no-op
and does not destroy the value:

```huc
current = current;
```

Using source and destination storage that overlaps but is not identical is
undefined behavior.
The compiler should diagnose obvious cases.

Assignment to an already inactive destination skips destination destruction
and begins a new active lifetime:

```huc
let mod first: File = File(probable_os_open("a.dat"));
let mod second: File = first; // destructive relocation; first inactive

first = File(probable_os_open("b.dat")); // first active again
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
destination becomes responsible for later cleanup. `drop` is intended
to release resources that are not already represented by automatically
destroyed fields.

All HUC0 structure members are public. Compiler-inserted destruction applies
wherever destruction of an active value is required.

After `drop` returns, fields are destroyed in reverse declaration order.

```huc
struct Session {
    let log: Logger;
    let socket: Socket;
    let mod native_transaction: i64;

    fn init(mod log: Logger, mod socket: Socket, native_transaction: i64)
        : log(log),
          socket(socket),
          native_transaction(native_transaction),
    {
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
2. destruction of the inline `socket` field;
3. destruction of the inline `log` field.

`drop` must not manually destroy Advanced fields that normal field cleanup
will destroy. HUC0's source-place rule forbids relocating an Advanced field
out of the still-addressable object, even from `drop`.

There is no exception unwinding in HUC0. A terminating panic need not execute
pending `drop` operations.

## 10. Parameters and argument order

Parameters are passed by value. Their behavior follows their type:

```huc
fn inspect(document: Document*) -> void {
    // raw observation; caller retains the value
}

fn revise(document: mod Document*) -> void {
    // writable observation; caller retains the value
}

fn summarize(metadata: Metadata) -> void {
    // copies when Metadata is Basic
}

fn consume(document: Document) -> void {
    // consumes when Document is Advanced; destroys it on return
}

fn relay(mod document: Document) -> void {
    probable_send(document); // transfers the Advanced parameter onward
}
```

For an Advanced `Document`:

```huc
let mod document: Document = Document(...);

inspect(&document); // observation; no relocation
revise(&document); // writable observation
consume(document); // relocation; caller's document becomes inactive
```

Argument expressions and their corresponding parameter initializations occur
strictly left-to-right. This is not a memory-safety mechanism. It makes
relocations, allocation, I/O, device access, counters, and temporary
construction follow source reading order.

```huc
fn compare_versions(left: Document, right: Document) -> i32;

let mod document: Document = Document(...);
let ordering: i32 = compare_versions(copy document, document);
```

HUC clones `document` for `left`, then relocates the original into `right`.
Reversing the arguments would try to clone an already inactive source.
Likewise, passing `document` by value twice does not create two active values.

C17 does not provide this argument-order guarantee. The backend must finish
initializing each argument's **parameter value** before starting the next
argument. Saving pointers to source places and deferring their transfers
until the final C call is insufficient. Typed parameter slots, separately
or in a call frame, preserve sequencing and exactly-once cleanup. Optimizers
may reorder pure computation when the program still behaves the same.

## 11. Return values

Returning a Basic value as the same Basic type follows the normal copy rule:

```huc
fn origin() -> Point {
    let point: Point = Point(0, 0);
    return point;
}
```

Returning an Advanced value destructively relocates it:

```huc
fn open_file(path: text::View) -> File {
    let mod result: File = File(probable_os_open(path));
    return result; // relocation; result becomes inactive
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
```

## 12. Containers and element value semantics

The following APIs illustrate a possible standard library; the container
design is not yet approved. They show how a container can use the
same Basic/Advanced rules without adding another ownership model.

The code in this section is HUC1-facing library source: `Vector<T>` is a
phase-1 family request. Before HUC0 checking, staging replaces every request
with the name of a concrete type. HUC0 itself never parses `Vector<T>`.

```huc
module examples.vector_values;

import std.collections as collections;

// Point, HeapText, and Widget refer to the example types defined above.

fn point_values() -> void {
    let mod points: collections::Vector<Point> =
        collections::Vector<Point>();
    let point: Point = Point(3, 4);

    points.push(point); // Point is Basic: copies into the vector
    points.push(copy point); // equivalent, but redundant for Basic Point
}

fn text_values() -> void {
    let mod texts: collections::Vector<HeapText> =
        collections::Vector<HeapText>();
    let mod text: HeapText = HeapText("first");

    texts.push(text); // destructive relocation; text becomes inactive
    text = HeapText("second");
    texts.push(copy text); // clone, then relocate the clone into the vector
}

```

A reallocation of `Vector<T>`:

- copies elements when `T` is Basic;
- uses fixed destructive relocation when `T` is Advanced;
- never calls `clone` merely to relocate elements;
- does not drop old element places that relocation made inactive;
- drops every element that remains active exactly once;
- may invalidate raw pointers into its storage without a diagnostic.

The container is responsible for its backing storage and initialized element
range. It must ensure that old slots whose values were transferred receive
no element cleanup. The only additional raw-storage lifetime intrinsics
planned are `construct_at` and `destruct_at`; their interfaces and detailed
semantics will be designed later. Ordinary relocation-source restrictions and
the absence of user-defined relocation hooks remain unchanged.

If `Vector<T>` itself supports logical copying, its library implementation can
declare `clone` and explicitly copy each element. Copying a vector of Advanced
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
```

This is the same container-level value model as C++ RAII, with two policy
changes: HUC makes expensive logical copying explicit and makes relocation of
an Advanced type automatic and non-overridable.

## 13. A complete resource-managing value example

The following HUC program combines construction, observation, mutation,
explicit copying, destructive relocation, value assignment, parameter
transfer, and destruction:

```huc
module examples.image_asset;

import std.io as io;
import std.memory as memory;

struct Image {
    let mod pixels: mod u8*;
    let width: usize;
    let height: usize;
    let stride: usize;

    fn init(width: usize, height: usize)
        : pixels(memory::allocate_zeroed_bytes(width * height * 4)),
          width(width),
          height(height),
          stride(width * 4),
    {
    }

    fn byte_count() -> usize {
        return this->stride * this->height;
    }

    fn fill(value: u8) mod -> void {
        memory::fill_bytes(this->pixels, value, this->byte_count());
    }

    fn clone() -> Image {
        let mod result: Image = Image(this->width, this->height);
        memory::copy_bytes(
            result.pixels,
            this->pixels,
            this->byte_count(),
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

fn print_shape(image: Image*) -> void {
    io::println(image->width, "x", image->height);
}

fn invert(image: mod Image*) -> void {
    probable_invert_bytes(image->pixels, image->byte_count());
}

fn upload(image: Image) -> void {
    probable_gpu_upload(image.pixels, image.width, image.height);
} // image is destroyed here

fn main() -> i32 {
    let mod working: Image = Image(1920, 1080);
    working.fill(0);

    print_shape(&working); // observe
    invert(&working); // writable observation

    let snapshot: Image = copy working; // logical copy through Image.clone

    let mod outgoing: Image = working; // working becomes inactive
    print_shape(&outgoing);
    upload(outgoing); // outgoing becomes inactive

    print_shape(&snapshot);
    return 0;
} // snapshot destroys its independently copied pixel buffer
```

An analogous C++20 program uses a move-aware RAII `Image` value:

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
void upload(Image image);

int main() {
    Image working(1920, 1080);
    working.fill(0);

    print_shape(&working);
    invert(&working);

    const Image snapshot = working;

    auto outgoing = std::move(working);
    print_shape(&outgoing);
    upload(std::move(outgoing));

    print_shape(&snapshot);
    return 0;
}
```

Both programs provide:

- inline values that manage resource storage;
- non-owning observation;
- mutable observation;
- deterministic RAII cleanup;
- deep logical copying;
- low-cost by-value transfer and replacement;
- by-value ownership transfer;
- return-value optimization opportunities.

HUC removes the need to spell `std::move`, delete copy members, or implement
move members. Its transfer operation is destructive relocation rather than a
customizable C++ move, so the source does not remain a live object.
In exchange, HUC does not offer custom implicit copy assignment, custom
relocation behavior, C++ references, or C++ value-category overloads.

## 14. C++ correspondence table

| HUC operation | Closest C++20 operation | Important difference |
|---|---|---|
| `let value: T = T(...)` | `const T value(...)` | HUC is fixed by default |
| `let mod value: T = T(...)` | `T value(...)` | HUC spells writable storage explicitly |
| `let b: T = a` for Basic `T` | `T b = a` | Both copy |
| `let b: T = a` for Advanced `T` | `T b = std::move(a)` | HUC destructively relocates and ends the source lifetime |
| `copy a` for Advanced `T` | Copy construction | HUC makes a potentially expensive copy explicit |
| `destination = copy source` | Copy assignment | HUC composes clone plus replace |
| `destination = source` for Advanced `T` | Move assignment | HUC drops the destination, then performs non-overridable relocation |
| `T*` | `const T*` by default | HUC pointer is always raw and unchecked |
| `mod T*` | `T*` | Writable pointee |
| `&value` | `std::addressof(value)` | HUC permits only eligible inline data places and cannot overload `&` |
| `fn clone() -> T` | Copy constructor, often clone function | Invoked only by explicit logical copying |
| `fn drop() mod -> void` | Destructor body | HUC destroys fields after `drop` |
| Fixed HUC destructive relocation | Move constructor/assignment | HUC ends the source lifetime and has no user hook |
| Inactive source storage after relocation | Valid-but-unspecified moved-from object | HUC source contains no live object until reinitialized |

“Comparable with C++” therefore means that ordinary C++ RAII and value-oriented
designs have direct HUC representations. It does not mean every C++ special
member, reference category, allocator hook, or overload pattern exists in HUC.

## 15. Required diagnostics

The compiler must reject:

- relocating from a fixed source slot;
- explicit relocation of an Advanced value from a pointee, field
  subobject, or array element;
- copying an Advanced structure that has no valid `clone`;
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
- any chained pointer form, including one hidden by an alias;
- unary `&` applied to a pointer slot or non-place result;
- `slot_off` applied to a non-place, such as a literal or function symbol.

Useful warnings include:

- a `clone` that returns or embeds a raw resource address from the source;
- a `drop` that manually destroys an automatically destroyed field;
- obvious non-identical overlapping relocation assignment;
- retaining a raw observer across value relocation or destruction;
- placing a known self-addressing inline type in a relocating container.

Warnings do not turn HUC into a memory-safe language and may be disabled.

## 16. Backend requirements

The HUC0-to-C17 transpiler must preserve these source semantics rather than
inheriting C defaults:

1. Basic/Advanced classification is decided by HUC semantic analysis.
2. `clone` runs only for an evaluated `copy` that requires it.
3. Relocation transfers the representation, makes the whole source inactive,
   and requires no clearing of source bytes; it is non-failing and independent
   of user overloads.
4. Inactive values are not dropped.
5. Exact self-relocation assignment is a no-op.
6. `drop` runs before reverse field cleanup.
7. Arguments and parameter initialization are left-to-right.
8. Generated lifecycle functions are implementation machinery, not extra HUC
   customization points.
9. Active/inactive state belongs to a storage place and must not add a hidden
   field to nominal `T`.
10. Cleanup covers every normal exit and active replacement. An inactive
    source receives no cleanup merely because its physical C storage leaves scope.
11. Relocation sources satisfy the root-place restriction; a possibly aliasing
    indirect destination is checked before replacement.
12. Raw-pointer copying leaves the source unchanged and never cleans up or
    extends the lifetime of a pointee.

The middle-end makes construction, copy, relocation, activation, deactivation,
and cleanup explicit before C emission. C assignment and byte copying are
permitted implementations only when they preserve the complete HUC operation,
including active state, and obey C's storage and aliasing rules. This keeps
the semantics usable by a future C++ or native-code backend.

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

These papers do not define HUC semantics. In particular, HUC's bitwise
relocation rule, `mod` requirement, and
unchecked address-dependence contract are HUC decisions.
