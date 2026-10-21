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
  reinitialized before it can be used as a value again. It must not be
  destroyed; section 7.3 explains who prevents that cleanup.

## 1. Values and raw pointers

HUC uses two related type forms:

| HUC type | Meaning | Binding behavior |
|---|---|---|
| `T` | An inline `T` value | Basic values copy; Advanced values transfer |
| `T*` | A nullable, unchecked, non-owning pointer | Copies one address |

HUC has no general reference type that aliases another variable. Raw pointers
never destroy or deallocate their pointees. Resource-managing structures use
the same `copy` method, `drop`, and relocation rules as other Advanced values.

Address-taking uses the non-overloadable unary `&` operator:

```huc
let value: Widget = Widget(7);
let observer: Widget* = &value;
```

Pointer chains such as `T**` are ordinary HUC0 raw-pointer types. Each `*`
adds another non-owning level; no level adds ownership or lifetime tracking.
Unary `&` accepts an addressable place: it returns
`T*` for a fixed place and `mut T*` for a writable place. Applying it to a
pointer slot adds one pointer level, so `&p` where `p` has type `T*` produces
`T**` (or `mut T**` for a writable slot). Its operand is evaluated once
without transferring a value, creating a temporary, extending storage
lifetime, or changing cleanup state. Binary `&` remains bitwise AND.

`&pointer` addresses the pointer slot, not its pointee. Ordinary functions
can receive this typed pointer directly. If an integer address is needed,
use `ptr_as<usize>(&place)`; convert it back with `ptr_as<T*>(address)`, using
the appropriate pointer type. The cast evaluates its operand once and
does not transfer a value, extend a lifetime, or change cleanup state.

The integer carries no access permissions or lifetime metadata. The programmer
must respect alignment, access type, valid representation, and cleanup.
Writing actually fixed storage remains undefined behavior after a cast. Byte
access through `u8*` or `c8*` performs no lifetime or cleanup operations.

`&` still requires a named Advanced root to be tracked Active, even when its
result is cast to an integer. Manual-storage code can retain a pointer before
transferring the value or use byte-array storage; taking a pointer does not keep
the old value alive. Section 7.5 defines the checked-use boundary.

Declarations use `name: Type`. `mut` before the name controls the slot;
`mut` inside a pointer type controls the pointee:

```huc
let observer: T*; // fixed field; constructor obligation
let observer: mut T*; // fixed field; writable T; constructor obligation
let mut observer: T*; // reseatable field; defaults to null
let mut observer: mut T*; // reseatable field; writable T; defaults to null
```

These are field forms. A fixed local requires an explicit initializer. Each
constructor must initialize fixed fields that lack declaration initializers.
Omitted mutable pointer fields default to null. HUC0 globals are restricted
to drop-free Basic scalars and raw pointers.

A fixed structure prevents assignment to its inline fields. A `mut T*`
field still allows mutation of the separate pointee, but not replacement of
the pointer stored in that field. Each layer keeps its own permissions.

Relocating a named Advanced value or assigning to a named slot requires
binding `mut`. Compiler-inserted destruction does not. Fresh unnamed results
need no binding `mut`: there is no source declaration on which to write it.

```huc
let file: File = open_file("input.dat");
consume(make_widget(7));
```

Direct constructor initialization, described in section 4, creates no
intermediate source place. Copying a raw pointer leaves it unchanged, even
when null, and never transfers or extends the pointee's lifetime.

## 2. Basic and Advanced types

Every complete runtime type is either **Basic** or **Advanced**.

The short rule is: **Basic values are copied; Advanced values are transferred.**
Ordinary binding leaves a Basic source usable. Transferring an inline Advanced
value makes its source inactive. Explicit `copy` requests a separate logical value
when the type supports it.

Basic types are:

- scalar arithmetic and boolean types;
- raw pointers;
- arrays of Basic elements;
- structures whose fields are all Basic, which declare neither a `copy` method
  nor `drop`, and which are marked neither `[nocopy]` nor `[norelocate]`.

Advanced types are:

- arrays of Advanced elements;
- structures containing at least one Advanced field;
- structures declaring a `copy` method or `drop`;
- structures marked `[nocopy]` or `[norelocate]`.

Having `init()` alone does not make a structure Advanced. A raw `T*` is Basic
even when `T` is Advanced; it copies only the observer address. In contrast,
an inline Advanced field makes its containing structure Advanced too.
An array is Basic exactly when its element type is Basic. A byte array is
therefore Basic, even when used as manual backing storage; a container must
declare `drop` or otherwise be Advanced to get destructive whole-value transfer.

Declaring a `copy` method intentionally makes an otherwise Basic structure
Advanced.
Otherwise ordinary binding could silently bypass the custom copy behavior:

```huc
struct Ticket {
    let id: u64;

    fn init(&mut self, id: u64) : id = id {
    }

    fn copy(&self) -> Ticket {
        return Ticket(probable_issue_related_ticket(self.id));
    }
}

let mut original: Ticket = Ticket(100);
let moved: Ticket = original; // destructive relocation; copy is not called
let copied: Ticket = copy moved; // calls Ticket's copy method
```

Basic and Advanced are category names, not keywords or declaration modifiers.
The compiler determines the category from the rules above.

A token containing only scalars can use `[nocopy]` to become Advanced when its
author wants to prevent copying:

```huc
struct [nocopy] UniqueTicket {
    let value: u64;

    fn init(&mut self, value: u64) : value = value {
    }
}
```

`UniqueTicket` can be relocated but cannot be copied (section 6.5). The
attribute is not a type-category keyword; the category still follows from the
rules above.

## 3. Special operations

HUC lets a type define three lifecycle methods:

```huc
fn init(&mut self, ...) : field = initializer, ... {
    // construction body
}

fn copy(&self) -> T {
    // return an independent logical duplicate
}

fn drop(&mut self) -> void {
    // release non-field resources
}
```

Their roles are:

| Operation | User customization | Invocation |
|---|---|---|
| Construction | `fn init(...)` | `T(...)` |
| Fieldwise copy of a Basic type | None | Ordinary binding or `copy` |
| Logical copy of an Advanced structure | `fn copy(&self) -> T`, or a generated one | Only requested by `copy` |
| Logical copy of an Advanced array | None | `copy` applies logical copy to each element |
| Destructive relocation initialization | None | Ordinary binding of an Advanced value |
| Destructive relocation assignment | None | Ordinary assignment of an Advanced value |
| Destruction | `fn drop(&mut self) -> void` | Inserted by the compiler, or `drop name;` for a local |
| Field destruction | None | Inserted after `drop`, in reverse field order |
| Ending a local without cleanup | None | `forget name;` |

Two attributes switch off default operations: `[nocopy]` forbids `copy`
(section 6.5), and `[norelocate]` forbids relocation (section 7.2).

This is deliberately smaller than C++'s special-member system. In particular,
there is no user-defined move constructor, relocation constructor, or
move-assignment operator.

## 4. Construction

A constructor is a runtime method named `init`:

```huc
struct Point {
    let x: i32;
    let y: i32;

    fn init(&mut self, x: i32, y: i32)
        : x = x,
          y = y,
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
- a function result in `return T(arguments)`;
- the supplied slot in `construct_at(pointer, T(arguments))`;
- the replacement slot after Advanced assignment destroys its old value.

HUC guarantees this; it is not an optional optimization. `init` sees the final
`this` address, and constructing a fixed Advanced value needs no writable
temporary first. Direct calls, Advanced copies, array literals, and selected
conditional arms also use their final result slots (section 8.2). Existing
Advanced values still follow the normal transfer rules.

Constructor initializer entries use `field = expression`. The left name is a
field; the expression uses ordinary lookup. Each entry initializes its field
as an ordinary `let` initializer would, using that field's storage. Entries:

1. must follow field declaration order;
2. are evaluated left-to-right;
3. initialize each field before the constructor body executes.

Every fixed field requires a declaration initializer or a constructor entry.
An omitted mutable scalar becomes zero, an omitted mutable pointer
becomes null, and an omitted mutable structure is default-initialized. Arrays
default-initialize their elements in increasing index order. `[nozero]` skips
omitted writes for scalars and scalar arrays; explicit initializers still run.

With no written `init`, a structure gets a generated memberwise init, with one
parameter per named field lacking both a declaration initializer and
`[nozero]`. Parameters have the field names and types in declaration order;
Advanced parameters are mutable bindings. It also gets a default no-argument
init when every named field has an initializer or is mutable and supports
default initialization. Identical zero-argument forms are one function. Any
written `init` suppresses both generated forms. Fixed-field initialization
obligations still apply. `is_default_initializable<T>()` reports whether a
type can be default-initialized; section 12 uses it in generic code. See
[the full construction rules](language-specification.md#46-structures).

`init` does not double as a copy or relocation hook. An overload accepting
another `T` is just an explicitly selected constructor:

```huc
struct ScaledPoint {
    let x: i32;
    let y: i32;

    fn init(&mut self, source: Point, scale: i32)
        : x = source.x * scale,
          y = source.y * scale,
    {
    }
}
```

## 5. Copying Basic values

A structure made entirely from Basic fields, with no `copy` method, no `drop`,
and neither `[nocopy]` nor `[norelocate]`, is copied by value, including its
representation bytes:

```huc
struct Point {
    let x: i32;
    let y: i32;

    fn init(&mut self, x: i32, y: i32)
        : x = x,
          y = y,
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
has no `copy` or `drop` method and no Advanced field, ordinary copying does
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
`mut`.

Writable Basic assignment is also direct:

```huc
let mut current: Point = Point(1, 2);
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

## 6. Custom logical copying with a `copy` method

An Advanced structure is not implicitly copied. It may support explicit logical
copying by declaring exactly one method with this signature, or by getting a
generated one (section 6.4):

```huc
fn copy(&self) -> T;
```

Rules:

- it is an ordinary runtime `fn`, without an angle header;
- it has no ordinary parameters besides its explicit receiver;
- its receiver is explicitly read-only (`&self`);
- it returns the same structure type in which it is declared;
- it must not modify the source object's own storage or leave the source invalid;
- it cannot be called directly or have its address taken;
- ordinary binding, assignment, argument passing, and return never call it;
- `copy source` calls it when `source` is an Advanced structure;
- declaring it makes the structure Advanced even when every field is Basic;
- mark the type `[nocopy]` when it must not be copyable (section 6.5);
- it cannot return a recoverable error, although allocation failure or panic
  may terminate the process.

An “independent logical duplicate” means that the original and its copy can
each be destroyed without invalidating the other. To achieve this, the `copy`
method must duplicate owned resources or create another valid ownership
share, for example by increasing a library-managed reference count.

Deep copying in HUC does **not** mean copying every reachable object. A `T*`
is an observer, so copying a raw-pointer field normally copies only its
address. Shared external state, interned data, immutable tables, and other
observed objects may remain shared. The type author decides what belongs to
the value and writes the necessary `copy` operations inside its `copy`
method. The compiler cannot prove that the method follows these rules.

Allocation and updates to external bookkeeping are allowed. For example, a
future HUC1 `Rc<T>` could implement `copy rc` by increasing a shared reference
count through an explicitly writable observer. The source handle's own
storage must remain unchanged, and it must still refer to the same resource.

A type that needs to report a copying failure should provide an ordinary
function such as `try_copy`, returning a future HUC1 `Result<T, E>`
specialization.
That function is not selected by the `copy` expression.

The initial language has one logical copy operation rather than separate
custom copy-construction and copy-assignment hooks. Advanced copy assignment
destroys the old destination, then builds the copy directly in its slot:

```huc
let mut destination: Text = Text("old");
let source: Text = Text("new");

destination = copy source;
```

Conceptually:

```text
retain(source)
destroy(destination)
CopyInto(destination, source)
```

These are abstract compiler operations, not callable HUC functions. The source
and everything the copy needs must survive destination destruction. Thus
Advanced `value = copy value` is undefined behavior, even when written directly.
To deliberately replace a value with its own copy, prepare the copy first:

```huc
let mut replacement: Text = copy value;
value = replacement;
```

Direct construction in `let b: T = copy a` likewise uses `b` as the copy's
result slot. A `copy` method that returns `T(...)`, another call, or
`copy source` forwards that slot; no inlining is required. Returning an
explicit local still relocates it. Section 8 gives the complete assignment
order.

### 6.1 Owning text example

The memory APIs below are probable standard-library APIs:

```huc
module examples.heap_text;

import std.memory as memory;
import std.text as text;

struct HeapText {
    let mut data: mut u8*;
    let size: usize;

    fn init(&mut self, source: text::View)
        : data = memory::allocate_raw_bytes(source.size()),
          size = source.size(),
    {
        memory::copy_bytes(self.data, source.data(), self.size);
    }

    fn view(&self) -> text::View {
        return text::View(self.data, self.size);
    }

    fn copy(&self) -> HeapText {
        return HeapText(self.view());
    }

    fn drop(&mut self) -> void {
        if (self.data) {
            memory::free_raw_bytes(self.data, self.size);
            self.data = null;
        }
    }
}

fn demonstrate() -> void {
    let mut first: HeapText = HeapText("alpha");
    let second: HeapText = copy first; // independent allocation
    let third: HeapText = first; // relocate; first becomes inactive
}
```

A C++20 counterpart that manages the allocation directly is:

```cpp
#include <cstddef>
#include <cstdint>
#include <cstring>
#include <string_view>
#include <utility>

class HeapText {
public:
    explicit HeapText(std::string_view source)
        : data_(new std::uint8_t[source.size()]),
          size_(source.size()) {
        if (size_ != 0) {
            std::memcpy(data_, source.data(), size_);
        }
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
          size_(std::exchange(source.size_, 0)) {}

    HeapText& operator=(HeapText&& source) noexcept {
        if (this != &source) {
            delete[] data_;
            data_ = std::exchange(source.data_, nullptr);
            size_ = std::exchange(source.size_, 0);
        }
        return *this;
    }

    ~HeapText() {
        delete[] data_;
    }

    std::string_view view() const {
        if (size_ == 0) {
            return {};
        }
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

This direct-allocation C++ example writes five special members and leaves a
moved-from object empty and copyable. The HUC version writes `copy` and `drop`;
HUC supplies fixed destructive relocation. This is not a requirement for every
C++ class: composing standard resource types, as in the `Image` example,
often removes custom cleanup and move code. A `std::string` member could
provide this text wrapper's resource management and copying with no custom
special members. A `unique_ptr` member supplies cleanup and ownership transfer,
but a copyable wrapper still needs to define how to duplicate its resource.
The first HUC backend emits LLVM IR with explicit lifetime operations; this
source comparison does not describe the generated backend code.

The C++ copy assignment above copies before releasing the old allocation and
explicitly guards self-assignment. HUC instead releases the old value before
copying; self-assignment is unchecked undefined behavior. The examples share
the ownership intent, not identical assignment ordering or alias guarantees.

### 6.2 Copying through a raw pointer

Copying a raw pointer never copies its pointee and is valid even when null:

```huc
let text: HeapText = HeapText("hello");
let observer: HeapText* = &text;
let another_observer: HeapText* = copy observer; // copies only the address
let duplicate: HeapText = copy *observer; // calls HeapText's copy method
```

The dereferenced form requires a live, accessible pointee that supports
logical copying. It creates an independent value and leaves the pointee
unchanged; it does not allocate storage for a separate wrapper object.

### 6.3 Specialized copy assignment

HUC's automatic Advanced `destination = copy source` destroys, then copies
into the destination. This does not reuse the old resource or buffer.
When an application needs buffer reuse or another specialized algorithm, it
uses a clearly named writable method:

```huc
struct ReusableBuffer {
    // fields, init, copy, and drop omitted

    fn assign_from(&mut self, source: ReusableBuffer*) -> void {
        probable_resize_in_place(&self, source->size());
        probable_copy_payload(&self, source);
    }
}

let mut buffer: ReusableBuffer = ReusableBuffer(...);
let other: ReusableBuffer = ReusableBuffer(...);
buffer.assign_from(&other);
```

`assign_from` is not an implicit copy-assignment hook. The call site explicitly
selects the specialized operation.

### 6.4 Generated copy methods

A structure that is Advanced only because of its fields usually needs no
written `copy` method. The compiler generates one when the structure:

1. is Advanced independently of the generated method;
2. declares neither a `copy` method nor `drop`;
3. is not marked `[nocopy]`, and not explicitly marked `[norelocate]`;
4. has only fields that support copying.

```huc
struct Packet {
    let tag: i32;
    let payload: HeapText;
}

let packet: Packet = Packet(7, HeapText("hello"));
let duplicate: Packet = copy packet; // generated copy method
```

The generated method copies the named fields in declaration order, each
directly into the matching destination field. It runs no constructor or
declaration initializer. A Basic structure stays Basic. A written `drop`
blocks generation, because copying the fields would duplicate whatever `drop`
releases.

Having no `drop` does not prove that copying the fields is correct:

```huc
struct TextAndView {
    let text: HeapText;
    let view: c8*; // intended to point into text
}
```

Relocation is fine, because moving the owning handle doesn't move its heap
buffer. A generated `copy` method, however, copies `text` into a new buffer
and copies `view` unchanged, so the duplicate's view still points into the
original. Such a type needs a written `copy` method or `[nocopy]`. The type
author remains responsible for relationships between fields. Rust instead
requires an explicit `#[derive(Clone)]`; HUC deliberately chooses a more
automatic default.

### 6.5 Opting out with `[nocopy]`

A field that doesn't support copying already prevents generation. To make a
type move-only on purpose, mark it `[nocopy]`:

```huc
struct [nocopy] UniquePacket {
    let payload: HeapText;
}
```

The type is Advanced, even when all its fields are Basic, so ordinary binding
moves it. No `copy` method is generated, a written one is an error, and a
`copy` expression on it is rejected. A structure with a `[nocopy]` field gets
no generated `copy` method either. Adding or removing copy support changes a
public type's copying contract, as changing its category does.
`is_copyable<T>()` reports whether a type supports copying; section 12 uses
it in a generic container.

## 7. Transferring Advanced values

Basic and Advanced are the type categories. **Transfer** is the everyday term
for moving an Advanced value to a destination. The precise term is
**destructive relocation**: the value continues in the destination, and the
inline source becomes inactive. This differs from C++ move construction, which
leaves the source alive. The compiler specification uses “relocation” when
that distinction matters.

Here `Packet` is Advanced because it contains an Advanced `HeapText`, even
though it declares neither a `copy` method nor `drop`:

```huc
struct Packet {
    let tag: i32;
    let payload: HeapText;

    fn init(&mut self, tag: i32, mut payload: HeapText)
        : tag = tag,
          payload = payload,
    {
    }
}

let mut source: Packet = Packet(1, HeapText("payload"));
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
cleanup runs. Advanced fixed arrays use the same whole-value relocation rule.
The operation invokes no user code and cannot fail in the HUC type system.

The rule is the same whether the structure is Advanced because of a `copy`
method, `drop`, `[nocopy]`, or an Advanced field. It does not change ordinary
binding of Basic types, which duplicates the value and leaves the source
active.

A fixed field cannot be assigned to directly, but it can be transferred as
part of a whole value. The containing source must be writable. For example,
transferring a writable structure also transfers its fixed Advanced fields.

The bytes of an inactive source need not be cleared and need not form a valid
representation of `T`. For directly named locals and parameters, the compiler
tracks whether the *storage place* is active. Assigning a new value to a place
tracked as inactive begins a new lifetime there without dropping an old value.
Using a place whose tracked state is Inactive or MaybeActive is a
compile-time error (section 7.5). Pointer extraction does not update an
aliased local's tracked state and leaves source cleanup to the programmer
(section 7.3). Accessing inactive storage outside this check is undefined
behavior, whether through a pointer or a name whose tracked state is stale.

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

This is HUC0's chosen default. Types that must not move are marked
`[norelocate]` instead (section 7.2). A future proposal may allow custom
relocation if real programs show that repairing addresses during a move is
needed often enough to justify more complex type rules, generic code, and
failure handling. That would need a separate feature, not another form of
`init`.

### 7.2 Address-dependent inline values

Fixed relocation has one important limit: it cannot repair values that depend
on the object's current address. A stored pointer to the object itself is one
example. Mark such a type `[norelocate]`:

```huc
struct [norelocate] SelfIndexed {
    let mut payload: i32;
    let mut self_pointer: SelfIndexed*;

    fn init(&mut self, payload: i32)
        : payload = payload,
          self_pointer = null,
    {
        self.self_pointer = &self;
    }

    fn has_expected_address(&self) -> bool {
        return self.self_pointer == &self;
    }
}

let mut first: SelfIndexed = SelfIndexed(10); // built in first
let second: SelfIndexed = first; // error: SelfIndexed is never relocated
```

A `[norelocate]` type is Advanced, even when all its fields are Basic, so it is
never copied implicitly either. Every relocation is a compile-time error:
binding an existing value, passing it by value, returning a local, extracting
it through a pointer, `take`, or relocating a structure that holds it inline.
Direct results still work, because they build in their final slot:
constructors, factories that return `T(...)` or another direct call, array
literals, and the selected arm of `?:`, even when the conditional's temporary
is only used for a member access.

```huc
fn make_indexed(payload: i32) -> SelfIndexed {
    return SelfIndexed(payload);
}

let a: SelfIndexed = make_indexed(1); // built in a
let pair: SelfIndexed[2] = [SelfIndexed(2), make_indexed(3)];
let n: i32 = (ready ? SelfIndexed(4) : make_indexed(5)).payload;
```

Assignment replaces in place: `first = SelfIndexed(6)` destroys the old value
and builds the new one in the same storage. `drop` and `forget` end a local
without moving it (section 9.4). A structure that holds a `[norelocate]` field
inline is non-relocatable too; a pointer to one is not. A generated memberwise
init that would relocate such a field is not generated, so the owner writes an
init that constructs the field directly.

An explicitly `[norelocate]` type gets no generated `copy` method, because
copying its fields would copy its self-pointer. It may declare one that builds
a correct new object, or add `[nocopy]` to forbid copying too:

```huc
struct [norelocate, nocopy] Mutex {
    let mut native: pthread_mutex_t; // used by address; never moved or copied
}
```

`Vector<SelfIndexed>` cannot grow by relocating elements; use stable-address
storage instead. A library can test `is_relocatable<T>()` to report this
clearly. A byte array hides its payloads from the type system, so relocating
an enclosing object that carries a live non-relocatable payload is undefined
behavior (section 9.3).

Without the attribute, HUC does not find or repair self-pointers. Relocating
such a value leaves its stored address stale, and relying on it is undefined
behavior.

### 7.3 Where a transfer may take its value from

An Advanced value can be transferred from these sources:

| Source | Who prevents later cleanup of the old value? |
|---|---|
| A whole local or parameter with binding `mut`, or a fresh temporary/result | The compiler updates that place's active state. |
| `*pointer` through `mut T*` | The programmer manages cleanup of the old slot. |
| Writable `value.field`, `p->field`, or `items[i]` | The programmer, exactly as for `*(&place)`; enclosing cleanup is unchanged. |

All these forms transfer the whole value, including its fields. The source
becomes inactive, the destination takes responsibility for cleanup, and no source
destructor, nulling, or replacement initialization occurs. The old bytes are
not a usable second value. The pointer expression is evaluated once; extraction
does not change the pointer's address value. Its binding can be fixed, but its
pointee must be writable, live, and correctly aligned.

```huc
// file_pointer has type mut File* and points to a live, manually managed File.
let value: File = *file_pointer; // allowed; the old slot is now inactive
// The storage manager must exclude that slot from cleanup.
```

A raw pointer does not carry the pointee's automatic cleanup state. Extraction
does not cancel cleanup of an aliased local or enclosing object, even when the
pointer was obtained with `&`. For example:

```huc
fn extract(file_pointer: mut File*) -> File {
    return *file_pointer;
}

fn incorrect() -> void {
    let mut file: File = File(probable_os_open("input.dat"));
    let value: File = extract(&file);
    // Incorrect: file's automatic cleanup has not been cancelled.
} // Reaching cleanup of the inactive file is undefined behavior.
```

The correct direct-local form is `let value: File = file;`: it lets the
compiler suppress `file`'s cleanup. Pointer extraction is useful when code
manages its own live slots, such as a vector that reduces its initialized
length after extracting the last element. This distinction is about cleanup
responsibility, not stack versus heap allocation. No per-element flags or
runtime ownership lookup are required.

The programmer must recognize this operation in every value-binding context:
`return *p`, `consume(*p)`, and `construct_at(q, *p)` extract an Advanced
pointee just as `let result: T = *p` does. A function taking `mut T*` can do
this, but must document the resulting lifetime obligation. For ordinary
in-place changes, call a method through the pointer. For an independent value,
use `copy *p` if cloning is supported. To consume an automatic local, prefer
passing it by value, so its compiler-managed cleanup state follows the transfer.
If a pointer-taking function temporarily extracts an automatic value, it must
reconstruct a live value with `construct_at` before the caller uses or destroys
that slot. Merely avoiding further reads in the caller does not prevent the
invalid automatic destruction.

Direct field and element moves use the same unchecked extraction rules:

```huc
let lease: Lease = session.lease; // writable field becomes inactive
let other: Lease = session_pointer->lease; // needs writable field access
let packet: Packet = packets[index]; // needs mut Packet*
```

Each selected place must be writable, as for `*(&place)`. An enclosing
`mut` root does not make a fixed field writable. Named containing roots still
need to be Active, but HUC does not track partly inactive objects. The enclosing
object must not later destroy the extracted slot. Refill it with
`construct_at`, or exclude it from cleanup, for example by reducing a
container's live length.

Ordinary assignment cannot refill an extracted Advanced field or element:
it would destroy the inactive value first. Relocating or copying an enclosing
inline object before restoring its field, or letting a method or `drop` read
that field, is also undefined behavior. These lifetime mistakes are not
required errors, even for direct field paths. Implementations may warn, but
must not reject otherwise well-typed runtime code for them. Type, permission,
and whole-root checks remain. No per-field flags or alias checks are added.
Unused byte-array capacity is different: it may accompany relocation of an
enclosing Advanced value under section 9.3.

Basic pointees are still copied, not consumed. `copy *file_pointer` copies a
live, copyable pointee and leaves it active; it does not require pointee `mut`.
Any eligible source may replace a valid indirect destination. The
source must be distinct, non-overlapping, and survive destination destruction.
Same-storage Advanced assignment is undefined behavior; the backend adds no
implicit address comparison or no-op path.

#### Taking a value and leaving a live source

Use `take place` when an Advanced slot should remain live:

```huc
let body: Buffer = take request.body; // leaves Buffer() in the writable field
let item: Buffer = take items[index]; // leaves Buffer() in that element
```

Evaluate the place once, relocate the old value out, construct `T()` directly
in the old slot, then yield the extracted result. The replacement constructor
observes the source slot's own address. This needs writable access, a live
source, and a valid zero-argument constructor. No source `drop` runs during
the extraction; normal cleanup later destroys the replacement.

For Basic values, `take` only copies and leaves the source unchanged. No
default constructor or writable access is needed, and a raw pointer is not
set to null. Reading a Basic bit-field or under-aligned packed field works
without taking its address. The operand must still be a place.

An Advanced `take` is fully evaluated before assignment destroys its old
destination. In `current = take spare`, `spare` already holds its replacement
when `current` is destroyed. Even `x = take x` can be valid: it destroys the
replacement and puts the extracted value back. Ordinary Advanced `x = x`
remains undefined. A discarded extracted result is cleaned up at the end of
its full expression.

`take` does not track fields or preserve every enclosing-object invariant.
The type's `T()` must be a suitable replacement; if not, use direct extraction
and `construct_at` with an explicit value. If the constructor panics, the
program terminates without rollback. See specification section 6.6.1 and
[the standalone example](../examples/take-and-extraction.huc0).

### 7.4 Native handle example

```huc
module examples.file_value;

struct File {
    let mut handle: i32;

    fn init(&mut self, handle: i32)
        : handle = handle {
    }

    fn drop(&mut self) -> void {
        if (self.handle >= 0) {
            probable_os_close(self.handle);
            self.handle = -1;
        }
    }
}

fn transfer() -> void {
    let mut input: File = File(probable_os_open("input.dat"));
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
flag if needed; it must not add a hidden field to `File`. That flag only
controls cleanup: section 7.5 rejects any use of the value in the uncertain
state.

### 7.5 Checked use after transfer

The rule is simple: **a local must be Active before it is used**.

For every Advanced local and by-value parameter, the compiler tracks whether
the value is live on all paths (Active), has been transferred on all paths
(Inactive), or differs between paths (MaybeActive). It already needs this
information to arrange cleanup. A use in either of the last two states is a
compile-time error in every compilation mode, not an optional warning.

The check follows direct transfers, not changes through arbitrary pointer
aliases. It also checks operations inside conditions without predicting
their true/false results. This keeps the rule conservative: some rejected
programs would never take the invalid path at runtime.

A use is anything that reads or observes the value: relocating it, `copy`,
a method call, field selection, or `&` applied to it or one of its fields.
Storing into a field checks its containing root before and after RHS
preparation, so `x.count = consume_and_count(x)` is rejected. Assigning a whole
new value does not use the old value. It makes
the place Active again, with a drop flag if needed to decide whether the old
value needs destruction. Uses on the right-hand side are still checked.
Scope-exit cleanup does not read an inactive value either; it skips that value.
`ptr_as<usize>(&value)` still performs the checked `&value` operation and is
rejected when the named root is Inactive or MaybeActive.

A source relocated into an indirect destination becomes Inactive; possible
self-aliasing does not add a defined no-op path. Assignment records source
transfer before destination activation. Consequently `x = x` ends with the
tracked root Active, but its execution is still undefined for Advanced `x`.
Flow bookkeeping is not a guarantee that unchecked lifetime rules were met.

In these examples, `Job` and `Document` are Advanced types, for example
because they declare `drop`:

```huc
fn submit_if(ready: bool) -> void {
    let mut job: Job = Job(7);

    if (ready) {
        submit(job); // job is Inactive on this path
    }

    inspect(&job); // error: job is MaybeActive
}
```

The analysis considers every branch possible, including branches whose
conditions happen to be correlated. Code that needs the value afterward must
make that visible to the compiler:

```huc
fn submit_if(ready: bool) -> void {
    let mut job: Job = Job(7);

    if (ready) {
        submit(job);
        return; // this path ends; job needs no cleanup
    }

    inspect(&job); // valid: Active on every path that reaches here
}
```

The same rule covers loops. `submit(job)` inside a `while` body is rejected
if `job` was declared outside the loop and a path can reach the next
iteration without reinitializing it. A `break` or `return` ends that path
instead of returning to the loop head.

The analysis follows HUC's left-to-right evaluation order within a call as
well, with the receiver before the arguments. This makes these calls
ill-formed:

```huc
merge(document, document.size()); // error: document already relocated
compare_versions(document, copy document); // error: same reason
document.absorb(document); // error: relocates the call's own receiver
attach(&document, document); // error: earlier argument points at document
```

For a borrowed absorb receiver, that observer or the pointer argument would refer to storage
whose value has already moved. The same holds when the transfer happens
inside a nested argument, as in `document.absorb(wrap(document))`.

These additional checks recognize a limited set of receiver and pointer
argument forms; they do not follow pointers through arbitrary functions.
For example, `merge(length_of(&document), document)` is allowed when
`length_of` returns a number: the observation finishes before the transfer.
By contrast, `attach(identity(&document), document)` escapes this check but
is invalid if `attach` accesses the inactive document through that pointer.

To fix the first example, obtain the size before transferring the document
and pass that saved number to `merge`. For `compare_versions`, copy the
first argument before transferring the original into the second.

The check does not prove pointer lifetimes. An existing raw observer is not
updated when a value moves. Extraction through `mut T*` also leaves an
aliased local's tracked state unchanged, so a later invalid use by name can
escape the check as well (section 7.3). See
[language specification section 6.12](language-specification.md#612-checked-use-of-relocated-roots)
for the complete rule.

The required call checks follow relocations already tracked during argument
binding. They are not extended to follow dependencies through assignment's
later destruction. That would be new analysis. For example,
`document = make_replacement(&document)` is accepted but must not use the old
document after it is destroyed. Section 8 explains this unchecked boundary.

## 8. Destroy-first assignment

A writable destination can be replaced by destructively relocating another
value:

```huc
let mut current: File = File(probable_os_open("old.dat"));
let mut replacement: File = File(probable_os_open("new.dat"));

current = replacement;
```

All Advanced `=` assignments use one order:

1. Evaluate the destination location once.
2. Prepare the RHS inputs in left-to-right order.
3. Destroy the old destination if it is still active after preparation.
4. Construct, call, copy, or relocate into that slot.
5. Mark the destination Active and finish normal temporary cleanup.

What preparation does depends on the outer RHS operation:

| RHS | Before destruction | After destruction |
|---|---|---|
| `T(arguments)` | Bind arguments | Construct in the slot |
| `make(arguments)` | Retain callee/receiver and bind arguments | Run call into slot |
| `copy source` | Retain or evaluate source; delay the copy itself | Copy into slot |
| Existing eligible Advanced place | Retain source location | Relocate into slot |
| Array literal | Prepare every element's inputs left to right | Build elements in increasing index order |
| `condition ? yes : no` | Evaluate condition once, then prepare only the selected arm by these rules | Finish that arm directly in the slot |
| Other expression | Evaluate the whole value | Relocate the result |

Parentheses do not change the selected form. This does not delay every nested
operation: `current = identity(copy other)` copies before destroying `current`,
then runs `identity`. A conditional forwards the destination to its selected
arm recursively; an outer copy or factory in that arm runs after destruction.
For `current = copy make_source()`, the factory builds a separate source that
stays alive through the delayed copy. Retaining a source place is not a
snapshot: legal changes made by the destination's destructor are visible to
the later operation.

Basic assignment stays read-then-store, including self-assignment. Compound
assignments are unchanged. Advanced assignment has type `void` and cannot be
overloaded. It offers no rollback if the delayed operation terminates.

### 8.1 Self-assignment and dependent inputs

Same-storage and overlapping Advanced assignment are undefined behavior:

```huc
current = current;        // Undefined behavior, not a no-op.
current = copy current;   // Undefined behavior, not a safe self-copy.
*destination = *source;   // Same rule if the Advanced slots alias.
```

The compiler adds no implicit address test or copy-first fallback. Ordinary
typing, permission, and moved-value checks still apply. Implementations may
warn about obvious self-aliasing, overlap, or dependencies, but these are not
required errors. Optional sanitizers do not change the language contract.
Undefined behavior need not produce a runtime error or crash.

Even different object addresses can be unsafe. The delayed operation must not
need a resource destroyed with the old destination:

```huc
text = Text(text.size()); // Scalar observation finishes before destruction.
text = Text(text.view()); // Undefined if construction reads freed characters.
tree = copy *tree.left;   // Undefined if tree's destruction destroys left.
text = text.rebuilt();    // The delayed receiver's old lifetime has ended.
```

No path or wrapper analysis is required. If `view()` returns an independently
owned value instead, using it can be valid. Passing a pointer, retaining a
view, or wrapping it in another function does not itself keep storage alive.
The retained destination address must also stay valid: RHS preparation must
not reallocate a container holding the destination element.

When the replacement needs the old value, complete it in a separate statement:

```huc
let mut replacement: Text = make_replacement(&text);
text = replacement;
```

Libraries that permit equal source and destination slots must handle them
explicitly. For example, a library can choose to guard a pointer-based copy
with `if (destination != source) { *destination = copy *source; }`. This is
the library's policy, not a hidden exception to `copy`.

### 8.2 Tracked state and result placement

Assignment to a destination the compiler tracks as inactive skips destination
destruction and begins a new active lifetime:

```huc
let mut first: File = File(probable_os_open("a.dat"));
let mut second: File = first; // destructive relocation; first inactive

first = File(probable_os_open("b.dat")); // first active again
```

Assignment is also allowed when the destination is MaybeActive. The compiler
uses a drop flag to destroy the old value only on paths where it is still
live. The destination is Active afterward.

In `x = wrap(x)`, binding the parameter transfers the old `x` before destruction,
so there is nothing left to drop. A conditional that transfers `x` also uses
its post-preparation state. Existing containing-root checks still reject
`packet.field = make_field(packet)` and `*(&x) = wrap(x)`.

A direct constructor, call, Advanced `copy`, array literal, or selected `?:`
result uses its final destination for initialization as well as assignment.
`return T(...)`, `return make()`, `return copy source`, and conditional direct
results forward that slot through nested calls. The final
constructor address is guaranteed even at `-O0` and without inlining. Returning
an explicit Advanced local still relocates it; named-return optimization is
not guaranteed. A named Advanced arm still relocates; the conditional is a
value, not a place. For example, `x = c ? copy x : copy other` has undefined
behavior when `c` selects the copy of the destroyed `x`.

The tracked-state reinitialization rule does not apply to a slot made
inactive through pointer extraction.
Extraction did not update any aliased local's cleanup state. Ordinary
assignment through a pointer, or through a local still tracked as active,
would try to destroy the old value. Use `construct_at(pointer, value)` to
initialize inactive manual storage without old-value destruction (section 9.2).
The new value must be live before any
pending automatic cleanup reaches the slot.

## 9. Destruction with `drop`

A structure may declare at most one destructor body:

```huc
fn drop(&mut self) -> void;
```

It must:

- be an unnumbered runtime method;
- have no ordinary parameters besides its explicit receiver;
- return exactly `void`;
- have a writable receiver;
- not be overloaded.

`drop` cannot be called directly or have its address taken. The compiler calls
it once when an active value is **destroyed**. Destructive relocation is not
destruction: it ends the source lifetime without calling source `drop`, and the
destination becomes responsible for later cleanup. `drop` is intended
to release resources that are not already represented by automatically
destroyed fields. The `drop name;` statement destroys a local early
(section 9.4).

All HUC0 structure members are public. Compiler-inserted destruction applies
wherever destruction of an active value is required.

After `drop` returns, fields are destroyed in reverse declaration order.

```huc
struct Session {
    let log: Logger;
    let socket: Socket;
    let mut native_transaction: i64;

    fn init(&mut self, mut log: Logger, mut socket: Socket, native_transaction: i64)
        : log = log,
          socket = socket,
          native_transaction = native_transaction,
    {
    }

    fn drop(&mut self) -> void {
        if (self.native_transaction != 0) {
            probable_abort_transaction(self.native_transaction);
            self.native_transaction = 0;
        }
    }
}
```

Destruction order is:

1. `Session.drop`;
2. destruction of the inline `socket` field;
3. destruction of the inline `log` field.

`drop` must not manually destroy Advanced fields that normal field cleanup
will destroy. Direct extraction from a writable field is allowed, including
inside `drop`, but it does not suppress subsequent automatic field cleanup.
Leaving that field inactive when `drop` returns is undefined behavior. Refill
it with `construct_at`, or use `take` when its default replacement is suitable.

There is no exception unwinding in HUC0. A terminating panic need not execute
pending `drop` operations.

### 9.1 When temporary values are destroyed

An unconsumed temporary stays alive until its full expression finishes. In
an ordinary statement, that usually means the semicolon. Remaining active
temporaries are then destroyed in reverse order of completed initialization.

Assume `make_text()` returns an Advanced `Text`, and `view()` returns a
non-owning view of its characters:

```huc
print(make_text().view());
```

The temporary `Text` stays alive through `view()` and `print()`. It is
destroyed after `print` returns. This works if `print` only uses the view
during that call; storing the view for later does not keep the text alive.

In contrast:

```huc
let view: View = make_text().view(); // temporary Text is destroyed here
print(view); // undefined behavior: the view's source is no longer alive
```

The `view` local survives into the next statement, but the text does not.
Keep the owning value in a local instead:

```huc
let text: Text = make_text(); // initializes text, not a discarded temporary
let view: View = text.view();
print(view);
```

Some boundaries are not semicolons:

- A condition's temporaries are destroyed after its result is computed,
  before the selected `if` branch or loop body starts. A loop cleans them up
  on every condition evaluation, including the final false one.
- A `for` initializer cleans up its temporaries before the first condition.
  Each expression in its comma-separated update list cleans up its own
  temporaries before the next update or condition; this is not a comma operator.
- A field initializer cleans up its temporaries after initializing that
  field, before the next field or the constructor body.
- A `return` binds its result, destroys the expression's remaining temporaries,
  then cleans up the function's remaining locals and parameters.

A value declared in a `for` initializer is not a temporary. It stays alive
across iterations and is cleaned up when the loop exits, if still active.
Body locals are cleaned up each iteration before the updates run, including
on `continue`. `break` and `return` skip the updates but still clean up active
body and header locals. A false first condition still cleans up the header
value. These are the ordinary scope and relocation rules, not a second
lifetime model for loops.

Individual arguments, nested calls, and short-circuit operands do not end
the caller's full expression. Their unconsumed temporaries stay alive until
that expression ends. An unevaluated operand creates nothing to destroy.

Transfers still happen at the usual point. For example, a `Text` passed by
value becomes the callee's parameter, not a caller-owned temporary waiting
for the semicolon. The parameter is cleaned up before the call returns unless
transferred onward. Consequently, this does not become valid:

```huc
fn bad_view(text: Text) -> View {
    return text.view(); // text is destroyed before the caller resumes
}

print(bad_view(make_text())); // the returned view already dangles
```

A transferred temporary is inactive and is not destroyed a second time.
Similarly, a value built directly in a local or return result follows that
destination's lifetime. These rules add no hidden copy or lifetime extension.
See [language specification section 6.5.1](language-specification.md#651-temporary-values)
for the precise boundaries.

### 9.2 Constructing and destroying values in manual storage

The two intrinsics separate a value's lifetime from its storage allocation:

```huc
construct_at(slot, File(handle)); // inactive slot becomes a live File
destruct_at(slot); // close the File; its slot becomes inactive again
```

`slot` has type `mut File*`. It must point to valid, writable storage of the
right size and alignment. Neither operation allocates or frees that storage,
nulls the pointer, or registers automatic cleanup. Constructors and `drop`
methods still perform whatever resource operations their bodies specify.

`construct_at(pointer, value)` infers the type from `mut T*` and returns
`void`. It evaluates the pointer once and prepares the initializer inputs as
in Advanced assignment. It then ends any old value in the slot without
cleanup and completes initialization there. Enclosing objects stay alive.
For the replacement:

- Basic values copy.
- Advanced values transfer from an eligible source, leaving that source
  inactive. A named source needs binding `mut`.
- `copy value` explicitly copies into the slot when supported.
- `T(arguments)` constructs directly in the slot; it does not first create
  and move a temporary. This also preserves the final `&self` address.
- Direct call results forward the slot through nested direct returns, as
  described in section 8.2, without requiring inlining.
- Array literals build in the element slots; `?:` forwards the destination
  to its selected arm.

There is no hidden by-value parameter between the initializer and the slot.
For a default structure constructor write `construct_at(slot, T())`; for a
scalar, supply its value. Arrays use a literal or another array value, not
`T[N]()`. Helper temporaries still follow the enclosing
expression's lifetime, but the newly constructed value is not destroyed at
the semicolon.

`destruct_at(pointer)` returns `void`. It performs full destruction: `drop`,
if present, followed by automatic field cleanup in reverse order. A Basic
value has no cleanup code, but its lifetime ends too. Source bytes need not
be cleared. This is distinct from extraction: `let result: T = *slot;`
transfers an Advanced value without running its destructor.

These are unchecked operations. Destroying an inactive value or passing an
invalid pointer is undefined behavior. Constructing over a live value is
allowed, but skips its cleanup and can leak resources. Neither construction
nor Advanced assignment has a same-storage no-op. Ordinary assignment drops
the old Advanced value; `construct_at` never does.

`construct_at(&item, Item(item.id))` reads the scalar argument before ending
the old lifetime. In `construct_at(&item, make(&item))`, the factory runs
after that lifetime ends and must not read the old value. Retaining a pointer
does not preserve the object.

**Automatic cleanup is not changed through aliases.** For example:

```huc
let mut file: File = File(first_handle);
let slot: mut File* = &file;

destruct_at(slot); // file's tracked state is unchanged, but its old value is gone
construct_at(slot, File(second_handle)); // restore a live File before scope exit
```

Without the reconstruction, scope exit would try to destroy an inactive
`file`. Conversely, constructing into storage whose local name is already
tracked as Inactive does not make that name active again. That value must
be accessed and cleaned up through manual-storage operations. The same rule
applies to fields: these operations do not disable automatic field cleanup.

Both intrinsics require a writable typed pointer. They can directly address
raw-pointer slots when given the corresponding pointer-chain type. Their names are unqualified,
cannot be overloaded, and cannot be used as function values. See
[language specification section 6.7](language-specification.md#67-manually-managed-storage)
for the full contract.

### 9.3 Reserving raw inline storage

An aligned `u8[N]` or `c8[N]` array provides inline storage for any complete
runtime object type. `[nozero]` leaves its bytes unwritten. No payload is
constructed until the programmer constructs it:

```huc
let [align(align_of<File>()), nozero] mut storage: u8[2 * size_of<File>()];
let slots: mut File* = ptr_as<mut File*>(&storage[0]);
construct_at(slots, File(first_handle));
construct_at(slots + 1, File(second_handle));

let file: File = *(slots + 1); // second payload becomes inactive
destruct_at(slots); // close the first File
// Byte-array cleanup performs no File drops; file has ordinary cleanup.
```

The whole payload must fit in the byte array at an address aligned for its
type. Typed pointer arithmetic can traverse this byte region, including
unused capacity, or form a one-past pointer. Neither taking the address nor
casting it reads the bytes. `size_of<T>()` and `align_of<T>()` provide target
layout constants. HUC0 counts and alignment arguments accept layout queries,
addition and multiplication; HUC1 emits concrete counts.

The byte array is Basic. Copying its bytes does not construct or transfer any
payload objects. Relocating an enclosing Advanced value does transfer its
live payloads at the same offsets, without scanning slots or clearing the
source. Pointers into the old region do not follow the relocation. A container
must track its initialized range, destroy those objects in `drop`, and copy
them explicitly if it offers a `copy` method. A `drop` or `[nocopy]` makes a
wrapper Advanced; omitting both can accidentally make the wrapper Basic.
Relocating an enclosing object that carries a live `[norelocate]` payload is
undefined behavior, so an inline owner of such payloads should itself be
`[norelocate]` (section 7.2).

Whole-aggregate representation copies and relocations can carry unwritten
bytes without reading them as scalar values. Reading an unwritten scalar is
still invalid. Neither array cleanup nor the compiler supplies payload drops.

An inline optional uses one aligned byte region plus a flag; its empty state
needs no default or sentinel payload. Separate regions plus a tag can hold a
sum type, at the cost of space for every alternative. This adds no union type.
See [the byte-storage contract](language-specification.md#675-byte-arrays-as-object-storage)
and [container examples](use-cases-and-examples.md#39a-raw-storage-for-containers).

For allocated storage, allocator APIs must honor the requested size and
alignment and permit constructing the chosen element type. Check capacity
multiplications for overflow. Allocation does not construct elements; freeing
storage does not run their cleanup.

`is_basic<T>()` reports the existing category as a compile-time `bool`. It lets
generic code distinguish a copied Basic element, which remains live, from an
extracted Advanced one. A copied Basic source need not be retired before
`construct_at` reuses its slot: the intrinsic ends its old lifetime. Explicit
`destruct_at` is also allowed and emits no user cleanup call. Do not destroy
the already-inactive Advanced source.

### 9.4 Ending a local early: `drop` and `forget`

Two statements end a local's lifetime before its scope ends:

```huc
drop value;   // run the full cleanup now
forget value; // run no cleanup at all
```

`drop` runs the cleanup that the scope end would have run: the `drop` method,
then fields in reverse order, or array elements in decreasing index order.
`forget` runs none, so owned resources leak unless their ownership moved
elsewhere. Both make the local Inactive: a later use is a compile-time error
until a whole new value is assigned, and the scope end skips it.

```huc
fn update(mutex: mut Mutex*, path: c8*) -> void {
    let mut guard: LockGuard = LockGuard(mutex);
    let mut file: File = open_locked(path);
    write_header(&file);
    drop guard;        // unlock now; file stays open
    write_body(&file); // runs without holding the lock
}                      // file closes here; guard is already Inactive

fn hand_off(mut file: File) -> void {
    let handle: i32 = file.handle;
    forget file;          // no sentinel; file's drop never closes the handle
    adopt_handle(handle); // adopt_handle now owns it
}
```

The operand is one named, mutable local or by-value parameter whose tracked
state is Active. Fields and pointees keep using `destruct_at` and
`construct_at`. A Basic operand runs no code but still ends the binding, so
generic code can write `drop value;` for any `T`. In a runtime branch or loop,
the statements work like a conditional transfer: the local becomes
MaybeActive at the join, and a drop flag decides its scope-end cleanup.
Neither statement moves the object, so both work on `[norelocate]` locals.

The object must actually still be live; the tracked state does not see
`destruct_at(&local)` or extraction through a pointer. After a field was
extracted, the two differ: `forget` cancels the remaining cleanup without
touching the missing field, while `drop` would destroy it, which is undefined
behavior.

For fully live, relocatable values, a by-value library function gives the
same result as `drop`, because a callee destroys its by-value parameters
before returning:

```huc
fn dispose<T: auto>(mut value: T) -> void {
}
```

The statements are still needed for `[norelocate]` locals, which cannot be
passed by value, and `forget` for an owner with an extracted field, which a
by-value helper would relocate with undefined behavior. Rust's `mem::drop` and
`mem::forget` are by-value library functions.

## 10. Parameters and argument order

Methods declare their receiver explicitly. `&self` and `&mut self` borrow
caller storage; `self` and `mut self` bind an owned first parameter before
ordinary arguments. Owned self is fixed unless marked mut. Both get automatic
cleanup, but only mut self supports onward Advanced relocation, drop, or forget.
The expression self is an object place; &self has writable pointee permission
only for the two mutable receiver forms. Bare names never implicitly find fields.

An owned call on a named Advanced root transfers it and updates its tracked
state. Pointer calls `p->finish()` and `(*p).finish()` both use unchecked
extraction for Advanced pointees: require mut T*, leave the slot inactive, and
do not cancel the caller's automatic cleanup. Restore the slot or exclude its
cleanup. Basic owned receivers copy. Direct-result receivers construct in the
parameter's final slot, including non-relocatable types.

In `x = x.transform()`, an owned receiver transfers x before assignment's
destruction step; no old value remains to destroy. A borrowed receiver has the
existing destroy-first lifetime hazard. Receiver-only overloads are forbidden;
the call cannot choose ownership behavior as an overload tie-breaker.
See [receiver rules](declarations-and-preparation.md#45-the-receiver-self-and-static-functions).


Parameters are passed by value. Their behavior follows their type:

```huc
fn inspect(document: Document*) -> void {
    // raw observation; caller retains the value
}

fn revise(document: mut Document*) -> void {
    // writable observation; caller retains the value
}

fn summarize(metadata: Metadata) -> void {
    // copies when Metadata is Basic
}

fn consume(document: Document) -> void {
    // consumes when Document is Advanced; destroys it on return
}

fn relay(mut document: Document) -> void {
    probable_send(document); // transfers the Advanced parameter onward
}
```

For an Advanced `Document`:

```huc
let mut document: Document = Document(...);

inspect(&document); // observation; no relocation
revise(&document); // writable observation
consume(document); // relocation; caller's document becomes inactive
```

Argument expressions and their corresponding parameter initializations occur
strictly left-to-right. It makes relocations, allocation, I/O, device access,
counters, and temporary construction follow source reading order. It also
gives the section 7.5 check one fixed order to follow.

```huc
fn compare_versions(left: Document, right: Document) -> i32;

let mut document: Document = Document(...);
let ordering: i32 = compare_versions(copy document, document);
```

HUC copies `document` for `left`, then relocates the original into `right`.
Reversing the arguments is a compile-time error: `document` is already
inactive when `copy document` runs. Passing `document` by value twice is
rejected for the same reason.

The LLVM backend must implement this argument-order guarantee. It must finish
initializing each argument's **parameter value** before starting the next
argument. Saving pointers to source places and deferring their transfers
until the final LLVM call is insufficient. Typed parameter slots, separately
or in a call frame, preserve sequencing and exactly-once cleanup. Optimizers
may reorder pure computation when the program still behaves the same.

### 10.1 Selecting a value or observing one

The conditional operator produces a value. For live, writable Advanced
locals, `(condition ? first : second).size()` transfers the selected value to
a temporary, calls its method, and destroys the temporary at the semicolon.
Both original locals are then MaybeActive, so later named use is rejected.
Compilers are encouraged to warn about this form (language specification
section 8.4).

For a borrowed `size` receiver, observe one without transferring it using either form:

```huc
(condition ? &first : &second)->size();
condition ? first.size() : second.size();
```

Each evaluates only the selected observation. The pointer form copies an
address, which is Basic; it does not extend the pointee's lifetime. A selected
named Advanced value still transfers in a direct-result position; a selected
constructor, call, copy, or array literal can build directly in the supplied
destination (section 8.2). Elsewhere it builds directly in the conditional's
temporary, which is what lets `[norelocate]` types use `?:`.

## 11. Return values

Returning a Basic value as the same Basic type follows the normal copy rule:

```huc
fn origin() -> Point {
    let point: Point = Point(0, 0);
    return point;
}
```

Returning a named Advanced local destructively relocates it:

```huc
fn open_file(path: text::View) -> File {
    let mut result: File = File(probable_os_open(path));
    return result; // relocation; result becomes inactive
}
```

For a named local, the implementation may construct directly in caller-provided
result storage only when it preserves observable behavior, including constructor
addresses and required `copy` method calls. This named-return optimization is
not guaranteed. A `copy` method runs exactly when required by an evaluated
`copy` expression. A `[norelocate]` local cannot be returned by value at all;
return a direct result instead (section 7.2).

In contrast, `return T(...)`, `return make()`, and `return copy x` for Advanced
`T` forward the caller's final result slot through direct calls and copies.
Array literals and direct-result conditional arms do too, even without
inlining or optimization, as specified in section 8.2.

The result is bound before return-expression temporary cleanup, which runs
before cleanup of remaining locals and parameters. A returned owner survives
in the caller; a returned pointer or view does not keep its source alive.

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
generic family request. Before HUC0 checking, staging replaces every request
with the name of a concrete type. HUC0 itself never parses `Vector<T>`.

```huc
module examples.vector_values;

import std.collections as collections;

// Point, HeapText, and Widget refer to the example types defined above.

fn point_values() -> void {
    let mut points: collections::Vector<Point> =
        collections::Vector<Point>();
    let point: Point = Point(3, 4);

    points.push(point); // Point is Basic: copies into the vector
    points.push(copy point); // equivalent, but redundant for Basic Point
}

fn text_values() -> void {
    let mut texts: collections::Vector<HeapText> =
        collections::Vector<HeapText>();
    let mut text: HeapText = HeapText("first");

    texts.push(text); // destructive relocation; text becomes inactive
    text = HeapText("second");
    texts.push(copy text); // copy, then relocate the copy into the vector
}

```

A reallocation of `Vector<T>`:

- copies elements when `T` is Basic;
- uses fixed destructive relocation when `T` is Advanced;
- cannot hold `[norelocate]` elements, because growth relocates them; a
  library can reject them early with `is_relocatable<T>()`;
- never calls a `copy` method merely to relocate elements;
- does not drop old element places that relocation made inactive;
- drops every element that remains active exactly once;
- may invalidate raw pointers into its storage without a diagnostic.

The container is responsible for its backing storage and initialized element
range. It must ensure that old slots whose values were transferred receive
no element cleanup. Extraction through a writable pointer provides the
transfer itself. For example, with the `File` type above:

```huc
// Preconditions: storage holds *length live Files, *length > 0,
// and length points to separate writable container bookkeeping.
// The container, not automatic element cleanup, manages these slots.
fn pop_file(storage: mut File*, length: mut usize*) -> File {
    let last: usize = *length - 1;
    let mut result: File = *(storage + last);
    *length = last; // exclude the inactive slot from later element cleanup
    return result; // direct local transfer; result's cleanup is suppressed
}
```

The returned `File` is responsible for closing the handle. The old slot has
no live `File` and receives no destructor call; neither its bytes nor the
source pointer need clearing. The backing allocation stays allocated.
The container must construct a new value before including that slot in its
initialized range again. No copy, replacement value, or per-element flag
is needed to pop.

`construct_at` and `destruct_at` complete the element-lifetime operations.
Extraction needs no separate intrinsic. For example, alongside `pop_file`:

```huc
// The live prefix has *length elements; storage has spare capacity for one more.
// length points to separate bookkeeping, not into the element storage.
fn push_file(storage: mut File*, length: mut usize*, mut file: File) -> void {
    construct_at(storage + *length, file);
    *length += 1; // publish the slot only after construction succeeds
}

// The container owns cleanup of exactly the live prefix.
fn clear_files(storage: mut File*, length: mut usize*) -> void {
    while (*length > 0) {
        *length -= 1;
        destruct_at(storage + *length);
    }
}
```

Relocation into a different inactive slot can be written
`construct_at(destination, *source)` through writable pointers. For an
Advanced element the old slot then needs no destruction; container bookkeeping
must exclude it from later cleanup. Deallocating backing storage is a separate
library operation. Direct `storage[index]` extraction has the same effect.
There is still no user-defined relocation hook. `take storage[index]` instead
leaves a live default replacement for an Advanced element; it is not the same
as removing a slot from the container's live range.

`Vector<T>` has a `drop`, so it gets no generated `copy` method. Its library
can write one that copies each element. HUC checks every method of a selected
generic body, so an unguarded `copy` method would make `Vector<File>` invalid
when `File` doesn't support copying, even if no program copies the vector.
Methods that default-initialize elements have the same problem. The library
guards such members with `if1`:

```huc
struct Vector<T: auto> {
    let mut data: mut T*;
    let mut length: usize;
    let mut capacity: usize;

    // init, reserve, push, and drop as usual

    if1 (is_copyable<T>()) {
        fn copy(&self) -> Vector<T> {
            let mut result: Vector<T> = Vector<T>();
            result.reserve(self.length);
            while (result.length < self.length) {
                result.push(copy *(self.data + result.length));
            }
            return result;
        }
    }

    if1 (is_default_initializable<T>()) {
        fn resize(&mut self, count: usize) -> void {
            self.reserve(count);
            while (self.length < count) {
                let mut fresh: T; // default initialization
                self.push(fresh);
            }
            while (self.length > count) {
                self.length -= 1;
                destruct_at(self.data + self.length);
            }
        }
    }
}
```

`Vector<File>` is then valid and simply lacks the methods its element type
can't support; only a use of a missing method is an error. Rust gets the same
effect from `impl<T: Clone> Clone for Vec<T>`. C++ declares the copy
constructor of `std::vector<T>` for every `T`, so
`std::is_copy_constructible_v<std::vector<std::unique_ptr<int>>>` is true and
the error appears only when a copy is instantiated.

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

A recursively embedded container must not query its incomplete element type
during preparation. Pointer-only Links<Node> works, but a Vector<Node> whose
selected body queries Node does not. Declare unconditional lifecycle members
on a concrete pointer-backed NodeList, then define their bodies in impl blocks
after Node completes. Moving a generic querying method into its generic impl
does not defer preparation past the structure request. See the worked
[recursive-container examples](declarations-and-preparation.md#5-recursive-containers).

## 13. A complete resource-managing value example

The following HUC program combines construction, observation, mutation,
explicit copying, destructive relocation, value assignment, parameter
transfer, and destruction:

```huc
module examples.image_asset;

import std.io as io;
import std.memory as memory;

struct Image {
    let mut pixels: mut u8*;
    let width: usize;
    let height: usize;
    let stride: usize;

    fn init(&mut self, width: usize, height: usize)
        : pixels = memory::allocate_zeroed_bytes(width * height * 4),
          width = width,
          height = height,
          stride = width * 4,
    {
    }

    fn byte_count(&self) -> usize {
        return self.stride * self.height;
    }

    fn fill(&mut self, value: u8) -> void {
        memory::fill_bytes(self.pixels, value, self.byte_count());
    }

    fn copy(&self) -> Image {
        let mut result: Image = Image(self.width, self.height);
        memory::copy_bytes(
            result.pixels,
            self.pixels,
            self.byte_count(),
        );
        return result;
    }

    fn drop(&mut self) -> void {
        if (self.pixels) {
            memory::free_raw_bytes(self.pixels, self.byte_count());
            self.pixels = null;
        }
    }
}

fn print_shape(image: Image*) -> void {
    io::println(image->width, "x", image->height);
}

fn invert(image: mut Image*) -> void {
    probable_invert_bytes(image->pixels, image->byte_count());
}

fn upload(image: Image) -> void {
    probable_gpu_upload(image.pixels, image.width, image.height);
} // image is destroyed here

fn main() -> i32 {
    let mut working: Image = Image(1920, 1080);
    working.fill(0);

    print_shape(&working); // observe
    invert(&working); // writable observation

    let snapshot: Image = copy working; // calls Image's copy method

    let mut outgoing: Image = working; // working becomes inactive
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
| `let mut value: T = T(...)` | `T value(...)` | HUC spells writable storage explicitly |
| `let b: T = a` for Basic `T` | `T b = a` | Both copy |
| `let b: T = a` for Advanced `T` | `T b = std::move(a)` | HUC destructively relocates and ends the source lifetime |
| `copy a` for Advanced `T` | Copy construction | HUC makes a potentially expensive copy explicit |
| `destination = copy source` | Copy assignment | HUC destroys the old value, then copies into its slot; self-copy is undefined |
| `destination = source` for Advanced `T` | Move assignment | HUC drops the destination, then performs non-overridable relocation |
| Advanced `destination = make(args)` | Assignment from a function result | HUC binds arguments, destroys destination, then runs the factory into that slot |
| `T*` | `const T*` by default | HUC pointer is always raw and unchecked |
| `mut T*` | `T*` | Writable pointee |
| `T**` and longer chains | Nested raw pointers with corresponding access permissions | HUC permits chains, including through aliases; none of the levels owns its pointee |
| `&value`, including a pointer slot | `std::addressof(value)` | HUC accepts addressable data places and cannot overload `&`; it does not materialize a temporary |
| `fn copy(&self) -> T` | Copy constructor | Invoked only by explicit logical copying; generated for eligible structures |
| `fn drop(&mut self) -> void` | Destructor body | Both run the body before reverse member cleanup; HUC has no inheritance cleanup or exception unwinding |
| `struct [nocopy] S` | `S(const S&) = delete;` | The type still moves, by fixed relocation |
| `struct [norelocate] S` | `S(S&&) = delete;` | Direct results still build in place; a written `copy` method stays allowed |
| `drop x;` | No direct equivalent; ending the scope early | Runs cleanup now; a later use of `x` is a compile-time error |
| `forget x;` | No direct equivalent; `release()` on an owning handle | Ends the whole object without cleanup |
| Fixed HUC destructive relocation | Move constructor/assignment | HUC ends the source lifetime and has no user hook |
| Inactive source storage after relocation | Source remains alive after move construction | HUC source has no live object until reinitialized; C++ moved-from guarantees depend on the type, with standard-library types generally valid but unspecified |
| Advanced binding from `*p`, including arguments and returns | Move construction from `std::move(*p)` | HUC ends the source lifetime without drop or source clearing; the programmer must prevent later source cleanup. Plain `T v = *p` in C++ normally copies instead |
| Advanced binding from a writable field or element | Move construction from `std::move(object.field)` or `std::move(items[i])` | HUC ends the source lifetime without cancelling enclosing cleanup; it is the same unchecked extraction as `*(&place)` |
| `take place` | Conceptually exchange with a default value for Advanced types | HUC constructs the replacement in the source slot after extraction; Basic values instead copy without resetting the source |
| `construct_at(p, value)` | `std::construct_at(p, arguments...)` | HUC takes one initializer, including guaranteed direct destination construction, returns `void`, and can end an old lifetime without cleanup; neither operation allocates storage |
| `destruct_at(p)` | `std::destroy_at(p)` | Both perform destruction without deallocation; HUC does not cancel an aliased local's scheduled cleanup |
| `T[N]`, `[e0, e1, ...]` | Fixed array / `std::array<T, N>` | HUC arrays are regular by-value types; category follows the element type and there is no pointer decay |
| Aligned `[nozero]` byte array | Uninitialized aligned byte buffer | Payload lifetimes are manual; only enclosing Advanced relocation transfers them |
| `size_of<T>()`, `align_of<T>()` | `sizeof(T)`, `alignof(T)` | Target layout constants; no value evaluation or construction |
| Unconsumed temporary | Usually destroyed at full-expression end | HUC specifies condition, initializer, and return boundaries and has no reference-based lifetime extension |
| Checked use of an Inactive or MaybeActive local | No corresponding C++ destructive-move state | HUC rejects named uses on uncertain paths; raw-pointer aliases remain unchecked |
| `(c ? a : b).size()` for writable Advanced locals | Same-type C++ lvalue arms can yield an lvalue | HUC transfers the selected value into a temporary; use `(c ? &a : &b)->size()` to observe |

“Comparable with C++” therefore means that ordinary C++ RAII and value-oriented
designs have direct HUC representations. It does not mean every C++ special
member, reference category, allocator hook, or overload pattern exists in HUC.

## 15. Required diagnostics

The compiler must reject:

- relocating from a fixed source slot;
- extraction of an Advanced value through a read-only `T*`;
- `take` of a non-place, or Advanced `take` without writable access or a
  valid zero-argument constructor;
- copying an Advanced structure that has no valid `copy` method, or one
  marked `[nocopy]`;
- a `copy` method with parameters, a writable receiver, or the wrong return
  type;
- more than one `copy` method in a structure, or one on a `[nocopy]` type;
- direct calls to a `copy` method;
- a `drop` with parameters or a non-`void` return;
- a `drop` without a writable receiver;
- more than one `drop` in a structure;
- direct calls to `drop`;
- attempts to declare a special move, relocation-constructor, or
  move-assignment hook;
- relocating a `[norelocate]` type from any source;
- `drop` or `forget` whose operand is not a mutable, Active, named local or
  by-value parameter;
- any named use of an Advanced local or by-value parameter that is Inactive
  or MaybeActive at that point, including a later argument of the same call
  (section 7.5);
- a call whose argument evaluation relocates the root of its receiver, or
  relocates a local or parameter after an earlier argument that is `&`
  applied to it or to one of its fields;
- unary `&` applied to a non-place result, literal, type, or function symbol;
- `ptr_as` with a source/target pair other than raw-pointer to raw-pointer,
  raw-pointer to `usize`, or `usize` to raw-pointer;
- `construct_at` or `destruct_at` with incorrect arity, a non-pointer or
  read-only pointer operand, or an invalid/incomplete pointee type;
- `construct_at` with an incompatible initializer or ineligible Advanced source;
- attempts to overload either lifetime intrinsic or use it as a function value;
- incomplete/invalid array element types, zero/nonconstant counts, or layout overflow;
- array literal count/type mismatches, or copying an uncopyable Advanced array;
- `[nozero]` on a non-scalar, non-scalar-array type;
- using the `void` result of `construct_at` as a value;
- invalid type operands or value arguments for `size_of`, `align_of`,
  `is_basic`, `is_relocatable`, `is_copyable`, or `is_default_initializable`;
- a type query evaluated during preparation about a structure whose body is
  still being processed.

Useful warnings include:

- a `copy` method that returns or embeds a raw resource address from the
  source;
- a `drop` that manually destroys an automatically destroyed field;
- extraction that leaves an inactive field or element reachable by cleanup;
- obvious self-assignment, overlapping Advanced assignment, or inputs that
  depend on the destroyed destination;
- retaining a raw observer across value relocation or destruction;
- placing a known self-addressing inline type that is not `[norelocate]` in a
  relocating container;
- `?:` selecting a named Advanced value into a temporary that is only observed
  or discarded.

Warnings do not turn HUC into a memory-safe language and may be disabled.

## 16. Backend requirements

The HUC0-to-LLVM compiler must preserve these source semantics rather than
assuming LLVM provides HUC value lifetimes:

1. Basic/Advanced classification is decided by HUC semantic analysis.
2. A `copy` method, written or generated, runs only for an evaluated `copy`
   that requires it.
3. Relocation transfers the representation, makes the whole source inactive,
   and requires no clearing of source bytes; it is non-failing and independent
   of user overloads.
4. Directly tracked inactive values are not dropped. Pointer, field, and
   element extraction leave prevention of source cleanup to the programmer.
   No per-field state or alias tracking is added.
5. Advanced assignment prepares RHS inputs, destroys the old active value,
   and executes the outer replacement operation in its slot. Same-storage
   assignment is undefined; no implicit identity branch is emitted.
6. `drop` runs before reverse field cleanup.
7. Arguments and parameter initialization are left-to-right.
8. Generated lifecycle functions are implementation machinery, not extra HUC
   customization points.
9. Active/inactive state belongs to a storage place and must not add a hidden
   field to nominal `T`.
10. Automatic cleanup follows tracked root state on every normal exit and
    replacement. Programmer-managed extraction must respect that state;
    backend storage reclamation alone never performs HUC cleanup.
11. Relocation accepts writable whole root sources, pointer dereferences,
    fields, and elements, with distinct source-cleanup responsibilities.
    Distinct, non-overlapping source/destination storage is an unchecked
    precondition.
12. Raw-pointer copying leaves the source unchanged and never cleans up or
    extends the lifetime of a pointee.
13. `construct_at` initializes its supplied slot without destroying an old
    value or creating an intermediate by-value parameter. `destruct_at` ends
    a live value with its full cleanup, without deallocating its storage.
    Neither operation changes an aliased root's tracked cleanup state.
14. Byte arrays provide aligned object storage without constructing payloads
    or adding runtime flags. Only relocation of an enclosing Advanced value
    transfers payload lifetimes; Basic byte copies do not. The owning library
    controls payload cleanup through its length or tag. Layout and category
    queries are compile-time constants.
15. Direct constructors, calls, Advanced copies, array literals, and selected
    conditional arms use final destination slots through nested direct returns,
    even without inlining. An existing Advanced local still relocates.
16. Record source transfer before destination activation. Self-assignment
    therefore ends with Active bookkeeping without making its execution valid.
17. Basic `take` is a read with no source changes. Advanced `take` extracts
    once, then constructs `T()` in the source slot before yielding its result.
    Keep that result separate from any retained assignment destination until
    the old destination has been destroyed. Source cleanup state is unchanged.
18. A `[norelocate]` value is never moved by lowered code. Direct results,
    including by-value arguments and conditional temporaries, keep their final
    slots across calls.
19. `drop` and `forget` update tracked root state like a transfer. `drop`
    emits the full cleanup at the statement; `forget` emits none. A drop flag
    is needed only where the local's state differs between paths.

The middle-end makes construction, copy, relocation, activation, deactivation,
and cleanup explicit before LLVM emission. Loads, stores, and memory-transfer
operations are permitted only when they preserve the complete HUC operation,
including active state. LLVM attributes must not assert stronger lifetime,
alias, or initialization guarantees than HUC provides. This keeps the value
model independent of LLVM and usable by alternative backends.

## 17. Design references

HUC's rule is its own language design, but the terminology is informed by
WG21's relocation work:

- [P1144R6, “Object relocation in terms of move plus destroy”](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p1144r6.html)
  analyzes relocation and representation-transfer optimizations.
- [P2786R13, “Trivial Relocatability For C++26”](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p2786r13.html)
  is historical proposal background, not a feature to assume in C++26.
- [P3920R0, “Wording for NB comment resolution on trivial relocation”](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p3920r0.html)
  records the November 2025 decision and wording to remove P2786's trivial
  relocation feature from C++26. HUC's comparison must not present it as an
  available standard C++ operation.
- [P2785R3, “Relocation in terms of a relocation constructor”](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2785r3.html)
  explores a user-defined relocation constructor—the customization point HUC0
  deliberately defers.

These papers do not define HUC semantics. In particular, HUC's bitwise
relocation rule, `mut` requirement, and
unchecked address-dependence contract are HUC decisions.
