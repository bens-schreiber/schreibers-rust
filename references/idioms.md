# Idioms

Rules `ID-1` … `ID-12`. The through-line: **make the reader's job easy at the
call site, not just at the definition.**

## ID-1: Enums, not boolean flags

A `bool` parameter forces the reader to remember what `true` meant. A variant
names it at every call site.

```rust
// DO
enum Casing {
    Preserve,
    Lower,
}

fn normalize(src: &str, casing: Casing) -> String { /* ... */ }

normalize(input, Casing::Lower);

// DON'T: what is `true` here?
fn normalize(src: &str, lowercase: bool) -> String { /* ... */ }

normalize(input, true);
```

This extends to struct fields and return types. Two related bools are almost
always one enum with three or four variants, and the enum makes the impossible
combination unrepresentable.

## ID-2: No `.unwrap()` / `.expect()` outside tests

Handle the error path: propagate with `?`, or destructure with `let ... else`
(ID-4). Test code is exempt: a panic there is a failed assertion.

When a panic is genuinely unreachable, justify it inline with a `// PANIC:`
comment stating *why* it cannot fire.

```rust
// DO
// PANIC: parsed from a literal above, cannot fail.
let n = "42".parse::<u32>().unwrap();

// DO: the ordinary path
let Some(cfg) = load_config() else {
    return Err(Error::MissingConfig);
};
```

Reserve `// SAFETY:` for `unsafe` blocks, where it is the language's own
convention and what clippy's `undocumented_unsafe_blocks` looks for. Using it on
a safe `unwrap` overloads a term that already means something specific.

## ID-3: Implement the standard traits

Do not hand-roll a method that duplicates a trait's contract. Callers, generic
code, and `?` all key off the traits.

| Instead of | Implement |
| --- | --- |
| `fn default() -> Self` | `Default` |
| `fn from_x(x: X) -> Self` | `From<X> for Self` |
| `fn try_from_x(x: X) -> Result<Self, E>` | `TryFrom<X> for Self` |
| `fn to_string(&self) -> String` | `Display` |
| `fn parse(s: &str) -> Result<Self, E>` | `FromStr` |
| `fn as_slice(&self) -> &[T]` | `AsRef<[T]>` |
| `fn next_item(&mut self) -> Option<T>` | `Iterator` |

```rust
// DO
impl Default for Config {
    fn default() -> Self { /* ... */ }
}

impl From<RawConfig> for Config {
    fn from(raw: RawConfig) -> Self { /* ... */ }
}

// DON'T
impl Config {
    fn default() -> Self { /* ... */ }
    fn from_raw(raw: RawConfig) -> Self { /* ... */ }
}
```

Implement `From`, never `Into`: the blanket impl gives you `Into` for free, and
implementing `Into` directly forfeits the reverse.

`Display` gives you `.to_string()` through the blanket `ToString` impl, so
implementing `ToString` by hand is always wrong.

## ID-4: Exit early

Flatten the happy path with `let ... else`, `if let`, and `?`. The success case
should stay at the function's base indentation from top to bottom.

```rust
// DO
fn load(path: &Path) -> Result<Config, Error> {
    let Some(ext) = path.extension() else {
        return Err(Error::NoExtension);
    };

    if ext != "toml" {
        return Err(Error::WrongExtension);
    }

    let raw = fs::read_to_string(path)?;

    Ok(Config::parse(&raw)?)
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

## ID-5: `matches!` for a bool

```rust
// DO
let is_terminal = matches!(state, State::Done | State::Failed);

// DON'T
let is_terminal = match state {
    State::Done | State::Failed => true,
    _ => false,
};
```

Once an arm needs to produce anything other than `true`/`false`, it is a `match`
again.

## ID-6: Name the local after the field

Then use field-init shorthand.

```rust
// DO
let name = row.get("name")?;
let age = row.get("age")?;

Person { name, age }

// DON'T
let person_name = row.get("name")?;
let the_age = row.get("age")?;

Person { name: person_name, age: the_age }
```

## ID-7: Pattern matching over field access

Destructure when it makes the control flow more explicit.

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

## ID-8: Import grouping

Three blocks, blank-line separated, widest scope to narrowest:

1. **`std` / `core` / `alloc`**
2. **External crates**
3. **Internal**: `crate::`, `super::`, `self::`

Alphabetize within each block.

Why grouping at all: the blocks make a file's dependency profile legible at a
glance, how much it leans on `std`, which third-party crates it pulls in, and
what it touches inside the crate, instead of burying that in one undifferentiated
wall.

```rust
// DO
use std::collections::HashMap;
use std::ops::Not;

use logos::Logos;
use serde::Deserialize;

use crate::ast::Node;
use crate::lexer::Token;

// DON'T: one undifferentiated wall
use crate::ast::Node;
use crate::lexer::Token;
use logos::Logos;
use serde::Deserialize;
use std::collections::HashMap;
use std::ops::Not;
```

### Working with rustfmt

This is exactly `rustfmt`'s `group_imports = "StdExternalCrate"`, so let the
formatter do it:

```toml
# rustfmt.toml
group_imports = "StdExternalCrate"
```

That option is nightly-only, so it needs `cargo +nightly fmt` to take effect.
On stable it is ignored (no error, the grouping just goes unenforced), so
apply ID-8 by hand there.

`reorder_imports` is stable and on by default. It alphabetizes *within* each
blank-line-separated block without merging blocks, which keeps ID-8's
alphabetization free on either toolchain.

## ID-9: `std::ops::Not` over prefix `!`

A leading `!` is one glyph attached to something long; it is easy to miss and
forces parentheses. `.not()` reads in the same direction as the rest of the
chain.

Use `.not()` when the negation wraps a call, a macro, or a parenthesized
expression:

```rust
// DO
use std::ops::Not;

is_epic.not().then(|| render());
matches!(state, State::Done).not()
name.is_empty().not()

// DON'T
(!is_epic).then(|| render());
!matches!(state, State::Done)
!name.is_empty()
```

Keep prefix `!` for a bare identifier in a plain condition, where there is
nothing to lose track of:

```rust
if !ready {
    return;
}
```

## ID-10: Alphabetize `Cargo.toml`

Every list stays alphabetized: `[dependencies]`, `[dev-dependencies]`,
`[build-dependencies]`, `[features]` (and the crates within each feature's
list), and `workspace.members`.

Why: an alphabetized list makes "is X already here?" a lookup instead of a scan,
and it removes an entire class of merge conflict: two branches adding
dependencies land in different places instead of both at the end.

```toml
# DO
[dependencies]
anyhow = "1"
logos = "0.14"
serde = { version = "1", features = ["derive"] }

# DON'T: insertion order
[dependencies]
serde = { version = "1", features = ["derive"] }
anyhow = "1"
logos = "0.14"
```

Section order itself follows Cargo convention (`[package]`, `[dependencies]`,
`[dev-dependencies]`, …), not alphabetical.

## ID-11: Type-hint on the RHS, via turbofish

When a call's return type needs a hint, put it on the call with `::<...>`, not
on the binding.

```rust
// DO
let names = names_iter.collect::<Vec<_>>();

// DON'T
let names: Vec<_> = names_iter.collect();
```

The LHS states a *name*; the RHS states an *expression*, and the hint is a fact
about that expression, not the binding. Keeping it on the RHS also means the
hint travels with the call if the result is later returned or passed inline
instead of bound at all, and it keeps every binding in the function shaped the
same way (`let x = ...;`) so the reader scans names on the left without
detouring through types.

## ID-12: Infer with `_` wherever the compiler can

Once one part of a type is pinned down elsewhere, an explicit repeat of the
rest is noise. Write `_` for any parameter the compiler can recover from
context.

```rust
// DO
let names = names_iter.collect::<Vec<_>>();
let cache = HashMap::<_, _>::new();

// DON'T
let names = names_iter.collect::<Vec<String>>();
let cache = HashMap::<String, Vec<u8>>::new();
```

This pairs with ID-11: the turbofish supplies only the piece of information the
compiler actually needs (`Vec` over, say, `HashSet`; `HashMap` over `BTreeMap`),
and `_` leaves every parameter to inference instead of restating what the rest
of the line already fixes. If the compiler can't infer it, it will say so, and
that error is the signal to spell out that one parameter, not the whole type.
