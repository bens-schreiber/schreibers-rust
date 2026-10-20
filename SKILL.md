---
name: schreibers-rust
description: Ben Schreiber's Rust style guide. Use whenever writing, editing, or reviewing Rust (.rs) code, or when asked to fix nits, clean up style, or make code idiomatic. Covers scoping and blocks, module vs. struct, visibility, idioms, imports, test structure and naming, and doc comments. Not for generated or vendored Rust.
---

# Schreiber's Rust Style

House style on top of `rustfmt` and `clippy` defaults. `references/rules.md` has
worked examples for the rules that need one.

Apply only to code you are writing or editing. Skip generated, vendored, and
macro-heavy files. When rules collide: correctness, then the file's existing
conventions, then the more specific rule. Reviewing: cite the ID, and raise a
rule only where following it measurably helps the reader.

## SC-1: keep bindings readable

The rule you will reach for most, so it gets its example here. Read it against
the code you write, not once at the start.

Keep a binding in the smallest scope that remains readable. Inline trivial
one-use values; keep names that clarify a multi-step computation, an error, or a
meaningful concept. Use a block or extract a function when that makes the scope
clearer.

```rust
// DO: no name for the token vector survives into the rest of the function
let mut parser = Parser {
    tokens: Lexer::new(source)
        .collect::<Result<Vec<_>, _>>()?
        .into_iter()
        .peekable(),
};

// DON'T: one line, read once, and still top-level state the reader must track
let tokens = Lexer::new(source).collect::<Result<Vec<_>, _>>()?;

let mut parser = Parser { tokens: tokens.into_iter().peekable() };
```

## Scoping

| ID | Rule |
| --- | --- |
| SC-1 | Above. |
| SC-2 | Use a block when it limits scope or clearly separates a distinct unit of work. Do not add blocks or naming comments when they do not improve readability. |
| SC-3 | Nest a helper `fn` in its only caller when short. Module scope if unit-tested, doc-tested, or long enough to displace the caller's body. |
| SC-5 | Closures go at the top of the function body. Not when hoisting would extend a capture across code that needs the same borrow. |
| SC-6 | No divider comments (`// ------`, `// ==== FOO ====`). Use a `{}` block inside a function, a `mod` or separate file outside one. |

## Formatting

| ID | Rule |
| --- | --- |
| FMT-1 | Use blank lines to separate independent statements and standalone comment blocks. Put them above and below bindings or blocks whose bodies span more than three lines. Keep joined `if`/`else if`/`else` branches and `match` arms together. |

## Modules and structs

| ID | Rule |
| --- | --- |
| MS-1 | A `mod`, not a shared prefix on free functions: `mod util { fn parse }`, not `fn util_parse`. |
| MS-2 | Use private visibility by default. When visibility must be widened, prefer plain `pub`; do not use `pub(crate)` or `pub(super)` unless absolutely necessary because neither private visibility nor plain `pub` can express the required boundary. |
| MS-3 | Visibility widened only for tests gets `// Visible for tests.` above it, without repeating the keyword. Prefer plain `pub`; use `pub(crate)` or `pub(super)` only under MS-2's exception. |
| MS-4 | A `mod` of free functions, not a zero-field struct that only namespaces an `impl`. |
| MS-7 | Order every module body: inner module documentation (`//!`), external module declarations (`mod foo;`), imports and re-exports, macro definitions, constants and statics (most public first), then everything else (plain `pub`, exceptional restricted visibility, private). A macro must appear before any item that uses it when textually required. |

## Idioms

| ID | Rule |
| --- | --- |
| ID-1 | An enum, not a `bool` parameter or field. Two bools with an impossible combination are one enum; independent flags stay separate. |
| ID-2 | Outside tests, handle the error path. An unreachable panic is `.expect("<why it cannot fail>")`, never bare `.unwrap()`, never a `// PANIC:` comment. `// SAFETY:` is for `unsafe` only. |
| ID-3 | Implement the standard traits instead of hand-rolling their contract: `Default`, `From`, `TryFrom`, `Display`, `FromStr`, `AsRef`, `Iterator`. `From`, never `Into`; `Into` as a bound is fine. `FromStr` and `Iterator` cannot yield a value that borrows from the input, so leave those signatures alone. |
| ID-4 | Exit early instead of nesting the happy path: `let ... else`, `if let`, `?`. |
| ID-5 | Name a local after the field it fills, then use field-init shorthand. |
| ID-6 | Pattern matching over field access plus conditionals. `matches!` when all you need is a `bool`. |
| ID-7 | Three blank-line-separated `use` blocks, alphabetized within each: `std`/`core`/`alloc`, external crates, internal (`crate`, `super`, `self`). Stable `rustfmt` will not group them; do it by hand. |
| ID-8 | `.not()` when negating a chain you continue (`x.is_empty().not().then(...)`) or a bare `matches!`. In `if`/`while` condition position prefix `!` always wins, `matches!` included. |
| ID-9 | Every list in `Cargo.toml` alphabetized: dependencies, dev-dependencies, build-dependencies, features, workspace members. |
| ID-10 | Compound generics off the LHS: `.collect::<Vec<_>>()`, not `let names: Vec<_> =`. Scalars, `const`/`static`, signatures, and expressions with no turbofish stay LHS-annotated. |
| ID-11 | `_` for any generic parameter the compiler can infer. |
| ID-12 | Bind a `for` loop's iterable to a named local first when it carries non-trivial filtering, mapping, or closure logic. |
| ID-13 | Comment a non-obvious `continue`, `break`, or early `return` immediately above it, inside the branch. Say why, never what the condition already says. |
| ID-14 | A closure passed to `.map()`, `.filter()` or another adaptor stays small enough to read as one expression. Once it needs statements, its own control flow, or more than a few lines, write a `for` loop, or extract a named `fn` when it has another caller. |

## Tests

| ID | Rule |
| --- | --- |
| TS-1 | `namespace_input_expectation`: subject, input condition, expected outcome. Drop `namespace` when the module or file supplies it. No `test_` prefix. |
| TS-2 | Label Arrange, Act and Assert when any one of them exceeds a line. Skip all three labels only when all three are one-liners. An Arrange living outside the body still counts as a phase. Never extend a label with narrative. |
| TS-3 | Bind a value in Arrange when used more than once, so a call and its assertion cannot drift. Used once, it stays inline. |
| TS-4 | Custom panic message on any `assert*` whose failure reason is not obvious. It goes in the message, not a comment above it. |
| TS-5 | Coalesce tests sharing most of their Arrange into one function, one case per `{}` block. Hoist only the shared Arrange; each case keeps its own Act and Assert. No shared Arrange means no TS-5, and `// Case:` is the only label those blocks take. |
| TS-6 | Three or more tests sharing a `namespace_*` prefix become a sibling `#[cfg(test)] mod namespace_tests`, prefix dropped (TS-1). Count the group as it will stand when you finish. TS-6 runs first; coalesce with TS-5 inside the new mod. |
| TS-7 | Test data that models a real domain thing (a site, host, region, tenant, SKU, path) gets a plausible value, not a placeholder. `"us-east-2b"`, not `"foo"`. Codebase convention wins. |
| TS-8 | Assert on typed values, never on a string the test assembles with `format!`. Return an enum, a struct, or a well-known `const`, with `Display` for the message. When the payload is not `PartialEq`, assert with `matches!` (ID-6). A literal expected string is fine in the one test covering the rendering itself. |

## Documentation

| ID | Rule |
| --- | --- |
| DOC-1 | Blank line between a documented field and the undocumented fields below it. |
| DOC-2 | `//!` header on every crate and non-obvious module, above the imports: purpose, public API, design quirks. |
| DOC-3 | Never an em dash, in doc comments, comments, or commit messages. Use a period, comma, colon, semicolon, or parentheses. |
| DOC-4 | Bullets whenever documentation or a comment enumerates distinct responsibilities, behaviors, or conditions. Never hide a list in commas or a run-on sentence. Indent bullet continuations under the bullet's *text*, or rustdoc drops them from the list. |

## Before you call it done

Run these over the diff and report what each caught, by ID, including the ones
that caught nothing. They are a net, not the rule: SC-1 and TS-6 cannot be
grepped and still need reading.

```sh
rg -n -A1 '// Case:' src/ tests/   # TS-5: next line is `{`, shared Arrange hoisted above
rg -n -A6 '#\[test\]' src/ tests/  # TS-2: any phase over one line means all three labels
rg -n 'assert\w*!.*format!' src/ tests/  # TS-8: assert the type, not the rendered string
rg -n '\.unwrap\(\)' src/                # ID-2: every hit outside a `#[cfg(test)] mod`
rg -n 'let \w+: (Vec|HashMap|HashSet|BTreeMap)<' src/ tests/  # ID-10: every hit
```
