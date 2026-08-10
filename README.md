# schreibers-rust - `v0.0.1`

> I spend too much time telling agents how to fix nits in Rust code. This skill encodes my style guide so they can do it themselves. 
>
> A work in progress that will be consistently updated as agents continue to find new ways to annoy me.

A Claude Code skill encoding Ben Schreiber's Rust style: scoping and block
structure, module vs. struct choices, visibility, idiomatic trait usage, import
grouping, test structure and naming, and doc comments.

It sits on top of `rustfmt` and `clippy` defaults.

# Summary of Rules

The `SKILL.md` file and `references/rules.md` contain rules intended to be read
by agents. Here is a summary of the rules for humans:

## Scoping

### SC-1: bind at the read count

Count the later scopes that read a local. Two or more, top level. Exactly one,
push it into the thing that reads it. Fix with a chain, then a block expression,
then an extracted `fn`.

```rust
// DO
let count = {
    let a_count = a.iter().count();
    let b_count = b.iter().count();
    a_count + b_count
};

// DON'T
let a_count = a.iter().count();
let b_count = b.iter().count();
let total = a_count + b_count;
```

### SC-2: fence unrelated units of work in `{}`

Several lines doing one thing, under a one-line comment naming it. The braces
are the point: neighboring units can then reuse the same binding names.

```rust
// DO
// Report the sales.
{
    let total: u32 = sales.iter().sum();
    println!("sold {total}");
}

// Report the refunds.
{
    let total: u32 = refunds.iter().sum();
    println!("refunded {total}");
}

// DON'T: no braces, so the second `total` needs a different name
let sales_total: u32 = sales.iter().sum();
println!("sold {sales_total}");

let refunds_total: u32 = refunds.iter().sum();
println!("refunded {refunds_total}");
```

### SC-3: nest a helper `fn` in its only caller

Module scope if it is unit-tested, doc-tested, or long enough to displace the
caller's body.

```rust
// DO: shout exists for banner and nowhere else
fn banner(lines: &str) -> String {
    fn shout(line: &str) -> String { line.to_uppercase() }

    lines.lines().map(shout).collect()
}

// DON'T: module scope for something with one caller and no test
fn shout(line: &str) -> String { line.to_uppercase() }

fn banner(lines: &str) -> String {
    lines.lines().map(shout).collect()
}
```

### SC-4: closure when it captures, nested `fn` when it doesn't

Never re-pass what a closure could capture.

```rust
// DO
let weighted = |raw: u32| raw * factor + bonus;

// DON'T: factor and bonus are in scope; the parameter list is noise
let weighted = |raw: u32, factor: u32, bonus: u32| raw * factor + bonus;
```

### SC-5: closures go at the top of the function body

The one exception to SC-1. Not when hoisting would extend a mutable capture
across code that needs the same borrow.

```rust
// DO
fn report(names: &[String], width: usize) -> String {
    let pad = |s: &str| format!("{s:width$}");

    names.iter().map(|name| pad(name)).collect()
}
```

### SC-6: no divider comments

Use a `{}` block inside a function, a `mod` or a separate file outside one.

```rust
// DO
mod parsing { /* ... */ }

// DON'T
// ===== PARSING =====
```

## Modules and structs

### MS-1: a `mod`, not a shared prefix on free functions

```rust
// DO
mod path {
    pub fn join(a: &str, b: &str) -> String { /* ... */ }
    pub fn split(s: &str) -> Vec<&str> { /* ... */ }
}

// DON'T
fn path_join(a: &str, b: &str) -> String { /* ... */ }
fn path_split(s: &str) -> Vec<&str> { /* ... */ }
```

### MS-2: narrowest visibility that compiles

Private, then `pub(super)`, then `pub(crate)`, then `pub`. `pub` only for
something used outside the crate.

```rust
// DO
pub(crate) fn slugify(title: &str) -> String { /* ... */ }

// DON'T: nothing outside the crate calls this
pub fn slugify(title: &str) -> String { /* ... */ }
```

### MS-3: mark visibility widened only for tests

```rust
// DO
// Visible for tests.
pub(crate) fn retry_count(&self) -> u32 { /* ... */ }

// DON'T: the next reader assumes this is real API
pub(crate) fn retry_count(&self) -> u32 { /* ... */ }
```

### MS-4: a `mod` of free functions, not a zero-field struct

```rust
// DO
mod escape {
    pub fn quote(s: &str) -> String { /* ... */ }
    pub fn unquote(s: &str) -> String { /* ... */ }
}

// DON'T: a struct that exists only to namespace an impl
struct Escape;

impl Escape {
    fn quote(s: &str) -> String { /* ... */ }
    fn unquote(s: &str) -> String { /* ... */ }
}
```

### MS-5: a struct once non-trivial parameters repeat

The trigger is repetition, not parameter count.

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

### MS-6: three methods serving one side feature is a component

Extracting is the fix, not more `impl` blocks on the same type.

```rust
// DO
struct History {
    past: Vec<String>,
}

impl History {
    fn push(&mut self, text: String) { /* ... */ }
    fn undo(&mut self) { /* ... */ }
    fn is_empty(&self) -> bool { /* ... */ }
}

struct Editor {
    text: String,
    history: History,
}

// DON'T: a history_* namespace bolted onto the editor
impl Editor {
    fn history_push(&mut self, text: String) { /* ... */ }
    fn history_undo(&mut self) { /* ... */ }
    fn history_is_empty(&self) -> bool { /* ... */ }
}
```

## Idioms

### ID-1: an enum, not a `bool` parameter or field

Two bools with an impossible combination are one enum; independent flags stay
separate.

```rust
// DO: sort(&mut xs, Order::Descending) reads at the call site
enum Order { Ascending, Descending }

fn sort(xs: &mut [u32], order: Order) { /* ... */ }

// DON'T: sort(&mut xs, true) says nothing
fn sort(xs: &mut [u32], descending: bool) { /* ... */ }
```

### ID-2: handle the error path outside tests

An unreachable panic is `.expect("<why it cannot fail>")`, never a bare
`.unwrap()` and never a `// PANIC:` comment. `// SAFETY:` is for `unsafe` only.

```rust
// DO
let n = "42".parse::<u32>().expect("parsed from a literal above, cannot fail");

// DO: the ordinary path
let Some(cfg) = load_config() else {
    return Err(Error::MissingConfig);
};

// DON'T: the reason never reaches the backtrace
// PANIC: parsed from a literal above, cannot fail.
let n = "42".parse::<u32>().unwrap();
```

### ID-3: implement the standard traits

`Default`, `From`, `TryFrom`, `Display`, `FromStr`, `AsRef`, `Iterator`. `From`,
never `Into`; `Into` as a bound is fine.

```rust
// DO
impl From<u32> for Celsius { /* ... */ }

// DON'T: hand-rolls the contract and forfeits the blanket impls
impl Celsius {
    fn from_u32(n: u32) -> Self { /* ... */ }
}
```

### ID-4: exit early instead of nesting the happy path

```rust
// DO
fn first_word(line: &str) -> Result<&str, Error> {
    let Some(word) = line.split_whitespace().next() else {
        return Err(Error::Empty);
    };

    if word.len() > 20 {
        return Err(Error::TooLong);
    }

    Ok(word)
}

// DON'T
fn first_word(line: &str) -> Result<&str, Error> {
    if let Some(word) = line.split_whitespace().next() {
        if word.len() > 20 {
            Err(Error::TooLong)
        } else {
            Ok(word)
        }
    } else {
        Err(Error::Empty)
    }
}
```

### ID-5: name a local after the field it fills

```rust
// DO
let name = input.trim().to_owned();

User { name }

// DON'T
let trimmed = input.trim().to_owned();

User { name: trimmed }
```

### ID-6: pattern matching over field access plus conditionals

`matches!` when all you need is a `bool`. A destructuring `match` is exhaustive:
add a variant and the compiler finds every site.

```rust
// DO
match shape {
    Shape::Circle { radius } => 3.14 * radius * radius,
    Shape::Square { side } => side * side,
}

// DON'T: adding a variant leaves this silently compiling
if shape.kind == Kind::Circle {
    3.14 * shape.radius * shape.radius
} else {
    shape.side * shape.side
}
```

### ID-7: three blank-line-separated `use` blocks

`std`/`core`/`alloc`, external crates, internal. Alphabetized within each.
Stable `rustfmt` will not group them, so do it by hand.

```rust
// DO
use std::collections::HashMap;
use std::fs;

use serde::Deserialize;

use crate::config::Config;

// DON'T
use crate::config::Config;
use serde::Deserialize;
use std::collections::HashMap;
use std::fs;
```

### ID-8: `.not()` when negating a chain you continue

Prefix `!` stays in `if`/`while` conditions, where the keyword already frames the
negation.

```rust
// DO
use std::ops::Not;

is_epic.not().then(|| render());
matches!(state, State::Done).not()

if !ready { /* ... */ }

// DON'T
(!is_epic).then(|| render());
!matches!(state, State::Done)

if ready.not() { /* ... */ }
```

### ID-9: alphabetize every list in `Cargo.toml`

```toml
# DO
[dependencies]
logos = "0.14"
serde = { version = "1", features = ["derive"] }
thiserror = "1"

# DON'T
[dependencies]
thiserror = "1"
logos = "0.14"
serde = { version = "1", features = ["derive"] }
```

### ID-10: compound generics off the LHS

Scalars, `const`/`static`, signatures, and expressions with no turbofish stay
LHS-annotated.

```rust
// DO
let words = line.split(' ').collect::<Vec<_>>();

// DON'T
let words: Vec<String> = line.split(' ').collect();
```

### ID-11: `_` for any generic parameter the compiler can infer

```rust
// DO
let ids = raw.split(',').map(str::parse).collect::<Result<Vec<_>, _>>()?;

// DON'T: the element type is already fixed by the signature
let ids = raw.split(',').map(str::parse).collect::<Result<Vec<u32>, ParseIntError>>()?;
```

## Tests

### TS-1: unit tests live in the file under test

`#[cfg(test)] mod tests` in the file; integration tests in the crate's `tests/`.

```rust
// DO: in src/slug.rs, next to the fn it covers
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn slugify_spaces_become_dashes() { /* ... */ }
}

// DON'T: the same test in tests/slug.rs, where private items are unreachable
```

### TS-2: `namespace_input_expectation`

Subject, input condition, expected outcome. Drop `namespace` when the module or
file supplies it. No `test_` prefix.

```rust
// DO
fn slugify_empty_input_returns_empty_string()
fn slugify_trailing_space_is_trimmed()

// DON'T
fn test_slugify()
fn slugify_works()
fn it_should_handle_the_case_where_the_input_has_a_trailing_space()
```

### TS-3: label all three phases when any one exceeds a line

Count the phases separately. Skip the labels only when all three are one-liners.
Never extend a label with narrative.

```rust
// DO: the Assert alone is over one line, so all three phases are labeled
#[test]
fn split_csv_row_yields_each_field() {
    // Arrange
    let row = "us-east-2b,3,ready";

    // Act
    let fields = split_csv(row);

    // Assert
    assert_eq!(fields[0], "us-east-2b");
    assert_eq!(fields[2], "ready");
}

// DON'T: labels on some phases, narrative on one of them
#[test]
fn split_csv_row_yields_each_field() {
    let row = "us-east-2b,3,ready";

    // Act: split it and check the fields came through
    let fields = split_csv(row);
    assert_eq!(fields[0], "us-east-2b");
    assert_eq!(fields[2], "ready");
}
```

### TS-4: bind what repeats, inline what doesn't

The reason to bind is drift: if a call and its assertion each spell out the same
literal, an edit to one silently passes.

```rust
// DO: "keyleth" is used twice, so it is bound once
const MEMBER: &str = "keyleth";

let found = roster.lookup(MEMBER);
assert_eq!(found, Some(MEMBER));

// DON'T: a name and a lookup, to say "scanlan" one time
const UNKNOWN_MEMBER: &str = "scanlan";
assert!(roster.lookup(UNKNOWN_MEMBER).is_none());
```

### TS-5: custom panic message when the failure reason isn't obvious

It goes in the message, not a comment above it.

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

### TS-6: coalesce tests that share most of their Arrange

One function, one case per `{}` block. Hoist only the shared Arrange; each case
keeps its own Act and Assert. No shared Arrange means no TS-6.

```rust
// DO
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

// DON'T: nothing hoisted, and each label restates the line under it
// Case: null
assert_eq!(parse("null"), Ok(Value::Null));

// Case: true
assert_eq!(parse("true"), Ok(Value::Bool(true)));
```

The cost is that the first failing case aborts the rest, so coalesce variations
on one behavior and keep distinct behaviors separate.

### TS-7: three tests sharing a prefix become a `mod`

The prefix then comes off the names (TS-2). Count the group as it will stand
when you finish, not mid-write. TS-6 merges by fixture, TS-7 splits by subject.

```rust
// DO
#[cfg(test)]
mod parse_tests {
    #[test]
    fn plain_rows_split_on_commas_and_newlines() { /* ... */ }

    #[test]
    fn quoted_field_unescapes_its_contents() { /* ... */ }

    #[test]
    fn unclosed_quote_reports_the_line_it_opened_on() { /* ... */ }
}

// DON'T
#[test]
fn parse_plain_rows_split_on_commas_and_newlines() { /* ... */ }

#[test]
fn parse_quoted_field_unescapes_its_contents() { /* ... */ }

#[test]
fn parse_unclosed_quote_reports_the_line_it_opened_on() { /* ... */ }
```

### TS-8: plausible values for real domain things

Sites, hostnames, regions, tenants, SKUs and paths get something the system
could plausibly have seen. Codebase convention wins.

```rust
// DO
const SITE: &str = "us-east-2b";

// DON'T
const SITE: &str = "foo";
```

Naming still beats theming: `const PADDED_NAME: &str = " grog";`, not
`const VAX: &str = " grog";`.

## Documentation

### DOC-1: markdown and line breaks in doc comments

One-line summary, blank `///`, then paragraphs, `#` headings, bullets. Indent
bullet continuations under the bullet's *text* or rustdoc drops them.

```rust
// DO
/// A single parsed manifest entry.
///
/// # Interning caveats
///
/// - The intern table is per-`Parser`, so entries from two parsers must be
///   compared by name.

// DON'T: the continuation is parsed as a new paragraph and leaves the list
/// A single parsed manifest entry.
/// - The intern table is per-`Parser`, so entries from two parsers must be
/// compared by name.
```

### DOC-2: blank line between a documented field and undocumented ones

```rust
// DO
pub struct Entry {
    /// Interned; unique within a single parser.
    pub name: Symbol,

    pub line: u32,
    pub column: u32,
}

// DON'T: does the comment cover the fields below it?
pub struct Entry {
    /// Interned; unique within a single parser.
    pub name: Symbol,
    pub line: u32,
}
```

### DOC-3: `//!` header on every crate and non-obvious module

Above the imports: purpose, public API, design quirks.

```rust
// DO
//! Reading and validating the config file.
//!
//! [`load`] is the entry point; everything else here supports it.
//!
//! # Design quirks
//!
//! - Comments are stripped while reading, so a loaded config cannot be
//!   written back out byte-for-byte.

use std::fs;
```

### DOC-4: do not document what the name says

If a name needs a comment to be understood, rename it.

```rust
// DO
/// Builds a [`Person`].
///
/// # Panics
///
/// Panics if [`Self::build`] is called before [`Self::name`].
pub struct PersonBuilder { /* ... */ }

// DON'T: pure restatement
/// PersonBuilder is a struct that follows the builder pattern and builds a
/// person.
pub struct PersonBuilder { /* ... */ }

// DON'T: a comment patching a bad name; rename instead
/// Pbldr is a person builder.
pub struct Pbldr { /* ... */ }
```

### DOC-5: conventional headings for conventional meanings only

`# Safety` (the contract for calling an `unsafe` fn soundly), `# Panics`,
`# Errors`, `# Examples`. Anything else gets your own heading.

```rust
// DO
/// # Errors
///
/// Returns [`Error::WrongExtension`] when the path is not a `.toml` file.
pub fn load(path: &Path) -> Result<Config, Error> { /* ... */ }

// DON'T: says something untrue to the reader and to clippy
/// # Safety
///
/// The caller should pass a valid path.
pub fn load(path: &Path) -> Result<Config, Error> { /* ... */ }
```

### DOC-6: never an em dash

In doc comments, comments, or commit messages. Use a period, comma, colon,
semicolon, or parentheses.

```rust
// DO
/// Rows are compared by name: two rows with the same name are equal.

// DON'T
/// Rows are compared by name — two rows with the same name are equal.
```

