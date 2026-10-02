---
name: rust-review
description: Review rust code in a set of files specified by the user.
---

## Scope

Restrict the review comments to code that is within the current project and which can be modified by the author to
improve it.

In the rules below, when referring to "type" generally we are referring to a type alias or struct definition
within the current crate.

### Files to Review

By default, if the user doesn't specify otherwise, review all files that have changed in git and have not been
committed yet.

Other sets of tiles the user can specify:
- a specific file (Rust file name specified)
- all files under a specified directory path (directory name supplied)
- project: review all rust-related files (including Cargo.toml) in the project
- a module name: find module(s) in the project that match the name. If more than one, ask the user to clarify which.
Then review all files in the specified module recursively down, including files that are behind features and files
that are referenced from elsewhere via a "path=" construct in module files

## Idiomatic Rust
Key characteristics of idiomatic Rust include:

- Embracing the Type System: Using enums and structs to model data, allowing the compiler to perform exhaustive 
matching and catch missing cases automatically.
- Functional Patterns: Preferring iterators, closures, and combinators (like `map`, `filter`, `fold`) over manual loops 
to write concise and parallelizable code.
- Error Handling: Utilizing Option and Result types with pattern matching (match, if let) and the ? operator 
instead of exceptions or manual null checks.
- Ownership and Borrowing: Writing code that works with the borrow checker to ensure memory safety without 
garbage collection, avoiding manual memory management.
- Trait-Based Design: Using traits for shared behavior and polymorphism instead of inheritance, and implementing 
standard traits (like Display, Debug, From) to integrate with the ecosystem.
- Tooling Compliance: Adhering to community standards such as cargo fmt for formatting and cargo clippy for style 
and best practices.

## Review Checklist (Summary)

### Issues the compiler cannot catch

- [ ] Boundary conditions are handled correctly
- [ ] State machine transitions are complete
- [ ] Race conditions in concurrent scenarios
- [ ] Public APIs are hard to misuse
- [ ] Type signatures clearly express intent
- [ ] Error type granularity is appropriate

### Readability

- [ ] No magic numbers; constants have descriptive names
- [ ] `cargo clippy` produces zero warnings
- [ ] `cargo fmt` has been applied
- [ ] Doc comments are complete
- [ ] Tests cover boundary conditions
- [ ] Public APIs have documented examples

### Ownership and Borrowing

- [ ] `clone()` is intentional and documented with a reason
- [ ] `Arc<Mutex<T>>` truly needs shared state
- [ ] `RefCell` usage is justified
- [ ] Lifetimes are not overly complex
- [ ] `Cow` considered to avoid unnecessary allocations

### Unsafe Code (Most Important)

- [ ] Every `unsafe` block has a `SAFETY` comment
- [ ] Every `unsafe fn` has a `# Safety` doc section
- [ ] Comments explain *why* it is safe, not just *what* it does
- [ ] Required invariants are listed
- [ ] Unsafe boundary is as small as possible
- [ ] Safe alternatives have been considered
- [ ] Unnecessary `unsafe` markers are removed

### Error Handling

- [ ] Libraries use `thiserror` for structured errors
- [ ] Applications use `anyhow` + `.context()`
- [ ] No `unwrap()` / `expect()` in production code
- [ ] Error messages are helpful for debugging
- [ ] `#[must_use]` return values are handled
- [ ] `#[source]` preserves the error chain

### Performance

- [ ] No unnecessary `collect()` or intermediate allocations
- [ ] Large data passed by reference
- [ ] Strings use `&str`, `with_capacity`, `join`, or `write!` as appropriate
- [ ] `impl Trait` vs `Box<dyn Trait>` choice is appropriate
- [ ] Hot paths avoid allocations

### Trait Design

- [ ] Traits introduced only when genuine polymorphism is needed
- [ ] Generics preferred over trait objects unless heterogeneous collections are required
- [ ] `impl Trait` used in return position when concrete type is unimportant

### Concurrency

- [ ] Lock acquisition order is consistent
- [ ] Channel buffer sizes are reasonable
- [ ] `JoinHandle` results are handled properly
- [ ] `join!` / `try_join!` preferred for structured concurrency

### Async

- [ ] No blocking in async (`std::fs`, `thread::sleep`)
- [ ] No `std::sync` lock held across `.await`
- [ ] Spawned tasks satisfy `'static`
- [ ] `spawn` is only used for genuinely parallel workloads
- [ ] Simple operations are awaited directly, not spawned
- [ ] Task lifecycle and shutdown strategy are considered
- [ ] Futures in `select!` are cancel-safe
- [ ] Async functions document their cancel safety
- [ ] Cancellation does not cause data loss or inconsistent state
- [ ] `tokio::pin!` is used correctly for futures that need reuse

### Function Design

- [ ] Constructors use `Type::new()` or `Default::default()`
- [ ] Free functions that take a type are candidates for methods
- [ ] Similar functions are consolidated where possible

### Type Design

- [ ] Strong types used instead of primitive obsession
- [ ] Groups of booleans replaced with enums where appropriate
- [ ] TypeState pattern considered for stateful types

### Cargo Review

- [ ] No unused or unnecessarily non-optional dependencies
- [ ] No duplicate crate versions that could be unified
- [ ] No redundant dependencies that serve the same purpose

---

## Readability

### Avoid magic numbers

Avoid the use of literals for important constants as their meaning is opaque. Replace them with named
constants whose names describe the value, with a comment on the definition if needed.

```rust
// BAD: meaning of 86400 is unclear at the call site
fn is_expired(timestamp: u64, now: u64) -> bool {
    now - timestamp > 86400
}

// GOOD: named constant makes intent obvious
/// Number of seconds in one day.
const SECONDS_PER_DAY: u64 = 86_400;

fn is_expired(timestamp: u64, now: u64) -> bool {
    now - timestamp > SECONDS_PER_DAY
}
```

---

## Function Design

### Constructors

If a function instantiates a new instance of a type from the current crate, consider making it an associated
function (`new` or similar) on the type. Also consider implementing `Default::default` when there is a natural
default state.

```rust
// BAD: free function acting as constructor
fn create_connection(host: &str, port: u16) -> Connection {
    Connection { host: host.to_string(), port, retries: 3 }
}

// GOOD: associated constructor function
impl Connection {
    fn new(host: &str, port: u16) -> Self {
        Self { host: host.to_string(), port, retries: 3 }
    }
}

// GOOD: implement Default when a natural default exists
impl Default for Connection {
    fn default() -> Self {
        Self { host: "localhost".to_string(), port: 8080, retries: 3 }
    }
}
```

### Free functions that could be methods

If a function takes an instance of (or a reference to) a struct defined in the crate, it is a candidate for
becoming a method on that type. Methods improve discoverability and allow method chaining.

```rust
// BAD: free function operating on a crate-local struct
fn validate_order(order: &Order) -> Result<(), OrderError> {
    if order.items.is_empty() { return Err(OrderError::Empty); }
    Ok(())
}

// GOOD: method on the type
impl Order {
    fn validate(&self) -> Result<(), OrderError> {
        if self.items.is_empty() { return Err(OrderError::Empty); }
        Ok(())
    }
}
```

---

## Duplication

### Repeated and similar functions

Identify repeated functions across the code base that can be pulled out to associated functions or helper
methods. Also look for functions of reasonable size that share most of their logic and differ only in a small
way -- these can often be combined with an additional parameter or a closure to vary the behavior.

```rust
// BAD: two nearly-identical functions
fn send_email_notification(user: &User, message: &str) -> Result<()> {
    let formatted = format!("Dear {}, {}", user.name, message);
    email_client().send(&user.email, &formatted)?;
    log::info!("Sent email to {}", user.name);
    Ok(())
}

fn send_sms_notification(user: &User, message: &str) -> Result<()> {
    let formatted = format!("Dear {}, {}", user.name, message);
    sms_client().send(&user.phone, &formatted)?;
    log::info!("Sent SMS to {}", user.name);
    Ok(())
}

// GOOD: unified with a channel abstraction
enum Channel { Email, Sms }

fn send_notification(user: &User, message: &str, channel: Channel) -> Result<()> {
    let formatted = format!("Dear {}, {}", user.name, message);
    match channel {
        Channel::Email => email_client().send(&user.email, &formatted)?,
        Channel::Sms   => sms_client().send(&user.phone, &formatted)?,
    }
    log::info!("Sent {:?} notification to {}", channel, user.name);
    Ok(())
}
```

---

## Type Design

### Groups of booleans

When several booleans are checked together to determine a state or setting -- especially when some
combinations are invalid -- replace them with an enum that succinctly captures the allowed states.

```rust
// BAD: invalid combinations are representable (e.g., connected && !bound)
struct Connection {
    is_bound: bool,
    is_connected: bool,
    is_authenticated: bool,
}

// GOOD: only valid states are representable
enum ConnectionState {
    Unbound,
    Bound { address: SocketAddr },
    Connected { address: SocketAddr, stream: TcpStream },
    Authenticated { address: SocketAddr, stream: TcpStream, token: Token },
}
```

### Strong types over primitives

Functions often create and process values with specific meaning -- IP addresses, file paths, user IDs -- but
represent them as common types like `String` or `u64`. Use strong types so the compiler prevents mix-ups.

```rust
// BAD: all three parameters are Strings and can be swapped silently
fn connect(host: String, path: String, token: String) { /* ... */ }

// An accidental swap compiles fine:
connect(token, host, path);  // wrong order, no compiler error

// GOOD: distinct newtypes prevent misuse
struct Host(String);
struct FilePath(String);
struct AuthToken(String);

fn connect(host: Host, path: FilePath, token: AuthToken) { /* ... */ }
// connect(token, host, path);  // compile error!
```

### TypeState pattern

When a type has flags indicating its current state and certain operations are only valid in certain states,
use generics and the TypeState pattern so the compiler enforces valid transitions.

```rust
use std::marker::PhantomData;

// State marker types (zero-sized)
struct Disconnected;
struct Connected;

struct Connection<S> {
    addr: String,
    _state: PhantomData<S>,
}

impl Connection<Disconnected> {
    fn new(addr: &str) -> Self {
        Connection { addr: addr.to_string(), _state: PhantomData }
    }

    fn connect(self) -> Result<Connection<Connected>, Error> {
        // ... perform connection ...
        Ok(Connection { addr: self.addr, _state: PhantomData })
    }
}

impl Connection<Connected> {
    fn send(&self, data: &[u8]) -> Result<(), Error> { /* ... */ Ok(()) }
    fn disconnect(self) -> Connection<Disconnected> {
        Connection { addr: self.addr, _state: PhantomData }
    }
}

// conn.send(b"hello");          // compile error: not connected
// conn.connect()?.send(b"hi");  // OK
```

---

## Cargo Review

Based on analysis of `Cargo.toml` and the resulting `Cargo.lock`.

### Unnecessary dependencies

Identify dependencies that are unused or are only needed in certain situations (dev, test, features) but are
not marked as optional. Unnecessary dependencies increase compile time and binary size.

```toml
# BAD: dependency only used in tests but listed as a normal dependency
[dependencies]
mockall = "0.11"

# GOOD: move to dev-dependencies
[dev-dependencies]
mockall = "0.11"

# BAD: dependency only used behind a feature gate but not marked optional
[dependencies]
serde_yaml = "0.9"

# GOOD: mark as optional and gate behind a feature
[dependencies]
serde_yaml = { version = "0.9", optional = true }

[features]
yaml = ["serde_yaml"]
```

### Duplicate crate versions

The resulting set of dependencies in `Cargo.lock` may contain multiple versions of the same crate. Where
possible, unify them by adjusting version requirements in `Cargo.toml` or updating intermediate crates
controlled by the project.

```toml
# BAD: two different versions of the same crate pulled in
# Cargo.lock contains both syn 1.0.109 and syn 2.0.38
[dependencies]
older-macro-lib = "0.5"   # depends on syn 1.x
newer-derive = "1.0"      # depends on syn 2.x

# GOOD: update older-macro-lib (if you control it) to syn 2.x,
# or find a version of the dependency that uses the same syn version
[dependencies]
older-macro-lib = "0.6"   # updated to syn 2.x
newer-derive = "1.0"
```

### Redundant dependencies

The project uses multiple dependencies that serve the same purpose (e.g., two error-handling crates, two
HTTP clients). Consolidating reduces code size, build time, and cognitive overhead.

```toml
# BAD: two HTTP clients
[dependencies]
reqwest = "0.11"
hyper = { version = "0.14", features = ["client"] }

# GOOD: pick one and use it consistently
[dependencies]
reqwest = "0.11"
```

---

## Ownership and Borrowing

### Avoid unnecessary clone()

`clone()` is "Rust's duct tape" -- used to bypass the borrow checker. During review, ask: is the clone
necessary? Could a borrow be used instead?

- Flag `clone()` calls that lack a justifying comment.
- If a clone is genuinely needed (e.g., data moved to a spawned task), require a comment explaining why.

```rust
// BAD: clone without justification
let owned = data.clone();
expensive_operation(owned)

// GOOD: pass by reference
expensive_operation(data)

// GOOD: justified clone with comment
// Clone needed: data will be moved to spawned task
let owned = data.clone();
tokio::spawn(async move { process(owned).await });
```

### Arc<Mutex<T>> usage

`Arc<Mutex<T>>` can hide unnecessary shared state. Review whether sharing is truly needed or if a
single-owner design would suffice. For concurrent access, consider finer-grained alternatives such as
`DashMap`.

```rust
// BAD: possibly unnecessary shared state
struct BadService {
    cache: Arc<Mutex<HashMap<String, Data>>>,
}

// GOOD: single owner when sharing is not required
struct GoodService {
    cache: HashMap<String, Data>,
}

// GOOD: finer-grained locking for concurrent access
use dashmap::DashMap;
struct ConcurrentService {
    cache: DashMap<String, Data>,
}
```

### Cow (Copy-on-Write) pattern

Use `Cow<'_, str>` (and similar) to avoid unnecessary allocations when the data may or may not need to be
owned. This is especially useful for functions that sometimes return borrowed data unchanged and sometimes
need to allocate a modified copy.

```rust
use std::borrow::Cow;

// BAD: always allocates a new String
fn bad_process_name(name: &str) -> String {
    if name.is_empty() { "Unknown".to_string() }
    else { name.to_string() }  // unnecessary allocation
}

// GOOD: borrow when possible, allocate only when modification is needed
fn normalize_name(name: &str) -> Cow<'_, str> {
    if name.chars().any(|c| c.is_uppercase()) {
        Cow::Owned(name.to_lowercase())
    } else {
        Cow::Borrowed(name)
    }
}
```

---

## Unsafe Code Review (Most Critical)

### Unnecessary unsafe markers

A function marked `unsafe` when it contains no operations that actually require `unsafe` forces callers
into an `unsafe` block for no benefit. Remove the `unsafe` qualifier when the function body is entirely safe.

```rust
// BAD: function is marked unsafe but contains no unsafe operations
unsafe fn add(a: i32, b: i32) -> i32 {
    a + b
}

// GOOD: no unsafe qualifier needed
fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

### Unnecessary unsafe code

Look for `unsafe` blocks that can be replaced by safe alternatives. The standard library and common crates
often provide safe APIs that make manual unsafe code unnecessary.

```rust
// BAD: unsafe for something the standard library handles safely
unsafe fn get_first(slice: &[u8]) -> u8 {
    *slice.as_ptr()
}

// GOOD: safe equivalent
fn get_first(slice: &[u8]) -> Option<u8> {
    slice.first().copied()
}

// BAD: unsafe transmute for conversion
let bytes: [u8; 4] = unsafe { std::mem::transmute(value) };

// GOOD: safe conversion
let bytes = value.to_ne_bytes();
```

### Every unsafe block must have a SAFETY comment

```rust
// BAD: no explanation
unsafe { *slice.get_unchecked(index) }

// GOOD: SAFETY comment explaining why this is sound
debug_assert!(index < slice.len());
// SAFETY: We verified index < slice.len() via debug_assert.
unsafe { *slice.get_unchecked(index) }
```

### Every unsafe fn must have a `# Safety` doc section

```rust
// BAD: unsafe fn without safety documentation -- red flag
unsafe fn bad_transmute<T, U>(t: T) -> U {
    std::mem::transmute(t)
}

// GOOD: documents invariants the caller must uphold
/// # Safety
/// - `T` and `U` must have the same size and alignment
/// - `T` must be a valid bit pattern for `U`
/// - No references to `t` may exist after this call
unsafe fn documented_transmute<T, U>(t: T) -> U {
    // SAFETY: Caller guarantees size/alignment match and bit pattern validity
    std::mem::transmute(t)
}
```

### Wrap unsafe in safe APIs

Prefer encapsulating unsafe code behind a safe public API that enforces the required invariants at the
boundary.

```rust
// GOOD: safe wrapper around unsafe
pub fn checked_get(slice: &[u8], index: usize) -> Option<u8> {
    if index < slice.len() {
        // SAFETY: bound's check performed above
        Some(unsafe { *slice.get_unchecked(index) })
    } else {
        None
    }
}
```

### FFI boundaries

Unsafe FFI calls should be wrapped in safe functions that validate inputs and translate error codes into
`Result`.

```rust
extern "C" {
    fn external_function(ptr: *const u8, len: usize) -> i32;
}

pub fn safe_wrapper(data: &[u8]) -> Result<i32, Error> {
    // SAFETY: data.as_ptr() is valid for data.len() bytes,
    // and external_function only reads from the buffer.
    let result = unsafe { external_function(data.as_ptr(), data.len()) };
    if result < 0 { Err(Error::from_code(result)) } else { Ok(result) }
}
```

---

## Error Handling

### Library vs. application error types

- **Libraries** should use `thiserror` to define structured, matchable error types.
- **Applications** should use `anyhow` with `.context()` for ergonomic error propagation.

```rust
// BAD: library using anyhow -- callers cannot match on error variants
pub fn parse_config(s: &str) -> anyhow::Result<Config> { /* ... */ }

// GOOD: library with `thiserror` crate
#[derive(Debug, thiserror::Error)]
pub enum ConfigError {
    #[error("invalid syntax at line {line}: {message}")]
    Syntax { line: usize, message: String },
    #[error("missing required field: {0}")]
    MissingField(String),
    #[error(transparent)]
    Io(#[from] std::io::Error),
}

pub fn parse_config(s: &str) -> Result<Config, ConfigError> { /* ... */ }
```

### Preserve error context

Always preserve the original error when adding context. Discarding the underlying error makes debugging
much harder.

```rust
// BAD: original error is lost
operation().map_err(|_| anyhow!("failed"))?;

// GOOD: use .context() to preserve the error chain
operation().context("failed to perform operation")?;

// GOOD: use .with_context() for lazy formatting
operation().with_context(|| format!("failed to process file: {}", filename))?;
```

### Error type design

Use `#[source]` to preserve the error chain and implement `From` for common conversions so the `?`
operator works ergonomically.

```rust
#[derive(Debug, thiserror::Error)]
pub enum ServiceError {
    #[error("database error")]
    Database(#[source] sqlx::Error),

    #[error("network error: {message}")]
    Network {
        message: String,
        #[source]
        source: reqwest::Error,
    },

    #[error("validation failed: {0}")]
    Validation(String),
}
```

---

## Performance

### Avoid unnecessary collect() and intermediate allocations

Do not materialize an intermediate `Vec` only to iterate over it again or to perform a simple check.
Use lazy iterator chains and iterator methods like `any`, `all`, `find`, and `sum` directly.

```rust
// BAD: unnecessary intermediate allocation
items.iter().filter(|x| **x > 0).collect::<Vec<_>>().iter().sum()

// GOOD: lazy iteration
items.iter().filter(|x| **x > 0).copied().sum()

// BAD: allocating a Vec just to check if any element matches
let filtered: Vec<_> = items.iter().filter(|i| i.is_valid()).collect();
!filtered.is_empty()

// GOOD: use iterator method
items.iter().any(|i| i.is_valid())
```

### Avoid unnecessary String and &str allocations

Strings are heap allocations. Prefer `&str` borrows when ownership is not needed, and use `&'static str`
for compile-time constants so they live in the read-only data segment rather than being allocated at runtime.

```rust
// BAD: String::from for a static string when no owned String is needed
fn bad_label() -> String { String::from("error message") }

// GOOD: return &'static str
fn good_label() -> &'static str { "error message" }
```

### String concatenation

Avoid repeated allocations when building strings in loops. Use `.join("")`, `String::with_capacity`, or
`write!`.

```rust
// BAD: re-allocates on every iteration
let mut s = String::new();
for item in items { s = s + item; }

// GOOD: join
items.join("")

// GOOD: pre-allocate
let total_len: usize = items.iter().map(|s| s.len()).sum();
let mut result = String::with_capacity(total_len);
for item in items { result.push_str(item); }
```

---

## Trait Design

### Avoid over-abstraction

Do not create traits for everything. Concrete types are simpler and faster. Only introduce a trait when
genuine polymorphism is required -- this is Rust, not Java.

```rust
// BAD: trait soup -- not Java, no need to interface everything
trait Processor { fn process(&self); }
trait Handler { fn handle(&self); }
trait Manager { fn manage(&self); }

// GOOD: concrete type when polymorphism is not needed
struct DataProcessor { config: Config }
impl DataProcessor {
    fn process(&self, data: &Data) -> Result<Output> { /* ... */ }
}
```

### Trait objects vs. generics

- Use **generics** (static dispatch) by default for performance and inlining.
- Use **trait objects** (`dyn Trait`) when you need heterogeneous collections or dynamic dispatch is required.
- Use `impl Trait` in return position when the concrete type is not important to the caller.

```rust
// Prefer generics (static dispatch, zero-cost)
fn process<H: Handler>(handler: &H) { handler.handle(); }

// Trait objects for heterogeneous collections
fn store_handlers(handlers: Vec<Box<dyn Handler>>) { /* ... */ }

// impl Trait return type -- hides concrete type from caller
fn create_handler() -> impl Handler { ConcreteHandler::new() }
```

---

## Async Code

### Avoid blocking operations in async context

Blocking calls (e.g., `std::fs`, `std::thread::sleep`) in async functions starve other tasks on the runtime.

```rust
// BAD: blocking in async
async fn bad_async() {
    let data = std::fs::read_to_string("file.txt").unwrap();  // blocks!
    std::thread::sleep(Duration::from_secs(1));                // blocks!
}

// GOOD: use async APIs
async fn good_async() -> Result<String> {
    let data = tokio::fs::read_to_string("file.txt").await?;
    tokio::time::sleep(Duration::from_secs(1)).await;
    Ok(data)
}

// GOOD: use spawn_blocking for unavoidable blocking work
async fn with_blocking() -> Result<Data> {
    let result = tokio::task::spawn_blocking(|| {
        expensive_cpu_computation()
    }).await?;
    Ok(result)
}
```

### Mutex and .await

Do not hold a `std::sync::Mutex` guard across an `.await` point -- this can cause deadlocks.

```rust
// BAD: holding std::sync::Mutex across .await
async fn bad_lock(mutex: &std::sync::Mutex<Data>) {
    let guard = mutex.lock().unwrap();
    async_operation().await;  // holding lock across await!
    process(&guard);
}

// GOOD option 1: minimize lock scope
async fn good_lock_scoped(mutex: &std::sync::Mutex<Data>) {
    let data = {
        let guard = mutex.lock().unwrap();
        guard.clone()  // release lock immediately
    };
    async_operation().await;
    process(&data);
}

// GOOD option 2: use tokio::sync::Mutex (designed for holding across await)
async fn good_lock_tokio(mutex: &tokio::sync::Mutex<Data>) {
    let guard = mutex.lock().await;
    async_operation().await;  // OK
    process(&guard);
}
```

**Selection guide:**
- `std::sync::Mutex`: low contention, short critical sections, never across `.await`
- `tokio::sync::Mutex`: when holding across `.await` is required or under high contention

### Async trait methods

Since Rust 1.75, async methods are supported natively in traits. For `dyn`-compatible scenarios, use
`Pin<Box<dyn Future>>` since async methods are not object-safe.

```rust
// Rust 1.75+: native async trait methods
trait Repository {
    async fn find(&self, id: i64) -> Option<Entity>;
    fn find_many(&self, ids: &[i64]) -> impl Future<Output = Vec<Entity>> + Send;
}

// For dyn-compatible scenarios, use Pin<Box<dyn Future>>
trait DynRepository: Send + Sync {
    fn find(&self, id: i64) -> Pin<Box<dyn Future<Output = Option<Entity>> + Send + '_>>;
}
```

### spawn vs. await

Do **not** spawn simple operations that can be directly awaited -- spawning adds overhead and loses
structured concurrency. Use `spawn` for truly parallel execution (multiple independent I/O operations) or
fire-and-forget background tasks.

```rust
// BAD: unnecessary spawn
let handle = tokio::spawn(async { simple_operation().await });
handle.await.unwrap();  // why not just await directly?

// GOOD: A direct `await`
simple_operation().await;

// GOOD: spawn for parallel execution
let task1 = tokio::spawn(fetch_from_service_a());
let task2 = tokio::spawn(fetch_from_service_b());
let (result1, result2) = tokio::try_join!(task1, task2)?;
```

### spawn's 'static requirement

Spawned futures must be `'static`. Solutions:
1. Clone the data.
2. Use `Arc` for shared ownership.
3. Use scoped task crates (`tokio-scoped`, `async-scoped`).

```rust
// BAD: borrowing non-'static data in spawn
tokio::spawn(async { process(data).await });  // Error: `data` is not 'static

// GOOD: clone or Arc
let owned = data.clone();
tokio::spawn(async move { process(&owned).await });

let data = Arc::clone(&data);
tokio::spawn(async move { process(&data).await });
```

### JoinHandle error handling

Always handle `JoinHandle` results -- do not silently discard panics or errors.

```rust
// BAD: ignoring spawn errors
let _ = handle.await;

// GOOD: handle both task errors and join errors
match handle.await {
    Ok(Ok(result)) => { /* task completed successfully */ }
    Ok(Err(e))     => { /* task returned an error */ }
    Err(join_err)  => {
        // task panicked or it was canceled
        if join_err.is_panic() {
            error!("Task panicked: {:?}", join_err);
        }
    }
}
```

### Prefer structured concurrency

Prefer `join!` / `try_join!` over raw `spawn` when all tasks share the same lifetime scope. With
`try_join!`, if any task fails the others are cancelled.

```rust
// GOOD: structured concurrency
tokio::try_join!(fetch_a(), fetch_b(), fetch_c())
```

When using `spawn`, consider the task lifecycle and have a shutdown strategy (graceful wait or abort).

### Cancellation safety

When a Future is dropped at an `.await` point, what state is it in?

- **Cancel-safe Future**: can be safely cancelled at any await point.
- **Cancel-unsafe Future**: cancellation may cause data loss or inconsistent state.

```rust
// BAD: cancel-unsafe - if it is canceled after receive_data, ack is never sent
async fn cancel_unsafe(conn: &mut Connection) -> Result<()> {
    let data = receive_data().await;
    conn.send_ack().await;
    Ok(())
}

// GOOD: use transactions or atomic operations for consistency
async fn cancel_safe(conn: &mut Connection) -> Result<()> {
    let transaction = conn.begin_transaction().await?;
    let data = receive_data().await;
    transaction.commit_with_ack(data).await?;
    Ok(())
}
```

### Cancel safety in select!

- `read_exact` is **not** cancel-safe: partial bytes read are lost when the future is dropped.
- `read` **is** cancel-safe: unread data remains in the stream.
- When a cancel-unsafe operation is needed in `select!`, move it into a separate task and select on the `JoinHandle`.

```rust
// BAD: cancel-unsafe future in select!
select! {
    result = stream.read_exact(&mut buffer) => { /* ... */ }
    _ = tokio::time::sleep(Duration::from_secs(5)) => { /* timeout */ }
}

// GOOD: cancel-safe API
select! {
    result = stream.read(&mut buffer) => {
        match result {
            Ok(0) => break,
            Ok(n) => handle_data(&buffer[..n]),
            Err(e) => return Err(e),
        }
    }
    _ = tokio::time::sleep(Duration::from_secs(5)) => {
        println!("Timeout, retrying...");
    }
}

// GOOD: use tokio::pin! for futures that need to be reused across loop iterations
let sleep = tokio::time::sleep(Duration::from_secs(10));
tokio::pin!(sleep);
loop {
    select! {
        _ = &mut sleep => { break; }
        data = receive_data() => { process(data).await; }
    }
}
```

### Document cancellation safety

Every public async function should document its cancel safety behavior.

```rust
/// # Cancel Safety
///
/// This method is **not** cancel safe. If it is canceled while reading,
/// partial data may be lost and the stream state becomes undefined.
/// Use `read_message_cancel_safe` if cancellation is expected.
async fn read_message(stream: &mut TcpStream) -> Result<Message> { /* ... */ }
```
