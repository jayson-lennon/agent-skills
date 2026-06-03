# Testing Patterns

Load this reference when writing any Rust test code.

---

## 1. One Test, One Behavior

Every test asserts exactly one semantic concept. When it fails, the test name alone tells you
_what_ broke. A test has exactly one `// When` and one `// Then` block. A `// Then` may have
`// And` lines, but only when they elaborate on the same behavior — never a different one.

### What Counts as One Concept

```rust
// ✅ ONE concept — checking multiple fields of the same result
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

### What Counts as Separate Concepts — Split Into Separate Tests

```rust
// ❌ BAD — two When/Then blocks in one test
#[test]
fn stream_token_appends_to_assistant_entry() {
    // ...setup...
    // When processing the first token.
    // Then the entry has "Hello".
    // When processing a second token.
    // Then the text is "Hello world".
}

// ✅ GOOD — split into two tests
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
// ❌ BAD — checking state change AND command emission in one test
#[test]
fn submit_message_clears_input_and_enqueues() {
    // ...setup...
    // When submitting a message.
    // Then the input buffer is cleared.
    // And EnqueueUserMessage was returned.
}

// ✅ GOOD — split into separate tests
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
// ❌ BAD — checking multiple entry type renders in one test
#[test]
fn render_mixed_entries() {
    // Given system, user, actor, and assistant entries.
    // When rendering.
    // Then line 6 is system (dark gray).
    // And line 7 is user (">" prefix, bold).
    // And line 8 is actor (yellow).
    // And line 9 is assistant (cyan).
}

// ✅ GOOD — one test per entry type
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

### The Rule

Duplicated test setup is acceptable. Do not combine tests to avoid setup duplication.
Each `#[test]` function answers exactly one question about the system.

---

## 2. BDD-Style Structure — Given / When / Then

Every test has three sections in order: Given, When, Then. Use `//` comments to mark them.
The test name reads as a standalone behavior description in the test report.

### Basic Pattern

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

### Testing Intent Handlers

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

### Testing Validators

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

### Testing Domain Actors

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

### The Rule

Every test has `// Given`, `// When`, `// Then` comments in that order.
For simple cases, inline the sections. For complex setup, use helper functions or builders.
The test name describes the behavior so it reads as a sentence in `cargo test` output.

---

## 3. Observable Behavior Only — No Testing Internals

Tests verify _what the system does_, not _how it does it_. If you can't test observable behavior,
that's a design problem — create an abstraction, don't test internals.

### What to Test

```rust
// ✅ GOOD — test the public API
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

### What NOT to Test

```rust
// ❌ BAD — testing private methods
#[test]
fn internal_sort_uses_quickselect() {
    let processor = Processor::new();
    // Testing a private implementation detail.
    // If the algorithm changes, this test breaks for no good reason.
    assert_eq!(processor.internal_sort_order(), Algorithm::QuickSelect);
}

// ❌ BAD — testing struct field values directly
#[test]
fn parser_sets_is_compound_flag() {
    let result = parse("compound expression");
    // Coupled to internal representation.
    assert!(result.is_compound);
}
```

### The Rule

If the only way to test something is to inspect internal state, the type needs a
semantic method that exposes the observable behavior. Ask: "if this implementation
changed entirely, would this test still be valid?" If no, it's testing internals.

---

## 4. Parameterized Tests with rstest

When the same assertion logic applies to many inputs, use `rstest` to parameterize.
Each `#[case]` must test the **same property** — do not use `rstest` to combine
different behaviors into one test.

### The Pattern

```rust
#[rstest::rstest]
#[case(Key::Tab, "Tab")]
#[case(Key::Enter, "Enter")]
#[case(Key::Esc, "Esc")]
fn key_display(#[case] key: Key, #[case] expected: &str) {
    assert_eq!(key.display(), expected);
}
```

### When NOT to Use rstest

```rust
// ❌ BAD — different behaviors crammed into cases
#[rstest::rstest]
#[case::empty("", Err(ParseError::Empty))]
#[case::valid("abc", Ok(ParseResult { /* ... */ }))]
fn parse(input: &str, expected: Result<ParseResult, ParseError>) { /* ... */ }
// These test different behaviors (validation vs success).
// Use two separate BDD-style tests.
```

### The Rule

Use `rstest` when the same assertion logic runs against different inputs.
For edge cases that don't fit a simple expected value, prefer a standalone BDD test.

---

## 5. Test Utilities

### Test Builders

Create domain-specific builders for complex setup. Builders live in each feature's
test module, not in a shared test utilities crate.

```rust
// ✅ GOOD — builder for test setup
fn test_session() -> SessionBuilder {
    SessionBuilder::new()
        .with_id(SessionId::from(Uuid::new_v4()))
        .with_entries(vec![ChatEntry::user("hello")])
}

fn test_session() -> Session {
    test_session().build()
}
```
