# HUC Value Semantics and Special Operations

Status: current design decision for HUC0

This document defines HUC's construction, copying, moving, assignment, and
destruction model. It also compares those operations with their C++20
counterparts.

The comparison is about observable value and resource behavior. HUC is not
source-compatible with C++, and it deliberately exposes fewer customization
points.

## 1. The three basic storage relationships

HUC uses three related type forms:

| HUC type | Meaning | Copy behavior |
|---|---|---|
| `T` | An inline `T` value | Determined by `T` |
| `T*` | A nullable, unchecked, non-owning pointer | Copies one address |
| `T&` | A nullable, unique owner of one allocated `T` | Moves by default; `copy` duplicates the pointee |

`T&` is not a C++ reference. HUC has no general aliasing-reference type and no
`T&&` type. A non-null `T&` owns an allocation, automatically destroys its
pointee, and then deallocates that storage. Moving it clears the source owner
to null.

Prefix `&` remains the address-of expression:

```huc
let Widget value = Widget(7);
let Widget* observer = &value;
let Widget& owner = new Widget(9);
```

The parser distinguishes the postfix `&` in a type from the prefix `&` in an
expression by grammatical position.

Mutability is independent at each layer:

```huc
let T* observer;          // fixed raw slot; read-only T
let mod T* observer;      // fixed raw slot; writable T
let T* mod observer;      // reseatable raw slot; read-only T
let mod T* mod observer;  // reseatable raw slot; writable T

let T& owner;             // fixed owner slot; read-only T
let mod T& owner;         // fixed owner slot; writable T
let T& mod owner;         // movable/reseatable owner; read-only T
let mod T& mod owner;     // movable/reseatable owner; writable T
```

A trailing `mod` is required when source code will move from, reset, release,
or reseat a pointer or owner slot. Compiler-inserted destruction at the end of
the slot's lifetime does not require source-level `mod`.

## 2. Copy and Move types

Every complete runtime type is classified as either **Copy** or **Move**.

Copy types are:

- scalar arithmetic and boolean types;
- enums and function handles;
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
let Ticket moved = original;       // move; clone is not called
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

With no `clone`, `UniqueTicket` can move but cannot be copied. A future explicit
non-copyable marker may replace this idiom, but the first compiler does not
need another type-category keyword.

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
| Move construction | None | Ordinary binding of a Move value |
| Move assignment | None | Ordinary assignment of a Move value |
| Destruction | `fn drop() mod -> void` | Inserted by the compiler |
| Field destruction | None | Inserted after `drop`, in reverse field order |

This is deliberately smaller than C++'s special-member system. In particular,
there is no user-defined move constructor or move-assignment operator.

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

Constructor initializer entries:

1. must follow field declaration order;
2. are evaluated left-to-right;
3. initialize each field before the constructor body executes.

Every fixed field requires a declaration initializer or a constructor entry.
An omitted mutable scalar becomes zero, an omitted mutable pointer or owner
becomes null, and an omitted mutable structure is default-constructed.

`init` does not double as a copy or move hook. An overload accepting another
`T` is just an explicitly selected constructor:

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

Structure fields are copied in declaration order and array elements in
increasing index order. This is semantic field copying, not a promise that
padding bytes are duplicated. Because a Copy type cannot contain a lifecycle
hook or Move field, ordinary copying does not invoke user copy code.

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
- it must leave the source unchanged;
- it cannot be called directly or have its address taken;
- ordinary binding, assignment, argument passing, and return never call it;
- `copy source` calls it when `source` is a Move structure;
- declaring it makes the structure Move even when every field is Copy;
- member visibility labels do not restrict the compiler's protocol call; omit
  `clone` when the type must not be copyable;
- it is infallible in the HUC type system, although allocation failure or panic
  may terminate the process.

A type needing recoverable duplication should expose a normally named function
such as `try_copy` returning a probable `Result[T, E]`. That function is not
selected by the `copy` expression.

The initial language has one logical copy operation rather than separate
custom copy-construction and copy-assignment hooks. Copy assignment is composed
from `clone`, destruction of the old destination, and synthesized move:

```huc
let Text mod destination = Text("old");
let Text source = Text("new");

destination = copy source;
```

Conceptually:

```text
temporary = source.clone()
destroy(destination)
move_construct(destination, temporary)
```

The temporary is constructed before the old destination is destroyed. Thus
`value = copy value` is well-defined.

A fixed-size array whose element type is Move uses a built-in elementwise copy
protocol. `copy array` copies elements in increasing index order and is valid
only when the element type itself supports logical copying. Array movement is
also synthesized in increasing index order and never invokes `clone`.

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
    let HeapText third = first;       // move; first becomes inactive
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
`drop`; HUC supplies the non-overridable move operations. This comparison does
not imply that the emitted C++ will have exactly this spelling.

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
buffer.assign_from(&other);
```

`assign_from` is not an implicit copy-assignment hook. The call site explicitly
selects the specialized operation.

## 7. Move is fixed and cannot be overridden

For a Move type, ordinary storage-binding operations move:

```huc
let Packet mod source = Packet(...);
let Packet destination = source; // synthesized move
```

The synthesized move:

1. copies each Copy field;
2. moves each Move field;
3. marks the source object inactive;
4. clears a moved-from owner source to null.

Inactive structure storage is not a usable object. It may be assigned a new
value, after which it begins a new active lifetime. HUC therefore does not need
C++'s “valid but unspecified” moved-from convention.

There is no recognized declaration such as:

```huc
// These do not override move behavior.
fn move(...) -> T;
fn move_init(...) -> T;
fn operator=(...) -> T;
```

A project may use a named method such as `relocate_storage` for domain-specific
work, but ordinary binding will never call it.

### 7.1 Why move is not customizable

Fixed move semantics provide:

- no hidden allocation or I/O during ordinary movement;
- no overload resolution based on value categories;
- no user code running merely because a value crosses a scope boundary;
- predictable fieldwise lowering;
- a direct rule for parameters, returns, assignments, and containers;
- a simple guarantee that a successful move cannot fail.

This is a deliberate limitation relative to C++. A type whose address is part
of its invariant cannot repair self-pointers in an automatic move hook. Such a
type should use stable heap ownership:

```huc
let SelfIndexed& mod stable = new SelfIndexed(...);
let SelfIndexed& elsewhere = stable; // only the owner moves; pointee address is stable
```

Alternatively it can expose an explicit named operation whose cost and
behavior are visible at the call site.

### 7.2 Native handle example

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
    let File output = input; // compiler-synthesized move; input is inactive
} // output.drop closes the handle exactly once
```

Declaring `drop` makes `File` a Move type. The synthesized move copies the
integer handle into the destination and deactivates the source. The source's
`drop` is skipped because it no longer contains an active `File`.

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

The HUC backend may use an active flag, control-flow-aware cleanup, or a
generated C++ representation that establishes an inactive sentinel. That is a
backend decision; the HUC programmer does not write a move hook.

## 8. Move assignment and self-move

A writable destination can be replaced by moving another value:

```huc
let File mod current = File(probable_os_open("old.dat"));
let File mod replacement = File(probable_os_open("new.dat"));

current = replacement;
```

For distinct active objects, Move assignment:

1. evaluates the destination location once;
2. evaluates the source location once;
3. destroys the old active destination;
4. move-constructs the destination from the source;
5. deactivates the source.

Exact self-move assignment is a no-op:

```huc
current = current;
```

Non-identical overlapping source and destination storage is undefined behavior.
The compiler should diagnose obvious cases.

Assignment to inactive storage begins a new active lifetime:

```huc
let File mod first = File(probable_os_open("a.dat"));
let File mod second = first; // first inactive

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
it once when an active lifetime ends. It is intended to release resources that
are not already represented by automatically destroyed fields.

Member visibility does not restrict compiler-inserted destruction. A private
resource type is still dropped correctly wherever its active lifetime ends.

After `drop` returns, fields are destroyed in reverse declaration order.

```huc
struct Session {
    let Logger& log;
    let Socket& socket;
    let i64 mod native_transaction;

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
will destroy. It may move such a field elsewhere; cleanup then observes that
field as null or inactive.

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
    // owns document; destroys it on return unless moved onward
}

fn relay(Document& mod document) -> void {
    probable_send(document); // moves the parameter onward
}
```

At a call:

```huc
let mod Document& mod document = new Document(...);

inspect(document); // owner-to-observer conversion; no move
revise(document);  // mutable owner-to-mutable-observer conversion
consume(document); // move; caller's owner becomes null
```

Argument expressions and their corresponding parameter initializations occur
strictly left-to-right.

This guarantee is not a memory-safety mechanism. HUC could leave the order
unspecified and remain unchecked, but then the same ownership expression could
mean different things across backends or optimization choices. The defined
order makes moves, allocation, I/O, device access, counters, and temporary
construction follow source reading order.

```huc
fn receive(Document* observed, Document& owned) -> void {
    // observed points to the object held by owned
}

let Document& mod document = new Document(...);
receive(document, document);
```

The first argument captures the address. The second argument then moves the
owner. Reversing that order would clear `document` before observation and
would produce a different result.

Another important example is:

```huc
fn compare_versions(Document left, Document right) -> i32;

let Document mod document = Document(...);
let i32 ordering = compare_versions(copy document, document);
```

HUC clones `document` for `left`, then moves the original into `right`.

C++20 does not provide the same left-to-right argument guarantee. The backend
must therefore sequence observable HUC argument evaluation, conceptually:

```cpp
auto&& huc_arg_0 = /* evaluate first HUC argument */;
auto&& huc_arg_1 = /* evaluate second HUC argument */;
receive(huc_arg_0, std::move(huc_arg_1));
```

The actual emitter must preserve lifetimes and types rather than necessarily
using `auto&&`. Optimizers remain free to reorder pure work when observable
behavior is unchanged.

## 11. Return values

Returning a Copy value copies it semantically:

```huc
fn origin() -> Point {
    let Point point = Point(0, 0);
    return point;
}
```

Returning a Move value transfers it:

```huc
fn open_file(text::View path) -> File {
    let File mod result = File(probable_os_open(path));
    return result; // result becomes inactive
}
```

Returning an owner transfers the owner:

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

```huc
module examples.vector_values;

import std.collections as collections;

// Point, HeapText, and Widget refer to the example types defined above.

fn point_values() -> void {
    let collections::Vector[Point] mod points =
        collections::Vector[Point]();
    let Point point = Point(3, 4);

    points.push(point);      // Point is Copy: copies into the vector
    points.push(copy point); // equivalent, but redundant for Copy Point
}

fn text_values() -> void {
    let collections::Vector[HeapText] mod texts =
        collections::Vector[HeapText]();
    let HeapText mod text = HeapText("first");

    texts.push(text); // HeapText is Move; text becomes inactive
    text = HeapText("second");
    texts.push(copy text); // clone, then move the clone into the vector
}

fn owner_values() -> void {
    let collections::Vector[Widget&] mod widgets =
        collections::Vector[Widget&]();
    let Widget& mod widget = new Widget(7);

    widgets.push(widget); // moves one pointer; widget becomes null
}
```

A reallocation of `Vector[T]`:

- copies elements when `T` is Copy;
- uses the fixed synthesized move when `T` is Move;
- never calls `clone` merely to relocate elements;
- drops each active old element exactly once;
- may invalidate raw pointers into its storage without a diagnostic.

If `Vector[T]` itself supports logical copying, its library implementation can
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
explicit copying, synthesized movement, owner assignment, parameter transfer,
and destruction:

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

export fn main() -> i32 {
    let mod Image& mod working = new Image(1920, 1080);
    working->fill(0);

    print_shape(working);           // observe; working still owns
    invert(working);                // mutate pointee; still owns

    let Image& snapshot = copy working; // deep copy through Image.clone

    let mod Image& mod outgoing = working; // move; working becomes null
    print_shape(outgoing);
    upload(outgoing);               // move; outgoing becomes null

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
- move construction and assignment;
- by-value ownership transfer;
- return-value optimization opportunities.

HUC removes the need to spell `std::move`, delete copy members, or implement
move members. In exchange, it does not offer custom implicit copy assignment,
custom move behavior, C++ references, or C++ value-category overloads.

## 14. C++ correspondence table

| HUC operation | Closest C++20 operation | Important difference |
|---|---|---|
| `let T value = T(...)` | `const T value(...)` | HUC is fixed by default |
| `let T mod value = T(...)` | `T value(...)` | HUC spells writable storage explicitly |
| `let T b = a` for Copy `T` | `T b = a` | Both copy |
| `let T b = a` for Move `T` | `T b = std::move(a)` | HUC selects move from the type and requires writable source storage |
| `copy a` for Move `T` | Copy construction | HUC makes a potentially expensive copy explicit |
| `destination = copy source` | Copy assignment | HUC composes clone plus replace |
| `destination = source` for Move `T` | Move assignment | HUC move is non-overridable |
| `T*` | `const T*` by default | HUC pointer is always raw and unchecked |
| `mod T*` | `T*` | Writable pointee |
| `T&` | `std::unique_ptr<T>` | HUC `&` is ownership, not reference |
| `new T(...)` | `std::make_unique<T>(...)` | Produces `T&` |
| Owner-to-`T*` conversion | `.get()` | HUC conversion is implicit in observer contexts |
| Owner parameter `T&` | `std::unique_ptr<T>` by value | HUC call moves automatically |
| `fn clone() -> T` | Copy constructor, often clone function | Invoked only by explicit logical copying |
| `fn drop() mod -> void` | Destructor body | HUC destroys fields after `drop` |
| Synthesized HUC move | Move constructor/assignment | HUC user cannot replace it |
| Inactive moved-from storage | Valid-but-unspecified moved-from object | HUC source may not use it until reinitialized |

“Comparable with C++” therefore means that ordinary C++ RAII and value-oriented
designs have direct HUC representations. It does not mean every C++ special
member, reference category, allocator hook, or overload pattern exists in HUC.

## 15. Required diagnostics

The compiler must reject:

- moving from a fixed source slot;
- copying a Move structure that has no valid `clone`;
- a `clone` with parameters, a writable receiver, or the wrong return type;
- more than one `clone` in a structure;
- direct calls to `clone`;
- a `drop` with parameters or a non-`void` return;
- a `drop` without a writable receiver;
- more than one `drop` in a structure;
- direct calls to `drop`;
- attempts to declare a special move constructor or move-assignment hook;
- ordinary use of inactive storage when detected;
- attempts to bind `T&` as though it were a C++ reference alias.

Useful warnings include:

- a `clone` that returns or embeds a raw resource address from the source;
- a `drop` that manually destroys an automatically destroyed field;
- obvious non-identical overlapping move assignment;
- copying an owner inside a loop where allocation is likely unintended;
- retaining a raw observer across an owner move or destruction.

Warnings do not turn HUC into a memory-safe language and may be disabled.

## 16. Backend obligations

The HUC0-to-C++20 transpiler must preserve these source semantics rather than
inheriting C++ defaults:

1. `T&` lowers to a pointer-sized owner representation, never a C++ reference.
2. Copy/Move classification is decided by HUC semantic analysis.
3. `clone` runs only for an evaluated `copy` that requires it.
4. moves are fieldwise, non-failing, and independent of user overloads;
5. owner moves clear the source;
6. inactive structure values are not dropped;
7. exact self-move assignment is a no-op;
8. `drop` runs before reverse field cleanup;
9. arguments and parameter initialization are left-to-right;
10. generated C++ special members are implementation machinery, not extra HUC
    customization points;
11. active/inactive state belongs to a storage place and must not add a hidden
    field to nominal `T` or to pointer-sized `T&`;
12. user `drop` must not be placed in an unconditional C++ destructor that
    would also run for an inactive raw-handle source.

The middle-end should make construction, copy, move, activation, deactivation,
and cleanup explicit before C++ emission. This keeps the semantics usable by a
future C or native-code backend.
