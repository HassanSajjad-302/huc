# HUC Use Cases and Detailed Examples

Status: design guide

Applies to: planned HUC 0.1 semantics

Related: [Implementation plan](implementation-plan.md)

This guide explains what HUC is useful for and how its runtime and staging
features work together. It intentionally contains both small semantic examples
and large application-style examples.

The compiler does not exist yet. These examples are design fixtures: as
features are implemented, each example must become either an executable test or
a golden HUC1-to-HUC0 expansion.

## 1. Reading the examples

### 1.1 Core versus probable library APIs

Core syntax and semantics in these examples are intended to be normative:

- `let`, `let1`, and `let2`;
- `mod`;
- `T*` and `T&`;
- `addressof` for inline-place observation;
- `std::slot_of` for unchecked, untyped slot addresses;
- `fn`, `fn1`, and `fn2`;
- `struct`, `struct1`, and `struct2`;
- `if1`, `for1`, and `while1`;
- `@`;
- `import` and `import2`;
- constructors named `init`;
- `clone`, `copy`, and `drop`;
- family specialization with angle patterns and predicates.

Names under modules such as these are probable standard-library APIs:

```text
std::io
std::fs
std::text
std::collections
std::net
std::result
std::memory
std::process
std::target
```

They illustrate a plausible library experience but do not freeze exact names,
error types, allocator APIs, or signatures.

### 1.2 What “not achievable elsewhere” means

Almost any language feature can be simulated in a sufficiently powerful
language using preprocessors, macros, compiler plugins, external generators, or
build scripts. This guide therefore does not make the false claim that other
languages are computationally incapable of producing the same machine code.

The useful distinction is:

- **Direct HUC expression:** the behavior is part of ordinary HUC syntax and
  has one specified selection, ownership, and generation model.
- **Simulation elsewhere:** comparable behavior normally needs a different
  mechanism, such as a macro language, build script, template trick, plugin, or
  convention.
- **HUC-specific contract:** the exact combination and observable rules are
  what HUC promises, even when individual pieces exist elsewhere.

Notable HUC-specific combinations are:

1. One partial-specialization model for structures, functions, and variables.
2. A `let1` specialization that chooses the variable’s complete declaration,
   including a different type or ownership mode.
3. Typed phase conditions at module and type-body scope whose discarded bodies
   are never parsed.
4. Compiler helper functions that emit raw HUC0 at their expansion call site,
   followed by a separately inspectable and independently parsed HUC0 round.
5. A normal `fn1` call that residualizes a runtime specialization and the same
   call prefixed by `@` that must execute completely during translation.
6. Fixed/read-only-by-default declarations, layer-specific `mod`, and a
   one-word unique owner expressed directly as `T&`.

## 2. Use-case catalog

The examples below cover these use cases:

| Area | Use case |
|---|---|
| Runtime | Fixed-by-default local and field storage |
| Runtime | Mutable scalar, pointer slot, pointee, and owner permissions |
| Runtime | Non-owning observation without lifetime tracking |
| Runtime | Explicit inline address-taking with `addressof` |
| Runtime | Automatic unique ownership and deterministic cleanup |
| Runtime | Explicit consumption through an owner parameter |
| Runtime | Owner-containing structures and destructive relocation |
| Runtime | Ordinary fieldwise-Copy value semantics |
| Runtime | Explicit deep copying with `copy` and `clone` |
| Runtime | Custom clone opting a type into the Move category |
| Runtime | Fixed, non-overridable destructive relocation |
| Runtime | Relocation assignment, exact self-relocation, and reinitialization |
| Runtime | Address-dependent values, self-pointers, and stable ownership |
| Runtime | Constructor-required fixed fields |
| Runtime | Deterministic default initialization |
| Runtime | Read-only and writable methods |
| Runtime | User `drop` plus automatic reverse field cleanup |
| Runtime | Direct correspondence with common C++20 RAII value types |
| Runtime | Left-to-right calls and expressions |
| Runtime | Conditional relocation without a borrow checker |
| Runtime | File, socket, and graphics-resource wrappers |
| HUC1 | Global declaration selection |
| HUC1 | Compile-time selection of structure layout |
| HUC1 | Discarding code that is not valid for the selected target |
| HUC1 | Repeating declarations and statements with phase loops |
| HUC1 | Structure primary, partial, and full specialization |
| HUC1 | Function partial specialization |
| HUC1 | Variable specialization with different resulting types |
| HUC1 | Compile-time predicates |
| HUC1 | Family argument deduction |
| HUC1 | Runtime residualization versus forced execution |
| HUC1 | Compiler-only helper functions and state |
| HUC1 | Call-site and module-scope raw generation |
| Pipeline | Per-module HUC1-to-HUC0 expansion |
| Pipeline | Standalone hand-written HUC0 |
| Pipeline | Generated C17 with explicit evaluation sequencing and cleanup |
| Systems | Target-specific layouts without preprocessor directives |
| Systems | Small-buffer and pointer-shape specialization |
| Systems | Embedded register-map generation |
| Systems | Protocol packet and codec generation |
| Systems | Command and plugin registries |
| Systems | ECS component storage generation |
| Future compiler API | Reflection-driven serialization |
| Future compiler API | RPC and schema generation |
| Future compiler API | Documentation, lint, and source-query tools |

## 3. Runtime examples

These examples describe HUC0 runtime behavior. A snippet with no numbered
syntax or angle-family request is directly valid HUC0. Snippets using probable
generic library spellings such as `Vector<T>` or `Result<T, E>` are HUC1
source: staging replaces each request with a concrete nominal name before the
same runtime semantics are checked by HUC0.

### 3.1 Fixed-by-default values

```huc
module examples.fixed_values;

fn calculate() -> i32 {
    let i32 base = 40;
    let i32 mod adjustment = 2;

    adjustment += 1;
    adjustment -= 1;

    // base += 1; // diagnostic: base is fixed
    return base + adjustment;
}
```

`let` means “the next declaration introduces storage.” It does not imply
mutability. `mod` is the only mutability marker.

The expected result is `42`. A backend may map `base` to a C `const` local,
but HUC’s rule is authoritative even if a backend chooses another
representation.

### 3.2 Pointer and owner permission layers

```huc
module examples.pointer_permissions;

struct Counter {
    let i32 mod value;

    fn init() {
    }

    fn read() -> i32 {
        return this->value;
    }

    fn increment() mod -> void {
        this->value += 1;
    }
}

fn observe(Counter* counter) -> i32 {
    return counter->read();
}

fn mutate(mod Counter* counter) -> void {
    counter->increment();
}

fn example() -> i32 {
    let mod Counter& mod owner = new Counter();

    let Counter* fixed_readonly = owner;
    let mod Counter* fixed_writable = owner;
    let Counter* mod reseatable_readonly = owner;
    let mod Counter* mod reseatable_writable = owner;

    mutate(fixed_writable);
    reseatable_readonly = null;
    reseatable_writable = owner;
    mutate(reseatable_writable);

    return observe(fixed_readonly);
}
```

This returns `2`.

Important distinctions:

- The leading `mod` controls access to the pointee.
- The trailing `mod` controls the pointer or owner slot.
- Observing an owner as a raw pointer does not relocate it.
- None of the raw pointers keep the `Counter` alive.

Address-taking for inline storage is explicit and never overloadable:

```huc
fn inline_address_example() -> i32 {
    let Counter mod counter = Counter();
    let mod Counter* observer = addressof(counter);
    observer->increment();
    return counter.read();
}
```

`addressof` preserves pointee permission, so a fixed `Counter` would produce
`Counter*` and the writable `counter` above produces `mod Counter*`.
`addressof` rejects pointer and owner slots because their addresses would
require one of the forbidden composed pointer types. Unary `&` is not HUC
syntax.

Untyped slot addresses are a separate, unchecked facility:

```huc
fn owner_slot_address_example() -> void {
    let Counter& mod owner = new Counter();
    let usize address = std::slot_of(owner);
    let mod u8* bytes = ptr_as<mod u8*>(address);
    // bytes addresses the owner word's representation, not the Counter.
    // Taking these addresses does not transfer ownership.
}
```

An ordinary function can receive `address`, but it receives no typed owner-slot
reference or automatic lifecycle handling. Taking a fixed slot's address is
also permitted; writing actually fixed storage remains undefined behavior,
without compiler permission tracking through the integer and cast. The caller
is responsible for storage lifetime, alignment, representation, and ownership
invariants. This does not make `Counter&*` a valid type.

### 3.3 Observation and consumption

```huc
module examples.consume;

struct FileHandle {
    let i32 mod native;

    fn init(i32 native) : native(native) {
    }

    fn valid() -> bool {
        return this->native >= 0;
    }

    fn drop() mod -> void {
        if (this->native >= 0) {
            probable_os_close(this->native);
            this->native = -1;
        }
    }
}

fn inspect(FileHandle* file) -> bool {
    return file->valid();
}

fn consume(FileHandle& file) -> void {
    // file owns the allocation for the duration of this call.
    if (file) {
        probable_log_file(file);
    }
} // file and its pointee are destroyed here

fn run(i32 native) -> bool {
    let FileHandle& mod file = new FileHandle(native);
    let bool was_valid = inspect(file); // observation
    consume(file);                      // owner relocation; file becomes null
    return was_valid;
}
```

`inspect(file)` converts the owner to a raw observer. `consume(file)` binds an
owner parameter and therefore relocates the owned address. The caller’s slot
must be `mod` because owner relocation clears it to null.

The language does not reject saving the raw pointer from `inspect`; using such
a saved pointer after `consume` would be undefined behavior.

### 3.4 Owner-containing structures

```huc
module examples.tree;

struct Node {
    let i32 value;
    let Node& mod left;
    let Node& mod right;

    fn init(i32 value) : value(value) {
        // left and right default to null.
    }

    fn sum() -> i32 {
        let i32 mod result = this->value;

        if (this->left) {
            result += this->left->sum();
        }

        if (this->right) {
            result += this->right->sum();
        }

        return result;
    }
}

fn make_tree() -> Node {
    let Node mod root = Node(10);
    root.left = new Node(20);
    root.right = new Node(30);
    return root; // fixed destructive relocation
}

fn run() -> i32 {
    let Node tree = make_tree();
    return tree.sum();
}
```

`Node` is in the Move category because it contains owners. Its fixed
destructive relocation:

1. copies `value`;
2. recursively relocates `left`;
3. recursively relocates `right`;
4. ends the source `Node` lifetime without running its cleanup.

Relocating each `T&` field leaves that source owner as a usable null owner. The
source `Node` as a whole is inactive, however; it is not a C++-style
moved-from object.

At the end of `run`, cleanup recursively destroys both children. No reference
count or tracing collector is involved.

### 3.5 Explicit deep copy

```huc
module examples.tree_copy;

struct Node {
    let i32 value;
    let Node& mod left;
    let Node& mod right;

    fn init(i32 value) : value(value) {
    }

    fn clone() -> Node {
        let Node mod result = Node(this->value);

        if (this->left) {
            result.left = copy this->left;
        }

        if (this->right) {
            result.right = copy this->right;
        }

        return result;
    }
}

fn duplicate(Node* source) -> Node {
    return copy *source;
}
```

Ordinary binding of `Node` destructively relocates it. Only `copy` asks for a
logical duplicate. This keeps potentially expensive allocation visible at the
call site. “Deep” follows logical ownership: `left` and `right` are owners, so
the clone duplicates their pointees recursively. The raw `source` parameter is
only an observer, and a raw-pointer field would normally retain the same
observed address rather than cloning whatever it reaches.

### 3.6 Constructor obligations and defaults

```huc
module examples.configuration;

struct Configuration {
    let i32 port;
    let bool secure;
    let i32 mod accepted_connections;
    let u8* mod scratch;

    fn init(i32 port, bool secure)
        : port(port),
          secure(secure) {
        // accepted_connections becomes 0.
        // scratch becomes null.
    }
}

fn create() -> Configuration {
    return Configuration(443, true);
}
```

Removing either `port(port)` or `secure(secure)` is a diagnostic because the
fields are fixed and lack declaration initializers. Mutable omitted fields have
deterministic defaults.

Initializer entries must appear in declaration order. HUC does not silently
display one order and execute another.

### 3.7 User cleanup plus field cleanup

The following probable standard-library wrapper owns a native graphics object
and a heap allocation:

```huc
module examples.texture;

import std.memory as memory;

struct PixelBuffer {
    let mod u8* mod bytes;
    let usize size;

    fn init(usize size)
        : bytes(memory::allocate_raw_bytes(size)),
          size(size) {
    }

    fn drop() mod -> void {
        memory::free_raw_bytes(this->bytes, this->size);
        this->bytes = null;
    }
}

struct Texture {
    let u32 mod gpu_id;
    let PixelBuffer staging;

    fn init(u32 gpu_id, PixelBuffer mod staging)
        : gpu_id(gpu_id),
          staging(staging) {
    }

    fn drop() mod -> void {
        if (this->gpu_id != 0) {
            probable_gpu_delete_texture(this->gpu_id);
            this->gpu_id = 0;
        }
    }
}
```

When `Texture` is destroyed:

1. `Texture.drop` releases the native GPU object.
2. `staging` is destroyed.
3. `PixelBuffer.drop` releases the byte allocation.
4. Remaining fields are cleaned in reverse declaration order.

### 3.8 Custom copy, fixed relocation, and custom destruction

HUC has three user-visible lifecycle declarations:

| Declaration | What it customizes |
|---|---|
| `fn init(...)` | Construction |
| `fn clone() -> T` | Explicit `copy` of a Move structure |
| `fn drop() mod -> void` | Cleanup before automatic field destruction |

`Move` is a type category, not the name of a user-overridable constructor.
Transferring a value in that category performs **destructive relocation**:

1. Visit each field in declaration order.
2. Copy a Copy field or recursively relocate a Move field.
3. After the last field, the source ceases to contain an active structure.
4. Neither the source's `drop` body nor its automatic field cleanup runs.

There is no user move constructor, move-assignment operator, or relocation
hook. Relocation assignment first cleans an active destination and then applies
the same fixed operation. Exact self-relocation assignment is a no-op.

`T&` is the primitive exception to the otherwise inactive-source rule.
Relocating an owner transfers its one address and leaves the source as an
active, usable null owner. This makes conditional owner consumption practical
without a borrow checker. It does not make a containing structure active after
that structure has been relocated.

```huc
module examples.native_lease;

struct NativeLease {
    let i32 mod handle;
    let u64 identity;

    fn init(i32 handle, u64 identity)
        : handle(handle),
          identity(identity) {
    }

    fn clone() -> NativeLease {
        return NativeLease(
            probable_duplicate_handle(this->handle),
            this->identity
        );
    }

    fn drop() mod -> void {
        if (this->handle >= 0) {
            probable_close_handle(this->handle);
            this->handle = -1;
        }
    }
}

fn lifecycle() -> void {
    let NativeLease mod original = NativeLease(
        probable_open_handle(),
        1001
    );

    let NativeLease duplicate = copy original; // calls clone
    let NativeLease relocated = original;       // destructive relocation
    // original is inactive; relocated holds the original handle.

    let NativeLease mod current = NativeLease(
        probable_open_handle(),
        2002
    );
    current = copy duplicate; // clone, drop current, relocate the temporary
    current = current;        // exact self-relocation is a no-op
}
```

Declaring either `clone` or `drop` puts `NativeLease` in the Move category. The
logical copy duplicates the operating-system resource. Ordinary binding never
calls `clone`, and destructive relocation never calls user code. In particular,
the relocation does not write `-1` into `original.handle`: the complete source
object is inactive, so its cleanup is skipped.

A conventional C++20 counterpart exposes the same resource behavior through
its special members:

```cpp
class NativeLease {
public:
    NativeLease(int handle, std::uint64_t identity)
        : handle_(handle), identity_(identity) {}

    NativeLease(const NativeLease& source)
        : handle_(probable_duplicate_handle(source.handle_)),
          identity_(source.identity_) {}

    NativeLease& operator=(const NativeLease& source) {
        if (this != &source) {
            NativeLease temporary(source);
            *this = std::move(temporary);
        }
        return *this;
    }

    NativeLease(NativeLease&& source) noexcept
        : handle_(std::exchange(source.handle_, -1)),
          identity_(source.identity_) {}

    NativeLease& operator=(NativeLease&& source) noexcept {
        if (this != &source) {
            close();
            handle_ = std::exchange(source.handle_, -1);
            identity_ = source.identity_;
        }
        return *this;
    }

    ~NativeLease() {
        close();
    }

private:
    void close() noexcept {
        if (handle_ >= 0) {
            probable_close_handle(handle_);
            handle_ = -1;
        }
    }

    int handle_;
    std::uint64_t identity_;
};

NativeLease original(probable_open_handle(), 1001);
NativeLease duplicate = original;
NativeLease moved = std::move(original);
```

The HUC and C++ values have comparable resource-ownership outcomes, explicit
deep copying, and deterministic cleanup, but the lifetime models differ.
A C++ move constructor constructs a destination while leaving a live source
whose destructor will run later; the example must therefore exchange the
source handle with `-1`. HUC destructive relocation ends the source lifetime
and suppresses source cleanup. Using the inactive HUC source is undefined
behavior, apart from the specified null-source behavior of a directly
relocated `T&`.

HUC makes the potentially expensive copy explicit and fixes relocation
behavior instead of selecting user code through C++ value categories. The
detailed comparison, including parameters, returns, owner values, assignment,
exact self-relocation, and a complete image type, is in
[Value Semantics and Special Operations](value-semantics.md).

### 3.9 Address stability, self-pointers, and containers

Fixed destructive relocation deliberately has no callback that can repair a
value after its address changes. This matters for:

- a structure that stores `this` in one of its own fields;
- an intrusive node referenced by address from neighboring nodes;
- an object registered as callback user data in a C API;
- a native object whose ABI requires a stable address.

The following type opts into the Move category with `drop` so it cannot be
implicitly copied. Its constructor establishes a self-pointer:

```huc
module examples.address_dependent;

struct SelfIndexed {
    let SelfIndexed* mod self;
    let u64 key;

    fn init(u64 key)
        : self(null),
          key(key) {
        this->self = this;
    }

    fn read_key_through_self() -> u64 {
        return this->self->key;
    }

    fn drop() mod -> void {
        // Empty cleanup still opts the structure into the Move category.
    }
}

fn unsafe_inline_relocation() -> u64 {
    let SelfIndexed mod original = SelfIndexed(42);
    let SelfIndexed relocated = original; // fixed destructive relocation

    // relocated.self still points at original's now-inactive storage.
    // Dereferencing it is undefined behavior.
    return relocated.read_key_through_self();
}
```

Structural relocation correctly copies the raw pointer field; it cannot know
that this particular pointer was meant to equal the object's current address.
The compiler need not reject the example. HUC is an unchecked systems
language, so preserving an address-dependent invariant remains the
programmer's responsibility.

The normal solution is to allocate the object once and relocate only its
owner:

```huc
fn safe_owner_relocation() -> u64 {
    let SelfIndexed& mod original = new SelfIndexed(42);
    let SelfIndexed& relocated = original;

    // original is a usable null owner. The allocation kept its address.
    return relocated->read_key_through_self();
}
```

Here the `T&` source becomes null, but the `SelfIndexed` pointee remains at the
address where `init` stored `this`. The pointee is destroyed exactly once when
`relocated` leaves scope.

Container choice follows the same rule. A contiguous
`Vector<SelfIndexed>` may destructively relocate inline elements when it grows,
erases, compacts, or sorts. That is an unchecked design error for
`SelfIndexed`, even though the operation is mechanically well-formed. Store
owners instead:

```huc
import std.collections as collections;

fn make_stable_index()
    -> collections::Vector<SelfIndexed&> {
    let collections::Vector<SelfIndexed&> mod values =
        collections::Vector<SelfIndexed&>();

    values.push(new SelfIndexed(10));
    values.push(new SelfIndexed(20));
    values.push(new SelfIndexed(30));
    return values;
}
```

The vector may relocate its `T&` elements, but each relocation transfers only
an owned address and leaves every pointee fixed. `Vector<SelfIndexed*>` is not
an ownership-equivalent substitute: `T*` does not keep its pointee alive.
A future standard library may also provide a stable-address node container,
but HUC0 does not need a special pin type for the initial milestone.

Relocating non-owner Move elements through raw storage is not an ordinary HUC0
source operation. Contiguous containers manage backing storage and initialized
element ranges explicitly, including preventing cleanup of retired source
slots. The only additional raw-storage lifetime intrinsics planned are
`std::construct_at` and `std::destruct_at`, whose interfaces and detailed
semantics will be designed later.

A fixed inline slot can be used only when the value is constructed directly in
its final storage and no later path relocates, reorders, returns, or captures it
by value. Mutable address-dependent values are therefore normally clearer as
`T&`. A future user-defined relocation hook should be considered only if real
address-repair use cases prove that explicit stable ownership is inadequate.

### 3.10 Left-to-right evaluation

```huc
module examples.order;

let i32 mod trace;

fn next(i32 digit) -> i32 {
    trace = trace * 10 + digit;
    return digit;
}

fn combine(i32 a, i32 b, i32 c) -> i32 {
    return a * 100 + b * 10 + c;
}

fn run() -> i32 {
    trace = 0;
    let i32 result = combine(next(1), next(2), next(3));

    if (trace != 123) {
        return -1;
    }

    return result;
}
```

HUC requires both `trace` and `result` to become `123`. The C17 backend must
introduce ordered temporaries if a direct C call would not preserve this
contract.

Conceptual generated C (module-qualified names shortened):

```c
int32_t huc_arg0 = next(1);
int32_t huc_arg1 = next(2);
int32_t huc_arg2 = next(3);
int32_t result = combine(huc_arg0, huc_arg1, huc_arg2);
```

### 3.11 Relocating the same owner twice in one call

```huc
module examples.double_relocation;

struct Item {
    let i32 value;

    fn init(i32 value) : value(value) {
    }
}

fn take(Item& first, Item& second) -> i32 {
    let i32 mod result = 0;

    if (first) {
        result += first->value;
    }

    if (second) {
        result += second->value;
    }

    return result;
}

fn run() -> i32 {
    let Item& mod item = new Item(7);
    return take(item, item);
}
```

The first argument relocates the address and clears `item`. The second argument
therefore receives null. The result is `7`. This behavior is defined by HUC’s
left-to-right rule rather than by backend argument ordering.

### 3.12 Conditional relocation remains unchecked

```huc
module examples.conditional_relocation;

struct Job {
    let i32 id;
    fn init(i32 id) : id(id) {}
}

fn consume(Job& job) -> void {
}

fn run(bool transfer) -> i32 {
    let Job& mod job = new Job(9);

    if (transfer) {
        consume(job);
    }

    if (job) {
        return job->id;
    }

    return 0;
}
```

HUC does not require a borrow checker to reject the final access. The owner is
either non-null or null. Dereferencing it without the explicit condition would
be unchecked and potentially undefined.

### 3.13 Probable file-processing application

This larger example uses illustrative standard-library APIs:

```huc
module examples.copy_file;

import std.fs as fs;
import std.io as io;
import std.result as result;
import std.text as text;

struct CopyStats {
    let usize mod bytes;
    let usize mod chunks;

    fn init() {
    }

    fn add(usize count) mod -> void {
        this->bytes += count;
        this->chunks += 1;
    }
}

fn copy_stream(fs::Reader& input, fs::Writer& output)
    -> result::Result<CopyStats, fs::Error> {
    let CopyStats mod stats = CopyStats();
    let fs::Buffer mod buffer = fs::Buffer(64 * 1024);

    while (true) {
        let result::Result<usize, fs::Error> mod count =
            input->read(buffer.writable_bytes());

        if (count.is_error()) {
            return result::Result<CopyStats, fs::Error>::error(
                count.take_error());
        }

        if (count.value() == 0) {
            break;
        }

        let result::Result<void, fs::Error> mod written =
            output->write_all(buffer.bytes(0, count.value()));

        if (written.is_error()) {
            return result::Result<CopyStats, fs::Error>::error(
                written.take_error());
        }

        stats.add(count.value());
    }

    return result::Result<CopyStats, fs::Error>::value(stats);
}

fn run(text::Text source, text::Text destination) -> i32 {
    let result::Result<fs::Reader&, fs::Error> mod opened_input =
        fs::open_reader(source);

    if (opened_input.is_error()) {
        io::error_line(opened_input.error().message());
        return 1;
    }

    let fs::Reader& mod input = opened_input.take_value();

    let result::Result<fs::Writer&, fs::Error> mod opened_output =
        fs::create_writer(destination);

    if (opened_output.is_error()) {
        io::error_line(opened_output.error().message());
        return 1;
    }

    let fs::Writer& mod output = opened_output.take_value();
    let result::Result<CopyStats, fs::Error> copied =
        copy_stream(input, output);

    if (copied.is_error()) {
        io::error_line(copied.error().message());
        return 1;
    }

    io::print("copied ");
    io::print(copied.value().bytes);
    io::print(" bytes in ");
    io::print(copied.value().chunks);
    io::print_line(" chunks");
    return 0;
}
```

The standard-library API is provisional, but the ownership intent is clear:

- `open_reader` returns an owner.
- `take_value` destructively relocates that owner and leaves its source null.
- `copy_stream` consumes both stream owners.
- all early returns deterministically clean still-active owners.
- buffers and errors use ordinary value semantics rather than special language
  exceptions.

## 4. HUC1 control-flow examples

### 4.1 Global declaration selection

```huc
module examples.platform_clock;

let2 bool use_monotonic_clock = true;

if1 (use_monotonic_clock) {
    fn platform_ticks() -> u64 {
        return probable_monotonic_ticks();
    }
} else {
    fn platform_ticks() -> u64 {
        return probable_wall_clock_ticks();
    }
}
```

Generated HUC0:

```huc
module examples.platform_clock;

fn platform_ticks() -> u64 {
    return probable_monotonic_ticks();
}
```

The selected declaration is a normal HUC0 function. No conditional exists at
runtime.

### 4.2 Type-layout selection

```huc
module examples.packet_header;

let2 bool compact_header = true;

struct PacketHeader {
    let u16 kind;

    if1 (compact_header) {
        let u16 length;
    } else {
        let u64 length;
        let u32 checksum;
    }

    fn init(
        u16 kind,
        if1 (compact_header) u16 else u64 length
    ) : kind(kind),
        length(length) {
        if1 (!compact_header) {
            this->checksum = 0;
        }
    }
}
```

The parameter-level `if1` shorthand shown above is a possible future grammar
extension and is not part of the initial grammar. The initial form should use
two selected constructor declarations:

```huc
struct PacketHeader {
    let u16 kind;

    if1 (compact_header) {
        let u16 length;

        fn init(u16 kind, u16 length)
            : kind(kind),
              length(length) {
        }
    } else {
        let u64 length;
        let u32 mod checksum;

        fn init(u16 kind, u64 length)
            : kind(kind),
              length(length) {
        }
    }
}
```

Compact HUC0:

```huc
struct PacketHeader {
    let u16 kind;
    let u16 length;

    fn init(u16 kind, u16 length)
        : kind(kind),
          length(length) {
    }
}
```

This is useful for protocol versions, embedded targets, optional debugging
fields, and architecture-dependent layouts.

### 4.3 An opaque discarded branch

```huc
module examples.opaque_branch;

let2 bool building_linux = true;

if1 (building_linux) {
    fn platform_name() -> c8* {
        return "linux";
    }
} else {
    this branch is deliberately not valid HUC !!!
    future_windows_syntax {
        tokens that the current parser does not understand;
    }
}
```

The HUC parser never sees the `else` contents. A delimiter scanner only finds
the matching brace. This is stronger skipping than `if constexpr`-style
semantic discarding, although traditional textual preprocessors can provide a
similar raw-token escape.

If `building_linux` becomes false, the selected body is parsed and compilation
fails until it is replaced with valid HUC.

### 4.4 Module-scope `while1`

```huc
module examples.generated_probes;

let2 usize mod index = 0;

while1 (index < 2) {
    if1 (index == 0) {
        fn generated_probe_zero() -> i32 {
            return 10;
        }
    } else {
        fn generated_probe_one() -> i32 {
            return 20;
        }
    }

    index += 1;
}
```

Generated HUC0:

```huc
module examples.generated_probes;

fn generated_probe_zero() -> i32 {
    return 10;
}

fn generated_probe_one() -> i32 {
    return 20;
}
```

The loop exists only during translation. Its body may execute compiler
statements such as `index += 1` while residualizing module declarations.

### 4.5 Future: `for1` over a compile-time pack

Open-ended phase packs are not in HUC 0.1. This fixture records a possible
later spelling and expected residual shape; it is not accepted by the initial
HUC1 grammar.

```huc
module examples.sum_constants;

fn1 sum_constants(i32... Values)() -> i32 {
    let i32 mod result = 0;

    for1 (let2 i32 value : Values) {
        result += value;
    }

    return result;
}

fn run() -> i32 {
    return sum_constants<10, 20, 12>();
}
```

Illustrative HUC0:

```huc
fn __huc_sum_constants_10_20_12() -> i32 {
    let i32 mod result = 0;
    result += 10;
    result += 20;
    result += 12;
    return result;
}

fn run() -> i32 {
    return __huc_sum_constants_10_20_12();
}
```

Pack syntax is planned but should be implemented after fixed-arity families.

## 5. Family specialization examples

### 5.1 Structure primary, partial, and full specialization

```huc
module examples.storage;

struct1 Storage(auto T) {
    let T value;

    fn init(T mod value) : value(value) {
    }

    fn get() -> T* {
        return addressof(this->value);
    }
}

struct1 Storage(auto T)<T*> {
    let T* value;

    fn init(T* value) : value(value) {
    }

    fn get() -> T* {
        return this->value;
    }
}

struct1 Storage()<u8> {
    let u8 value;

    fn init(u8 value) : value(value) {
    }

    fn get() -> u8 {
        return this->value;
    }

    fn hexadecimal_digit() -> c8 {
        if (this->value < 10) {
            return as<c8>('0' + this->value);
        }
        return as<c8>('a' + this->value - 10);
    }
}
```

Requests:

```huc
let Storage<i32> number = Storage<i32>(42);       // primary
let Storage<i32*> pointer = Storage<i32*>(null);  // partial
let Storage<u8> byte = Storage<u8>(15);           // full
```

The full specialization wins over the primary for `u8`; the pointer pattern
wins for every raw-pointer type.

### 5.2 Structural pattern plus predicate

This example assumes a probable library `InlineArray<T, N>` family; primitive
`[N]T` arrays are deferred from HUC 0.1.

```huc
struct1 Buffer(auto T, usize N) {
    let InlineArray<T, N> mod values;

    fn init() {
    }
}

struct1 Buffer(auto T, usize N)<T*, N>(N <= 16) {
    let InlineArray<T*, N> mod values;

    fn init() {
    }
}
```

The partial specialization is viable only for pointer elements and small
capacities. The predicate filters; it does not make the pattern rank above a
different structurally more-specific pattern.

### 5.3 Function partial specialization

```huc
module examples.hash;

fn1 hash_value(auto T)(T value) -> u64 {
    return probable_hash_bytes(addressof(value), size_of<T>());
}

fn1 hash_value(auto T)<T*>(T* value) -> u64 {
    if (!value) {
        return 0;
    }

    return probable_hash_bytes(value, size_of<T>());
}

fn run(i32 value, i32* pointer) -> u64 {
    let u64 first = hash_value(value);    // T deduced as i32
    let u64 second = hash_value(pointer); // pointer partial selected
    return first ^ second;
}
```

[C++ does not permit partial specialization of function templates
directly](https://eel.is/c++draft/temp.spec.partial); overloading is normally
used instead. HUC deliberately uses the same declared family-specialization
model for `fn1` and `struct1`.

### 5.4 `let1` with different result types

```huc
module examples.specialized_slot;

struct Widget {
    let i32 id;
    fn init(i32 id) : id(id) {}
}

fn run() -> i32 {
    let1(auto T) slot {
        let T slot = T();
    }

    let1(auto T)<T*> slot {
        let T* mod slot = null;
    }

    let1()<Widget> slot {
        let Widget& mod slot = new Widget(99);
    }

    let i32 first = slot<i32>;
    let Widget* second = slot<Widget*>;
    let i32 third = slot<Widget>->id;

    if (second) {
        return -1;
    }

    return first + third;
}
```

Illustrative HUC0 at the family declaration site:

```huc
let i32 __huc_slot_i32 = i32();
let Widget* mod __huc_slot_Widget_ptr = null;
let Widget& mod __huc_slot_Widget = new Widget(99);

let i32 first = __huc_slot_i32;
let Widget* second = __huc_slot_Widget_ptr;
let i32 third = __huc_slot_Widget->id;
```

The same family source name denotes:

- an inline `i32`;
- a raw `Widget*`;
- an owning `Widget&`.

This is one of HUC’s most unusual direct facilities. C++ variable templates can
also be explicitly/partially specialized, but HUC integrates the selected
ordinary `let` declaration, ownership syntax, phase control, and HUC0
materialization into the same numbered model.

### 5.5 `let1` selecting mutability

```huc
fn configure(bool writable) -> void {
    let1(bool Writable) setting {
        if1 (Writable) {
            let i32 mod setting;
        } else {
            let i32 setting = 7;
        }
    }

    if1 (writable) {
        setting<true> = 12;
        probable_use(setting<true>);
    } else {
        probable_use(setting<false>);
    }
}
```

The family decides whether the residual variable slot itself is writable.
Because `writable` must be a compile-time value for `if1`, a runtime boolean is
not accepted here; the example’s parameter would need to be a family binder or
phase-2 input in the finalized grammar.

A correct family form is:

```huc
fn1 configure(bool Writable)() -> void {
    let1(bool Choice) setting {
        if1 (Choice) {
            let i32 mod setting;
        } else {
            let i32 setting = 7;
        }
    }

    if1 (Writable) {
        setting<true> = 12;
        probable_use(setting<true>);
    } else {
        probable_use(setting<false>);
    }
}
```

The invalid first version is retained deliberately to show the phase boundary.

## 6. Residualization and forced execution

### 6.1 One `fn1`, two call modes

```huc
module examples.biased_add;

fn1 biased_add(i32 Bias)(i32 left, i32 right) -> i32 {
    return left + right + Bias;
}

fn runtime_case(i32 left, i32 right) -> i32 {
    return biased_add<5>(left, right);
}

fn compile_case() -> i32 {
    return @biased_add<5>(10, 20);
}
```

HUC0:

```huc
fn __huc_biased_add_5(i32 left, i32 right) -> i32 {
    return left + right + 5;
}

fn runtime_case(i32 left, i32 right) -> i32 {
    return __huc_biased_add_5(left, right);
}

fn compile_case() -> i32 {
    return 35;
}
```

The normal call specializes a runtime function. `@` requires complete
translation-time execution and embeds its materializable result.

### 6.2 Phase-2 helper and evaluation island

`InlineArray` below is a probable later library family; it is used to show
phase-value materialization, not as a HUC 0.1 primitive.

```huc
module examples.phase_helper;

fn2 clamp_build_value(i32 value, i32 low, i32 high) -> i32 {
    if (value < low) {
        return low;
    }

    if (value > high) {
        return high;
    }

    return value;
}

let2 i32 selected_capacity = @clamp_build_value(1000, 16, 256);

struct Cache {
    let InlineArray<u8, selected_capacity> mod bytes;
}
```

The outer `@` starts compiler execution. Calls made inside
`clamp_build_value` would not need another `@`.

The HUC0 structure has a concrete capacity of `256`; no compiler-only binding
survives.

## 7. Raw generation examples

### 7.1 A helper generating at its call site

```huc
module examples.callsite_emit;

import2 compiler;

fn2 declare_counter() -> void {
    compiler::emit_huc("let i32 mod generated_counter;");
}

fn run() -> i32 {
    @declare_counter();
    generated_counter = 42;
    return generated_counter;
}
```

Generated HUC0:

```huc
module examples.callsite_emit;

fn run() -> i32 {
    let i32 mod generated_counter;
    generated_counter = 42;
    return generated_counter;
}
```

The helper is defined at module scope, but its inherited insertion cursor is
inside `run`.

### 7.2 Module generation from a helper

```huc
module examples.module_emit;

import2 compiler;

fn2 generate_health_check() -> void {
    compiler::emit_huc_module(
        "fn health_check() -> i32 { return 200; }");
}

@generate_health_check();
```

Generated HUC0:

```huc
module examples.module_emit;

fn health_check() -> i32 {
    return 200;
}
```

`emit_huc_module` may emit declarations only. It cannot insert arbitrary code
into an already checked foreign module.

### 7.3 Emitted HUC1 is rejected in Round 2

```huc
@compiler::emit_huc_module(
    "if1 (true) { fn illegal() -> i32 { return 1; } }");
```

The HUC1 translator writes the string without parsing it. The HUC0 translator
then rejects `if1`. This strict boundary keeps raw generation simple and makes
the intermediate artifact honest.

## 8. Large case study: target-specialized small vector

This forward-looking case study assumes the later `InlineArray<T, N>` library
facility. The 0.1 core deliberately defers primitive fixed arrays.

This example combines:

- a family selected by element type and inline capacity;
- fixed-by-default fields;
- owner-backed spill storage;
- phase-selected layout;
- runtime destructive relocation and cleanup;
- probable allocator APIs.

For clarity, this bootstrap sketch assumes that `T` is Copy. A production
standard-library version would state that requirement through the future
compiler type-query API or add element-wise relocation/drop handling.

```huc
module case_studies.small_vector;

import std.memory as memory;
import std.io as io;

struct1 HeapArray(auto T) {
    let mod T* mod values;
    let usize capacity;

    fn init(usize capacity)
        : values(memory::allocate_raw_items<T>(capacity)),
          capacity(capacity) {
    }

    fn drop() mod -> void {
        memory::free_raw_items<T>(
            this->values,
            this->capacity);
        this->values = null;
    }
}

struct1 SmallVector(auto T, usize Inline) {
    let InlineArray<T, Inline> mod inline_values;
    let HeapArray<T>& mod heap_values;
    let usize mod size;
    let usize mod capacity;

    fn init()
        : capacity(Inline) {
    }

    fn using_heap() -> bool {
        return this->heap_values != null;
    }

    fn data() -> T* {
        if (this->heap_values) {
            return this->heap_values->values;
        }

        return addressof(this->inline_values[0]);
    }

    fn data_mut() mod -> mod T* {
        if (this->heap_values) {
            return this->heap_values->values;
        }

        return addressof(this->inline_values[0]);
    }

    fn reserve(usize requested) mod -> void {
        if (requested <= this->capacity) {
            return;
        }

        let usize mod next_capacity = this->capacity * 2;
        if (next_capacity < requested) {
            next_capacity = requested;
        }

        let HeapArray<T>& mod replacement =
            new HeapArray<T>(next_capacity);

        let usize mod index = 0;
        while (index < this->size) {
            replacement->values[index] = this->data_mut()[index];
            index += 1;
        }

        this->heap_values = replacement;
        this->capacity = next_capacity;
    }

    fn push(T mod value) mod -> void {
        this->reserve(this->size + 1);
        this->data_mut()[this->size] = value;
        this->size += 1;
    }

    fn get(usize index) -> T* {
        // Bounds remain unchecked in the core language.
        return addressof(this->data()[index]);
    }

}

struct1 SmallVector(auto T, usize Inline)<T*, Inline>(Inline <= 4) {
    let InlineArray<T*, Inline> mod values;
    let usize mod size;

    fn init() {
    }

    fn push(T* value) mod -> void {
        // This compact specialization deliberately has no spill path.
        // Writing past Inline is undefined in the unchecked profile.
        this->values[this->size] = value;
        this->size += 1;
    }

    fn get(usize index) -> T* {
        return this->values[index];
    }
}

fn run() -> i32 {
    let SmallVector<i32, 8> mod numbers =
        SmallVector<i32, 8>();

    numbers.push(10);
    numbers.push(20);
    numbers.push(12);

    let SmallVector<i32*, 4> mod pointers =
        SmallVector<i32*, 4>();

    pointers.push(numbers.get(0));
    pointers.push(numbers.get(1));

    io::print_line(*pointers.get(0) + *pointers.get(1));
    return *numbers.get(2);
}
```

This is illustrative rather than a finalized safe container:

- The pointer specialization is chosen structurally.
- The predicate limits the compact form to capacities up to four.
- The general form has one owner word for spill storage.
- HUC does not add bounds or dangling-pointer checks.
- The backend sees only concrete HUC0 structures.

## 9. Large case study: embedded register map

A hardware description often needs many nearly identical volatile accessors.
The bootstrap reflection library is not available yet, so raw HUC0 generation
can serve as the initial mechanism.

The probable compiler text helper `format` below is illustrative.

```huc
module case_studies.uart_registers;

import2 compiler;

struct2 RegisterSpec {
    let2 compiler::Text name;
    let2 usize offset;
    let2 bool writable;

    fn2 init(
        compiler::Text name,
        usize offset,
        bool writable)
        : name(name),
          offset(offset),
          writable(writable) {
    }
}

fn2 emit_register(RegisterSpec spec) -> void {
    if (spec.writable) {
        compiler::emit_huc_module(
            compiler::format(
                "fn write_{0}(mod u32* base, u32 value) -> void { "
                "base[{1}] = value; "
                "}",
                spec.name,
                spec.offset / 4));
    }

    compiler::emit_huc_module(
        compiler::format(
            "fn read_{0}(u32* base) -> u32 { "
            "return base[{1}]; "
            "}",
            spec.name,
            spec.offset / 4));
}

@emit_register(RegisterSpec("status", 0x00, false));
@emit_register(RegisterSpec("control", 0x04, true));
@emit_register(RegisterSpec("baud", 0x08, true));

fn initialize_uart(mod u32* base, u32 baud) -> void {
    write_control(base, 0);
    write_baud(base, baud);
    write_control(base, 1);
}
```

Expected generated declarations conceptually include:

```huc
fn read_status(u32* base) -> u32 {
    return base[0];
}

fn write_control(mod u32* base, u32 value) -> void {
    base[1] = value;
}

fn read_control(u32* base) -> u32 {
    return base[1];
}

fn write_baud(mod u32* base, u32 value) -> void {
    base[2] = value;
}

fn read_baud(u32* base) -> u32 {
    return base[2];
}
```

Usefulness:

- offsets are computed once;
- read-only registers receive no writer;
- the HUC0 output is inspectable;
- the runtime operations remain unchecked and compile to raw loads/stores;
- future structured generation can replace raw formatting without changing the
  two-stage architecture.

`compiler::Text`, `compiler::format`, and user-defined `struct2 init` details
belong to the future compiler-library proposal; this case study records the
intended experience, not bootstrap API commitments beyond raw emission.

## 10. Large case study: protocol messages

This example uses structure families to choose wire storage and phase control
to include optional protocol features.

```huc
module case_studies.protocol;

import std.io as io;
import std.net as net;
import std.result as result;

let2 bool protocol_has_checksum = true;

struct1 WireInteger(usize Bits) {
    if1 (Bits == 8) {
        let u8 value;
        fn init(u8 value) : value(value) {}
    } else {
        if1 (Bits == 16) {
            let u16 value;
            fn init(u16 value) : value(value) {}
        } else {
            let u32 value;
            fn init(u32 value) : value(value) {}
        }
    }
}

struct MessageHeader {
    let WireInteger<16> kind;
    let WireInteger<16> payload_size;

    if1 (protocol_has_checksum) {
        let WireInteger<32> checksum;
    }

    if1 (protocol_has_checksum) {
        fn init(u16 kind, u16 payload_size, u32 checksum)
            : kind(WireInteger<16>(kind)),
              payload_size(WireInteger<16>(payload_size)),
              checksum(WireInteger<32>(checksum)) {
        }
    } else {
        fn init(u16 kind, u16 payload_size)
            : kind(WireInteger<16>(kind)),
              payload_size(WireInteger<16>(payload_size)) {
        }
    }
}

struct Packet {
    let MessageHeader header;
    let net::ByteBuffer& mod payload;

    fn init(MessageHeader header, net::ByteBuffer& mod payload)
        : header(header),
          payload(payload) {
    }
}

fn send_packet(net::Socket* socket, Packet& packet)
    -> result::Result<void, net::Error> {
    let result::Result<void, net::Error> mod header_result =
        socket->write(probable_bytes_of(addressof(packet->header)));

    if (header_result.is_error()) {
        return header_result;
    }

    return socket->write(
        packet->payload->bytes(
            0,
            packet->header.payload_size.value));
}

fn run(
    net::Socket* socket,
    net::ByteBuffer& mod payload,
    usize size
) -> i32 {
    let u32 checksum = probable_checksum(payload->bytes(0, size));
    let MessageHeader header =
        MessageHeader(7, as<u16>(size), checksum);
    let Packet& mod packet = new Packet(header, payload);

    let result::Result<void, net::Error> sent =
        send_packet(socket, packet);

    if (sent.is_error()) {
        io::error_line(sent.error().message());
        return 1;
    }

    return 0;
}
```

The HUC0 output has exactly one header layout and constructor. Runtime packet
ownership is independent of the compile-time layout mechanism.

## 11. Large case study: generated command registry

This case demonstrates module-scope generation and direct runtime dispatch. Raw
generation is intentionally verbose; the future structured compiler API would
make it hygienic.

```huc
module case_studies.commands;

import2 compiler;
import std.io as io;
import std.text as text;

fn command_build(text::Text arguments) -> i32 {
    io::print("building ");
    io::print_line(arguments);
    return 0;
}

fn command_clean(text::Text arguments) -> i32 {
    io::print("cleaning ");
    io::print_line(arguments);
    return 0;
}

fn command_test(text::Text arguments) -> i32 {
    io::print("testing ");
    io::print_line(arguments);
    return 0;
}

fn2 emit_dispatcher() -> void {
    compiler::emit_huc_module(
        "fn dispatch(text::Text name, "
        "text::Text arguments) -> i32 {"
        "  if (name == \"build\") { return command_build(arguments); }"
        "  if (name == \"clean\") { return command_clean(arguments); }"
        "  if (name == \"test\") { return command_test(arguments); }"
        "  io::error_line(\"unknown command\");"
        "  return 2;"
        "}");
}

@emit_dispatcher();
```

Usefulness:

- the registry is generated once;
- the generated HUC0 dispatch function can be inspected and tested;
- no runtime reflection table is required;
- future reflection can discover functions carrying command metadata rather
  than manually listing them.

## 12. Large case study: ECS component storage

Entity-component systems often need type-directed storage. The following
illustrates a family that chooses inline storage for small Copy components and
owner storage for Move components.

The predicate helpers are probable future compiler type queries.

```huc
module case_studies.ecs;

import2 compiler;
import std.collections as collections;
import std.memory as memory;

struct Position {
    let f32 mod x;
    let f32 mod y;
    fn init(f32 x, f32 y) : x(x), y(y) {}
}

struct Sprite {
    let u32 texture;
    let memory::ByteBuffer& mod pixels;

    fn init(u32 texture, memory::ByteBuffer& mod pixels)
        : texture(texture),
          pixels(pixels) {
    }
}

struct1 ComponentStore(auto T) {
    if1 (@compiler::type<T>().is_move()) {
        let collections::Vector<T&> mod values;

        fn add(T& mod value) mod -> usize {
            this->values.push(value);
            return this->values.size() - 1;
        }

        fn get(usize index) -> T* {
            return this->values[index];
        }
    } else {
        let collections::Vector<T> mod values;

        fn add(T value) mod -> usize {
            this->values.push(value);
            return this->values.size() - 1;
        }

        fn get(usize index) -> T* {
            return this->values.pointer_at(index);
        }
    }

    fn init() {
    }
}

struct World {
    let ComponentStore<Position> mod positions;
    let ComponentStore<Sprite> mod sprites;

    fn init() {
    }
}
```

This exact predicate depends on the future reflection API and is therefore
non-bootstrap. It demonstrates why HUC reserves stable type metadata:

- Copy components can be stored inline.
- Move components can be stored through unique owners.
- users interact with a uniform `get` observer;
- each concrete store is generated in HUC0;
- runtime storage contains no reflection metadata.

## 13. Future compiler/reflection use cases

The APIs in this section are deliberately non-normative. They document the
capabilities the future `import2 compiler` design must support.

### 13.1 Environment inspection

```huc
import2 compiler;

fn2 report_environment() -> void {
    let2 compiler::Environment environment =
        compiler::current_environment();

    compiler::log(environment.target().triple());
    compiler::log(environment.language_revision());

    for1 (let2 compiler::Module module : environment.modules()) {
        compiler::log(module.qualified_name());
    }
}

@report_environment();
```

Required design properties:

- immutable snapshots;
- stable module/declaration IDs;
- explicit target and feature data;
- deterministic enumeration;
- tracked access to external build inputs;
- no exposure of host pointers or internal compiler data structures.

### 13.2 Serializer generation

```huc
import2 compiler;

fn2 generate_serializer(compiler::Type type) -> void {
    let2 compiler::Class record = type.as_class();
    let2 compiler::Code mod body = compiler::Code();

    for1 (let2 compiler::Field field : record.fields()) {
        if1 (!field.attributes().contains("skip")) {
            body.append(
                compiler::statement(
                    "writer.write_field({name}, value.{field})",
                    field.name(),
                    field.identifier()));
        }
    }

    compiler::emit_module(
        compiler::function(
            "serialize_" + record.name(),
            body.commit()));
}
```

The future structured API should eventually replace strings and provide:

- typed statements and expressions;
- identifier hygiene;
- declaration construction;
- explicit validation/commit;
- origin tracking;
- current-scope and module-scope insertion.

### 13.3 RPC generation

A compiler tool could:

1. enumerate functions carrying an `rpc` attribute;
2. inspect parameter and result types;
3. generate request and response message structures;
4. generate client stubs;
5. generate server dispatch;
6. generate a schema document as a tracked build artifact;
7. emit all runtime code as ordinary HUC0.

The important HUC boundary is that the reflection handles remain compiler-only.
The executable contains only explicitly generated runtime types and functions.

### 13.4 Database and configuration schemas

Reflection-driven generators could produce:

- SQL column mappings;
- migration validators;
- command-line parsers;
- environment-variable loaders;
- JSON/TOML codecs;
- configuration documentation;
- default-value tables.

Compile-time queries must register any read schema file as a build dependency,
including its content hash.

### 13.5 Static analysis and “questions”

The compiler library should allow phase code to ask questions such as:

- Which types contain an owner?
- Which functions consume an owner parameter?
- Which structures have a `drop` method?
- Which declarations and attributes exist in each imported module?
- Which statements perform unchecked pointer arithmetic?
- Which generated declarations originated from this family?
- Which concrete specializations exist?
- Which modules depend on a selected declaration?

Answers are semantic objects or immutable query results, not raw access to
compiler implementation containers.

Possible tools include:

- API documentation generators;
- ABI reports;
- ownership audits;
- custom project lints;
- call-graph exporters;
- generated tests;
- serialization completeness checks;
- plugin/command registries.

## 14. Comparison with other language mechanisms

| HUC facility | Typical alternative elsewhere | HUC’s direct contract |
|---|---|---|
| `T&` | `unique_ptr`, owned boxes, library wrapper | One-word unique ownership integrated with relocation-by-binding |
| `init` / `clone` / `drop` | C++ special members, RAII wrappers | Custom construction, explicit logical copy, and cleanup |
| Fixed destructive relocation | C++ move constructors and move assignment | Fieldwise non-failing transfer that ends the source lifetime, with no user hook or value-category overload |
| Layered `mod` | const qualifiers, mutable references, capabilities | Slot and pointee permissions remain visually separate |
| `if1` in a type body | conditional members, macros, template/static conditionals | Selected declarations become normal HUC0; discarded body is not parsed |
| `fn1` partial specialization | overloads, traits, macros | Same primary/partial/full mechanism as structure and variable families |
| `let1` | variable templates, macros, generated locals | Selected leaf is a complete ordinary `let`, including type and ownership |
| `@fn1(...)` | constant evaluation or interpreter call | Same family either residualizes or must fully execute |
| Raw call-site emission | macros, mixins, compiler plugins | Compiler helper inherits a lexical HUC0 insertion cursor |
| Printable HUC0 boundary | internal compiler IR or generated source | Second public translator parses the exact intermediate source |
| Future semantic reflection | procedural macros, plugins, build generators | Stable compiler-only semantic snapshots plus explicit emission |

HUC’s goal is not to win a feature-count comparison. It aims to make these
mechanisms compose without C++ template substitution, value-category rules,
preprocessor macros, and separate ownership idioms all overlapping.

## 15. Example acceptance policy

As implementation progresses:

1. Every core HUC0 example becomes an HUC0 parser/type-check/execute test.
2. Every HUC1 example receives an exact expected `.huc0` golden output.
3. Every undefined-behavior example is marked and is not assigned a required
   runtime result.
4. Every probable standard-library name remains explicitly provisional until
   its library proposal is accepted.
5. Every future reflection example remains non-normative until the compiler
   API is separately specified.
6. Examples that intentionally show an invalid form must immediately provide
   and explain the corrected form.
7. Generated names shown as `__huc_*` are illustrative; the architecture’s
   stable mangling algorithm determines their final spelling.
