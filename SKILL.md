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

## SC-1: bind at the read count

The rule you will reach for most, so it gets its example here. Read it against
the code you write, not once at the start.

Count the later scopes that read a local. **Two or more:** top level. **Exactly
one:** inside the thing that reads it, never at top level. The trigger is the
read count, not the line count: a one-line binding read once is the most common
instance, not an exception. Sibling branches of one `if` or `match` are one
reader between them, so a local read in two arms binds directly above that
construct rather than at the top of the function.

Fix it in this order: a method chain where one exists, a block expression where
the computation needs statements, an extracted `fn` where it deserves a name.

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

Leave the binding where it is when pushing it in would cost the reader more
than it saves. Common cases, not a closed list:

- A short prelude to the function's tail expression.
- A binding whose name is what keeps a nearby `.expect()` or error message
  readable (ID-2).
- A derivation several adapters long whose name is the only thing that says what
  it computes. A short, self-evident chain does not qualify.

## Scoping

| ID | Rule |
| --- | --- |
| SC-1 | Above. |
| SC-2 | Fence unrelated units of work inside a function in `{}`, each under a one-line comment naming it. A naming comment over loose statements is a divider comment (SC-6) and lets neighboring units shadow each other. |
| SC-3 | Nest a helper `fn` in its only caller when short. Module scope if unit-tested, doc-tested, or long enough to displace the caller's body. |
| SC-4 | Closure when it captures from the enclosing scope, nested `fn` when everything arrives as parameters. Never re-pass what a closure could capture. |
| SC-5 | Closures go at the top of the function body. Not when hoisting would extend a capture across code that needs the same borrow. |
| SC-6 | No divider comments (`// ------`, `// ==== FOO ====`). Use a `{}` block inside a function, a `mod` or separate file outside one. |

## Formatting

| ID | Rule |
| --- | --- |
| FMT-1 | Within a block, put one blank line after each direct-child statement ending in `}` or `};` when another independent statement follows. Do not separate joined `if`/`else if`/`else` branches or individual `match` arms. |

## Modules and structs

| ID | Rule |
| --- | --- |
| MS-1 | A `mod`, not a shared prefix on free functions: `mod util { fn parse }`, not `fn util_parse`. |
| MS-2 | Narrowest visibility that compiles: private, `pub(super)`, `pub(crate)`, `pub`. `pub` only for something used outside the crate. |
| MS-3 | Visibility widened only for tests gets `// Visible for tests.` above it, without repeating the keyword. |
| MS-4 | A `mod` of free functions, not a zero-field struct that only namespaces an `impl`. |
| MS-5 | A struct with a real `impl`, not a `mod`, once the same non-trivial parameters thread through several functions. |
| MS-6 | Three or more methods in an `impl` serving one side feature rather than the type's main job are their own component, by MS-4 or MS-5. Extract it; do not add `impl` blocks. No honest seam means no finding. |
| MS-7 | Order every module body: `//!` docs, `mod foo;` declarations, imports and re-exports, macros, constants and statics, then everything else. Within the constants, and again within everything else, most public first: `pub`, `pub(crate)`, `pub(super)`, `pub(in ...)`, private. Types and traits sit anywhere their visibility tier allows. A macro stays above any `mod` that uses it, which outranks this order. |

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
| ID-8 | Prefer `matches!(..).not()` over `!matches!(..)` everywhere, including `if`/`while` conditions. |
| ID-9 | Every list in `Cargo.toml` alphabetized: dependencies, dev-dependencies, build-dependencies, features, workspace members. |
| ID-10 | Compound generics off the LHS: `.collect::<Vec<_>>()`, not `let names: Vec<_> =`. Scalars, `const`/`static`, signatures, and expressions with no turbofish stay LHS-annotated. |
| ID-11 | `_` for any generic parameter the compiler can infer. |
| ID-12 | Bind a `for` loop's iterable to a named local first when it carries non-trivial filtering, mapping, or closure logic. |
| ID-13 | Comment a non-obvious `continue`, `break`, or early `return` immediately above it, inside the branch. Say why, never what the condition already says. |
| ID-14 | A closure passed to `.map()`, `.filter()` or another adaptor stays small enough to read as one expression. Once it needs statements, its own control flow, or more than a few lines, write a `for` loop, or extract a named `fn` when it has another caller. |

## Tests

| ID | Rule |
| --- | --- |
| TS-1 | Unit tests in a `#[cfg(test)] mod tests` in the file under test; integration tests in the crate's `tests/`. |
| TS-2 | `namespace_input_expectation`: subject, input condition, expected outcome. Drop `namespace` when the module or file supplies it. No `test_` prefix. |
| TS-3 | Label Arrange, Act and Assert when any one of them exceeds a line. Skip all three labels only when all three are one-liners. An Arrange living outside the body still counts as a phase. Never extend a label with narrative. |
| TS-4 | Bind a value in Arrange when used more than once, so a call and its assertion cannot drift. Used once, it stays inline. |
| TS-5 | Custom panic message on any `assert*` whose failure reason is not obvious. It goes in the message, not a comment above it. |
| TS-6 | Coalesce tests sharing most of their Arrange into one function, one case per `{}` block. Hoist only the shared Arrange; each case keeps its own Act and Assert. No shared Arrange means no TS-6, and `// Case:` is the only label those blocks take. |
| TS-7 | Three or more tests sharing a `namespace_*` prefix become a sibling `#[cfg(test)] mod namespace_tests`, prefix dropped (TS-2). Count the group as it will stand when you finish. TS-7 runs first; coalesce with TS-6 inside the new mod. |
| TS-8 | Test data that models a real domain thing (a site, host, region, tenant, SKU, path) gets a plausible value, not a placeholder. `"us-east-2b"`, not `"foo"`. Codebase convention wins. |
| TS-9 | Assert on typed values, never on a string the test assembles with `format!`. Return an enum, a struct, or a well-known `const`, with `Display` for the message. When the payload is not `PartialEq`, assert with `matches!` (ID-6). A literal expected string is fine in the one test covering the rendering itself. |

## Documentation

| ID | Rule |
| --- | --- |
| DOC-1 | Blank line between a documented field and the undocumented fields below it. |
| DOC-2 | `//!` header on every crate and non-obvious module, above the imports: purpose, public API, design quirks. |
| DOC-3 | Never an em dash, in doc comments, comments, or commit messages. Use a period, comma, colon, semicolon, or parentheses. |
| DOC-4 | Bullets whenever documentation or a comment enumerates distinct responsibilities, behaviors, or conditions. Never hide a list in commas or a run-on sentence. Indent bullet continuations under the bullet's *text*, or rustdoc drops them from the list. |

## Before you call it done

Run these over the diff and report what each caught, by ID, including the ones
that caught nothing. They are a net, not the rule: SC-1, MS-6 and TS-7 cannot be
grepped and still need reading.

```sh
rg -n -A1 '// Case:' src/ tests/   # TS-6: next line is `{`, shared Arrange hoisted above
rg -n -A6 '#\[test\]' src/ tests/  # TS-3: any phase over one line means all three labels
rg -n 'assert\w*!.*format!' src/ tests/  # TS-9: assert the type, not the rendered string
rg -n '\.unwrap\(\)' src/                # ID-2: every hit outside a `#[cfg(test)] mod`
rg -n 'let \w+: (Vec|HashMap|HashSet|BTreeMap)<' src/ tests/  # ID-10: every hit
```
