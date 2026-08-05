# Modules and Structs

Rules `MS-1` … `MS-6`. The through-line: **let the container carry the shared
context: a name prefix, a repeated parameter, and a speculative `pub` are all
the same failure.**

## MS-1: A `mod` instead of a shared name prefix

If several free functions share a prefix, that prefix is a module trying to
exist.

```rust
// DO
mod util {
    pub fn parse() { /* ... */ }
    pub fn render() { /* ... */ }
    pub fn validate() { /* ... */ }
}

// DON'T
pub fn util_parse() { /* ... */ }
pub fn util_render() { /* ... */ }
pub fn util_validate() { /* ... */ }
```

Callers read `util::parse()`, and the prefix stops being repeated in every
definition.

## MS-2: Narrowest visibility that compiles

Walk up only as far as you must: private → `pub(super)` → `pub(crate)` → `pub`.

Why: `pub` is not a convenience, it is an API surface someone now has to keep
stable. `pub(crate)` says "internal, refactor freely"; `pub` says "breaking this
is a semver event."

```rust
// DO: used elsewhere in the crate, never outside it
pub(crate) fn parse(src: &str) -> Ast { /* ... */ }

// DON'T: pub, but nothing outside this module calls it
pub fn parse(src: &str) -> Ast { /* ... */ }
```

## MS-3: Say when visibility exists only for tests

If an item is widened purely so a test can reach it, record that directly above
it. Otherwise the next reader assumes there is a real consumer and preserves the
visibility forever.

```rust
// DO
// Visible for tests.
pub(crate) fn normalize(raw: &str) -> String { /* ... */ }
```

A unit test in a `#[cfg(test)] mod tests` in the same file can already see
private items (TS-1), so this note should be rare: it is for integration tests
and cross-module test helpers.

## MS-4: A `mod`, not a field-less struct used as a namespace

```rust
// DON'T: Util holds no data; it exists only to hang methods off of
struct Util;

impl Util {
    fn parse() { /* ... */ }
}

// DO: same fix as MS-1: a mod, not a type standing in for one
mod util {
    pub fn parse() { /* ... */ }
}
```

## MS-5: A struct once parameters repeat

The inverse of MS-4. When the same set of non-trivial parameters is threaded
through several functions, those parameters are state: store them once.

```rust
// DO: the context is stored once, methods just read self
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

The trigger is repetition, not parameter count: two functions sharing two
parameters is a struct; one function taking four one-off arguments is not.

## MS-6: Split past ~400 lines

Split any `impl` block or `mod` once it passes roughly 400 lines. Cut along a
real seam (a responsibility, a sub-resource, a lifecycle stage) so each piece
stays cohesive; do not slice at line 400 for its own sake.
For a large `impl`, prefer several `impl` blocks in separate files over one
sprawling block:

```rust
// src/renderer/mod.rs
struct Renderer { /* ... */ }

// src/renderer/layout.rs
impl Renderer {
    // everything about measuring and placing
}

// src/renderer/paint.rs
impl Renderer {
    // everything about emitting output
}
```

If no honest seam exists, the type is doing too much: that is the finding, and
it outranks the line count.
