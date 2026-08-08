# Documentation

Rules `DOC-1` … `DOC-6`. The through-line: **write for someone who just opened
this file, and say only what the code cannot.**

## DOC-1: Markdown and line breaks, liberally

A doc comment should read well twice: as source, and as rendered rustdoc.

Structure: a one-line summary, a blank `///` line, then paragraphs, headings,
and bullet lists as needed.

```rust
// DO

/// A single parsed manifest entry.
///
/// Entries are interned, so two entries with the same name share storage.
/// This means comparison is pointer-equality and is cheap enough to do in a
/// hot loop.
///
/// # Interning caveats
///
/// - The intern table is per-`Parser`, so entries from two parsers must be
///   compared by name.
///   Comparing them by pointer silently returns `false`.
/// - Entries are never freed before the parser is dropped.
pub struct Entry {
    /// Interned; unique within a single parser.
    pub name: Symbol,

    pub line: u32,
    pub column: u32,
}
```

```rust
// DON'T: a wall of prose with no summary line, headings, or breaks

/// A single parsed manifest entry. Entries are interned, so two entries with the same name share storage, which means comparison is pointer-equality. Note the intern table is per-Parser so entries from two parsers must be compared by name, and entries are never freed before the parser is dropped.
pub struct Entry {
    /// Interned; unique within a single parser.
    pub name: Symbol,
    pub line: u32,
    pub column: u32,
}
```

The first line is special: rustdoc uses it alone as the summary in index and
search results. Keep it to one sentence that stands on its own.

Note the bullet continuations above: they indent to sit under the bullet's
*text*, not the marker. rustdoc parses the misaligned form as a new paragraph
and drops it out of the list.

## DOC-2: Blank line after a documented field

Separate a documented field from the undocumented fields below it, so the
comment's scope is unambiguous.

```rust
// DON'T: does the comment cover line and column too?
pub struct Entry {
    /// Interned; unique within a single parser.
    pub name: Symbol,
    pub line: u32,
    pub column: u32,
}
```

## DOC-3: `//!` headers on crates and non-obvious modules

Every crate root, and every module whose job is not self-evident, opens with a
`//!` block at the very top of the file, above the imports. Cover:

- **Purpose**: what this unit is for.
- **Public API**: the handful of entry points a caller actually starts from.
- **Design quirks**: invariants, ordering requirements, and anything that will
  surprise the next reader.

```rust
//! Lexing for the manifest grammar.
//!
//! [`tokenize`] is the entry point; everything else here supports it.
//!
//! # Design quirks
//!
//! - Trivia (whitespace, comments) is stripped during lexing, not parsing, so
//!   the token stream cannot reconstruct the original source byte-for-byte.
//! - The lexer is allocation-free: every [`Token`] borrows from the input.

use logos::Logos;

use crate::ast::Node;
```

Link to items with rustdoc's `[`Item`]` syntax so the header stays navigable.

## DOC-4: Do not restate the name

If the name already says it, say nothing. If the name does not say it, fix the
name.

```rust
// DO: the name carries it
pub struct PersonBuilder { /* ... */ }

// DON'T: pure restatement
/// PersonBuilder is a struct that follows the builder pattern and builds a
/// person. Did I mention it's called PersonBuilder?
pub struct PersonBuilder { /* ... */ }

// DON'T: a comment patching a bad name; rename instead
/// Pbldr is a person builder.
pub struct Pbldr { /* ... */ }
```

The useful doc comment for a well-named item covers what the name *cannot*:
invariants, failure modes, cost, ordering.

```rust
// DO: adds what the name cannot
/// Builds a [`Person`].
///
/// # Panics
///
/// Panics if [`Self::build`] is called before [`Self::name`].
pub struct PersonBuilder { /* ... */ }
```

## DOC-5: Conventional headings keep their conventional meanings

rustdoc has established headings that readers and lints rely on:

| Heading | Means |
| --- | --- |
| `# Safety` | The contract a caller must uphold to call an `unsafe` fn soundly |
| `# Panics` | Conditions under which this panics |
| `# Errors` | What the `Err` variants mean |
| `# Examples` | Compiled, tested doctests |

Do not use these for anything else: `# Safety` on a safe function tells the
reader (and `clippy::missing_safety_doc`) something untrue. For anything else,
write your own heading:

```rust
// DO
/// # Interning caveats

// DON'T: nothing unsafe here
/// # Safety
/// Caution: may be fascinating!
```

## DOC-6: Never write an em dash

Not in doc comments, not in regular comments, not in commit messages or any
other prose this skill governs. Use a period, comma, colon, semicolon, or
parentheses instead: whichever one the sentence actually calls for.

```rust
// DO
/// Entries are interned: two entries with the same name share storage.

// DON'T
/// Entries are interned — two entries with the same name share storage.
```
