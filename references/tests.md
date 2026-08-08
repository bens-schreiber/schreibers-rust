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
// DO: three phases, none of them a single line
// Arrange
let mut lexer = Lexer::new();
lexer.set_mode(Mode::Strict);
let src = fixture("manifest.txt");

// Act
let tokens = lexer.tokenize(&src);
let idents = tokens.iter().filter(|t| t.is_ident());

// Assert
assert_eq!(tokens.len(), 5);
assert_eq!(idents.count(), 2);

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

## TS-4: Bind what repeats, inline what doesn't

The reason to bind is **drift**: if a call and its assertion each spell out the
same literal, an edit to one silently passes. One binding makes that impossible.

```rust
// DO: "keyleth" is used twice, so it is bound once
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

// DON'T: written twice; an edit to one silently passes
let found = roster.lookup("keyleth");
assert_eq!(found, Some("keyleth"));
```

A value used **once** stays inline. Hoisting it just makes the reader jump to
the top of the test to learn what the call actually receives.

```rust
// DO
assert!(roster.lookup("scanlan").is_none());

// DON'T: a name and a lookup, to say "scanlan" one time
const UNKNOWN_MEMBER: &str = "scanlan";
assert!(roster.lookup(UNKNOWN_MEMBER).is_none());
```

Same test for derived values: compute an expectation in `Arrange` only when it
is genuinely reused or when the derivation itself is the point. Otherwise put
the expression in the assertion.

Use `const` where the type allows it, `let` otherwise.

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
fn lookup_returns_member_only_on_exact_match() {
    // Arrange
    let roster = Roster::new();

    // Case: empty query
    {
        assert!(roster.lookup("").is_none());
    }

    // Case: exact match
    {
        const MEMBER: &str = "vex";

        assert_eq!(roster.lookup(MEMBER), Some(MEMBER));
    }

    // Case: unknown member
    {
        assert!(roster.lookup("scanlan").is_none());
    }
}
```

`// Case:` is the only label these blocks need; each phase here is one line, so
TS-3's labels would be noise. The case-local `const` in the second block is
there because that case uses the value twice (TS-4).

The cost of coalescing is that the first failing case aborts the rest. Coalesce
when the cases are variations on one behavior; keep them separate when each case
is a distinct behavior you want reported independently.

## TS-7: Split a namespace group once it gets big

Once a `namespace_*` group passes about four functions, move it into its own
`mod` or file so the prefix can be dropped (TS-2). Do this when you are already
restructuring that group. Adding one test to an existing module is not a reason
to reorganize and rename the tests around it.

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

## TS-8: Vox Machina names for pure placeholders

The trigger is narrow. Ask: *would I otherwise have written `foo`, `bar`,
`alice`, or `bob` here?* If yes, write `vex`, `vax`, `percy`, `keyleth`, `grog`,
`pike`, or `scanlan` instead. If no, TS-8 does not apply.

A value that models a real thing in the domain is not a placeholder, even when
the specific value is invented for the test. Sites, hostnames, regions, tenants,
SKUs, and paths all read as real data to whoever comes next, and a fantasy name
there is a puzzle, not a joke. Existing codebase convention beats both.

```rust
// DO: reads like a site this system could actually have
const SITE: &str = "us-east-2b";

// DON'T: nobody will know whether "vox" is a real site name
const SITE: &str = "vox";
```

Naming still beats theming: if a value carries meaning the test depends on, name
the binding for that meaning.

```rust
// DO: the leading space is the point of the test
const PADDED_NAME: &str = " grog";

// DON'T: hides what makes the input interesting
const VAX: &str = " grog";
```
