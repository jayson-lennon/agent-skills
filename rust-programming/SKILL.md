---
name: rust-programming
description: Rust code quality rules that eliminate technical debt. Activates on any Rust coding task. Enforces aggressive newtypes, subsystem-level context structs, loop isolation, block scoping, service traits with DI, and error_stack error handling. Every rule has concrete positive and negative code examples.
---

# Rust Programming

You are an expert Rust software developer prioritizing developer UX and code maintainability.

**Load [references/testing-patterns.md](references/testing-patterns.md) when writing tests.**

---

## 1. Newtypes - Aggressive Wrapping

Wrap any primitive that carries domain meaning, even if the wrapped type is "obvious."
The type system is documentation. A function signature `fn send(id: Uuid)` tells you nothing
about _what_ the ID identifies. `fn send(id: SessionId)` does.

### When to Wrap

```rust
// ✅ DO - two strings that mean different things
struct ProviderName(String);
struct ModelName(String);
// Prevents passing a model name where a provider is expected.

// ✅ DO - IDs are always wrapped, and use Uuid not String
struct SessionId(Uuid);
struct WorkflowId(Uuid);
// Even though both are Uuids, you never want to pass
// a session ID to a function expecting a workflow ID.

// ✅ DO - numeric types with different units or semantics
struct TokenCount(u32);
struct TokenLimit(u32);
struct MaxRetries(usize);
// Prevents accidentally comparing a count to a limit
// without it being obviously intentional.

// ✅ DO - IDs over wire formats or persistence layers
struct MessageId(Uuid);
struct ActorRefId(Uuid);
// Any ID that crosses a boundary gets its own type.
```

### When NOT to Wrap

```rust
// ❌ DON'T - wrapping when the inner type already carries meaning
struct FilePath(PathBuf);  // PathBuf IS a file path. No ambiguity.
struct ByteCount(usize);   // usize with "byte" in the name is clear.

// ❌ DON'T - wrapping types used only within a single function
fn process(data: &[u8]) {
    // A local `let chunk_size = 1024;` is fine.
    // Don't make `struct ChunkSize(usize)` for this.
}

// ❌ DON'T - wrapping just to wrap
struct Port(u16);  // Only if it appears alongside other u16s.
                    // If it's the only u16 in scope, the name "port" is enough.
```

### The Rule

If a type appears in a function signature alongside another value of the same primitive type,
it must be a newtype. If it crosses a module boundary or appears in a protocol message, it
must be a newtype. IDs preferably use `Uuid`.

---

## 2. Context Structs - Subsystem-Level Capability Bundles

When multiple functions across a subsystem share the same set of dependencies,
bundle them into a context struct. Adding a new capability to the subsystem changes
the struct - zero call sites change.

### The Pattern

```rust
// ✅ GOOD - RenderCtx provides subsystem capabilities
struct RenderCtx<'a> {
    state: &'a AppState,
    theme: &'a Theme,
    config: &'a Config,
}

// Function-specific params stay in the signature.
// The ctx gives everything the render subsystem needs.
// `width` is specific to THIS function call, not the subsystem.
fn render_sidebar(ctx: &RenderCtx, width: u16) -> Vec<Line> { /* ... */ }
fn render_status_bar(ctx: &RenderCtx, position: StatusBarPosition) -> Vec<Line> { /* ... */ }
fn render_chat_entry(ctx: &RenderCtx, entry: &ChatEntry) -> Vec<Line> { /* ... */ }
// Adding a new capability to RenderCtx changes the struct.
// Zero call sites change. But `width`, `position`, `entry`
// stay as parameters because they vary per call.
```

### The Anti-Pattern

```rust
// ❌ BAD - capabilities threaded as individual parameters
// Every new capability means changing every call site.
fn render_sidebar(
    state: &AppState,
    theme: &Theme,
    config: &Config,
    registry: &WidgetRegistry,
) -> Vec<Line> { /* ... */ }

fn render_status_bar(
    state: &AppState,
    theme: &Theme,
    config: &Config,
    registry: &WidgetRegistry,
) -> Vec<Line> { /* ... */ }
```

```rust
// ❌ DON'T - context for a single function or two.
// This isn't a subsystem, it's just hiding parameters.
struct OrderContext<'a> {
    db: &'a Db,
    user_id: &'a UserId,
}
fn place_order(ctx: &OrderContext) { /* ... */ }
// This function is the ONLY consumer. Just pass db and user_id directly.
```

### The Rule

Create a context struct when **three or more functions** in a subsystem share the same
dependency set. The context struct holds "what the subsystem needs"; individual parameters
hold "what this specific function needs." If only one or two functions consume it, use
plain parameters.

---

## 3. Loop Isolation - One Loop Per Function

Loops grow fast. A function with two sequential `for`, `while`, or `loop` blocks forces the
reader to hold multiple phases of computation in their head. Extract each one into its own
named function so the top level reads as a recipe. Prefer iterators over explicit loops where
reasonable - multiple iterator chains in one function are fine.

### What Counts as a Loop

```rust
// These are LOOPS - subject to the one-per-function rule:
for item in collection { /* ... */ }
while condition { /* ... */ }
loop { /* ... */ }

// These are ITERATORS - multiple per function is fine:
collection.iter().filter(/* ... */).map(/* ... */).collect()
collection.iter().any(/* ... */)
collection.iter().for_each(/* ... */)
// Iterator chains are expressions. They compose well and stay readable.
```

### The Pattern

```rust
// ✅ GOOD - each loop is a named function
fn rebuild_index(state: &mut State, entries: &[Entry]) {
    let active = active_entries(entries);
    populate_index(&mut state.index, &active);
    notify_watchers(&state.watchers, &active);
}
// The top level reads as a 3-step recipe.
// Each helper does exactly one pass over the data.
```

```rust
// ✅ GOOD - multiple iterator chains in one function is fine
fn summarize(entries: &[Entry]) -> Summary {
    let active_count = entries.iter().filter(|e| e.is_active()).count();
    let total_bytes: usize = entries.iter().map(|e| e.size()).sum();
    Summary { active_count, total_bytes }
}
// Both are iterator chains - expressions that compose cleanly.
// No need to extract these into separate functions.
```

### The Anti-Pattern

```rust
// ❌ BAD - two sequential for-loops doing conceptually different work
fn rebuild_index(state: &mut State, entries: &[Entry]) {
    // Loop 1: validate and collect
    let mut valid = vec![];
    for entry in entries {
        if entry.status == Status::Active {
            valid.push(entry.clone());
        }
    }
    // Loop 2: insert into index
    for entry in &valid {
        state.index.insert(entry.id, entry.data.clone());
    }
    // Loop 3: notify watchers
    for entry in &valid {
        state.watchers.notify(&entry.id);
    }
}
// Reader has to hold all three phases in their head.
// The function name "rebuild_index" doesn't convey "validate, insert, notify."
```

### The Rule

One `for`, `while`, or `loop` per function. Multiple iterator chains are fine - they're
expressions that compose cleanly and stay readable. When a function needs multiple explicit
loops, each loop becomes a named helper and the parent reads as step-by-step pseudocode.

## 4. Block Scoping - Limit Variable Lifetimes

When a value requires multiple setup steps or intermediate bindings, wrap the sequence
in a block expression. The final binding is immutable and temporaries don't leak.

### Create-Then-Configure

```rust
// ❌ BAD - mutable binding lives past setup
let mut services = ServiceBuilder::new();
services.register(auth_backend);
services.register(cache_backend);
services.register(storage_backend);
let services = services.build();
// `services` was mut, now shadowed. But the pattern invites
// accidental reuse of the pre-build value in long functions.

// ✅ GOOD - setup is scoped, final binding is immutable
let services = {
    let mut builder = ServiceBuilder::new();
    builder.register(auth_backend);
    builder.register(cache_backend);
    builder.register(storage_backend);
    builder.build()
};
// `builder` doesn't exist outside the block. Can't be misused.
```

### Intermediate Values

```rust
// ❌ BAD - intermediate bindings remain in scope after final value is computed
let raw = compute_raw_offset(cursor, viewport);
let clamped = raw.clamp(0, max_scroll);
let adjusted = snap_to_line_boundary(clamped);
// `raw` and `clamped` still exist. Nothing stops later code from using them.

// ✅ GOOD - only the final value escapes the block
let adjusted = {
    let raw = compute_raw_offset(cursor, viewport);
    let clamped = raw.clamp(0, max_scroll);
    snap_to_line_boundary(clamped)
};
// `raw` and `clamped` don't exist out here. Only `adjusted` remains.
// Easy to extract into a function later since the block is already self-contained.
```

### The Rule

If a value requires mutability for setup, scope the setup in a block so the final
binding is immutable. If computing a value requires intermediate `let` bindings that
have no purpose after the computation, wrap them in a block. A block is a
pre-extracted function - if the block grows beyond 10 lines, extract it.

---

## 5. Service Traits and Dependency Injection

Every external dependency or service must have a trait abstraction. Production code
implements the trait with the real backend. Tests implement it with fakes. No function
should directly depend on a concrete service type.

### Service Trait

```rust
// ✅ GOOD - trait defines the capability, error colocated with it
use wherror::Error;

#[derive(Debug, Error)]
#[error(debug)]
pub struct SessionStoreError;

pub trait SessionStore {
    fn save(&self, session: &Session) -> Result<(), Report<SessionStoreError>>;
    fn load(&self, id: &SessionId) -> Result<Option<Session>, Report<SessionStoreError>>;

    /// Returns the name of this backend for debugging.
    fn name(&self) -> &'static str;
}
```

### Service Wrapper

```rust
// ✅ GOOD - shared, cloneable wrapper around a trait object
use std::sync::Arc;
use derive_more::Debug;

#[derive(Debug, Clone)]
pub struct SessionStoreService {
    #[debug("SessionStore<{}>", self.backend.name())]
    backend: Arc<dyn SessionStore>,
}

impl SessionStoreService {
    pub fn new(backend: Arc<dyn SessionStore>) -> Self {
        Self { backend }
    }

    pub fn save(&self, session: &Session) -> Result<(), Report<SessionStoreError>> {
        self.backend.save(session)
    }
}
```

### Services Container

```rust
// ✅ GOOD - created once at startup, shared throughout the application
#[derive(Debug, Clone)]
pub struct Services {
    pub session_store: SessionStoreService,
    pub llm_service: LlmServiceFactoryService,
    pub config_storage: ConfigStorageService,
    // ...
}
// Every field is a service wrapper - cheap to clone, shared via Arc.
// Tests construct with fakes. Production constructs with real backends.
```

### The Anti-Pattern

```rust
// ❌ BAD - depending on a concrete type
fn save_session(store: &SqliteSessionStore, session: &Session) { /* ... */ }
// Can't test without a real database. Can't swap implementations.

// ❌ BAD - standalone traits.rs file
// Traits live with their related types, not in a separate traits.rs.
```

### The Rules

- Every service has a trait. Every trait has a colocated error type.
- Service structs wrap `Arc<dyn Trait>` - never expose the trait object directly.
- Service wrappers include a `name()` method for debugging.
- The `Services` container is the DI container - one instance, shared everywhere.
- Traits live in the same module as the types that implement them, never in standalone files.
- `#[async_trait]` for async methods.

---

## 6. Error Handling - error_stack with wherror

Use `wherror::Error` for error types and `error_stack::Report` for error reporting.
Errors carry context. Every fallible operation attaches _what was happening_.

### Error Type

```rust
use wherror::Error;

#[derive(Debug, Error)]
#[error(debug)]
pub struct SessionStoreError;
// Colocated with SessionStore trait, never in a standalone error.rs.
```

### Result with Context

```rust
use error_stack::{Report, ResultExt};

pub fn load_config(path: &Path) -> Result<Config, Report<ConfigError>> {
    let content = std::fs::read_to_string(path)
        .change_context(ConfigError)
        .attach(printable::path(path))
        .attach("failed to read config file")?;
    Ok(parse_config(&content))
}
```

### Document Errors on Public Functions

```rust
/// Persists the session to storage.
///
/// # Errors
///
/// Returns an error if the storage backend fails to write.
pub fn save(&self, session: &Session) -> Result<(), Report<SessionStoreError>>
```

### The Rules

- Use `wherror::Error` with `#[error(debug)]` - no manual `Display` impls.
- Return `Result<T, Report<YourError>>` from fallible functions.
- Use `.change_context()` to convert underlying errors into domain errors.
- Use `.attach()` to add context about what was happening.
- Colocate errors with their related types - never standalone `error.rs` or `errors.rs`.
- Document errors with `# Errors` doc section on public functions.

---

## Cross-References

- **[Testing Patterns](references/testing-patterns.md)** - Load when writing any test code.
  Covers BDD structure, one-test-one-behavior, observable behavior only, rstest parameterization,
  and test utilities.
