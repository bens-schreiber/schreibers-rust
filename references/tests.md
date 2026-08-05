# Tests

Rules `TS-1` … `TS-8`. The through-line: **a test that fails should tell you what
broke without you reading its body.**

## TS-1: Where tests live

Unit tests live in a `#[cfg(test)] mod tests` in the file under test, where they
can reach private items. Integration tests live in the crate's `tests/`
directory and see only the public API.

```rust
// src/lexer.rs

pub fn tokenize(src: &str) -> Vec<Token> { /* ... */ }

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn tokenize_empty_input_returns_no_tokens() {
        assert!(tokenize("").is_empty());
    }
}
```

## TS-2: `namespace_input_expectation`

Three segments: the subject under test, the input condition, the expected
outcome. Drop `namespace` when the enclosing module or file already supplies it
(TS-7).

```rust
// DO
fn tokenize_empty_input_returns_no_tokens()
fn tokenize_unterminated_string_returns_lex_error()
fn parse_nested_groups_preserves_depth()

// DON'T: no input condition, no expectation
fn test_tokenize()
fn tokenize_works()
fn it_should_handle_the_case_where_the_string_is_not_terminated()
```

Do not prefix with `test_`; `#[test]` already says that.

## TS-3: Arrange / Act / Assert labels

Label the three phases once they are not obvious at a glance. Skip the labels
when each phase is a single line.

The label stands alone; never extend it with narrative. If a phase needs
explaining, the explanation belongs in an assertion message (TS-5) or in a
better-named test or constant.

```rust
// DO
// Arrange
const SRC: &str = "let vex = 1;";
let lexer = Lexer::new();

// Act
let tokens = lexer.tokenize(SRC);

// Assert
assert_eq!(tokens.len(), 5);

// DON'T: narrative bolted onto the label
// Arrange: set up the lexer and the source string we will use
```

```rust
// DO: small enough that labels would be noise
#[test]
fn tokenize_empty_input_returns_no_tokens() {
    assert!(tokenize("").is_empty());
}
```

## TS-4: Every value is a named constant in Arrange

No literal appears inside `Act` or `Assert`. A value used by both a call and an
assertion must come from a single binding, so the two cannot drift apart in
future edits.

```rust
// DO
#[test]
fn roster_known_member_is_returned() {
    // Arrange
    const MEMBER: &str = "keyleth";
    let roster = Roster::new();

    // Act
    let found = roster.lookup(MEMBER);

    // Assert
    assert_eq!(found, Some(MEMBER));
}

// DON'T: "keyleth" written twice; an edit to one silently passes
#[test]
fn roster_known_member_is_returned() {
    let roster = Roster::new();

    let found = roster.lookup("keyleth");

    assert_eq!(found, Some("keyleth"));
}
```

`const` where the type allows it, `let` otherwise. Derived expectations may be
computed in `Arrange` from the same constant:

```rust
// Arrange
const NAMES: [&str; 3] = ["vex", "vax", "percy"];
let expected_len = NAMES.len();
```

## TS-5: Custom panic messages when the reason is not obvious

Put the explanation in the assertion message, not in a comment above it. A
comment is invisible in CI output; the message is the failure.

```rust
// DO
assert!(
    tokens.is_empty(),
    "a comment-only source yields no tokens: trivia is stripped before parsing",
);

// DON'T
// A comment-only source yields no tokens because trivia is stripped.
assert!(tokens.is_empty());
```

`assert_eq!` already prints both sides, so it needs a message only when *why*
they should be equal is the non-obvious part.

## TS-6: Coalesce cases that share an Arrange

When several tests differ only in their input, make them one test with a
`// Case: <name>` block per case. Hoist only the shared Arrange; each case keeps
its own Act and Assert inside its own block (SC-3).

```rust
#[test]
fn lookup_various_inputs_returns_matching_member() {
    // Arrange
    let roster = Roster::new();

    // Case: empty query
    {
        // Act
        let found = roster.lookup("");

        // Assert
        assert!(found.is_none());
    }

    // Case: exact match
    {
        // Arrange
        const MEMBER: &str = "vex";

        // Act
        let found = roster.lookup(MEMBER);

        // Assert
        assert_eq!(found, Some(MEMBER));
    }

    // Case: unknown member
    {
        // Act
        let found = roster.lookup("scanlan");

        // Assert
        assert!(found.is_none());
    }
}
```

Note the case-local `Arrange` in the second block: values used by one case are
declared in that case, per TS-4.

The cost of coalescing is that the first failing case aborts the rest. Coalesce
when the cases are variations on one behavior; keep them separate when each case
is a distinct behavior you want reported independently.

## TS-7: Split a namespace group past three tests

Once a `namespace_*` group exceeds three functions, move it into its own `mod`
or file so the prefix can be dropped (TS-2).

```rust
// DO
#[cfg(test)]
mod tests {
    mod tokenize {
        use super::super::*;

        #[test]
        fn empty_input_returns_no_tokens() { /* ... */ }

        #[test]
        fn unterminated_string_returns_lex_error() { /* ... */ }

        #[test]
        fn nested_comments_are_stripped() { /* ... */ }

        #[test]
        fn trailing_newline_is_ignored() { /* ... */ }
    }

    mod parse {
        // ...
    }
}
```

**TS-6 vs. TS-7:** TS-6 merges functions that share a fixture; TS-7 splits
functions that share a subject. Apply TS-6 within a group, TS-7 across groups:
they never contend for the same edit.

## TS-8: Vox Machina names for arbitrary data

Use `vex`, `vax`, `percy`, `keyleth`, `grog`, `pike`, `scanlan` for genuinely
arbitrary dummy values, instead of `foo`/`bar`/`alice`/`bob`.

```rust
// DO
const MEMBER: &str = "keyleth";

// DON'T
const MEMBER: &str = "foo";
```

The rule covers *arbitrary* data only. When a value carries meaning the test
depends on, name it for that meaning:

```rust
// DO: the leading space is the point of the test
const PADDED_NAME: &str = " grog";

// DON'T: hides what makes the input interesting
const VAX: &str = " grog";
```
