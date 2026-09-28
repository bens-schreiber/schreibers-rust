# schreibers-rust - `v0.1.0`

> I spend too much time telling agents how to fix nits in Rust code. This skill encodes my style guide so they can do it themselves. 
>
> A work in progress that will be consistently updated as agents continue to find new ways to annoy me.

A skill encoding Ben Schreiber's Rust style: scoping and block
structure, module vs. struct choices, visibility, idiomatic trait usage, import
grouping, test structure and naming, and doc comments.

It sits on top of `rustfmt` and `clippy` defaults.

`SKILL.md` and `references/rules.md` are written for agents to read. What follows
is the same set of rules, written for people. Every rule gets a DO and a DON'T,
so you can skim the code and skip the prose.

## Contents

**Formatting**

- [FMT-1](#fmt-1-blank-line-after-a-block-ending-statement) blank line after a block-ending statement

**Scoping**

- [SC-1](#sc-1-bind-at-the-read-count) bind at the read count
- [SC-2](#sc-2-fence-unrelated-units-of-work-in) fence unrelated units of work in `{}`
- [SC-3](#sc-3-nest-a-helper-fn-in-its-only-caller) nest a helper `fn` in its only caller
- [SC-4](#sc-4-closure-when-it-captures-nested-fn-when-it-doesnt) closure when it captures, nested `fn` when it doesn't
- [SC-5](#sc-5-closures-go-at-the-top-of-the-function-body) closures go at the top of the function body
- [SC-6](#sc-6-no-divider-comments) no divider comments

**Modules and structs**

- [MS-1](#ms-1-a-mod-not-a-shared-prefix-on-free-functions) a `mod`, not a shared prefix on free functions
- [MS-2](#ms-2-narrowest-visibility-that-compiles) narrowest visibility that compiles
- [MS-3](#ms-3-mark-visibility-widened-only-for-tests) mark visibility widened only for tests
- [MS-4](#ms-4-a-mod-of-free-functions-not-a-zero-field-struct) a `mod` of free functions, not a zero-field struct
- [MS-5](#ms-5-a-struct-once-non-trivial-parameters-repeat) a struct once non-trivial parameters repeat
- [MS-6](#ms-6-three-methods-serving-one-side-feature-is-a-component) three methods serving one side feature is a component
- [MS-7](#ms-7-most-public-first-in-every-module) most public first in every module

**Idioms**

- [ID-1](#id-1-an-enum-not-a-bool-parameter-or-field) an enum, not a `bool` parameter or field
- [ID-2](#id-2-handle-the-error-path-outside-tests) handle the error path outside tests
- [ID-3](#id-3-implement-the-standard-traits) implement the standard traits
- [ID-4](#id-4-exit-early-instead-of-nesting-the-happy-path) exit early instead of nesting the happy path
- [ID-5](#id-5-name-a-local-after-the-field-it-fills) name a local after the field it fills
- [ID-6](#id-6-pattern-matching-over-field-access-plus-conditionals) pattern matching over field access plus conditionals
- [ID-7](#id-7-three-blank-line-separated-use-blocks) three blank-line-separated `use` blocks
- [ID-8](#id-8-matchesnot-over-matches) `matches!(..).not()` over `!matches!(..)`
- [ID-9](#id-9-alphabetize-every-list-in-cargotoml) alphabetize every list in `Cargo.toml`
- [ID-10](#id-10-compound-generics-off-the-lhs) compound generics off the LHS
- [ID-11](#id-11--for-any-generic-parameter-the-compiler-can-infer) `_` for any generic parameter the compiler can infer
- [ID-12](#id-12-name-complex-for-iterables) name complex `for` iterables
- [ID-13](#id-13-explain-non-obvious-control-flow-exits-at-the-exit) explain non-obvious control-flow exits at the exit
- [ID-14](#id-14-no-large-iterator-adaptor-closures) no large iterator-adaptor closures

**Tests**

- [TS-1](#ts-1-unit-tests-live-in-the-file-under-test) unit tests live in the file under test
- [TS-2](#ts-2-namespaceinputexpectation) `namespace_input_expectation`
- [TS-3](#ts-3-label-all-three-phases-when-any-one-exceeds-a-line) label all three phases when any one exceeds a line
- [TS-4](#ts-4-bind-what-repeats-inline-what-doesnt) bind what repeats, inline what doesn't
- [TS-5](#ts-5-custom-panic-message-when-the-failure-reason-isnt-obvious) custom panic message when the failure reason isn't obvious
- [TS-6](#ts-6-coalesce-tests-that-share-most-of-their-arrange) coalesce tests that share most of their Arrange
- [TS-7](#ts-7-three-tests-sharing-a-prefix-become-a-mod) three tests sharing a prefix become a `mod`
- [TS-8](#ts-8-plausible-values-for-real-domain-things) plausible values for real domain things
- [TS-9](#ts-9-assert-on-the-type-not-the-rendered-string) assert on the type, not the rendered string

**Documentation**

- [DOC-1](#doc-1-blank-line-between-a-documented-field-and-undocumented-ones) blank line between a documented field and undocumented ones
- [DOC-2](#doc-2--header-on-every-crate-and-non-obvious-module) `//!` header on every crate and non-obvious module
- [DOC-3](#doc-3-never-an-em-dash) never an em dash
- [DOC-4](#doc-4-lists-look-like-lists) lists look like lists

## Formatting

Whitespace only. There is one rule, because `rustfmt` handles the rest.

### FMT-1: blank line after a block-ending statement

Put a blank line after a `}` when the next line starts something new. Joined
`if` branches and the arms of a `match` are one statement, so they stay
together.

```rust
// DO
for item in items {
    process(item);
}

let total = {
    let base = subtotal();
    base + base * TAX_RATE
};

report(total);

// DO: joined constructs are one statement, so nothing separates them
if ready {
    start();
} else {
    wait();
}

report(total);

// DON'T: two independent statements touching
for item in items {
    process(item);
}
report(items);
```

## Scoping

Where a thing lives, and how long the reader has to remember it. Most of these
are about keeping names out of scopes that do not need them.

### SC-1: bind at the read count

Count the later scopes that read a local:

- **Two or more:** bind it at top level.
- **Exactly one:** push it into the thing that reads it.
- **Both arms of one `if` or `match`:** that is one reader. Bind it directly
  above the construct, not at the top of the function.

Fix it in this order:

1. A method chain, where one exists.
2. A block expression, where the computation needs statements.
3. An extracted `fn`, where it deserves a name.

```rust
// DO
let total = {
    let a_count = a.iter().count();
    let b_count = b.iter().count();
    a_count + b_count
};

// DON'T
let a_count = a.iter().count();
let b_count = b.iter().count();
let total = a_count + b_count;
```

Keep the binding when pushing it in would cost the reader more than it saves.
The cases that come up most:

- A short prelude to the function's tail expression.
- A binding whose name is what keeps a nearby `.expect()` readable.
- A long derivation whose name is the only thing that says what it computes. A
  chain like `x.iter().filter(..).map(..).sum()` gets a name even at one read,
  rather than sitting inline in a call. A short, obvious chain does not.

That list is not closed. The test is always whether the name buys the reader
more than the extra line of state costs them.

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

Move it to module scope only if it is:

- Unit-tested.
- Doc-tested.
- Long enough to displace the caller's body.

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

Define them first, before the work that uses them. The exception is when
hoisting would stretch a capture across code that needs the same borrow.

```rust
// DO
fn report(names: &[String], width: usize) -> String {
    let pad = |s: &str| format!("{s:width$}");

    names.iter().map(|name| pad(name)).collect()
}

// DON'T: a statement above the closure, so the reader meets it mid-body
fn report(names: &[String], width: usize) -> String {
    let heading = format!("{} names", names.len());
    let pad = |s: &str| format!("{s:width$}");

    heading + &names.iter().map(|name| pad(name)).collect::<String>()
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

When to reach for a `mod`, when to reach for a struct, and how visible to make
either one.

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

Take the first one that compiles:

1. Private
2. `pub(super)`
3. `pub(crate)`
4. `pub`

Reach for `pub` only when something outside the crate calls it.

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

Extracting is the fix, not more `impl` blocks on the same type. A shared prefix
is a hint, never the trigger: methods named for the type's main job are that
job. If the extracted type would need the parent's data handed back to it, there
was no seam, and no finding.

```rust
// DO: History owns its state outright, so it needs nothing from the Editor
struct History {
    past: Vec<String>,
}

impl History {
    fn push(&mut self, text: String) { self.past.push(text); }
    fn undo(&mut self) -> Option<String> { self.past.pop() }
    fn is_empty(&self) -> bool { self.past.is_empty() }
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

### MS-7: most public first in every module

Apply this order independently in crate roots and every file-backed, inline, or
nested module:

1. `//!` module documentation
2. External module declarations (`mod foo;`), whatever their visibility
3. Imports and re-exports
4. Macro definitions
5. Constants and statics, most public first
6. Everything else, most public first: `pub`, `pub(crate)`, `pub(super)`,
   `pub(in ...)`, private

Types and traits go wherever their visibility tier allows, so group them however
explains the code best. Visibility beats declaration order, with one exception:
a `macro_rules!` has to sit above any `mod` that uses it, or the build breaks.

```rust
// DO
mod client {
    //! Connects to the service.

    // `protocol` expands `invalid!`, so the macro has to come first.
    macro_rules! invalid {
        () => { Error::Invalid };
    }

    mod protocol;

    use crate::Error;

    pub const MAX_RETRIES: usize = 3;
    const BACKOFF_MS: u64 = 50;

    pub struct Client;

    pub(crate) fn default_client() -> Client { /* ... */ }

    fn validate_endpoint() -> Result<(), Error> { /* ... */ }
}

// DON'T: `mod` under the imports, private helper above what it serves,
// and a macro below the module that expands it
mod client {
    use crate::Error;

    mod protocol;

    macro_rules! invalid {
        () => { Error::Invalid };
    }

    fn validate_endpoint() -> Result<(), Error> { /* ... */ }

    pub(crate) fn default_client() -> Client { /* ... */ }

    pub struct Client;
}
```

## Idioms

The small stuff. Which construct to use when Rust gives you several that work.

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

Implement the trait instead of hand-rolling its contract:

- `Default` instead of `fn default() -> Self`.
- `From` instead of `fn from_x(x: X) -> Self`.
- `TryFrom` instead of `fn try_from_x(x: X) -> Result<Self, E>`.
- `Display` instead of `fn to_string(&self) -> String`.
- `FromStr` instead of `fn parse(s: &str) -> Result<Self, E>`.
- `AsRef` instead of `fn as_slice(&self) -> &[T]`.
- `Iterator` instead of `fn next_item(&mut self) -> Option<T>`.

Write `From`, never `Into`. `Into` as a bound is fine.

Two of these do not always fit. Neither `FromStr` nor `Iterator` can hand back a
value that borrows from its input, so a zero-copy parser stays a plain function
rather than paying for an allocation to satisfy the trait.

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

In this order, alphabetized within each block:

1. `std`, `core`, `alloc`
2. External crates
3. Internal (`crate`, `super`, `self`)

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

### ID-8: `matches!(..).not()` over `!matches!(..)`

```rust
// DO
use std::ops::Not;

let stale = matches!(state, State::Done).not();
if matches!(state, State::Done).not() { /* ... */ }

// DON'T
let stale = !matches!(state, State::Done);
if !matches!(state, State::Done) { /* ... */ }
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

Scalars, `const` and `static`, signatures, and expressions with nowhere to hang
a turbofish all stay annotated on the left.

```rust
// DO
let words = line.split(' ').collect::<Vec<_>>();

// DON'T: the same collect, with the type shoved to the left
let words: Vec<&str> = line.split(' ').collect();
```

### ID-11: `_` for any generic parameter the compiler can infer

```rust
// DO: the return type already pins both, so `_` just carries them through
fn parse_ids(raw: &str) -> Result<Vec<u32>, ParseIntError> {
    raw.split(',').map(str::parse).collect::<Result<Vec<_>, _>>()
}

// DON'T: the turbofish restates what the signature said one line up
fn parse_ids(raw: &str) -> Result<Vec<u32>, ParseIntError> {
    raw.split(',').map(str::parse).collect::<Result<Vec<u32>, ParseIntError>>()
}
```

### ID-12: name complex `for` iterables

If the thing you are looping over does any real filtering or mapping, give it a
name first. The loop header should say what it walks, not how the list was
built.

```rust
// DO
let eligible_users = users
    .iter()
    .filter(|user| user.is_active() && user.has_access());

for user in eligible_users {
    // ...
}

// DON'T
for user in users
    .iter()
    .filter(|user| user.is_active() && user.has_access())
{
    // ...
}
```

### ID-13: explain non-obvious control-flow exits at the exit

Put the reason directly above the `continue`, `break` or `return`, inside the
branch. A comment above the `if` is too far away, and one that restates the
condition says nothing.

```rust
// DO
if record.version < minimum_version {
    // Older records cannot contain the fields required below.
    continue;
}

if retries == retry_limit {
    // Further attempts would exceed the upstream request deadline.
    break;
}

if cache.is_fresh() {
    // Refreshing would discard locally validated metadata.
    return Ok(cache);
}

// DON'T: the explanation is outside the branch.
// Older records cannot contain the fields required below.
if record.version < minimum_version {
    continue;
}

// DON'T: the comment merely restates the condition.
if record.version < minimum_version {
    // Skip old records.
    continue;
}
```

### ID-14: no large iterator-adaptor closures

A closure inside `.map()`, `.filter()` or another adaptor should read as one
expression. Once it needs statements of its own, its own control flow, or an
early exit, the chain has stopped being a chain. Write the loop.

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

Do not pull the body out into a one-off helper just to keep the chain intact.
The loop is the fix. If the list you end up looping over is itself built by a
chain, give that a name too.

## Tests

Where tests live, what to call them, and how to keep them from breaking on
changes that are not bugs.

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

The subject, then the input condition, then what you expect to happen. Drop
`namespace` when the module or file already supplies it. No `test_` prefix.

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

Count the phases separately. A one-line Act against a ten-line Assert still
gets all three labels. Skip them only when all three phases are one-liners.
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
// DO: "keyleth" is spelled once, so the call and the assertion cannot drift
const MEMBER: &str = "keyleth";

assert_eq!(roster.lookup(MEMBER), Some(MEMBER));

// DON'T: two literals to keep in sync, and editing one still passes
assert_eq!(roster.lookup("keyleth"), Some("keyleth"));

// DON'T: a name and a binding, to say "scanlan" one time
const UNKNOWN_MEMBER: &str = "scanlan";
let found = roster.lookup(UNKNOWN_MEMBER);

assert!(found.is_none());
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
when you finish, not mid-write.

When both this and TS-6 apply, split first and coalesce second: make the `mod`,
then look for shared fixtures inside it.

```rust
// DO
#[cfg(test)]
mod parse_tests {
    use super::*;

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

Sites, hostnames, regions, tenants, SKUs and paths all get something the system
could plausibly have seen. Codebase convention wins.

```rust
// DO
const SITE: &str = "us-east-2b";

// DON'T
const SITE: &str = "foo";
```

Name the binding for the property the test depends on, not for the value it
holds: `const PADDED_USERNAME: &str = "  hgrant";`, not
`const HGRANT: &str = "  hgrant";`.

### TS-9: assert on the type, not the rendered string

Never assert against a string the test assembles with `format!`. Rebuilding the
message in the test asserts nothing, and hardcoding it breaks on every wording
change.

Have the code return an enum, a struct, or a well-known `const` instead, then
give it `Display` when a message is needed. If the payload cannot derive
`PartialEq`, match on the shape with `matches!` rather than flattening it to a
string.

```rust
// DO
#[derive(Debug, PartialEq)]
pub enum Error {
    WrongExtension { found: String },
}

impl Display for Error {
    fn fmt(&self, f: &mut Formatter<'_>) -> fmt::Result {
        match self {
            Self::WrongExtension { found } => write!(f, "`{found}` is not a .toml file"),
        }
    }
}

#[test]
fn load_non_toml_path_is_rejected() {
    const PATH: &str = "config.json";

    let error = load(Path::new(PATH)).unwrap_err();

    assert_eq!(error, Error::WrongExtension { found: PATH.into() });
}

// DON'T: passes whatever either format! says, and breaks on any reword
#[test]
fn load_non_toml_path_is_rejected() {
    let error = load(Path::new("config.json")).unwrap_err();

    assert_eq!(error, format!("`{}` is not a .toml file", "config.json"));
}
```

The wording itself gets one test, and only one, written against a literal:

```rust
#[test]
fn wrong_extension_displays_the_offending_name() {
    let error = Error::WrongExtension { found: "config.json".into() };

    assert_eq!(error.to_string(), "`config.json` is not a .toml file");
}
```

## Documentation

Comments and rustdoc. Mostly about shape, since what to say is your call.

### DOC-1: blank line between a documented field and undocumented ones

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

### DOC-2: `//!` header on every crate and non-obvious module

Above the imports. Say what the module is for, which entry points a caller
starts from, and which quirks will surprise the next reader.

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

// DON'T: no purpose, no entry point, no quirks
//! Config module.

use std::fs;
```

### DOC-3: never an em dash

In doc comments, comments, or commit messages. Use a period, comma, colon,
semicolon, or parentheses.

```rust
// DO
/// Rows are compared by name: two rows with the same name are equal.

// DON'T
/// Rows are compared by name — two rows with the same name are equal.
```

### DOC-4: lists look like lists

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

Indent bullet continuations under the bullet's *text*, not the marker, or
rustdoc parses them as a new paragraph and drops them from the list.

```rust
// DO
/// - The intern table is per-`Parser`, so entries from two parsers must be
///   compared by name.

// DON'T
/// - The intern table is per-`Parser`, so entries from two parsers must be
/// compared by name.
```
