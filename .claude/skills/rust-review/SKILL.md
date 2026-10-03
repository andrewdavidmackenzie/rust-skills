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
- files modified in a pull request. if the user specifies a pull request number, use gh client or git to determine 
code modified in the PR and analyse that code, and the files including those changes, but not the entire repo.
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

## Process Checklist

These items are process-level checks that do not map to individual BAD/GOOD code examples.

- [ ] `cargo clippy` produces zero warnings
- [ ] `cargo fmt` has been applied
- [ ] Doc comments are complete on all public items
- [ ] Tests cover boundary conditions
- [ ] Public APIs have documented examples
- [ ] Panic strategy is documented at project level

## Review Checklist (Summary)

Every item below has a corresponding detailed rule section with BAD and GOOD code examples further
in this document.

### Readability

- [ ] No magic numbers; constants have descriptive names
- [ ] `match` used instead of `if`/`else` chains for enums
- [ ] `matches!` macro used for simple boolean pattern checks
- [ ] `let-else` used instead of `match`/`if let` when only the failure branch diverges

### Ownership and Borrowing

- [ ] `clone()` is intentional and documented with a reason
- [ ] `Arc<Mutex<T>>` truly needs shared state
- [ ] `Cow` considered to avoid unnecessary allocations
- [ ] Variables are immutable by default; `mut` is limited to tight scopes
- [ ] Temporary mutability uses scope blocks or shadowing to re-bind as immutable

### Unsafe Code (Most Important)

- [ ] Every `unsafe` block has a `SAFETY` comment
- [ ] Every `unsafe fn` has a `# Safety` doc section
- [ ] Unsafe boundary is as small as possible; unsafe wrapped in safe APIs
- [ ] Safe alternatives have been considered; unnecessary `unsafe` removed
- [ ] FFI calls are wrapped in safe functions that validate inputs

### Error Handling

- [ ] Libraries use `thiserror` for structured errors
- [ ] Applications use `anyhow` + `.context()`
- [ ] No `unwrap()` / `expect()` in production code
- [ ] Error context is preserved (no discarding original errors)
- [ ] `#[source]` preserves the error chain
- [ ] Library error enums do not leak inner dependency types via `#[from]`
- [ ] `From` impls that can fail are `TryFrom` instead
- [ ] Panics rely only on local invariants; non-local invariants use `Result`

### Performance

- [ ] No unnecessary `collect()` or intermediate allocations
- [ ] Strings use `&str`, `with_capacity`, `join`, or `write!` as appropriate
- [ ] Hot paths avoid allocations

### Trait Design

- [ ] Traits introduced only when genuine polymorphism is needed
- [ ] Generics preferred over trait objects unless heterogeneous collections are required
- [ ] `impl Trait` used in return position when concrete type is unimportant
- [ ] Premature generics resisted; concrete types used until polymorphism is actually needed

### Concurrency

- [ ] Lock acquisition order is consistent to prevent deadlocks
- [ ] No lock guards held across `if let` / `while let` else branches (sneaky deadlock)
- [ ] `try_join!` not used when all tasks must complete regardless of individual failures
- [ ] `JoinHandle` results are handled properly

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
- [ ] Enums used to eliminate invalid states (booleans, correlated fields)
- [ ] TypeState pattern considered for stateful types
- [ ] Newtypes validate in constructors; inner fields are private
- [ ] `Default` is only derived when all default field values are meaningful
- [ ] `#[must_use]` applied to important builder/config types
- [ ] Visibility is minimized: fields, methods, structs, and modules are private unless public access is required

### Safe Rust Pitfalls

- [ ] No `as` for narrowing numeric conversions; `From::from` or `TryFrom` used instead
- [ ] Arithmetic that can overflow uses `checked_*` / `saturating_*` methods
- [ ] Array/slice access uses `.get()` or slice patterns instead of direct indexing
- [ ] `split_at_checked` preferred over `split_at` for untrusted indices

### Defensive Programming

- [ ] `match` arms enumerate all variants explicitly (no wildcard `_` catch-all on owned enums)
- [ ] Struct destructuring used in trait impls (e.g. `PartialEq`, `Debug`) to catch new fields
- [ ] `..Default::default()` avoided; all fields set explicitly so new fields trigger compiler errors
- [ ] Boolean parameters replaced with enums for clarity at call sites
- [ ] `Debug` implemented manually for types with sensitive data (passwords, tokens)
- [ ] `Serialize`/`Deserialize` not blindly derived on types with sensitive or validated fields

### Simplicity

- [ ] Premature optimization avoided; complexity not added to dodge cold-path allocations
- [ ] Abstractions kept shallow; layered traits/generics avoided when concrete types suffice

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

## Pattern Matching

### Prefer `match` over `if`/`else` chains for enums

`match` on enums gives exhaustiveness checking and often better compiler optimizations.

```rust
// BAD: no exhaustiveness checking
if status == Status::Active { /* ... */ }
else if status == Status::Inactive { /* ... */ }
else { /* catch-all, hides new variants */ }

// GOOD: exhaustive, compiler-checked
match status {
    Status::Active => { /* ... */ }
    Status::Inactive => { /* ... */ }
    Status::Suspended => { /* ... */ }
}
```

### Use `matches!` for simple boolean checks

When a pattern match only needs to return `true`/`false`, prefer the `matches!` macro.

```rust
// BAD: verbose for a boolean check
let is_success = match response.status {
    Status::Ok | Status::Created => true,
    _ => false,
};

// GOOD: concise
let is_success = matches!(response.status, Status::Ok | Status::Created);
```

### Use `let-else` for early returns

When you need to destructure-or-diverge, `let-else` (Rust 1.65+) is cleaner than
`match` or `if let` with an early return.

```rust
// BAD: nested match just for early return
let config = match load_config() {
    Some(c) => c,
    None => return Err(Error::NoConfig),
};

// GOOD: let-else is flat and direct
let Some(config) = load_config() else {
    return Err(Error::NoConfig);
};
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

### Use enums to eliminate invalid states

When booleans or related fields can form invalid combinations, replace them with an enum so only
valid states are representable. This applies to groups of booleans, correlated `Option` fields,
and any set of fields whose valid values depend on each other.

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

// BAD: ssl=true with ssl_cert=None is invalid but representable
struct Config {
    ssl: bool,
    ssl_cert: Option<String>,
}

// GOOD: invalid combination is impossible
enum Security {
    Plaintext,
    Tls { cert_path: String },
}

struct Config {
    security: Security,
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
// BAD: runtime flag that can be forgotten or misused
struct Connection {
    addr: String,
    connected: bool,
}

impl Connection {
    fn send(&self, data: &[u8]) -> Result<(), Error> {
        if !self.connected { return Err(Error::NotConnected); } // runtime check
        /* ... */
        Ok(())
    }
}

// GOOD: compile-time enforcement via TypeState
use std::marker::PhantomData;

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

### Guard newtypes with private fields and validated constructors

Newtypes should validate in constructors. Keep the inner field private so construction must go
through the validated path. Use `#[non_exhaustive]` or a private field to prevent external
struct-literal construction.

```rust
// BAD: public field allows bypassing validation
pub struct Username(pub String);

// anyone can write: Username("".to_string())

// GOOD: private field forces use of validated constructor
pub struct Username(String);  // field is private

impl Username {
    pub fn new(raw: &str) -> Result<Self, UsernameError> {
        if raw.is_empty() { return Err(UsernameError::Empty); }
        if raw.len() > 30 { return Err(UsernameError::TooLong); }
        if !raw.chars().all(|c| c.is_alphanumeric() || c == '_') {
            return Err(UsernameError::InvalidChars);
        }
        Ok(Self(raw.to_string()))
    }

    pub fn as_str(&self) -> &str { &self.0 }
}
```

### Be intentional with `Default`

Do not blindly `#[derive(Default)]` on types where the zero/empty default creates an invalid or
surprising state. Either implement `Default` manually with sensible values, or don't implement
it at all.

```rust
// BAD: port 0 and 0 connections are probably not what you want
#[derive(Default)]
struct ServerConfig {
    port: u16,               // defaults to 0
    max_connections: usize,  // defaults to 0
}

// GOOD: explicit constructor with validated defaults
impl ServerConfig {
    pub fn new(port: u16) -> Self {
        Self { port, max_connections: 100 }
    }
}
```

### Minimize visibility

Keep fields, methods, structs, and modules as private as possible. Only mark items `pub` when
external access is genuinely required. This limits the API surface, prevents misuse, and makes
refactoring safer.

```rust
// BAD: everything public for no reason
pub struct DatabasePool {
    pub connection_string: String,  // anyone can mutate this
    pub pool: Vec<Connection>,      // internals exposed
}

// GOOD: private by default, expose only what's needed
pub struct DatabasePool {
    connection_string: String,
    pool: Vec<Connection>,
}

impl DatabasePool {
    pub fn new(connection_string: &str) -> Self { /* ... */ }
    pub fn get_connection(&self) -> Option<&Connection> { /* ... */ }
    // internals stay private
}
```

### Use `#[must_use]` on important types

Mark types that must be consumed (configs, guards, builders) with `#[must_use]` to prevent
accidental discard.

```rust
// BAD: caller can silently ignore the transaction
pub struct Transaction { /* ... */ }

let txn = db.begin(); // oops, forgot to commit or rollback

// GOOD: compiler warns if the value is unused
#[must_use = "Transaction must be committed or rolled back"]
pub struct Transaction { /* ... */ }

let txn = db.begin(); // warning: unused `Transaction` that must be used
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

## Immutability

### Prefer immutable bindings

Use `let` without `mut` by default. Only add `mut` when mutation is genuinely needed, and confine
it to the smallest possible scope.

```rust
// BAD: mut used when not needed
let mut result = compute_value();
println!("{result}");

// GOOD: immutable binding
let result = compute_value();
println!("{result}");
```

### Temporary mutability pattern

When data needs mutation only during initialization, shadow the binding to make it immutable
afterward.

```rust
// BAD: data stays mut for entire function even though mutation is done
let mut data = load_items();
data.sort();
data.dedup();
// ... 200 lines of code where data is accidentally mutated ...

// GOOD: temporary mutability with shadowing
let data = {
    let mut data = load_items();
    data.sort();
    data.dedup();
    data  // returned immutable
};
// `data` is immutable from here on
```

---

## Unsafe Code Review (Most Critical)

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
boundary. Keep the unsafe boundary as small as possible.

```rust
// BAD: unsafe exposed to all callers, who must uphold invariants themselves
pub unsafe fn get_element(slice: &[u8], index: usize) -> u8 {
    *slice.get_unchecked(index)
}

// GOOD: safe wrapper around unsafe; invariants enforced at boundary
pub fn checked_get(slice: &[u8], index: usize) -> Option<u8> {
    if index < slice.len() {
        // SAFETY: bound's check performed above
        Some(unsafe { *slice.get_unchecked(index) })
    } else {
        None
    }
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

### FFI boundaries

Unsafe FFI calls should be wrapped in safe functions that validate inputs and translate error codes into
`Result`.

```rust
// BAD: raw FFI call exposed directly; callers must ensure pointer validity
extern "C" { fn external_function(ptr: *const u8, len: usize) -> i32; }

pub fn bad_wrapper(data: &[u8]) -> i32 {
    unsafe { external_function(data.as_ptr(), data.len()) }
    // no error handling, no safety comment
}

// GOOD: safe wrapper validates inputs and translates errors
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

### No unwrap/expect in production code

`unwrap()` and `expect()` cause panics on `None`/`Err`. In production code, propagate errors with
`?` or handle them explicitly.

```rust
// BAD: panics if the file doesn't exist
let content = std::fs::read_to_string("config.toml").unwrap();

// GOOD: propagate the error
let content = std::fs::read_to_string("config.toml")
    .context("failed to read config file")?;

// GOOD: handle explicitly
let content = match std::fs::read_to_string("config.toml") {
    Ok(c) => c,
    Err(e) => {
        log::warn!("Config not found, using defaults: {e}");
        String::new()
    }
};
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

### Error type design with `#[source]`

Use `#[source]` to preserve the error chain and implement `From` for common conversions so the `?`
operator works ergonomically.

```rust
// BAD: source error not preserved -- debugging is harder
#[derive(Debug, thiserror::Error)]
pub enum ServiceError {
    #[error("database error: {0}")]
    Database(String),  // original error lost
}

// GOOD: #[source] preserves the error chain
#[derive(Debug, thiserror::Error)]
pub enum ServiceError {
    #[error("database error")]
    Database(#[source] sqlx::Error),

    #[error("validation failed: {0}")]
    Validation(String),
}
```

### Library errors must not leak inner dependency types

Using `#[from]` to expose third-party error types (e.g., `sqlx::Error`) in your public API forces
consumers to depend on those crates and creates version-coupling. Wrap them instead.

```rust
// BAD: leaks sqlx::Error to consumers
#[derive(Debug, thiserror::Error)]
pub enum MyError {
    #[error("database error: {0}")]
    Database(#[from] sqlx::Error),  // consumers must depend on sqlx!
}

// GOOD: wrap with a boxed trait object or newtype
#[derive(Debug, thiserror::Error)]
pub enum MyError {
    #[error("database error: {0}")]
    Database(Box<dyn std::error::Error + Send + Sync>),
}

impl From<sqlx::Error> for MyError {
    fn from(err: sqlx::Error) -> Self {
        Self::Database(Box::new(err))
    }
}
```

### `From` impls that can fail should be `TryFrom`

If a type conversion can fail, resist implementing `From` (which hides the fallibility). Use `TryFrom`
instead so callers are forced to handle errors.

```rust
// BAD: From that panics or uses unwrap_or -- hides failure
impl From<&str> for Port {
    fn from(s: &str) -> Self {
        Port(s.parse().unwrap_or(0))  // silent fallback!
    }
}

// GOOD: TryFrom makes fallibility explicit
impl TryFrom<&str> for Port {
    type Error = PortError;
    fn try_from(s: &str) -> Result<Self, Self::Error> {
        let n: u16 = s.parse().map_err(|_| PortError::InvalidFormat)?;
        if n == 0 { return Err(PortError::Zero); }
        Ok(Port(n))
    }
}
```

### Panic strategy

Panics should only occur on **bugs**, never on expected error conditions. Panics relying on
**local invariants** (checked in the same scope) are acceptable. Panics relying on **non-local
invariants** (caller must uphold) are risky -- prefer `Result`.

```rust
// BAD: non-local invariant -- caller must ensure validity
/// Caller must ensure `i < slice.len()` (otherwise will panic)
pub fn get_item(slice: &[u8], i: usize) -> u8 {
    slice[i]  // panics if caller violates contract
}

// GOOD: return Option instead of trusting the caller
pub fn get_item(slice: &[u8], i: usize) -> Option<u8> {
    slice.get(i).copied()
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
// BAD: trait object when only one concrete type is ever used
fn process(handler: &dyn Handler) { handler.handle(); }

// GOOD: generics (static dispatch, zero-cost)
fn process<H: Handler>(handler: &H) { handler.handle(); }

// Trait objects for heterogeneous collections (appropriate use)
fn store_handlers(handlers: Vec<Box<dyn Handler>>) { /* ... */ }

// impl Trait return type -- hides concrete type from caller
fn create_handler() -> impl Handler { ConcreteHandler::new() }
```

### Resist premature generics

Only make code generic when you need to swap implementations *right now*. Premature generics
increase compile time (monomorphization), worsen error messages, and add cognitive load.

```rust
// BAD: generic for no current reason
fn process<T: AsRef<str> + Send + Sync + ?Sized>(input: &T) -> &str {
    input.as_ref()
}

// GOOD: concrete until polymorphism is actually needed
fn process(input: &str) -> &str {
    input
}
```

---

## Simplicity

### Avoid premature optimization

Do not add complexity to avoid allocations in code that is not on a hot path. A few `.clone()` calls
or an extra `Vec` allocation rarely matter. Measure before optimizing.

```rust
// BAD: complex lifetime gymnastics to avoid one allocation on a cold path
fn process<'a>(data: &'a [Item<'a>]) -> Vec<&'a str> { /* ... */ }

// GOOD: simple owned types until profiling says otherwise
fn process(data: &[Item]) -> Vec<String> { /* ... */ }
```

### Keep abstractions shallow

Avoid layering traits, generics, and indirection when a concrete type suffices. Good abstractions
"click" -- adding new functionality feels obvious and tests need little mocking.

```rust
// BAD: over-layered indirection for a simple task
trait DataFormat { fn parse(&self, s: &str) -> Result<Vec<Record>>; }
struct UniversalParser<T: DeserializeOwned> {
    format: Box<dyn DataFormat>,
    _marker: PhantomData<T>,
}

// GOOD: direct implementation when only one format is used
fn parse_csv(content: &str) -> Result<Vec<Customer>, csv::Error> {
    let mut reader = csv::Reader::from_reader(content.as_bytes());
    reader.deserialize().collect()
}
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
// BAD: raw spawns without lifecycle management
let h1 = tokio::spawn(fetch_a());
let h2 = tokio::spawn(fetch_b());
// no join, no error handling -- tasks are fire-and-forget by accident

// GOOD: structured concurrency
tokio::try_join!(fetch_a(), fetch_b(), fetch_c())?;
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
// BAD: no documentation of cancel safety
pub async fn read_message(stream: &mut TcpStream) -> Result<Message> { /* ... */ }

// GOOD: cancel safety behavior documented
/// # Cancel Safety
///
/// This method is **not** cancel safe. If it is canceled while reading,
/// partial data may be lost and the stream state becomes undefined.
/// Use `read_message_cancel_safe` if cancellation is expected.
pub async fn read_message(stream: &mut TcpStream) -> Result<Message> { /* ... */ }
```

---

## Safe Rust Pitfalls

### Avoid `as` for narrowing conversions

The `as` keyword silently truncates values on narrowing conversions (e.g., `i64` to `i32`). Use
`From::from()` for guaranteed-lossless conversions and `TryFrom` when data loss is possible.

```rust
// BAD: silent truncation
let large: i64 = 1_000_000_000_000;
let small: i32 = large as i32;  // silently wraps!

// GOOD: compile-time safe widening
let x: i32 = 42;
let y: i64 = i64::from(x);  // always safe

// GOOD: checked narrowing
let y = i32::try_from(large)
    .map_err(|_| Error::Overflow("value too large for i32"))?;
```

### Use checked arithmetic

Arithmetic on integers can overflow. In debug builds Rust panics; in release builds it silently wraps.
Use `checked_*`, `saturating_*`, or `wrapping_*` methods when overflow is a realistic possibility.

```rust
// BAD: can overflow in release mode
fn total_price(price: u32, quantity: u32) -> u32 {
    price * quantity
}

// GOOD: explicit overflow handling
fn total_price(price: u32, quantity: u32) -> Option<u32> {
    price.checked_mul(quantity)
}
```

Consider enabling overflow checks in release builds for non-performance-critical applications:

```toml
[profile.release]
overflow-checks = true
```

### Prefer `.get()` over direct indexing

Direct indexing (`arr[i]`) panics on out-of-bounds access. Use `.get()` which returns `Option`,
or use slice pattern matching to access elements safely.

```rust
// BAD: panics if index is out of bounds
let elem = arr[index];

// GOOD: returns Option
let elem = arr.get(index);

// GOOD: slice pattern matching -- compiler enforces correctness
match items.as_slice() {
    [] => handle_empty(),
    [single] => handle_one(single),
    [first, rest @ ..] => handle_many(first, rest),
}
```

### Prefer `split_at_checked` over `split_at`

`split_at` panics if the index is out of bounds. `split_at_checked` returns `Option` instead.

```rust
// BAD: panics if mid > arr.len()
let (left, right) = arr.split_at(mid);

// GOOD: returns Option for safe handling
match arr.split_at_checked(mid) {
    Some((left, right)) => process(left, right),
    None => handle_out_of_bounds(),
}
```

---

## Defensive Programming

### Avoid wildcard catch-all on owned enums

Wildcard `_` arms in `match` on enums you control hide new variants added later. Spell out all
variants explicitly so the compiler forces you to handle additions.

```rust
// BAD: new variants will silently fall through
match state {
    State::Active => { /* ... */ }
    State::Inactive => { /* ... */ }
    _ => { /* catch-all hides future variants */ }
}

// GOOD: exhaustive -- compiler warns on new variants
match state {
    State::Active => { /* ... */ }
    State::Inactive => { /* ... */ }
    State::Suspended | State::Deleted => { /* shared logic */ }
}
```

### Destructure structs in trait impls

When manually implementing traits like `PartialEq`, `Debug`, or `Hash`, destructure `self` so the
compiler errors when a new field is added but not handled.

```rust
// BAD: new fields silently ignored
impl PartialEq for Order {
    fn eq(&self, other: &Self) -> bool {
        self.id == other.id && self.total == other.total
        // forgot self.discount!
    }
}

// GOOD: destructure forces handling all fields
impl PartialEq for Order {
    fn eq(&self, other: &Self) -> bool {
        let Self { id, total, discount, created_at: _ } = self;
        let Self { id: other_id, total: other_total, discount: other_discount, created_at: _ } = other;
        id == other_id && total == other_total && discount == other_discount
    }
}
```

### Avoid `..Default::default()` for struct initialization

Using `..Default::default()` to fill in remaining fields hides new fields from the compiler.
Set all fields explicitly so the compiler forces you to handle additions.

```rust
// BAD: new fields silently get default values
let config = AppConfig {
    name: "myapp".into(),
    ..Default::default()  // what else is being set? will new fields be correct?
};

// GOOD: explicit -- compiler errors when a new field is added
let config = AppConfig {
    name: "myapp".into(),
    port: 8080,
    max_retries: 3,
    timeout: Duration::from_secs(30),
};
```

### Replace boolean parameters with enums

Boolean parameters are unreadable at call sites and error-prone. Use enums instead.

```rust
// BAD: what does `true` mean here?
process_data(&data, true, false);

// GOOD: self-documenting call site
enum Compression { Enabled, Disabled }
enum Encryption { Enabled, Disabled }

process_data(&data, Compression::Enabled, Encryption::Disabled);
```

### Implement Debug manually for sensitive types

Blindly deriving `Debug` on types containing passwords, tokens, or keys can leak secrets into logs.
Implement `Debug` manually and redact sensitive fields. Destructure `self` to catch new fields.

```rust
// BAD: password printed in plaintext in logs
#[derive(Debug)]
struct Credentials {
    username: String,
    password: String,  // will appear in debug output!
}

// GOOD: manual Debug with redaction and destructuring
struct Credentials {
    username: String,
    password: String,
}

impl std::fmt::Debug for Credentials {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        let Self { username, password: _ } = self;
        f.debug_struct("Credentials")
            .field("username", username)
            .field("password", &"[REDACTED]")
            .finish()
    }
}
```

### Guard Serialize/Deserialize on validated types

Do not blindly derive `Serialize`/`Deserialize` on newtypes with validation invariants. Use
`#[serde(try_from = "RawType")]` to run validation on deserialization.

```rust
// BAD: deserialization bypasses validation -- empty email is accepted
#[derive(Deserialize)]
pub struct EmailAddress(String);

// GOOD: validation runs on every deserialization
#[derive(Deserialize)]
#[serde(try_from = "String")]
pub struct EmailAddress(String);

impl TryFrom<String> for EmailAddress {
    type Error = EmailError;
    fn try_from(raw: String) -> Result<Self, Self::Error> {
        if is_valid_email(&raw) { Ok(Self(raw)) }
        else { Err(EmailError::Invalid(raw)) }
    }
}
```

---

## Concurrency

### Consistent lock acquisition order

When multiple locks must be held simultaneously, always acquire them in the same order across all
code paths to prevent deadlocks.

```rust
// BAD: different acquisition order in different functions
fn update_a_then_b(a: &Mutex<A>, b: &Mutex<B>) {
    let _a = a.lock().unwrap();
    let _b = b.lock().unwrap();  // order: a, b
}

fn update_b_then_a(a: &Mutex<A>, b: &Mutex<B>) {
    let _b = b.lock().unwrap();
    let _a = a.lock().unwrap();  // order: b, a -- DEADLOCK risk!
}

// GOOD: always acquire in the same order (e.g. a before b)
fn update_both(a: &Mutex<A>, b: &Mutex<B>) {
    let _a = a.lock().unwrap();
    let _b = b.lock().unwrap();  // always a, then b
}
```

### Beware `if let` / `while let` lock scoping

Temporaries created in an `if let` condition live for the *entire* `if`/`else` block, including the
`else` branch. This means a read-lock acquired in the `if let` condition is still held in the `else`
branch, causing deadlocks if a write-lock is taken there.

```rust
// BAD: read lock held across else branch -- deadlocks!
if let Some(val) = *map.read().unwrap() {
    println!("Found: {val}");
} else {
    let mut guard = map.write().unwrap();  // DEADLOCK: read lock still held
    *guard = Some(42);
}

// GOOD: drop the lock before the else branch
let has_value = map.read().unwrap().is_some();
if has_value {
    println!("Found value");
} else {
    let mut guard = map.write().unwrap();
    *guard = Some(42);
}
```

### `try_join!` cancels siblings on failure

When using `try_join!`, if any task fails, the remaining tasks are **cancelled**. If all tasks
must run to completion regardless of individual failures, use `join!` and handle errors afterward.

```rust
// BAD: if stop_a fails, stop_b and stop_c are cancelled
try_join!(stop_service_a(), stop_service_b(), stop_service_c())?;

// GOOD: all services get a chance to stop
let (a, b, c) = join!(stop_service_a(), stop_service_b(), stop_service_c());
a?; b?; c?;
```

---

## Recommended Clippy Lints

These lints catch many of the issues described above at compile time. Consider adding them to
your project's `Cargo.toml` or crate root:

```toml
[lints.clippy]
# Numeric safety
checked_conversions = "warn"
cast_possible_truncation = "warn"
cast_sign_loss = "warn"
cast_possible_wrap = "warn"
arithmetic_side_effects = "warn"

# Unwraps and panics
unwrap_used = "warn"
expect_used = "warn"
indexing_slicing = "warn"

# Defensive programming
wildcard_enum_match_arm = "warn"
fn_params_excessive_bools = "warn"
must_use_candidate = "warn"
fallible_impl_from = "deny"

# Path safety
join_absolute_paths = "warn"

# Serde
serde_api_misuse = "deny"
```
