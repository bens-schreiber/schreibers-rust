# Idioms

Rules `ID-1` … `ID-11`. The through-line:
**make the reader's job easy at the call site, not just at the definition.**

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

This extends to struct fields and return types. Two bools whose combination has
an impossible state are one enum, which makes that state unrepresentable.
Independent flags (`verbose` and `dry_run`) stay separate bools: collapsing them
gives you a four-variant cartesian enum and breaks the `serde` / `clap` derives.

## ID-2: Don't panic outside tests

Handle the error path: propagate with `?`, or destructure with `let ... else`
(ID-4). Test code is exempt: a panic there is a failed assertion.

When a panic is genuinely unreachable, say why in an `.expect()` message. Never
a bare `.unwrap()`, and never a `// PANIC:` comment: a comment is invisible in
the log that will eventually carry the panic.

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

Reserve `// SAFETY:` for `unsafe` blocks, where it is the language's own
convention and what clippy's `undocumented_unsafe_blocks` looks for.

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

Implement `From`, not `Into`: the blanket impl gives you `Into` for free, and
implementing `Into` directly forfeits the reverse. `Into` as a *generic bound*
(`name: impl Into<String>`) is a different thing and is often the better
signature.

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

## ID-5: Name the local after the field

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

## ID-6: Pattern matching over field access

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

When the only thing you want out of the pattern is a `bool`, that is `matches!`
(`let is_terminal = matches!(state, State::Done | State::Failed);`). Once an arm
has to produce anything else, it is a `match` again.

## ID-7: Import grouping

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

`group_imports = "StdExternalCrate"` does exactly this, but is nightly-only:
it needs `cargo +nightly fmt`, and stable ignores it silently, so on stable the
grouping is by hand. `reorder_imports` is stable and on by default, so the
alphabetization within each block is free on either toolchain.

## ID-8: `std::ops::Not` over prefix `!`

A leading `!` is one glyph attached to something long; it is easy to miss and
forces parentheses. `.not()` reads in the same direction as the rest of the
chain.

Use `.not()` when you are continuing the chain afterward, or negating a
`matches!`. It needs `use std::ops::Not;` in the file.

```rust
// DO
use std::ops::Not;

is_epic.not().then(|| render());
matches!(state, State::Done).not()

// DON'T
(!is_epic).then(|| render());
!matches!(state, State::Done)
```

Keep prefix `!` in condition position, where the `if` already frames it:

```rust
// DO
if !ready {
    return;
}

if !name.is_empty() { /* ... */ }

// DON'T: nothing gained, and the negation is now at the far end of the line
if name.is_empty().not() { /* ... */ }
```

## ID-9: Alphabetize `Cargo.toml`

Every list stays alphabetized: `[dependencies]`, `[dev-dependencies]`,
`[build-dependencies]`, `[features]` (and the crates within each feature's
list), and `workspace.members`.

Why: it makes "is X already here?" a lookup instead of a scan, and stops two
branches from both appending at the end.

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

## ID-10: Keep compound generic types off the LHS

The thing to avoid is a container type sprawling across the left of a `let`,
where it pushes the name away from the reader. When a call's return type needs a
hint, prefer the turbofish.

```rust
// DO
let names = names_iter.collect::<Vec<_>>();

// DON'T
let names: Vec<_> = names_iter.collect();
```

The hint is a fact about the expression, not the binding, and on the RHS it
travels with the call if the result is later returned or passed inline.

Annotated on the LHS is fine, and sometimes required: `const` / `static` /
signatures, plain scalars (`let count: usize = ...`), and expressions with no
turbofish to hang a hint on (`.into()`, struct-literal fields). Judgement call.

## ID-11: Infer with `_` wherever the compiler can

Supply only the part the compiler actually needs (`Vec` over `HashSet`,
`HashMap` over `BTreeMap`) and leave the rest to inference. If it can't infer,
it will say so, and that error names the one parameter to spell out.

```rust
// DO
let cache = HashMap::<_, _>::new();

// DON'T
let cache = HashMap::<String, Vec<u8>>::new();
```
