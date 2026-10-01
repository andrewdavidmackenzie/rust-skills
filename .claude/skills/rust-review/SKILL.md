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
- a specific file (rust file name specified)
- all files under a specified directory path (directory name supplied)
- project: review all rust-related files (including Cargo.toml) in the project
- a module name: find module(s) in the project that match the name. If mopre than one ask the user to clarify which.
Then review all files in the specified module recurseivly down, including files that are behind features and files
that are referenced from elsewhere via a "path=" construct in module files

rm ca   
## Review Checklist (Summary)

### Issues the compiler cannot catch

- [ ] Boundary conditions handled correctly
- [ ] State machine transitions are complete
- [ ] Race conditions in concurrent scenarios
- [ ] Public APIs are hard to misuse
- [ ] Type signatures clearly express intent
- [ ] Error type granularity is appropriate

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

### Async / Concurrency

- [ ] No blocking in async (`std::fs`, `thread::sleep`)
- [ ] No `std::sync` lock held across `.await`
- [ ] Spawned tasks satisfy `'static`
- [ ] Lock acquisition order is consistent
- [ ] Channel buffer sizes are reasonable

### Cancellation Safety

- [ ] Futures in `select!` are cancel-safe
- [ ] Async functions document their cancel safety
- [ ] Cancellation does not cause data loss or inconsistent state
- [ ] `tokio::pin!` is used correctly for futures that need reuse

### spawn vs await

- [ ] `spawn` is only used for genuinely parallel workloads
- [ ] Simple operations are awaited directly, not spawned
- [ ] `JoinHandle` results are handled properly
- [ ] Task lifecycle and shutdown strategy are considered
- [ ] `join!` / `try_join!` preferred for structured concurrency

### Error Handling

- [ ] Libraries use `thiserror` for structured errors
- [ ] Applications use `anyhow` + `.context()`
- [ ] No `unwrap()` / `expect()` in production code
- [ ] Error messages are helpful for debugging
- [ ] `#[must_use]` return values are handled
- [ ] `#[source]` preserves the error chain

### Performance

- [ ] No unnecessary `collect()`
- [ ] Large data passed by reference
- [ ] Strings use `with_capacity` or `write!`
- [ ] `impl Trait` vs `Box<dyn Trait>` choice is appropriate
- [ ] Hot paths avoid allocations
- [ ] `Cow` considered to reduce cloning

### Code Quality

- [ ] `cargo clippy` produces zero warnings
- [ ] `cargo fmt` has been applied
- [ ] Doc comments are complete
- [ ] Tests cover boundary conditions
- [ ] Public APIs have documented examples


## Cargo review
Based on analysis of Cargo.toml and the resulting Cargo.lock

### Unnecessary dependencies

### Dependencies could be combined
The resulting set of dependencies in Cargo.lock has multiple versions of the same crate that could be combined by
modifying versions of the crate or parents of the crate controlled by the corrent project.

### Similar Dependencies
The project uses multiple dependencies of a similar nature (e.g. error handling) and code modifications could 
merge the dependencies into one reducing code size and build time.

## Function level checks

### Constructors
If a function instantiates a new instance of a type from the current crate, consider it as a possible constructor
("new" or similar) that could be an associated function of the type in question. Consider a possible implementation
of the Default::default trait for that type.

### Free functions / Associated functions that could be methods
If a function takes an instance of or a reference to a struct that is defined in the crate, consider it as a candidate
for becoming a method of that type.

## Function signature checks

## Line level checks

## Documentation / Doc-Test checks

## Overall design checks

### Groups of booleans
When there are a number of booleans that are checked together in some way to determine a state, or a setting,
(especially when there are some combinations of the booleans that are not valid) consider the use of an enum to 
succinctly capture the allowed states.

## Find errors at compile time
### Low use of types 
Functions create and process "types" with specific meaning and use but they are represented by common types
(e.g. Strings) that can be interchanged in function signatures and return types. Use strong types so that for example 
an IP address cannot be confused with a file path.

### TypeState pattern
When a type has boolean or other flags within it to indicate it's state, and that is used to allow or deny certain
operations (e.g. a Connection that has a "bound" or "connected" flag and sending data methods cannot be used unless
in the "connected" state) then consider the use of generics and the typestate pattern to avoid use of the type
when it is in the incorrect stage

## Duplication
### Repeated functions
Identify repeated functions across the code base that can be pulled out to associated functions or helper methods.

### Similar functions
Identify similar functions (of a reasonable size/complexity) across the code based that share a lot of lines of code 
and that could be combined with the use of an additional parameter to vary the behaviour.

## Ownership and Borrowing

### Avoid unnecessary clone()

`clone()` is "Rust's duct tape" -- used to bypass the borrow checker. During review, ask: is the clone necessary? 
Could a borrow be used instead?

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

`Arc<Mutex<T>>` can hide unnecessary shared state. Review whether sharing is truly needed or if a single-owner design 
would suffice. For concurrent access, consider finer-grained alternatives such as `DashMap`.

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

Use `Cow<'_, str>` (and similar) to avoid unnecessary allocations when the data may or may not need to be owned.

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

## Unsafe Code Review (Most Critical)

### Unnecessary unsafe marker
Determine if a function is marked as unsafe when it is not necessary

### Unnecessary unsafe code
Find blocks or sections of unsafe code be replaced by safe code equivalents.

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
    // SAFETY: Caller guarantees size/alignment match and bit validity
    std::mem::transmute(t)
}
```

### Wrap unsafe in safe APIs

Prefer encapsulating unsafe code behind a safe public API that enforces the required invariants at the boundary.

```rust
// GOOD: safe wrapper around unsafe
pub fn checked_get(slice: &[u8], index: usize) -> Option<u8> {
    if index < slice.len() {
        // SAFETY: bounds check performed above
        Some(unsafe { *slice.get_unchecked(index) })
    } else {
        None
    }
}
```

### FFI boundaries

Unsafe FFI calls should be wrapped in safe functions that validate inputs and translate error codes into `Result`.

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

---

## Cancellation Safety

### What is cancellation safety?

When a Future is dropped at an `.await` point, what state is it in?

- **Cancel-safe Future**: can be safely cancelled at any await point.
- **Cancel-unsafe Future**: cancellation may cause data loss or inconsistent state.

```rust
// BAD: cancel-unsafe -- if cancelled after receive_data, ack is never sent
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

Every public async function should document its cancel safety behavior:

```rust
/// # Cancel Safety
///
/// This method is **not** cancel safe. If cancelled while reading,
/// partial data may be lost and the stream state becomes undefined.
/// Use `read_message_cancel_safe` if cancellation is expected.
async fn read_message(stream: &mut TcpStream) -> Result<Message> { /* ... */ }
```

---

## spawn vs await

### When to use spawn

- Do **not** spawn simple operations that can be directly awaited -- spawning adds overhead and loses structured 
concurrency.
- Use `spawn` for truly parallel execution (multiple independent I/O operations).
- Use `spawn` for fire-and-forget background tasks.

```rust
// BAD: unnecessary spawn
let handle = tokio::spawn(async { simple_operation().await });
handle.await.unwrap();  // why not just await directly?

// GOOD: direct await
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
        // task panicked or was cancelled
        if join_err.is_panic() {
            error!("Task panicked: {:?}", join_err);
        }
    }
}
```

### Prefer structured concurrency

Prefer `join!` / `try_join!` over raw `spawn` when all tasks share the same lifetime scope. With `try_join!`, 
if any task fails the others are cancelled.

```rust
// GOOD: structured concurrency
tokio::try_join!(fetch_a(), fetch_b(), fetch_c())
```

When using `spawn`, consider the task lifecycle and have a shutdown strategy (graceful wait or abort).

---

## Error Handling

### Library vs application error types

- **Libraries** should use `thiserror` to define structured, matchable error types.
- **Applications** should use `anyhow` with `.context()` for ergonomic error propagation.

```rust
// BAD: library using anyhow -- callers cannot match on error variants
pub fn parse_config(s: &str) -> anyhow::Result<Config> { /* ... */ }

// GOOD: library with thiserror
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

```rust
// BAD: original error is lost
operation().map_err(|_| anyhow!("failed"))?;

// GOOD: use .context() to preserve the error chain
operation().context("failed to perform operation")?;

// GOOD: use .with_context() for lazy formatting
operation().with_context(|| format!("failed to process file: {}", filename))?;
```

### Error type design

- Use `#[source]` to preserve the error chain.
- Implement `From` for common conversions.

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

### Avoid unnecessary collect()

Do not materialize an intermediate `Vec` only to iterate over it again. Use lazy iterator chains.

```rust
// BAD: unnecessary intermediate allocation
items.iter().filter(|x| **x > 0).collect::<Vec<_>>().iter().sum()

// GOOD: lazy iteration
items.iter().filter(|x| **x > 0).copied().sum()
```

### String concatenation

Avoid repeated allocations in loops. Use `.join("")`, `String::with_capacity`, or `write!`.

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

### Avoid unnecessary String usage

Strings are allocation son the HEAP. Avoid using a string when a &str can suffice. If they are constant then
&'staic str to have them in the text segment and not allocated.

### Avoid unnecessary allocations

```rust
// BAD: allocating a Vec just to check if any element matches
let filtered: Vec<_> = items.iter().filter(|i| i.is_valid()).collect();
!filtered.is_empty()

// GOOD: use iterator method
items.iter().any(|i| i.is_valid())

// BAD: String::from for a static string when no owned String is needed
fn bad_static() -> String { String::from("error message") }

// GOOD: return &'static str
fn good_static() -> &'static str { "error message" }
```

---

## Trait Design

### Avoid over-abstraction

Do not create traits for everything. Concrete types are simpler and faster. Only introduce a trait when genuine 
polymorphism is required.

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

### Trait objects vs generics

- Use **generics** (static dispatch) by default for performance and inlining.
- Use **trait objects** (`dyn Trait`) when you need heterogeneous collections or dynamic dispatch is required.
- Use `impl Trait` in return position when the concrete type is not important to the caller.

```rust
// Prefer generics
fn good_process<H: Handler>(handler: &H) { handler.handle(); }

// Trait objects for heterogeneous collections
fn store_handlers(handlers: Vec<Box<dyn Handler>>) { /* ... */ }

// impl Trait return type
fn create_handler() -> impl Handler { ConcreteHandler::new() }
```
