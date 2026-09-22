# Worked examples

Elaboration on `SKILL.md`. Only rules whose shape needs more than a table row
appear here; the rest are complete as written. SC-1 carries its own DO/DON'T in
`SKILL.md` and is elaborated rather than repeated.

## FMT-1: separate block-ending statements

```rust
// DO
for item in items {
    process(item);
}

let result = {
    prepare();
    finish()
};

report(result);

// DO: these constructs are joined
if ready {
    start();
} else {
    wait();
}

match state {
    State::Ready => start(),
    State::Waiting => wait(),
}

// DON'T: independent sibling statements touch
for item in items {
    process(item);
}
report(items);
```

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

### When to leave the binding alone

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

A long derivation also stays bound when its *name* is the only thing that says
what the chain computes. Several adapters inlined into a call argument make the
reader run the chain in their head to learn the result:

```rust
// DO: one read, and the name is the whole explanation
let stale_sessions = sessions
    .iter()
    .filter(|session| session.last_seen < cutoff)
    .filter(|session| session.pending_writes.is_empty())
    .count();

report.record(stale_sessions);

// DON'T: four lines of mechanism in an argument, with no statement of intent
report.record(
    sessions
        .iter()
        .filter(|session| session.last_seen < cutoff)
        .filter(|session| session.pending_writes.is_empty())
        .count(),
);
```

This does not reopen SC-1 for short chains. `let count = names.len();` read once
is still the rule's central case: the name restates the call and buys nothing.

These are the cases that come up most, not a closed list. The test is always the
same: does the name buy the reader more than the extra line of state costs them?
Where it does, keep it.

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
two names for the same concept, and the second binding would silently shadow the
first:

```rust
// DO
// Case: object
{
    let value = parse("{}");
    assert_eq!(value, Ok(Value::Object(Vec::new())));
}

// DON'T: no braces, so the next `value` silently shadows this one
// Case: object
let value = parse("{}");
assert_eq!(value, Ok(Value::Object(Vec::new())));
```

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

## MS-6: componentize, do not re-file the `impl`

More `impl` blocks on the same type, in one file or several, still hand every
method `&mut self` on the whole struct. Extract the cluster instead; the method
prefixes it sheds (`comment_queue` to `queue`) confirm the seam.

A shared prefix is a hint, never the trigger. Methods named for the type's main
job (`parse_header`, `parse_body`, `parse_footer` on a `Parser`) are that job,
not a side feature, and extracting them produces a component that has to borrow
its way back into the original. If the extracted type would need the parent's
data passed in, there was no seam.

```rust
// DO: comment state is reachable only from the methods that own it
struct CommentCtx {
    pending: Vec<Comment>,
    last_flushed_line: u32,
}

struct LanguageFormatter {
    comments: CommentCtx,
    indent: u32,
}

// DON'T: a second impl block or comment_* namespace 
impl LanguageFormatter {
    fn comment_queue(&mut self, comment: Comment) { /* ... */ }
}
```

Struct or `mod` is MS-4 and MS-5, on the cluster's state: `CommentCtx` holds
state across calls, whereas an `escape` cluster that only transforms arguments
is a `mod` of free functions. Files follow the components, so
`src/formatter/comments.rs` holds `CommentCtx` and its `impl` together.

## MS-7: most public first in every module

Apply this ordering independently to the crate root and every file-backed,
inline, or nested module:

1. Inner module documentation (`//!`)
2. External module declarations (`mod foo;`), regardless of visibility
3. Imports and re-exports
4. Macro definitions
5. Constants and statics, most public first
6. Everything else, most public first: `pub`, `pub(crate)`, `pub(super)`,
   `pub(in ...)`, private

An external module declaration has no body in the current file, such as
`mod client;` or `pub(crate) mod protocol;`. An inline module such as
`mod client { ... }` remains in its visibility tier.

Types and traits go anywhere their visibility tier allows, so group them however
explains the code best. Inherent `impl` blocks take the same tiers internally
and sit at the tier of their most-visible method. Keep outer documentation and
attributes attached to the item they describe.

One exception outranks the order: `macro_rules!` is textually scoped, so a macro
stays above any `mod` that uses it. Moving a `mod` above it stops compiling.

Otherwise visibility outranks declaration and composition order, and a visible
item stays above the less-visible helper it calls.

```rust
mod client {
    //! Connects to the service.

    // `protocol` expands `invalid!`, so the macro has to come first.
    macro_rules! invalid {
        () => { Error::Invalid };
    }

    mod protocol;
    pub(crate) mod transport;

    use crate::Error;

    pub const MAX_RETRIES: usize = 3;
    const BACKOFF_MS: u64 = 50;

    pub struct Client;

    impl Client {
        pub fn connect(&self) -> Result<(), Error> {
            validate_endpoint()
        }

        pub(crate) fn reset(&mut self) { /* ... */ }

        fn retry(&self) { /* ... */ }
    }

    pub(crate) fn default_client() -> Client { /* ... */ }

    pub(super) fn shared_client() -> Client { /* ... */ }

    fn validate_endpoint() -> Result<(), Error> { /* ... */ }

    mod tests {
        // Inline and private, so it remains with private items.
    }
}
```

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

`FromStr` and `Iterator` are the two that will not always fit. Neither can yield
a value that borrows from its input, so a zero-copy `fn parse(src: &str) ->
Result<Token<'_>, E>` stays a plain function. Forcing the trait there buys an
allocation you did not need.
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
// DO: expression position, where the negation would otherwise jump to the front
use std::ops::Not;

is_epic.not().then(|| render());
let stale = matches!(state, State::Done).not();

// DON'T: same expressions, negation stranded at the far left
(!is_epic).then(|| render());
let stale = !matches!(state, State::Done);
```

Condition position is the other way round. The `if` already frames the negation,
so prefix `!` always wins there, `matches!` included:

```rust
// DO
if !ready { wait(); }
if !matches!(state, State::Done) { poll(); }

// DON'T: the negation lands at the far end of the line
if ready.not() { wait(); }
```

## ID-14: no large iterator-adaptor closures

A closure passed to `.map()`, `.filter()`, `.filter_map()` or another adaptor
should read as one expression. Once the body needs statements, its own control
flow, or an early exit, the chain has stopped being a chain: write the `for`
loop. A body you have to scroll or indent past is already too big.

```rust
// DO: normalization has another genuine caller.
fn normalize_name(name: &str) -> String {
    name.trim().to_lowercase()
}

let names = raw_names
    .iter()
    .copied()
    .map(normalize_name)
    .collect::<Vec<_>>();
let owner = normalize_name(raw_owner);

// DO: single-use logic stays local without filling an adaptor closure.
let mut records = Vec::new();
for row in rows {
    let fields = row.split(',').collect::<Vec<_>>();
    if fields.len() != EXPECTED_FIELDS {
        return Err(Error::WrongFieldCount);
    }

    records.push(Record {
        id: fields[0].parse()?,
        name: fields[1].to_owned(),
    });
}

// DON'T: a single-use miniature function body hidden inside the chain.
let records = rows
    .iter()
    .map(|row| {
        let fields = row.split(',').collect::<Vec<_>>();
        if fields.len() != EXPECTED_FIELDS {
            return Err(Error::WrongFieldCount);
        }

        let id = fields[0].parse()?;
        let name = fields[1].to_owned();
        Ok(Record { id, name })
    })
    .collect::<Result<Vec<_>, Error>>()?;
```

Do not extract a single-use helper merely to keep the iterator chain: this rule
is more specific than SC-3. ID-12 still governs the iterable when the replacement
loop itself contains non-trivial filtering, mapping, or closure logic.

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

## TS-3: when the labels go on

Count the phases separately. A one-line Act against a ten-line Assert still gets
all three labels. An Arrange living outside the body, such as a module-level
`const` or a fixture `fn`, still counts as a phase: label the line that pulls it
in, or where there is none, label the Act and Assert alone.

```rust
// DO: the Assert alone is over one line, so all three phases are labeled
#[test]
fn settings_document_parses_to_its_entries() {
    // Arrange
    let source = SETTINGS;

    // Act
    let value = parse(source).expect("SETTINGS is a valid document");

    // Assert
    assert_eq!(value["retries"], Value::Number(3.0));
    assert_eq!(value["region"], Value::String("us-east-2b".into()));
}

// DON'T: every phase is one line, so the labels are three lines of nothing
#[test]
fn empty_document_parses_to_an_empty_object() {
    // Arrange
    let source = "{}";

    // Act
    let value = parse(source);

    // Assert
    assert_eq!(value, Ok(Value::Object(Vec::new())));
}
```

## TS-4: bind what repeats, inline what doesn't

The reason to bind is **drift**: if a call and its assertion each spell out the
same literal, an edit to one silently passes.

```rust
// DO: "keyleth" is spelled once, so the call and the assertion cannot drift
const MEMBER: &str = "keyleth";

assert_eq!(roster.lookup(MEMBER), Some(MEMBER));

// DON'T: two literals to keep in sync, and an edit to one still passes
assert_eq!(roster.lookup("keyleth"), Some("keyleth"));

// DON'T: a name and a binding, to say "scanlan" one time
const UNKNOWN_MEMBER: &str = "scanlan";
let found = roster.lookup(UNKNOWN_MEMBER);

assert!(found.is_none());
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

The `// Arrange` label marks the hoisted fixture, which is the whole reason the
cases live together. Inside the blocks every phase is one line, so TS-3 asks for
no labels and `// Case:` is all they take. The case-local `const` is there
because that case uses the value twice (TS-4).

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
    use super::*;

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

// DON'T: a placeholder says nothing about the shape of a real site
const SITE: &str = "foo";
```

Name the binding for the property the test depends on, not for the value it
happens to hold:

```rust
// DO: the name says why this value is in the test
const PADDED_USERNAME: &str = "  hgrant";

// DON'T: the name restates the value and hides what is being exercised
const HGRANT: &str = "  hgrant";
```

## TS-9: assert on the type, not the rendered string

A test that rebuilds the message with the same `format!` the code uses passes no
matter what either one says. A test that hardcodes the rendered message breaks
on every wording change. Both put the wording in the assertion's way.

```rust
// DON'T: the error is a string, so every test has to spell one out
fn load(path: &Path) -> Result<Config, String> {
    Err(format!("`{}` is not a .toml file", path.display()))
}

#[test]
fn load_non_toml_path_is_rejected() {
    let error = load(Path::new("config.json")).unwrap_err();

    assert_eq!(error, format!("`{}` is not a .toml file", "config.json"));
}
```

An enum carries the facts and `Display` carries the wording (ID-3). The
assertion then names the variant and its fields, so rewording touches no test:

```rust
// DO
use std::fmt::{self, Display, Formatter};
use std::path::PathBuf;

#[derive(Debug, PartialEq)]
pub enum Error {
    WrongExtension { found: String },
    Missing { path: PathBuf },
}

impl Display for Error {
    fn fmt(&self, f: &mut Formatter<'_>) -> fmt::Result {
        match self {
            Self::WrongExtension { found } => write!(f, "`{found}` is not a .toml file"),
            Self::Missing { path } => write!(f, "cannot read `{}`", path.display()),
        }
    }
}

#[test]
fn load_non_toml_path_is_rejected() {
    const PATH: &str = "config.json";

    let error = load(Path::new(PATH)).unwrap_err();

    assert_eq!(error, Error::WrongExtension { found: PATH.into() });
}
```

The rendering gets exactly one test of its own, against a literal rather than a
rebuilt `format!`:

```rust
#[test]
fn wrong_extension_displays_the_offending_name() {
    let error = Error::WrongExtension { found: "config.json".into() };

    assert_eq!(error.to_string(), "`config.json` is not a .toml file");
}
```

### When the error is not `PartialEq`

Any variant wrapping `io::Error`, `serde_json::Error` or `Box<dyn Error>` cannot
derive `PartialEq`, and `assert_eq!` is off the table. Do not flatten the source
error into a `String` to get comparison back. Match on the shape instead (ID-6):

```rust
// DO
#[derive(Debug)]
pub enum Error {
    WrongExtension { found: String },
    Unreadable(io::Error),
}

#[test]
fn load_unreadable_path_reports_the_source_error() {
    let error = load(Path::new("/nonexistent/config.toml")).unwrap_err();

    assert!(
        matches!(error, Error::Unreadable(_)),
        "a missing file surfaces as Unreadable, not WrongExtension",
    );
}

// DON'T: a String payload just to make the assertion compile
pub enum Error {
    Unreadable(String),
}
```

The same holds for any value a test reaches for: prefer the enum variant, the
struct, or a `const` the code already exports over a string the test assembles.

## DOC-2: what goes in a `//!` header

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

## DOC-3: never an em dash

```rust
// DO
/// Entries are interned: two entries with the same name share storage.

// DON'T
/// Entries are interned — two entries with the same name share storage.
```

## DOC-4: lists look like lists

Always use bullets when documentation or an explanatory comment enumerates
distinct responsibilities, behaviors, or conditions. Do not hide the list in
commas or a run-on sentence. Commas remain fine in ordinary prose.

```rust
// DO
// Responsible for:
// - Parsing each record.
// - Validating its fields.
// - Writing it to storage.
//
// Under these conditions:
// - The input has passed schema validation.
// - The destination is writable.

// DON'T
// Responsible for parsing each record, validating its fields, and writing it to
// storage, as long as the input is valid and the destination is writable.
```

Bullet continuations indent to sit under the bullet's *text*, not the marker.
The misaligned form is parsed as a new paragraph and drops out of the list:

```rust
// DO
/// - The intern table is per-`Parser`, so entries from two parsers must be
///   compared by name.

// DON'T: the continuation leaves the list
/// - The intern table is per-`Parser`, so entries from two parsers must be
/// compared by name.
```
