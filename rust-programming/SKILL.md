---
name: rust-programming
description: Rust code quality rules that eliminate technical debt and improve program reliability. Activate on all Rust coding tasks.
---

# Rust Programming

You are an expert Rust software developer prioritizing developer UX and code maintainability.

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

### When a Global Context Already Exists

The "context struct" is often a single application-wide container (commonly named
`Services`). When that global context exists, it IS the context struct for every
function that needs a service. Reaching into it and passing a piece onward is the same
anti-pattern as threading individual capabilities — it re-fragments the bundle the
context exists to prevent.

```rust
// ❌ BAD - pulling a piece out of the global context
fn resolve(fs: &FsService, asset: &Path) -> ResolvedAsset { /* ... */ }
// Caller: resolve(&services.fs, asset)
// Adding a second dependency means changing the signature AND every call site.

// ✅ GOOD - the function takes the whole context
fn resolve(services: &Services, asset: &Path) -> ResolvedAsset { /* ... */ }
// Caller: resolve(&services, asset). New capabilities are free.
```

A per-subsystem `Ctx` (e.g. `RenderCtx`) is warranted only when it carries
**additional local state** beyond the global services — a theme, a viewport,
per-request data. If your `Ctx` only forwards `&Services` plus a value or two,
fold those values into `Services` and delete the `Ctx`.

```rust
// ❌ BAD - a Ctx that only forwards Services + one borrowed value
struct AuditCtx<'a> { services: &'a Services, config: &'a Config, root: &'a Path }
fn run_audit(ctx: &AuditCtx) { /* reads ctx.services, ctx.config, ctx.root */ }
// `config` and `root` belong IN Services. This struct adds indirection for nothing.

// ✅ GOOD - fold the values into Services, take the whole thing
fn run_audit(services: &Services) { /* reads services.config() */ }
```

### The Rule

Create a context struct when **three or more functions** in a subsystem share the same
dependency set. The context struct holds "what the subsystem needs"; individual parameters
hold "what this specific function needs." If only one or two functions consume it, use
plain parameters.

The "three or more functions" heuristic decides when to create a *new subsystem* context.
It does **not** gate the global `Services` container, which is passed to any function that
even remotely needs it — passing the whole thing everywhere is the *point*: it eliminates
the churn of adding a capability later.

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

### The Canonical Case: Assembling `Services`

`Services` is the textbook create-then-configure. At the single program entry point,
build every backend in one block. Throwaway pieces — a root-discovery probe, the
`FsService` used to load config/registry — live only inside the block; the final
`Services` is immutable and escapes. Build each real backend **once** and reuse it.

```rust
let services = {
    let fs = FsService::new(Arc::new(RealFs::new()));     // built once
    let registry = RegistryService::load(&fs, &root)?;     // bootstrap exception
    let clock = ClockService::new(Arc::new(RealClock::new()));
    let config = ConfigService::load(&fs, &root)?;         // bootstrap exception
    Services { fs, registry, clock, config }               // immutable, escapes
};
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

`Services` is the application-wide **dependency injection container** — one instance
constructed at program entry and shared by reference everywhere. It bundles every
service the program might need, so adding a capability later is a one-field change with
zero call-site churn.

```rust
// ✅ GOOD - created once at startup, shared throughout the application
#[derive(Debug, Clone)]
pub struct Services {
    pub session_store: SessionStoreService,   // trait-backed (behavior)
    pub llm: LlmService,                       // trait-backed (behavior)
    pub config: ConfigService,                 // Arc<Config> (data)
    // ...
}
// Every field satisfies ONE of two clauses:
//  - cheap to clone (Arc<T> of data), OR
//  - behind a trait (Arc<dyn Trait> service wrapper).
// Both are valid. Never a bare owned HashMap/struct that deep-copies on clone.
```

### Behavior Services vs Data Services — Which Clause Applies?

Decide per field with one question: **is there a *behavior* to swap, or only *data*?**

- **Behavior service** — the *implementation* varies (filesystem, clock, network, DB).
  Trait + `Arc<dyn Trait>` wrapper. Tests swap the backend.
- **Data service** — pure data loaded *through* an already-abstracted service, with no
  behavior to swap (a parsed config cache, a registry read as data). `Arc<T>` concrete,
  cheap to clone. No trait: the *data* is the test seam, swapped via a builder.

```rust
// Behavior: implementation varies → trait + wrapper
pub trait ClockBackend: Send + Sync {
    fn now(&self) -> u64;
    fn name(&self) -> &'static str;
}
pub struct ClockService { backend: Arc<dyn ClockBackend> }   // RealClock / FakeClock

// Data: loaded via FsBackend, nothing to swap → Arc<T>, no trait
pub struct ConfigService {
    root: Arc<Path>,
    config: Arc<Config>,
}
// Tests swap the *data* (Config::default(), a builder), not a backend.
```

A project may mandate trait-backing for uniformity even on data fields — that's a valid
project override, not the default the skill teaches.

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
- Behavior service structs wrap `Arc<dyn Trait>` - never expose the trait object directly.
- Every `Services` field is either a trait-backed service wrapper or an `Arc<T>` of data - never a bare owned collection that deep-copies on clone.
- Service wrappers include a `name()` method for debugging.
- The `Services` container is the **application-wide DI container** - **one instance, shared everywhere**.
- Traits live in the same module as the types that implement them, never in standalone files.
- `#[async_trait]` for async methods.

### Bootstrap Exceptions — When a Raw Service Piece Is Correct

Some functions must take a raw `&FsService` rather than `&Services` because they run
*before* the container exists: they are the factories that build the backends (root
discovery, config load, registry load). These are confined to:

- the single assembly block at program entry, and
- test seeding.

They are not a reach-in violation — you cannot inject a container that hasn't been built.
But they must never appear in the runtime call graph *below* the assembly block.

```rust
// Bootstrap factory: takes &FsService because Services doesn't exist yet.
pub fn load(fs: &FsService, root: &Path) -> Result<Config, Report<ConfigError>> { /* ... */ }
// Called ONLY from the main assembly block or test seeding. Never from a command.
```

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
        .attach(path.to_string_lossy())
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

## 7. Testing Patterns

### One Test, One Behavior

Every test asserts exactly one semantic concept. When it fails, the test name alone tells you
_what_ broke. A test has exactly one `// When` and one `// Then` block. A `// Then` may have
`// And` lines, but only when they elaborate on the same behavior - never a different one.

#### What Counts as One Concept

```rust
// ✅ ONE concept - checking multiple fields of the same result
#[test]
fn parse_returns_correct_id_and_name() {
    // Given a raw config string.
    let raw = "id: abc\nname: test";

    // When parsing.
    let result = parse(raw);

    // Then the id is "abc".
    assert_eq!(result.id, "abc");
    // And the name is "test".
    assert_eq!(result.name, "test");
}
// Both assertions confirm "the parse result is correct." One concept.
```

#### What Counts as Separate Concepts - Split Into Separate Tests

```rust
// ❌ BAD - two When/Then blocks in one test
#[test]
fn stream_token_appends_to_assistant_entry() {
    // ...setup...
    // When processing the first token.
    // Then the entry has "Hello".
    // When processing a second token.
    // Then the text is "Hello world".
}

// ✅ GOOD - split into two tests
#[test]
fn first_stream_token_creates_assistant_entry() {
    // ...setup...
    // When processing StreamToken("Hello").
    // Then the session has an Assistant entry with "Hello".
}

#[test]
fn subsequent_stream_token_appends_to_existing_entry() {
    // ...setup with one token already processed...
    // When processing another StreamToken(" world").
    // Then the text is "Hello world".
}
```

```rust
// ❌ BAD - checking state change AND command emission in one test
#[test]
fn submit_message_clears_input_and_enqueues() {
    // ...setup...
    // When submitting a message.
    // Then the input buffer is cleared.
    // And EnqueueUserMessage was returned.
}

// ✅ GOOD - split into separate tests
#[test]
fn submit_message_clears_input_buffer() {
    // ...setup...
    // When handling Intent::SubmitMessage.
    // Then the input buffer is empty.
}

#[test]
fn submit_message_returns_enqueue_command() {
    // ...setup...
    // When handling Intent::SubmitMessage.
    // Then the result contains EnqueueUserMessage.
}
```

```rust
// ❌ BAD - checking multiple entry type renders in one test
#[test]
fn render_mixed_entries() {
    // Given system, user, actor, and assistant entries.
    // When rendering.
    // Then line 6 is system (dark gray).
    // And line 7 is user (">" prefix, bold).
    // And line 8 is actor (yellow).
    // And line 9 is assistant (cyan).
}

// ✅ GOOD - one test per entry type
#[test]
fn render_system_entry_is_dark_gray() {
    // Given a ChatLogElement with a system entry.
    // When rendering.
    // Then the system entry line has dark gray foreground.
}

#[test]
fn render_user_entry_has_prefix() {
    // Given a ChatLogElement with a user entry.
    // When rendering.
    // Then the user entry line starts with ">".
}
```

#### The Rule

Duplicated test setup is acceptable. Do not combine tests to avoid setup duplication.
Each `#[test]` function answers exactly one question about the system.

---

### BDD-Style Structure - Given / When / Then

Every test has three sections in order: Given, When, Then. Use `//` comments to mark them.
The test name reads as a standalone behavior description in the test report.

#### Basic Pattern

```rust
#[test]
fn pop_returns_none_when_stack_empty() {
    // Given an empty stack.
    let mut stack = Stack::default();

    // When popping from the stack.
    let item = stack.pop();

    // Then we get nothing back.
    assert!(item.is_none());
}
```

#### Testing Intent Handlers

```rust
#[test]
fn quit_sets_should_quit_in_state() {
    // Given default app state.
    let mut state = AppState::default();

    // When handling Intent::Quit.
    let result = IntentHandler::handle(&Intent::Quit, &mut state);

    // Then should_quit is set to true.
    assert!(state.should_quit);
    // And no commands are emitted.
    assert!(result.commands.is_empty());
}
```

#### Testing Validators

```rust
#[test]
fn submit_message_rejected_when_buffer_empty() {
    // Given an empty input buffer.
    let state = AppState::default();

    // When validating submit message.
    let result = validate_submit_message(&state);

    // Then validation fails with EmptyBuffer.
    assert!(matches!(result, Err(SubmitMessageError::EmptyBuffer)));
}
```

#### Testing Domain Actors

```rust
#[test]
fn stream_token_appends_to_assistant_entry() {
    // Given a projector with an active session.
    let state = State::new(AppState::default());
    let sink = RecordingSink::new();
    let session_actor = SessionPersistenceActor::activate(&mut ctx);

    // When handling StreamToken("Hello").
    session_actor.handle_stream_token(&StreamToken { /* ... */ }, &sink);

    // Then the session has an Assistant entry with "Hello".
    let s = state.read();
    assert_eq!(s.active_session().last_entry_text(), "Hello");
}
```

#### The Rule

Every test has `// Given`, `// When`, `// Then` comments in that order.
For simple cases, inline the sections. For complex setup, use helper functions or builders.
The test name describes the behavior so it reads as a sentence in `cargo test` output.

---

### Observable Behavior Only - No Testing Internals

Tests verify _what the system does_, not _how it does it_. If you can't test observable behavior,
that's a design problem - create an abstraction, don't test internals.

#### What to Test

```rust
// ✅ GOOD - test the public API
#[test]
fn save_then_load_roundtrips_session() {
    // Given a session store.
    let store = InMemorySessionStore::new();
    let session = Session::new(SessionId::new());

    // When saving and loading.
    store.save(&session).expect("save");
    let loaded = store.load(&session.id()).expect("load");

    // Then the loaded session matches the original.
    assert_eq!(loaded.unwrap().id(), session.id());
}
```

#### What NOT to Test

```rust
// ❌ BAD - testing private methods
#[test]
fn internal_sort_uses_quickselect() {
    let processor = Processor::new();
    // Testing a private implementation detail.
    // If the algorithm changes, this test breaks for no good reason.
    assert_eq!(processor.internal_sort_order(), Algorithm::QuickSelect);
}

// ❌ BAD - testing struct field values directly
#[test]
fn parser_sets_is_compound_flag() {
    let result = parse("compound expression");
    // Coupled to internal representation.
    assert!(result.is_compound);
}
```

#### The Rule

If the only way to test something is to inspect internal state, the type needs a
semantic method that exposes the observable behavior. Ask: "if this implementation
changed entirely, would this test still be valid?" If no, it's testing internals.

---

### Parameterized Tests with rstest

When the same assertion logic applies to many inputs, use `rstest` to parameterize.
Each `#[case]` must test the **same property** - do not use `rstest` to combine
different behaviors into one test.

#### The Pattern

```rust
#[rstest::rstest]
#[case(Key::Tab, "Tab")]
#[case(Key::Enter, "Enter")]
#[case(Key::Esc, "Esc")]
fn key_display(#[case] key: Key, #[case] expected: &str) {
    assert_eq!(key.display(), expected);
}
```

#### When NOT to Use rstest

```rust
// ❌ BAD - different behaviors crammed into cases
#[rstest::rstest]
#[case::empty("", Err(ParseError::Empty))]
#[case::valid("abc", Ok(ParseResult { /* ... */ }))]
fn parse(input: &str, expected: Result<ParseResult, ParseError>) { /* ... */ }
// These test different behaviors (validation vs success).
// Use two separate BDD-style tests.
```

#### The Rule

Use `rstest` when the same assertion logic runs against different inputs.
For edge cases that don't fit a simple expected value, prefer a standalone BDD test.

---

### Test Utilities

#### Test Builders

Create domain-specific builders for complex setup. Builders live in each feature's
test module colocated per fake/struct. Prefer a shared test module for shared test features.

```rust
// ✅ GOOD - builder for test setup
fn test_session() -> SessionBuilder {
    SessionBuilder::new()
        .with_id(SessionId::from(Uuid::new_v4()))
        .with_entries(vec![ChatEntry::user("hello")])
}

fn test_session() -> Session {
    test_session().build()
}
```

#### Services Test Builder

When the global context is trait-backed, tests swap the whole container — not each
call site. Provide a single builder, gated behind a test feature, that defaults to
fakes and overrides per test. This replaces every raw struct-literal construction of
`Services`.

```rust
let svc = Services::test()
    .registry_specs([LicenseSpec::new("MIT")])
    .config(Config::default(), PathBuf::from("/proj"))
    .build();   // defaults: FakeFs, FakeClock::fixed(0), empty registry
```
