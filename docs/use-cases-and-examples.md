# HUC Use Cases and Detailed Examples

Status: design guide

Applies to: planned HUC 0.1 semantics

Related: [Implementation plan](implementation-plan.md)

This guide explains what HUC is useful for and how its runtime and staging
features work together. It includes small examples of individual rules and
larger examples that resemble real applications.

The compiler does not exist yet. These examples describe the intended design.
As features are implemented, each example must become either an executable
test or an expansion test that compares generated HUC0 with expected output
(a golden test).

## 1. Reading the examples

### 1.1 Core versus probable library APIs

The following syntax and behavior describe language rules, not library proposals:

- `let`, `let1`, and `let2`;
- `mod`;
- inline values and raw `T*` pointers;
- unary `&` for inline-place observation;
- `slot_off` for unchecked, untyped slot addresses;
- `construct_at` and `destruct_at` for manually managed value lifetimes;
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

They show how libraries could work. Their exact names, error types, allocator
APIs, and signatures are not yet settled.

Some terms used in the explanations:

- A **slot** or **place** is a storage location, such as a variable or field.
- A **pointee** is the object a pointer points to.
- **Relocation** transfers an Advanced value and makes its source inactive.
- A **family** describes a set of declarations selected by compile-time
  arguments. A **specialization** is one selected version.
- **Residual code** is the runtime code left after compile-time work.
  **Residualization** produces that code. **Materialization** writes a
  compiler-known value as valid HUC0 runtime syntax.

### 1.2 What “not achievable elsewhere” means

Other languages can often produce the same machine code using preprocessors,
macros, compiler plugins, external generators, or build scripts. This guide
does not claim that these results are possible only in HUC.

The useful distinction is:

- **Direct HUC expression:** the behavior is part of ordinary HUC syntax and
  has one specified selection, ownership, and generation model.
- **Simulation elsewhere:** comparable behavior normally needs a different
  mechanism, such as a macro language, build script, template trick, plugin, or
  convention.
- **HUC-specific rules:** HUC promises a particular combination of features
  and behavior, even when individual pieces exist elsewhere.

Notable HUC-specific combinations are:

1. One partial-specialization model for structures, functions, and variables.
2. A `let1` specialization that chooses the variable’s complete declaration,
   including a different type or mutation permission.
3. Typed phase conditions at module and type-body scope whose discarded bodies
   are never parsed.
4. Compiler helper functions that emit raw HUC0 at their expansion call site,
   followed by a separately inspectable and independently parsed HUC0 round.
5. A normal `fn1` call that generates and calls a runtime specialization, while
   the same call prefixed by `@` must run completely at compile time.
6. Fixed/read-only-by-default declarations, layer-specific `mod`, and fixed
   destructive relocation of Advanced values.

## 2. Use-case catalog

The examples below cover these use cases:

| Area | Use case |
|---|---|
| Runtime | Fixed-by-default local and field storage |
| Runtime | Mutable scalar, pointer slot, and pointee permissions |
| Runtime | Non-owning observation without lifetime tracking |
| Runtime | Explicit inline address-taking with unary `&` |
| Runtime | Deterministic cleanup of active values |
| Runtime | Full-expression temporary lifetimes without observer lifetime extension |
| Runtime | Consumption through an Advanced by-value parameter |
| Runtime | Advanced-containing structures and destructive relocation |
| Runtime | Ordinary copying of Basic values |
| Runtime | Explicit deep copying with `copy` and `clone` |
| Runtime | Custom clone opting a type into the Advanced category |
| Runtime | Fixed, non-overridable destructive relocation |
| Runtime | Relocation assignment, exact self-relocation, and reinitialization |
| Runtime | Address-dependent values, self-pointers, and stable storage |
| Runtime | Constructor-required fixed fields |
| Runtime | Deterministic default initialization |
| Runtime | Read-only and writable methods |
| Runtime | User `drop` plus automatic reverse field cleanup |
| Runtime | Direct correspondence with common C++20 RAII value types |
| Runtime | Left-to-right calls and expressions |
| Runtime | Checked conditional relocation without a borrow checker |
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
source: staging replaces each request with the name of a concrete type before the
same runtime semantics are checked by HUC0.

### 3.1 Fixed-by-default values

```huc
module examples.fixed_values;

fn calculate() -> i32 {
    let base: i32 = 40;
    let mod adjustment: i32 = 2;

    adjustment += 1;
    adjustment -= 1;

    // base += 1; // diagnostic: base is fixed
    return base + adjustment;
}
```

`let` declares storage. It does not mean the value can be changed. Use `mod`
to allow changes.

The expected result is `42`. A backend may map `base` to a C `const` local,
but the HUC rule still applies if a backend uses another representation.

### 3.2 Pointer permission layers

```huc
module examples.pointer_permissions;

struct Counter {
    let mod value: i32;

    fn init() {
    }

    fn read() -> i32 {
        return this->value;
    }

    fn increment() mod -> void {
        this->value += 1;
    }
}

fn observe(counter: Counter*) -> i32 {
    return counter->read();
}

fn mutate(counter: mod Counter*) -> void {
    counter->increment();
}

fn example() -> i32 {
    let mod counter: Counter = Counter();

    let fixed_readonly: Counter* = &counter;
    let fixed_writable: mod Counter* = &counter;
    let mod reseatable_readonly: Counter* = &counter;
    let mod reseatable_writable: mod Counter* = &counter;

    mutate(fixed_writable);
    reseatable_readonly = null;
    reseatable_writable = &counter;
    mutate(reseatable_writable);

    return observe(fixed_readonly);
}
```

This returns `2`.

Important distinctions:

- `mod` inside the type controls access to the pointee.
- `mod` before the binding name controls the pointer slot.
- Taking an inline value's address does not relocate it.
- None of the raw pointers keep the `Counter` alive.

Address-taking for inline storage is explicit and never overloadable:

```huc
fn inline_address_example() -> i32 {
    let mod counter: Counter = Counter();
    let observer: mod Counter* = &counter;
    observer->increment();
    return counter.read();
}
```

Unary `&` preserves pointee permission, so a fixed `Counter` would produce
`Counter*` and the writable `counter` above produces `mod Counter*`.
Unary `&` adds one pointer level when applied to a pointer slot. Prefix `&`
takes an address; binary `&` remains bitwise AND. Neither operation depends on
spacing.

Untyped slot addresses are a separate, unchecked facility:

```huc
fn pointer_slot_address_example() -> void {
    let mod counter: Counter = Counter();
    let mod pointer: Counter* = &counter;
    let address: usize = slot_off(pointer);
    let bytes: mod u8* = ptr_as<mod u8*>(address);
    // bytes addresses the pointer slot, not the Counter.
    // Taking these addresses does not transfer a value.
}
```

An ordinary function can receive `address`, but it receives no typed slot
reference or automatic lifecycle handling. Taking a fixed slot's address is
also permitted; writing actually fixed storage remains undefined behavior.
The caller is responsible for storage lifetime, alignment, valid access, and
cleanup. Pointer chains remain ordinary non-owning pointer types; this does
not add ownership or automatic cleanup.

Copying a raw pointer leaves its source unchanged, including when null. It
does not transfer or extend the pointee's lifetime.

### 3.3 Observation and consumption

```huc
module examples.consume;

struct FileHandle {
    let mod native: i32;

    fn init(native: i32) : native(native) {
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

fn inspect(file: FileHandle*) -> bool {
    return file->valid();
}

fn consume(file: FileHandle) -> void {
    if (file.valid()) {
        probable_log_file(&file);
    }
} // FileHandle.drop closes the transferred handle

fn run(native: i32) -> bool {
    let mod file: FileHandle = FileHandle(native);
    let was_valid: bool = inspect(&file);
    consume(file); // file becomes inactive
    return was_valid;
}
```

`inspect(&file)` observes the inline value. `consume(file)` transfers the
Advanced value and its cleanup obligation. The caller's source slot requires
binding `mod`.

Using `file` after `consume(file)` is a compile-time error. HUC does not
prevent saving the raw pointer from `inspect`, though. Using that pointer after
the source is transferred would access inactive storage, which is undefined
behavior.

### 3.4 Advanced-containing structures

Using the `FileHandle` type from the preceding example:

```huc
struct Session {
    let id: i32;
    let input: FileHandle;
    let output: FileHandle;

    fn init(id: i32, mod input: FileHandle, mod output: FileHandle)
        : id(id),
          input(input),
          output(output),
    {
    }
}

fn run(input_handle: i32, output_handle: i32) -> void {
    let mod source: Session =
        Session(10, FileHandle(input_handle), FileHandle(output_handle));
    let destination: Session = source;
}
```

`Session` is Advanced because its fields include Advanced values.
Transferring it moves the whole representation and makes the source and all
its fields inactive. No source bytes are cleared and no source cleanup runs.
At scope exit, `destination.output` is destroyed before `destination.input`.
Ordinary transfer out of an individual Advanced field remains prohibited.

### 3.5 Explicit deep copy

```huc
module examples.buffer_copy;

import std.memory as memory;

struct ByteBuffer {
    let mod data: mod u8*;
    let size: usize;

    fn init(size: usize)
        : data(memory::allocate_zeroed_bytes(size)),
          size(size),
    {
    }

    fn clone() -> ByteBuffer {
        let mod result: ByteBuffer = ByteBuffer(this->size);
        memory::copy_bytes(result.data, this->data, this->size);
        return result;
    }

    fn drop() mod -> void {
        memory::free_raw_bytes(this->data, this->size);
    }
}

fn duplicate(source: ByteBuffer*) -> ByteBuffer {
    return copy *source;
}
```

The pointer passed to `duplicate` must designate a live buffer. The explicit
`copy` calls `clone`, which duplicates the bytes into independent storage.
Both buffers may then be destroyed independently. Ordinary binding instead
transfers the Advanced value without calling `clone`.

Copying a raw `ByteBuffer*` alone copies only its address; it neither clones
the buffer nor keeps it alive.

### 3.6 Required field initialization and defaults

```huc
module examples.configuration;

struct Configuration {
    let port: i32;
    let secure: bool;
    let mod accepted_connections: i32;
    let mod scratch: u8*;

    fn init(port: i32, secure: bool)
        : port(port),
          secure(secure),
    {
        // accepted_connections becomes 0.
        // scratch becomes null.
    }
}

fn create() -> Configuration {
    return Configuration(443, true);
}
```

Removing either `port(port)` or `secure(secure)` is a compile-time error:
these fields are fixed and have no declaration initializers. Omitted mutable
fields receive the defaults specified by the language.

Initializer entries must appear in field declaration order and execute in
that same order.

### 3.7 User cleanup plus field cleanup

The following probable standard-library wrapper owns a native graphics object
and a heap allocation:

```huc
module examples.texture;

import std.memory as memory;

struct PixelBuffer {
    let mod bytes: mod u8*;
    let size: usize;

    fn init(size: usize)
        : bytes(memory::allocate_raw_bytes(size)),
          size(size),
    {
    }

    fn drop() mod -> void {
        memory::free_raw_bytes(this->bytes, this->size);
        this->bytes = null;
    }
}

struct Texture {
    let mod gpu_id: u32;
    let staging: PixelBuffer;

    fn init(gpu_id: u32, mod staging: PixelBuffer)
        : gpu_id(gpu_id),
          staging(staging),
    {
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

HUC lets a type define three lifecycle methods:

| Declaration | What it customizes |
|---|---|
| `fn init(...)` | Construction |
| `fn clone() -> T` | Explicit `copy` of an Advanced structure |
| `fn drop() mod -> void` | Cleanup before automatic field destruction |

`Advanced` is a type category, not the name of a user-overridable constructor.
Transferring a value in that category performs **destructive relocation**:

1. Transfer the whole value's representation with bitwise relocation semantics.
2. Make the entire source and all its subobjects inactive.
3. Run neither the source's `drop` body nor its automatic field cleanup.

Source fields require no clearing. This rule does not depend on whether
`clone`, `drop`, or an Advanced field made the structure Advanced.

There is no user move constructor, move-assignment operator, or relocation
hook. Relocation assignment first cleans an active destination and then applies
the same fixed operation. Exact self-relocation assignment is a no-op.

```huc
module examples.native_lease;

struct NativeLease {
    let mod handle: i32;
    let identity: u64;

    fn init(handle: i32, identity: u64)
        : handle(handle),
          identity(identity),
    {
    }

    fn clone() -> NativeLease {
        return NativeLease(
            probable_duplicate_handle(this->handle),
            this->identity,
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
    let mod original: NativeLease = NativeLease(
        probable_open_handle(),
        1001,
    );

    let duplicate: NativeLease = copy original; // calls clone
    let relocated: NativeLease = original; // destructive relocation
    // original is inactive; relocated holds the original handle.

    let mod current: NativeLease = NativeLease(
        probable_open_handle(),
        2002,
    );
    current = copy duplicate; // clone, drop current, relocate the temporary
    current = current; // exact self-relocation is a no-op
}
```

Declaring either `clone` or `drop` puts `NativeLease` in the Advanced category. The
logical copy duplicates the operating-system resource. Ordinary binding never
calls `clone`, and destructive relocation never calls user code. In particular,
the relocation does not write `-1` into `original.handle`: the whole source
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
and skips source cleanup. Using this directly transferred local again before
reinitializing it is a compile-time error. Access through a raw pointer or an
untyped address is still unchecked and is undefined when it accesses the
inactive value.

HUC makes the potentially expensive copy explicit and fixes relocation
behavior instead of selecting user code through C++ value categories. The
detailed comparison, including parameters, returns, resource-managing values,
assignment, exact self-relocation, and a complete image type, is in
[Value Semantics and Special Operations](value-semantics.md).

### 3.9 Address stability, self-pointers, and containers

Fixed destructive relocation deliberately has no callback that can repair a
value after its address changes. This matters for:

- a structure that stores `this` in one of its own fields;
- an intrusive node referenced by address from neighboring nodes;
- an object registered as callback user data in a C API;
- a native object whose ABI requires a stable address.

The following type opts into the Advanced category with `drop` so it cannot be
implicitly copied. Its constructor establishes a self-pointer:

```huc
module examples.address_dependent;

struct SelfIndexed {
    let mod self: SelfIndexed*;
    let key: u64;

    fn init(key: u64)
        : self(null),
          key(key),
    {
        this->self = this;
    }

    fn read_key_through_self() -> u64 {
        return this->self->key;
    }

    fn drop() mod -> void {
        // Empty cleanup still opts the structure into the Advanced category.
    }
}

fn unsafe_inline_relocation() -> u64 {
    let mod original: SelfIndexed = SelfIndexed(42);
    let relocated: SelfIndexed = original; // fixed destructive relocation

    // relocated.self still points at original's now-inactive storage.
    // Dereferencing it is undefined behavior.
    return relocated.read_key_through_self();
}
```

Bitwise relocation preserves the raw pointer field; it cannot know
that this particular pointer was meant to equal the object's current address.
The compiler need not reject the example. HUC is an unchecked systems
language, so the programmer must ensure that any stored addresses remain
correct.

One solution for a local value is to construct it in fixed storage and pass
only observers:

```huc
fn stable_local() -> u64 {
    let original: SelfIndexed = SelfIndexed(42);
    let observer: SelfIndexed* = &original;
    return observer->read_key_through_self();
}
```

The value remains at its construction address until cleanup. A pointer does
not keep it alive, so observers must not be used after that lifetime ends.

A contiguous `Vector<SelfIndexed>` may relocate elements when it grows,
erases, compacts, or sorts. That breaks the stored self-pointer even if the
compiler accepts the operation. Use stable-address storage or redesign the
representation. Mutable address-dependent values have the same requirement:
do not relocate them while their address-dependent invariants are needed.

Contiguous containers manage backing storage and initialized element ranges
explicitly, including preventing cleanup of retired source slots. They can
extract an Advanced element with `let mod value: T = *(data + index);` through
a `mod T*`, then update their bookkeeping to exclude that inactive slot from
cleanup. Extraction does not clear the source bytes or change the pointer
address. The returned value takes responsibility for the resource.

The compiler does not cancel automatic local or field cleanup through pointer
aliases. Passing `&local` to an extracting function and then letting that
inactive local receive automatic cleanup is undefined behavior. Direct
Advanced moves from fields or array indexing remain rejected; the explicit
pointer form leaves source cleanup to the programmer instead of tracking
partial lifetimes.

Use `construct_at(pointer, value)` to initialize an extracted or otherwise
inactive slot, and `destruct_at(pointer)` to destroy a live element without
freeing its backing storage. Both require `mod T*` and leave container
bookkeeping to the programmer. Ordinary indirect assignment instead assumes
a live destination and destroys its old value before replacement.

A future custom relocation hook should be considered only if real
address-repair cases justify changing the fixed-transfer model.

### 3.10 Left-to-right evaluation

```huc
module examples.order;

let mod trace: i32;

fn next(digit: i32) -> i32 {
    trace = trace * 10 + digit;
    return digit;
}

fn combine(a: i32, b: i32, c: i32) -> i32 {
    return a * 100 + b * 10 + c;
}

fn run() -> i32 {
    trace = 0;
    let result: i32 = combine(next(1), next(2), next(3));

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

### 3.11 Copy before transfer in one call

```huc
module examples.copy_then_transfer;

struct Item {
    let value: i32;

    fn init(value: i32) : value(value) {
    }

    fn clone() -> Item {
        return Item(this->value);
    }
}

fn take(first: Item, second: Item) -> i32 {
    return first.value + second.value;
}

fn run() -> i32 {
    let mod item: Item = Item(7);
    return take(copy item, item);
}
```

The result is `14`. The first argument clones the value before the second
argument transfers it. `take(item, item)` and `take(item, copy item)` are
compile-time errors: the first binding leaves `item` inactive before the
second argument names it. There is no implicit empty value after transfer.

### 3.12 Conditional relocation is checked

```huc
module examples.conditional_relocation;

struct Job {
    let id: i32;
    fn init(id: i32) : id(id) {}

    fn drop() mod -> void {
        probable_finish_job(this->id);
    }
}

fn consume(job: Job) -> void {
}

fn run(transfer: bool) -> i32 {
    let mod job: Job = Job(9);

    if (transfer) {
        consume(job);
        return 0; // job is inactive, so no second cleanup
    }

    return job.id; // this path still has an active job
}
```

Every path that reaches `return job.id` still holds an active `job`, because
the transferring path has already returned. The compiler accepts this.
Without the early return, `job` would be MaybeActive after the `if`, and
reading it there is a compile-time error rather than undefined behavior:

```huc
fn run_incorrectly(transfer: bool) -> i32 {
    let mod job: Job = Job(9);

    if (transfer) {
        consume(job);
    }

    return job.id; // error: job is MaybeActive here
}
```

The checker does not predict condition results or connect separate conditions.
It considers both outcomes even if other reasoning could prove that
`transfer` is false. Adding a separate `if (!transfer)` around the read is
therefore not enough; use the `else` branch of the original `if`, return from
the transferring branch, or reinitialize `job` there. Each option makes the
active paths clear without proving relationships between conditions.
The compiler still arranges exactly-once cleanup, using control-flow knowledge
or a separate drop flag when necessary.

### 3.12.1 Temporary views and expression boundaries

Assume `make_text()` returns an owning Advanced `Text`, and `view()` returns
a non-owning `View` into its characters:

```huc
print(make_text().view()); // temporary Text survives print, then is destroyed

let view: View = make_text().view(); // Text is destroyed at this semicolon
print(view); // invalid: view no longer refers to live text
```

A view does not keep its source alive. Name the owning value when the view
must remain usable across statements:

```huc
let text: Text = make_text();
let view: View = text.view();
print(view); // valid while text remains alive and unchanged
```

The whole call expression, not each argument or nested call, is the lifetime
boundary for its unconsumed temporaries. A condition's temporaries are instead
destroyed before its branch or loop body starts. Transferred values follow
their destination lifetimes; the caller does not destroy a transferred
temporary again at the semicolon. See
[temporary lifetime rules](value-semantics.md#91-when-temporary-values-are-destroyed).

### 3.13 Probable file-processing application

This larger example uses illustrative standard-library APIs:

```huc
module examples.copy_file;

import std.fs as fs;
import std.io as io;
import std.result as result;
import std.text as text;

struct CopyStats {
    let mod bytes: usize;
    let mod chunks: usize;

    fn init() {
    }

    fn add(count: usize) mod -> void {
        this->bytes += count;
        this->chunks += 1;
    }
}

fn copy_stream(mod input: fs::Reader, mod output: fs::Writer)
    -> result::Result<CopyStats, fs::Error> {
    let mod stats: CopyStats = CopyStats();
    let mod buffer: fs::Buffer = fs::Buffer(64 * 1024);

    while (true) {
        let mod count: result::Result<usize, fs::Error> =
            input.read(buffer.writable_bytes());

        if (count.is_error()) {
            return result::Result<CopyStats, fs::Error>::error(
                count.take_error(),
            );
        }

        if (count.value() == 0) {
            break;
        }

        let mod written: result::Result<void, fs::Error> =
            output.write_all(buffer.bytes(0, count.value()));

        if (written.is_error()) {
            return result::Result<CopyStats, fs::Error>::error(
                written.take_error(),
            );
        }

        stats.add(count.value());
    }

    return result::Result<CopyStats, fs::Error>::value(stats);
}

fn run(mod source: text::Text, mod destination: text::Text) -> i32 {
    let mod opened_input: result::Result<fs::Reader, fs::Error> =
        fs::open_reader(source);

    if (opened_input.is_error()) {
        io::error_line(opened_input.error().message());
        return 1;
    }

    let mod input: fs::Reader = opened_input.take_value();

    let mod opened_output: result::Result<fs::Writer, fs::Error> =
        fs::create_writer(destination);

    if (opened_output.is_error()) {
        io::error_line(opened_output.error().message());
        return 1;
    }

    let mod output: fs::Writer = opened_output.take_value();
    let copied: result::Result<CopyStats, fs::Error> =
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

The standard-library API is not settled, but the intended value behavior is:

- `open_reader` returns a result containing an Advanced stream value.
- `copy_stream` consumes both stream values.
- early returns clean up still-active values.
- buffers and errors use ordinary value semantics.

The illustrative result-extraction methods must manage their initialized
payload state explicitly. Pointer extraction can transfer a manually managed
payload, but it does not disable automatic field cleanup. `construct_at` and
`destruct_at` provide payload lifetime operations; allocation and representation
remain library design choices. Direct moves from Advanced fields remain rejected.

## 4. HUC1 control-flow examples

### 4.1 Global declaration selection

```huc
module examples.platform_clock;

let2 use_monotonic_clock: bool = true;

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

let2 compact_header: bool = true;

struct PacketHeader {
    let kind: u16;

    if1 (compact_header) {
        let length: u16;
    } else {
        let length: u64;
        let checksum: u32;
    }

    fn init(
        kind: u16,
        length: if1 (compact_header) u16 else u64,
    ) : kind(kind),
        length(length),
    {
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
    let kind: u16;

    if1 (compact_header) {
        let length: u16;

        fn init(kind: u16, length: u16)
            : kind(kind),
              length(length),
        {
        }
    } else {
        let length: u64;
        let mod checksum: u32;

        fn init(kind: u16, length: u64)
            : kind(kind),
              length(length),
        {
        }
    }
}
```

Compact HUC0:

```huc
struct PacketHeader {
    let kind: u16;
    let length: u16;

    fn init(kind: u16, length: u16)
        : kind(kind),
          length(length),
    {
    }
}
```

This is useful for protocol versions, embedded targets, optional debugging
fields, and architecture-dependent layouts.

### 4.3 An opaque discarded branch

```huc
module examples.opaque_branch;

let2 building_linux: bool = true;

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
the matching brace. Unlike `if constexpr`-style discarding, this skips parsing
the body itself. Traditional textual preprocessors can also skip text that is
not valid language syntax.

If `building_linux` becomes false, the selected body is parsed and compilation
fails until it is replaced with valid HUC.

### 4.4 Module-scope `while1`

```huc
module examples.generated_probes;

let2 mod index: usize = 0;

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

The loop runs only at compile time. Its body may execute compiler statements
such as `index += 1` while producing runtime module declarations.

### 4.5 Future: `for1` over a compile-time pack

Open-ended phase packs are not in HUC 0.1. This example shows possible future
syntax and its expected runtime output. The initial HUC1 grammar does not
accept it.

```huc
module examples.sum_constants;

fn1 sum_constants(Values: i32...)() -> i32 {
    let mod result: i32 = 0;

    for1 (let2 value: i32 in Values) {
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
    let mod result: i32 = 0;
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

struct1 Storage(T: auto) {
    let value: T;

    fn init(mod value: T) : value(value) {
    }

    fn get() -> T* {
        return &this->value;
    }
}

struct1 Storage(T: auto)<T*> {
    let value: T*;

    fn init(value: T*) : value(value) {
    }

    fn get() -> T* {
        return this->value;
    }
}

struct1 Storage()<u8> {
    let value: u8;

    fn init(value: u8) : value(value) {
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
let number: Storage<i32> = Storage<i32>(42); // primary
let pointer: Storage<i32*> = Storage<i32*>(null); // partial
let byte: Storage<u8> = Storage<u8>(15); // full
```

The full specialization wins over the primary for `u8`; the pointer pattern
wins for every raw-pointer type.

### 5.2 Structural pattern plus predicate

This example assumes a probable library `InlineArray<T, N>` family; primitive
`[N]T` arrays are deferred from HUC 0.1.

```huc
struct1 Buffer(T: auto, N: usize) {
    let mod values: InlineArray<T, N>;

    fn init() {
    }
}

struct1 Buffer(T: auto, N: usize)<T*, N>(N <= 16) {
    let mod values: InlineArray<T*, N>;

    fn init() {
    }
}
```

This partial specialization can be selected only for pointer elements and
small capacities. Its predicate filters candidates; it does not make this
pattern outrank another pattern that is structurally more specific.

### 5.3 Function partial specialization

```huc
module examples.hash;

fn1 hash_value(T: auto)(value: T) -> u64 {
    return probable_hash_bytes(&value, size_of<T>());
}

fn1 hash_value(T: auto)<T*>(value: T*) -> u64 {
    if (!value) {
        return 0;
    }

    return probable_hash_bytes(value, size_of<T>());
}

fn run(value: i32, pointer: i32*) -> u64 {
    let first: u64 = hash_value(value); // T deduced as i32
    let second: u64 = hash_value(pointer); // pointer partial selected
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
    let id: i32;
    fn init(id: i32) : id(id) {}
}

fn run() -> i32 {
    let1 slot(T: auto) {
        let slot: T = T();
    }

    let1 slot(T: auto)<T*> {
        let mod slot: T* = null;
    }

    let1 slot()<Widget> {
        let mod slot: Widget = Widget(99);
    }

    let first: i32 = slot<i32>;
    let second: Widget* = slot<Widget*>;
    let third: i32 = slot<Widget>.id;

    if (second) {
        return -1;
    }

    return first + third;
}
```

Illustrative HUC0 at the family declaration site:

```huc
let __huc_slot_i32: i32 = i32();
let mod __huc_slot_Widget_ptr: Widget* = null;
let mod __huc_slot_Widget: Widget = Widget(99);

let first: i32 = __huc_slot_i32;
let second: Widget* = __huc_slot_Widget_ptr;
let third: i32 = __huc_slot_Widget.id;
```

The same family name can produce:

- an inline `i32`;
- a raw `Widget*`;
- a writable inline `Widget`.

This is one of HUC’s most unusual direct facilities. C++ variable templates can
also be explicitly/partially specialized, but HUC integrates the selected
ordinary `let` declaration, mutation permissions, phase control, and HUC0
materialization into the same numbered model.

### 5.5 `let1` selecting mutability

```huc
fn configure(writable: bool) -> void {
    let1 setting(Writable: bool) {
        if1 (Writable) {
            let mod setting: i32;
        } else {
            let setting: i32 = 7;
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

The family decides whether the generated variable itself is writable.
Because `writable` must be a compile-time value for `if1`, a runtime boolean is
not accepted here; the example’s parameter would need to be a family binder or
phase-2 input in the finalized grammar.

A correct family form is:

```huc
fn1 configure(Writable: bool)() -> void {
    let1 setting(Choice: bool) {
        if1 (Choice) {
            let mod setting: i32;
        } else {
            let setting: i32 = 7;
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

fn1 biased_add(Bias: i32)(left: i32, right: i32) -> i32 {
    return left + right + Bias;
}

fn runtime_case(left: i32, right: i32) -> i32 {
    return biased_add<5>(left, right);
}

fn compile_case() -> i32 {
    return @biased_add<5>(10, 20);
}
```

HUC0:

```huc
fn __huc_biased_add_5(left: i32, right: i32) -> i32 {
    return left + right + 5;
}

fn runtime_case(left: i32, right: i32) -> i32 {
    return __huc_biased_add_5(left, right);
}

fn compile_case() -> i32 {
    return 35;
}
```

The normal call produces a specialized runtime function and calls it at
runtime. `@` runs the call completely at compile time and writes its result
as a HUC0 value.

### 6.2 Phase-2 helper and evaluation island

`InlineArray` below is a probable later library family; it is used to show
phase-value materialization, not as a HUC 0.1 primitive.

```huc
module examples.phase_helper;

fn2 clamp_build_value(value: i32, low: i32, high: i32) -> i32 {
    if (value < low) {
        return low;
    }

    if (value > high) {
        return high;
    }

    return value;
}

let2 selected_capacity: i32 = @clamp_build_value(1000, 16, 256);

struct Cache {
    let mod bytes: InlineArray<u8, selected_capacity>;
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
    compiler::emit_huc("let mod generated_counter: i32;");
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
    let mod generated_counter: i32;
    generated_counter = 42;
    return generated_counter;
}
```

The helper is defined at module scope, but it writes code at its call site
inside `run`. That output position is its inherited insertion cursor.

### 7.2 Module generation from a helper

```huc
module examples.module_emit;

import2 compiler;

fn2 generate_health_check() -> void {
    compiler::emit_huc_module(
        "fn health_check() -> i32 { return 200; }",
    );
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
    "if1 (true) { fn illegal() -> i32 { return 1; } }",
);
```

The HUC1 translator writes the string without parsing it. The HUC0 translator
then rejects `if1`. Raw generation does not run staging a second time: its
output must already be valid HUC0.

## 8. Large case study: target-specialized small vector

This forward-looking case study assumes the later `InlineArray<T, N>` library
facility. The 0.1 core deliberately defers primitive fixed arrays.

This example combines:

- a family selected by element type and inline capacity;
- fixed-by-default fields;
- explicitly managed spill storage;
- phase-selected layout;
- runtime destructive relocation and cleanup;
- probable allocator APIs.

For clarity, this bootstrap sketch assumes that `T` is Basic. A production
standard-library version would state that requirement through the future
compiler type-query API or add element-wise relocation/drop handling.

```huc
module case_studies.small_vector;

import std.memory as memory;
import std.io as io;

struct1 HeapArray(T: auto) {
    let mod values: mod T*;
    let capacity: usize;

    fn init(capacity: usize)
        : capacity(capacity) {
        if (capacity != 0) {
            this->values = memory::allocate_raw_items<T>(capacity);
        }
    }

    fn drop() mod -> void {
        if (this->values) {
            memory::free_raw_items<T>(this->values, this->capacity);
            this->values = null;
        }
    }
}

struct1 SmallVector(T: auto, Inline: usize) {
    let mod inline_values: InlineArray<T, Inline>;
    let mod heap_values: HeapArray<T>;
    let mod size: usize;
    let mod capacity: usize;

    fn init()
        : heap_values(HeapArray<T>(0)),
          capacity(Inline),
    {
    }

    fn using_heap() -> bool {
        return this->heap_values.values != null;
    }

    fn data() -> T* {
        if (this->using_heap()) {
            return this->heap_values.values;
        }

        return &this->inline_values[0];
    }

    fn data_mut() mod -> mod T* {
        if (this->using_heap()) {
            return this->heap_values.values;
        }

        return &this->inline_values[0];
    }

    fn reserve(requested: usize) mod -> void {
        if (requested <= this->capacity) {
            return;
        }

        let mod next_capacity: usize = this->capacity * 2;
        if (next_capacity < requested) {
            next_capacity = requested;
        }

        let mod replacement: HeapArray<T> =
            HeapArray<T>(next_capacity);

        let mod index: usize = 0;
        while (index < this->size) {
            replacement.values[index] = this->data_mut()[index];
            index += 1;
        }

        this->heap_values = replacement;
        this->capacity = next_capacity;
    }

    fn push(mod value: T) mod -> void {
        this->reserve(this->size + 1);
        this->data_mut()[this->size] = value;
        this->size += 1;
    }

    fn get(index: usize) -> T* {
        // Bounds remain unchecked in the core language.
        return &this->data()[index];
    }

}

struct1 SmallVector(T: auto, Inline: usize)<T*, Inline>(Inline <= 4) {
    let mod values: InlineArray<T*, Inline>;
    let mod size: usize;

    fn init() {
    }

    fn push(value: T*) mod -> void {
        // This compact specialization deliberately has no spill path.
        // Writing past Inline is undefined in the unchecked profile.
        this->values[this->size] = value;
        this->size += 1;
    }

    fn get(index: usize) -> T* {
        return this->values[index];
    }
}

fn run() -> i32 {
    let mod numbers: SmallVector<i32, 8> =
        SmallVector<i32, 8>();

    numbers.push(10);
    numbers.push(20);
    numbers.push(12);

    let mod pointers: SmallVector<i32*, 4> =
        SmallVector<i32*, 4>();

    pointers.push(numbers.get(0));
    pointers.push(numbers.get(1));

    io::print_line(*pointers.get(0) + *pointers.get(1));
    return *numbers.get(2);
}
```

This illustrates the design; it is not a finished, safety-checked container:

- The pointer specialization is chosen structurally.
- The predicate limits the compact form to capacities up to four.
- The general form contains an inline resource-managing spill buffer.
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
    let2 name: compiler::Text;
    let2 offset: usize;
    let2 writable: bool;

    fn2 init(
        name: compiler::Text,
        offset: usize,
        writable: bool,
    )
        : name(name),
          offset(offset),
          writable(writable),
    {
    }
}

fn2 emit_register(spec: RegisterSpec) -> void {
    if (spec.writable) {
        compiler::emit_huc_module(
            compiler::format(
                "fn write_{0}(base: mod u32*, value: u32) -> void { "
                "base[{1}] = value; "
                "}",
                spec.name,
                spec.offset / 4,
            ),
        );
    }

    compiler::emit_huc_module(
        compiler::format(
            "fn read_{0}(base: u32*) -> u32 { "
            "return base[{1}]; "
            "}",
            spec.name,
            spec.offset / 4,
        ),
    );
}

@emit_register(RegisterSpec("status", 0x00, false));
@emit_register(RegisterSpec("control", 0x04, true));
@emit_register(RegisterSpec("baud", 0x08, true));

fn initialize_uart(base: mod u32*, baud: u32) -> void {
    write_control(base, 0);
    write_baud(base, baud);
    write_control(base, 1);
}
```

Expected generated declarations conceptually include:

```huc
fn read_status(base: u32*) -> u32 {
    return base[0];
}

fn write_control(base: mod u32*, value: u32) -> void {
    base[1] = value;
}

fn read_control(base: u32*) -> u32 {
    return base[1];
}

fn write_baud(base: mod u32*, value: u32) -> void {
    base[2] = value;
}

fn read_baud(base: u32*) -> u32 {
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

let2 protocol_has_checksum: bool = true;

struct1 WireInteger(Bits: usize) {
    if1 (Bits == 8) {
        let value: u8;
        fn init(value: u8) : value(value) {}
    } else {
        if1 (Bits == 16) {
            let value: u16;
            fn init(value: u16) : value(value) {}
        } else {
            let value: u32;
            fn init(value: u32) : value(value) {}
        }
    }
}

struct MessageHeader {
    let kind: WireInteger<16>;
    let payload_size: WireInteger<16>;

    if1 (protocol_has_checksum) {
        let checksum: WireInteger<32>;
    }

    if1 (protocol_has_checksum) {
        fn init(kind: u16, payload_size: u16, checksum: u32)
            : kind(WireInteger<16>(kind)),
              payload_size(WireInteger<16>(payload_size)),
              checksum(WireInteger<32>(checksum)),
        {
        }
    } else {
        fn init(kind: u16, payload_size: u16)
            : kind(WireInteger<16>(kind)),
              payload_size(WireInteger<16>(payload_size)),
        {
        }
    }
}

struct Packet {
    let header: MessageHeader;
    let mod payload: net::ByteBuffer;

    fn init(header: MessageHeader, mod payload: net::ByteBuffer)
        : header(header),
          payload(payload),
    {
    }
}

fn send_packet(socket: net::Socket*, packet: Packet)
    -> result::Result<void, net::Error> {
    let mod header_result: result::Result<void, net::Error> =
        socket->write(probable_bytes_of(&packet.header));

    if (header_result.is_error()) {
        return header_result;
    }

    return socket->write(
        packet.payload.bytes(
            0,
            packet.header.payload_size.value,
        ),
    );
}

fn run(
    socket: net::Socket*,
    mod payload: net::ByteBuffer,
    size: usize,
) -> i32 {
    let checksum: u32 = probable_checksum(payload.bytes(0, size));
    let header: MessageHeader =
        MessageHeader(7, as<u16>(size), checksum);
    let mod packet: Packet = Packet(header, payload);

    let sent: result::Result<void, net::Error> =
        send_packet(socket, packet);

    if (sent.is_error()) {
        io::error_line(sent.error().message());
        return 1;
    }

    return 0;
}
```

The HUC0 output has exactly one header layout and constructor. Runtime packet
transfer and cleanup are independent of the compile-time layout mechanism.

## 11. Large case study: generated command registry

This case demonstrates module-scope generation and direct runtime dispatch.
Raw generation requires detailed text construction. The future structured
compiler API is intended to add name hygiene: generated names would not
accidentally clash with or refer to existing names.

```huc
module case_studies.commands;

import2 compiler;
import std.io as io;
import std.text as text;

fn command_build(arguments: text::Text) -> i32 {
    io::print("building ");
    io::print_line(arguments);
    return 0;
}

fn command_clean(arguments: text::Text) -> i32 {
    io::print("cleaning ");
    io::print_line(arguments);
    return 0;
}

fn command_test(arguments: text::Text) -> i32 {
    io::print("testing ");
    io::print_line(arguments);
    return 0;
}

fn2 emit_dispatcher() -> void {
    compiler::emit_huc_module(
        "fn dispatch(name: text::Text, "
        "mod arguments: text::Text) -> i32 {"
        "  if (name == \"build\") { return command_build(arguments); }"
        "  if (name == \"clean\") { return command_clean(arguments); }"
        "  if (name == \"test\") { return command_test(arguments); }"
        "  io::error_line(\"unknown command\");"
        "  return 2;"
        "}",
    );
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
illustrates a family that selects value-parameter handling for Basic and
Advanced components, while storing both inline.

The predicate helpers are probable future compiler type queries.

```huc
module case_studies.ecs;

import2 compiler;
import std.collections as collections;
import std.memory as memory;

struct Position {
    let mod x: f32;
    let mod y: f32;
    fn init(x: f32, y: f32) : x(x), y(y) {}
}

struct Sprite {
    let texture: u32;
    let mod pixels: memory::ByteBuffer;

    fn init(texture: u32, mod pixels: memory::ByteBuffer)
        : texture(texture),
          pixels(pixels),
    {
    }
}

struct1 ComponentStore(T: auto) {
    if1 (@compiler::type<T>().is_advanced()) {
        let mod values: collections::Vector<T>;

        fn add(mod value: T) mod -> usize {
            this->values.push(value);
            return this->values.size() - 1;
        }

        fn get(index: usize) -> T* {
            return this->values.pointer_at(index);
        }
    } else {
        let mod values: collections::Vector<T>;

        fn add(value: T) mod -> usize {
            this->values.push(value);
            return this->values.size() - 1;
        }

        fn get(index: usize) -> T* {
            return this->values.pointer_at(index);
        }
    }

    fn init() {
    }
}

struct World {
    let mod positions: ComponentStore<Position>;
    let mod sprites: ComponentStore<Sprite>;

    fn init() {
    }
}
```

This exact predicate needs the future reflection API, so it is outside the
first implementation. It shows why HUC plans to provide stable type metadata:

- Basic components can be stored inline.
- Advanced components transfer into storage and receive exactly-once cleanup.
- users interact with a uniform `get` observer;
- each concrete store is generated in HUC0;
- runtime storage contains no reflection metadata.

## 13. Future compiler/reflection use cases

The APIs in this section are proposals, not current language rules. They
describe the capabilities required from the future `import2 compiler` design.

### 13.1 Environment inspection

```huc
import2 compiler;

fn2 report_environment() -> void {
    let2 environment: compiler::Environment =
        compiler::current_environment();

    compiler::log(environment.target().triple());
    compiler::log(environment.language_revision());

    for1 (let2 module: compiler::Module in environment.modules()) {
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

fn2 generate_serializer(type: compiler::Type) -> void {
    let2 record: compiler::Class = type.as_class();
    let2 mod body: compiler::Code = compiler::Code();

    for1 (let2 field: compiler::Field in record.fields()) {
        if1 (!field.attributes().contains("skip")) {
            body.append(
                compiler::statement(
                    "writer.write_field({name}, value.{field})",
                    field.name(),
                    field.identifier(),
                ),
            );
        }
    }

    compiler::emit_module(
        compiler::function(
            "serialize_" + record.name(),
            body.commit(),
        ),
    );
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

- Which types contain Advanced fields?
- Which functions consume an Advanced parameter?
- Which structures have a `drop` method?
- Which declarations and attributes exist in each imported module?
- Which statements perform unchecked pointer arithmetic?
- Which generated declarations originated from this family?
- Which concrete specializations exist?
- Which modules depend on a selected declaration?

Answers are objects describing checked code or read-only query results, not
direct access to the compiler's internal containers.

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
| `init` / `clone` / `drop` | C++ special members, RAII wrappers | Custom construction, explicit logical copy, and cleanup |
| Fixed destructive relocation | C++ move constructors and move assignment | Bitwise non-failing transfer of inline Advanced values that ends the source lifetime, with no user hook or value-category overload |
| Layered `mod` | const qualifiers, mutable references, capabilities | Slot and pointee permissions remain visually separate |
| `if1` in a type body | conditional members, macros, template/static conditionals | Selected declarations become normal HUC0; discarded body is not parsed |
| `fn1` partial specialization | overloads, traits, macros | Same primary/partial/full mechanism as structure and variable families |
| `let1` | variable templates, macros, generated locals | Selected leaf is a complete ordinary `let`, including type and mutation permissions |
| `@fn1(...)` | constant evaluation or interpreter call | Same family either residualizes or must fully execute |
| Raw call-site emission | macros, mixins, compiler plugins | Compiler helper inherits a lexical HUC0 insertion cursor |
| Printable HUC0 boundary | internal compiler IR or generated source | Second public translator parses the exact intermediate source |
| Future semantic reflection | procedural macros, plugins, build generators | Stable compiler-only semantic snapshots plus explicit emission |

HUC's goal is not to have the most features. It aims to make these mechanisms
work together without the overlap between C++ template substitution,
value-category rules, preprocessor macros, and separate ownership patterns.

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
