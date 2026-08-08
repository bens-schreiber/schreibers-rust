# Worked examples

Elaboration on `SKILL.md`. Only rules whose shape needs more than a table row
appear here; the rest are complete as written. SC-1, SC-2, TS-3 and TS-6 carry
their DO/DON'T in `SKILL.md` itself and are not repeated.

## SC-1: the block-expression fix

The chain is first choice (see `SKILL.md`). Reach for a block when the
computation needs statements. A block cannot return a borrow of something bound
inside it, which is the other reason to try a chain first.

```rust
// DO
let total = {
    let base = order.subtotal();
    base + base * TAX_RATE
};

// DON'T: base stays in scope for the whole function despite being dead
let base = order.subtotal();
let total = base + base * TAX_RATE;
```

### The two exceptions

A short prelude to the function's tail expression stays flat. Wrapping the whole
body in a block re-indents everything and buys nothing:

```rust
// DO
fn render(cfg: &Config) -> String {
    let header = cfg.title.to_uppercase();
    let body = cfg.rows.join("\n");

    format!("{header}\n{body}")
}
```

A binding also stays when its *name* is what keeps a nearby `.expect()` readable.
Chaining it away buries the reason inside a nested call, which costs more (ID-2)
than the top-level name does:

```rust
// DO
let code = u32::from_str_radix(&digits, 16).expect("four hex digits fit in a u32");

char::from_u32(code).ok_or(Error::InvalidUnicodeEscape { offset })
```

## SC-2: what the block buys you

```rust
// Source and report the daily totals.
{
    let orders = db.orders_for(today)?;
    let total: Money = orders.iter().map(Order::total).sum();
    println!("{today}: {total}");
}

// Source and report the daily refunds.
{
    let refunds = db.refunds_for(today)?;
    let total: Money = refunds.iter().map(Refund::amount).sum();
    println!("{today} refunded: {total}");
}
```

Both blocks bind `total` and neither leaks. Without the braces you would need
two names for the same concept.

## SC-3: nest a helper at its only call site

A nested `fn` is unreachable from `mod tests`, which is why a helper needing its
own test stays at module scope.

```rust
// DO: parse_row exists for load_manifest and nowhere else
fn load_manifest(src: &str) -> Result<Vec<Row>, ParseError> {
    fn parse_row(line: &str) -> Result<Row, ParseError> { /* ... */ }

    src.lines().map(parse_row).collect()
}
```

## SC-4: closure when it captures

```rust
// DO: the closure captures the fixtures it needs
let weighted = |raw: u32| raw * weights.factor + bonus;

// DON'T: weights and bonus are in scope; the parameter list is pure noise
let weighted = |raw: u32, weights: &Weights, bonus: u32| raw * weights.factor + bonus;
```

## MS-5: a struct once parameters repeat

The trigger is repetition, not parameter count: two functions sharing two
parameters is a struct; one function taking four one-off arguments is not.

```rust
// DO
struct Renderer {
    theme: Theme,
    fonts: FontSet,
}

impl Renderer {
    fn header(&self) -> String { /* ... */ }
    fn row(&self, data: &Row) -> String { /* ... */ }
}

// DON'T: theme and fonts are re-passed to everything
mod render {
    fn header(theme: &Theme, fonts: &FontSet) -> String { /* ... */ }
    fn row(theme: &Theme, fonts: &FontSet, data: &Row) -> String { /* ... */ }
}
```

## MS-6: splitting a large `impl`

Prefer several `impl` blocks in separate files over one sprawling block:
`src/renderer/mod.rs` holds the struct, `src/renderer/layout.rs` an `impl` for
measuring and placing, `src/renderer/paint.rs` an `impl` for emitting output.

## ID-2: the reason goes in the panic message

```rust
// DO
let n = "42".parse::<u32>().expect("parsed from a literal above, cannot fail");

// DO: the ordinary path
let Some(cfg) = load_config() else {
    return Err(Error::MissingConfig);
};

// DON'T: the reason is not in the backtrace
// PANIC: parsed from a literal above, cannot fail.
let n = "42".parse::<u32>().unwrap();
```

`// SAFETY:` is the language's own convention for `unsafe` blocks and what
clippy's `undocumented_unsafe_blocks` looks for. Do not spend it elsewhere.

## ID-3: which trait replaces which method

| Instead of | Implement |
| --- | --- |
| `fn default() -> Self` | `Default` |
| `fn from_x(x: X) -> Self` | `From<X> for Self` |
| `fn try_from_x(x: X) -> Result<Self, E>` | `TryFrom<X> for Self` |
| `fn to_string(&self) -> String` | `Display` |
| `fn parse(s: &str) -> Result<Self, E>` | `FromStr` |
| `fn as_slice(&self) -> &[T]` | `AsRef<[T]>` |
| `fn next_item(&mut self) -> Option<T>` | `Iterator` |

Implementing `Into` directly forfeits the reverse; `From` gives you both.
`Display` gives you `.to_string()` through the blanket `ToString` impl, so
implementing `ToString` by hand is always wrong.

## ID-4: exit early

```rust
// DO
fn load(path: &Path) -> Result<Config, Error> {
    let Some(ext) = path.extension() else {
        return Err(Error::NoExtension);
    };

    if ext != "toml" {
        return Err(Error::WrongExtension);
    }

    Ok(Config::parse(&fs::read_to_string(path)?)?)
}

// DON'T
fn load(path: &Path) -> Result<Config, Error> {
    if let Some(ext) = path.extension() {
        if ext == "toml" {
            match fs::read_to_string(path) {
                Ok(raw) => Config::parse(&raw),
                Err(e) => Err(e.into()),
            }
        } else {
            Err(Error::WrongExtension)
        }
    } else {
        Err(Error::NoExtension)
    }
}
```

## ID-6: pattern matching over field access

```rust
// DO
match event {
    Event::Click { x, y } => self.hit_test(x, y),
    Event::Key { code, .. } => self.dispatch(code),
    Event::Close => self.shutdown(),
}

// DON'T
if event.kind == Kind::Click {
    self.hit_test(event.x.unwrap(), event.y.unwrap())
} else if event.kind == Kind::Key {
    self.dispatch(event.code.unwrap())
} else {
    self.shutdown()
}
```

A destructuring `match` is exhaustive: add a variant and the compiler finds every
site. A chain of field comparisons silently keeps compiling.

## ID-7: import grouping

```rust
// DO
use std::collections::HashMap;
use std::ops::Not;

use logos::Logos;
use serde::Deserialize;

use crate::ast::Node;
use crate::lexer::Token;
```

`group_imports = "StdExternalCrate"` does this but is nightly-only, and stable
ignores it silently. `reorder_imports` is stable and on by default, so the
alphabetization within each block is free on either toolchain.

## ID-8: `.not()` over prefix `!`

```rust
// DO
use std::ops::Not;

is_epic.not().then(|| render());
matches!(state, State::Done).not()

// DON'T
(!is_epic).then(|| render());
!matches!(state, State::Done)
```

In condition position the `if` already frames the negation, so prefix `!` stays:
`if !ready` is right and `if ready.not()` puts the negation at the far end of the
line.

## TS-2: names

```rust
// DO
fn tokenize_empty_input_returns_no_tokens()
fn tokenize_unterminated_string_returns_lex_error()

// DON'T: no input condition, no expectation
fn test_tokenize()
fn tokenize_works()
fn it_should_handle_the_case_where_the_string_is_not_terminated()
```

## TS-4: bind what repeats, inline what doesn't

The reason to bind is **drift**: if a call and its assertion each spell out the
same literal, an edit to one silently passes.

```rust
// DO: "keyleth" is used twice, so it is bound once
const MEMBER: &str = "keyleth";

let found = roster.lookup(MEMBER);
assert_eq!(found, Some(MEMBER));

// DON'T: a name and a lookup, to say "scanlan" one time
const UNKNOWN_MEMBER: &str = "scanlan";
assert!(roster.lookup(UNKNOWN_MEMBER).is_none());
```

Same test for derived values: compute an expectation in Arrange only when it is
reused or when the derivation itself is the point. Use `const` where the type
allows it, `let` otherwise.

## TS-5: put the explanation in the message

```rust
// DO
assert!(
    tokens.is_empty(),
    "a comment-only source yields no tokens: trivia is stripped before parsing",
);

// DON'T: a comment is invisible in CI output
// A comment-only source yields no tokens because trivia is stripped.
assert!(tokens.is_empty());
```

`assert_eq!` already prints both sides, so it needs a message only when *why*
they should be equal is the non-obvious part.

## TS-6: a full coalesced test

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
}
```

`// Case:` is the only label these blocks need; each phase here is one line, so
TS-3's labels would be noise. The case-local `const` is there because that case
uses the value twice (TS-4).

The cost of coalescing is that the first failing case aborts the rest. Coalesce
when the cases are variations on one behavior; keep them separate when each is a
distinct behavior you want reported independently.

## TS-7: split a prefix group into a `mod`

The prefix is the symptom; the `mod` is the fix, and it is what lets TS-2 drop
the prefix. Count the group as it will stand when you finish. A file whose tests
are being written in one pass still gets counted, because the group is complete
by the end of that pass.

```rust
// DO: the mod supplies the namespace, so the names carry only input and outcome
#[cfg(test)]
mod parse_tests {
    #[test]
    fn plain_rows_split_on_commas_and_newlines() { /* ... */ }

    #[test]
    fn quoted_field_unescapes_its_contents() { /* ... */ }

    #[test]
    fn unclosed_quote_reports_the_line_it_opened_on() { /* ... */ }

    #[test]
    fn ragged_row_reports_the_first_rows_field_count() { /* ... */ }
}

#[cfg(test)]
mod lex_tests {
    // ...
}

// DON'T: 3+ tests carrying a prefix that a mod should have supplied
#[test]
fn parse_plain_rows_split_on_commas_and_newlines() { /* ... */ }

#[test]
fn parse_quoted_field_unescapes_its_contents() { /* ... */ }
```

## TS-8: plausible values for real domain things

A value that models a real thing in the domain reads as real data to whoever
comes next, even when the value is invented for the test. Sites, hostnames,
regions, tenants, SKUs and paths all get something the system could plausibly
have seen.

```rust
// DO: reads like a site this system could actually have
const SITE: &str = "us-east-2b";

// DON'T: nobody will know whether "vox" is a real site name
const SITE: &str = "vox";
```

Naming still beats theming: if a value carries meaning the test depends on, name
the binding for that meaning. `const PADDED_NAME: &str = " grog";`, not
`const VAX: &str = " grog";`.

## DOC-1: structure, and the bullet-continuation trap

```rust
/// A single parsed manifest entry.
///
/// Entries are interned, so two entries with the same name share storage and
/// comparison is pointer-equality.
///
/// # Interning caveats
///
/// - The intern table is per-`Parser`, so entries from two parsers must be
///   compared by name.
///   Comparing them by pointer silently returns `false`.
pub struct Entry {
    /// Interned; unique within a single parser.
    pub name: Symbol,

    pub line: u32,
    pub column: u32,
}
```

The first line is the rustdoc summary in index and search results: keep it to one
sentence that stands on its own. Bullet continuations indent to sit under the
bullet's *text*, not the marker; the misaligned form is parsed as a new paragraph
and drops out of the list.

The blank line before `line` is DOC-2. Without it, the reader cannot tell whether
the comment on `name` covers the two fields below it.

## DOC-3: what goes in a `//!` header

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
```

Purpose, the handful of entry points a caller starts from, then the invariants
and ordering requirements that will surprise the next reader. Link items with
`[`Item`]` so the header stays navigable.

## DOC-4: what a good doc comment adds

```rust
// DON'T: pure restatement
/// PersonBuilder is a struct that follows the builder pattern and builds a
/// person.
pub struct PersonBuilder { /* ... */ }

// DON'T: a comment patching a bad name; rename instead
/// Pbldr is a person builder.
pub struct Pbldr { /* ... */ }

// DO: adds what the name cannot
/// Builds a [`Person`].
///
/// # Panics
///
/// Panics if [`Self::build`] is called before [`Self::name`].
pub struct PersonBuilder { /* ... */ }
```

## DOC-5: the conventional headings

| Heading | Means |
| --- | --- |
| `# Safety` | The contract a caller must uphold to call an `unsafe` fn soundly |
| `# Panics` | Conditions under which this panics |
| `# Errors` | What the `Err` variants mean |
| `# Examples` | Compiled, tested doctests |

`# Safety` on a safe function tells the reader (and `clippy::missing_safety_doc`)
something untrue. For anything else, write your own heading.

## DOC-6: never an em dash

```rust
// DO
/// Entries are interned: two entries with the same name share storage.

// DON'T
/// Entries are interned — two entries with the same name share storage.
```
