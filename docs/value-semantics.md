# HUC Value Semantics and Special Operations

Status: current design decision for HUC0

This document explains how HUC creates, copies, transfers, replaces, and
destroys values. It gives the exact rules and compares them with C++20.

The comparison concerns how values and resources behave, not source
compatibility. HUC deliberately lets users customize fewer operations.

A few terms used throughout:

- **Source** and **destination**: where a value comes from and where it goes.
- **Slot** or **place**: a storage location, such as a variable or field.
- **Inline value**: a `T` stored directly, rather than a `T*` or `T&` holding
  its address.
- **Pointee**: the object a pointer or owner points to.
- **Binding**: giving a value to a destination, such as a new local or parameter.
- **Active**: the slot contains a live value. An **inactive** slot must be
  reinitialized before it can be used as a value again; it receives no cleanup.

## 1. The three basic storage relationships

HUC uses three related type forms:

| HUC type | Meaning | Transfer behavior |
|---|---|---|
| `T` | An inline `T` value | Determined by `T` |
| `T*` | A nullable, unchecked, non-owning pointer | Copies one address |
| `T&` | A nullable, unique owner of one allocated `T` | Relocates by default; `copy` duplicates the pointee |

`T&` is not a C++ reference. HUC has no general reference type that aliases
another variable, and no `T&&` type. A non-null `T&` owns an allocation,
automatically destroys its pointee, and then deallocates that storage.
Directly relocating an owner slot clears that source slot to null. Relocating
an enclosing inline aggregate instead makes the entire source inactive without
requiring owner-field nulling.

HUC has no unary `&` expression. Address-taking uses the core,
non-overloadable `addressof` intrinsic:

```huc
let Widget value = Widget(7);
let Widget* observer = addressof(value);
let Widget& owner = new Widget(9);
```

The pointer-like forms cannot be combined:
`T**`, `T*&`, `T&*`, and `T&&` are not HUC0 types, including through aliases.
`addressof` accepts only an inline non-pointer place: it returns `T*` for a
fixed place and `mod T*` for a writable place. Applying it to a raw-pointer or
owner slot is a compile-time error. Using `owner` where `T*` is expected gives
access to the pointee, not to the owner slot itself. To replace a caller's
owner through an ordinary typed function, pass ownership in and return the
replacement (consume-and-return). Binary `&` remains bitwise AND.

The separate non-overloadable intrinsic `std::slot_of(place) -> usize` exposes
the address of an addressable storage slot, including a raw-pointer or owner
slot. For an owner, this is the address of the owner word, not its pointee.
`usize` is target-pointer-sized. The operation evaluates the place once without
loading or transferring its value. It does not clone, destroy, allocate,
extend a lifetime, or change cleanup state.

An ordinary function may receive this integer and reconstruct a raw pointer
with `ptr_as<T*>(address)`. This gives low-level access to the stored bytes,
not a new typed owner reference: `T&*` and the other combined forms remain
forbidden.
Byte access through `u8*` or `c8*` can inspect or manipulate representation;
it does not itself perform HUC ownership or lifetime operations. In particular,
duplicating an owner word does not create a second valid owner.

Taking a fixed slot's address is allowed. The compiler does not track mutation
permissions through these integers and casts or insert permission checks.
The programmer must ensure that access respects the storage's permissions,
lifetime, and alignment, and leaves valid values with unique ownership and
correct cleanup. Writing a fixed slot remains undefined behavior; ordinary
typed `mod` rules are unchanged. See the language specification's
pointer-layering and cast sections for the full rules.

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
drop-free Basic scalars and raw pointers, so owners are never global in 0.1.

A fixed structure prevents assignment to its fields and mutation of values
stored inline in those fields. A pointer or owner field with leading `mod`
still allows mutation of the separate object it points to.
For example, a read-only method may mutate the pointee of a `mod T*` field but
may not replace the pointer stored in that field. Each layer keeps its own
permissions.

A trailing `mod` is required when source code will relocate from, reset,
release, or reseat a **named** pointer, owner, or other Advanced slot.
Compiler-inserted destruction at the end of the slot's lifetime does not
require source-level `mod`.

A fresh unnamed result can be transferred directly. Function results,
`new T(...)` owner results, evaluated `copy` results, and other temporary
Advanced values need no trailing `mod`: there is no source declaration on
which to write it.

```huc
let File file = open_file("input.dat");
consume(new Widget(7));
```

Only relocation from named storage tests its trailing `mod`. Direct
constructor initialization, described in section 4, creates no intermediate
source place at all.

## 2. Basic and Advanced types

Every complete runtime type is either **Basic** or **Advanced**.

The short rule is: **Basic values are copied; Advanced values are transferred.**
Ordinary binding leaves a Basic source usable. Transferring an inline Advanced
value makes its source inactive; directly transferring an owner leaves its
source active and null. Explicit `copy` requests a separate logical value
when the type supports it.

Basic types are:

- scalar arithmetic and boolean types;
- raw pointers;
- arrays whose element type is Basic;
- structures whose fields are all Basic and which declare neither `clone` nor
  `drop`.

Advanced types are:

- unique owners;
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

Basic and Advanced are category names, not keywords or declaration modifiers.
The compiler determines the category from the rules above.

A token containing only scalars can use an empty `drop` to become Advanced
when its author wants to prevent copying:

```huc
struct UniqueTicket {
    let u64 value;

    fn init(u64 value) : value(value) {
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
| Construction | `fn init(...)` | `T(...)` or `new T(...)` |
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

These expressions construct values directly. `Point(0, 0)` constructs
`origin` in its own storage, and `new Point(10, 20)` constructs in the final
allocation.
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

## 5. Copying Basic values

A structure made entirely from Basic fields, with no `clone` and no `drop`, is
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
let Point mod current = Point(1, 2);
let Point replacement = Point(8, 9);

current = replacement;       // fieldwise copy of a Basic value
current = copy replacement;  // equivalent for a Basic type
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
describe the emitted backend code: the first backend emits C17 with explicit
lifetime operations, not C++ special members.

### 6.2 Unique-owner copying

`copy` on an owner performs an explicit deep copy of the owned object. The
source owner must be non-null; evaluating `copy` on a null owner is undefined
behavior. The operation has no implicit null-preserving case:

```huc
let HeapText& mod first = new HeapText("hello");
let HeapText& second = copy first;
```

For a valid source, the operation:

1. evaluates `first`;
2. allocates uninitialized storage for one `HeapText`;
3. logically copies `*first` into that storage, using fieldwise copying for a
   Basic pointee or `clone` for an Advanced structure;
4. returns a new independent owner.

The pointee type must support logical copying at compile time. A null value
does not bypass this type requirement, and no runtime null check is required.
The non-null precondition applies even if the pointee's copy operation would
not otherwise read its storage.

Code that needs null-preserving duplication must test the source explicitly
and produce a null result itself. This includes `clone` methods for structures
with optional owner fields. Null owners remain valid values for observation,
relocation, and destruction; only copying their nonexistent pointee is invalid.

Copying a raw `HeapText*` never copies the pointee and is valid even when the
pointer is null:

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

## 7. Transferring Advanced values

Basic and Advanced are the type categories. **Transfer** is the everyday term
for moving an Advanced value to a destination. The precise term is
**destructive relocation**: the value continues in the destination, and the
inline source becomes inactive. This differs from C++ move construction, which
leaves the source alive. The compiler specification uses “relocation” when
that distinction matters. Direct owner transfer has a separate rule below.

Here `Packet` is Advanced because it contains an owner, even though it declares
neither `clone` nor `drop`:

```huc
struct Packet {
    let i32 tag;
    let Widget& payload;

    fn init(i32 tag, Widget& mod payload)
        : tag(tag),
          payload(payload) {
    }
}

let Packet mod source = Packet(1, new Widget(7));
let Packet destination = source; // destructive relocation
```

Before this binding, `source` contains an active `Packet` and `destination`
does not. Relocation continues the value in `destination` and ends the active
`Packet` lifetime in `source`. It does not create a second live object and it
does not leave a moved-from `Packet`.

For a non-owner Advanced value, relocation transfers its stored representation:

1. Transfer the whole value's representation to the destination.
2. Make the entire source, including every field subobject, inactive.
3. Leave the destination responsible for the value's eventual cleanup.

Owner words inside the aggregate transfer unchanged. In particular, the
operation need not write null to `source.payload`; that field is inactive,
not a second live owner. Neither it nor `source` may be used as a value before
the whole source is reinitialized. No source `drop` body or source-field
cleanup runs. Arrays in the Advanced category use the same whole-value rule.
The operation invokes no user code and cannot fail in the HUC type system.

The rule is the same whether the structure is Advanced because of `clone`, `drop`,
or an Advanced field. It does not change ordinary binding of Basic types, which
duplicates the value and leaves the source active.

A fixed field cannot be assigned to directly, but it can be transferred as
part of a whole value. The containing source must be writable. For example,
transferring a writable structure also transfers its fixed `T&` fields.

The bytes of an inactive source need not be cleared and need not form a valid
representation of `T`. The compiler tracks whether the *storage place* is
active. Assigning a new value to an inactive place begins a new lifetime there.
Using it as a value before reinitialization is undefined behavior.

Direct relocation of a primitive `T&` is the deliberate exception to the
inactive-source rule. Relocating an owner slot transfers its address and
writes null to the source. The source is then an active, usable null owner:

```huc
let Widget& mod first = new Widget(7);
let Widget& second = first;

if (!first) {
    // defined: first is a null owner, not inactive Widget& storage
}
```

This exception also applies when an owner field or element is moved
individually while its containing object stays active. It does not apply to
owner words transferred as part of a whole aggregate relocation: the source
aggregate and all its fields become inactive, with no required null stores.
Null owners remain valid to relocate directly or inside an aggregate.

C++ move construction has a different lifetime model: it constructs a
destination object while the source object remains alive and is destroyed
later. A C++ move constructor must therefore establish a destructible source
state. HUC destructive relocation ends the non-owner source lifetime
immediately, so it needs neither a moved-from-state contract nor a source
destructor call.

Bitwise relocation is the language rule, not an optimization of separate field
transfers. It does not require a memory-copy instruction or give padding bytes
a defined meaning. The backend may copy memory, use loads and stores, keep
values in registers, or remove the transfer entirely. The program must still
behave the same: the source becomes inactive, and cleanup stays correct.

For example, `let T a = b;` for an inline Advanced `T` may keep the value in
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
        // Empty: this opts the scalar-only structure into Advanced.
    }
}

let SelfIndexed mod first = SelfIndexed(10);
let SelfIndexed second = first; // bitwise relocation; first becomes inactive

// second.self still contains the old inline address. Relying on the broken
// invariant is undefined behavior.
```

HUC0 does not automatically find or repair self-pointers. A value that needs
such a pointer to stay correct must instead:

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

`Vector<SelfIndexed>` breaks this requirement when growth relocates
elements. `Vector<SelfIndexed&>` or a stable node container is appropriate.
The same caveat applies to by-value parameters, returns, aggregate fields, and
assignment. HUC is unchecked: obvious cases may be warned about, but the
compiler is not required to prove address independence.

### 7.3 Where a transfer may take its value from

Transferring a non-owner Advanced value requires a whole value whose active
state the compiler tracks directly:

- a named local or parameter with the required trailing `mod`;
- a fresh temporary or function-result place.

Transferring a whole structure or array includes all its fields or elements.
It does not remove them one at a time while leaving the containing value active.

HUC0 rejects explicit non-owner move-out from a pointee or subobject:

```huc
let File value = *file_owner;          // error: indirect Advanced source
let Lease lease = session.lease;       // error: Advanced subobject source
let Packet packet = packets[index];    // error: Advanced element source
```

Otherwise the containing object, owner, or array could remain active even
though one of the values it must later destroy is already inactive. In the
first example, `file_owner` would still be non-null and would later try to
drop an inactive `File`. Tracking that extra state would break the one-word
owner model.

This restriction does not affect:

- Basic values, which are copied rather than consumed;
- `copy *file_owner`, which clones while leaving the pointee active;
- relocation of the whole `file_owner`, which leaves its source owner null;
- relocation of a primitive owner field, because that source subobject remains
  an active null owner;
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
let File mod current = File(probable_os_open("old.dat"));
let File mod replacement = File(probable_os_open("new.dat"));

current = replacement;
```

For distinct active objects, relocation assignment:

1. evaluates the destination location once;
2. evaluates the source location once;
3. checks whether they refer to exactly the same storage;
4. destroys the old active destination;
5. relocates the source into the now-inactive destination.

The source is then inactive, except that a primitive owner source is usable
null. Exact self-relocation assignment is a no-op and does not destroy the
value:

```huc
current = current;
```

Using source and destination storage that overlaps but is not identical is
undefined behavior.
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
destination becomes responsible for later cleanup. `drop` is intended
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

`drop` must not manually destroy owner or Advanced fields that normal field cleanup
will destroy. It may relocate a primitive owner field elsewhere, after which
field cleanup observes a null owner. HUC0's source-place rule forbids
relocating a non-owner Advanced field out of the still-addressable object, even
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
    // copies when Metadata is Basic
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

C17 does not provide the same left-to-right argument guarantee. The backend
must therefore initialize each argument's **parameter value** before it begins
the next argument. For the owner example, generated code is conceptually:

```c
const Document *huc_param_0 = document;
Document *huc_param_1 = document;
document = NULL;
receive_impl(huc_param_0, &huc_param_1);
```

The first declaration completes the owner-to-observer conversion. Binding the
second parameter then transfers the pointer and clears `document`. Merely
saving pointers to source places and postponing both conversions until the
final C call would be incorrect. The pointer to `huc_param_1` transports an
already initialized parameter slot, and the called function's generated code
is responsible for its cleanup. This internal C pointer-to-pointer does not
add a combined pointer type to HUC. The emitter uses typed storage for each
parameter, either separately or in a call frame, and preserves cleanup order.
Optimizers may reorder pure computation when the program still behaves the same.

## 11. Return values

Returning a Basic value follows the normal copy rule:

```huc
fn origin() -> Point {
    let Point point = Point(0, 0);
    return point;
}
```

Returning an Advanced value destructively relocates it:

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
    let collections::Vector<Point> mod points =
        collections::Vector<Point>();
    let Point point = Point(3, 4);

    points.push(point);      // Point is Basic: copies into the vector
    points.push(copy point); // equivalent, but redundant for Basic Point
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

- copies elements when `T` is Basic;
- uses fixed destructive relocation when `T` is Advanced;
- never calls `clone` merely to relocate elements;
- does not drop old element places that relocation made inactive;
- drops every element that remains active exactly once;
- may invalidate raw pointers into its storage without a diagnostic.

The container is responsible for its backing storage and initialized element
range. It must ensure that old slots whose values were transferred receive
no element cleanup. The only additional raw-storage lifetime intrinsics
planned are `std::construct_at` and `std::destruct_at`; their interfaces and detailed
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

std::vector<std::unique_ptr<Widget>> widgets;
auto widget = std::make_unique<Widget>(7);
widgets.push_back(std::move(widget));
```

This is the same container-level value model as C++ RAII, with two policy
changes: HUC makes expensive logical copying explicit and makes relocation of
an Advanced type automatic and non-overridable.

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

- inline values with Basic or Advanced classification;
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
| `let T b = a` for Basic `T` | `T b = a` | Both copy |
| `let T b = a` for Advanced `T` | `T b = std::move(a)` | HUC destructively relocates and ends the non-owner source lifetime |
| `copy a` for Advanced `T` | Copy construction | HUC makes a potentially expensive copy explicit |
| `destination = copy source` | Copy assignment | HUC composes clone plus replace |
| `destination = source` for Advanced `T` | Move assignment | HUC drops the destination, then performs non-overridable relocation |
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
- explicit relocation of a non-owner Advanced value from a pointee, field
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
- any chained pointer/owner form, including one hidden by an alias;
- `addressof` applied to a pointer or owner slot;
- `std::slot_of` applied to a non-place, such as a literal or function symbol;
- attempts to bind `T&` as though it were a C++ reference alias.

Useful warnings include:

- a `clone` that returns or embeds a raw resource address from the source;
- a `drop` that manually destroys an automatically destroyed field;
- obvious non-identical overlapping relocation assignment;
- copying an owner inside a loop where allocation is likely unintended;
- retaining a raw observer across owner relocation or destruction;
- placing a known self-addressing inline type in a relocating container.

Warnings do not turn HUC into a memory-safe language and may be disabled.

## 16. Backend requirements

The HUC0-to-C17 transpiler must preserve these source semantics rather than
inheriting C defaults:

1. `T&` lowers to one typed C pointer; ownership is enforced by HUC analysis,
   not by the C type system.
2. Basic/Advanced classification is decided by HUC semantic analysis.
3. `clone` runs only for an evaluated `copy` that requires it.
4. non-owner relocation transfers the representation, makes the whole source
   inactive, and requires no clearing of embedded owner words; it is
   non-failing and independent of user overloads;
5. direct owner relocation clears the source to a usable null owner, including
   an owner field moved individually;
6. inactive structure values are not dropped;
7. exact self-relocation assignment is a no-op;
8. `drop` runs before reverse field cleanup;
9. arguments and parameter initialization are left-to-right;
10. generated initialization, clone, drop, and relocation functions are
    implementation machinery, not extra HUC customization points;
11. active/inactive state belongs to a storage place and must not add a hidden
    field to nominal `T` or to pointer-sized `T&`;
12. explicit cleanup covers every normal exit and active replacement; no drop
    call may be emitted for an inactive raw-handle source merely because its
    physical C storage leaves scope;
13. non-owner relocation sources must satisfy the root-place restriction, and
    a possibly aliasing indirect destination must be checked before replacement.

The middle-end must make construction, copy, relocation, activation,
deactivation, and cleanup explicit before C emission. C assignment and byte
copying are permitted implementations only when they preserve the complete
HUC operation, including direct-owner nulling and active state, and obey C's
storage and aliasing rules. This keeps the semantics usable by a future C++ or
native-code backend without changing HUC value semantics.

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
relocation rule, direct-owner nulling exception, `mod` requirement, and
unchecked address-dependence contract are HUC decisions.
