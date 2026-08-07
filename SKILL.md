---
name: schreibers-rust
description: Ben Schreiber's Rust style guide. Use whenever writing, editing, or reviewing Rust (.rs, Cargo.toml) code, governing scoping and block structure, module vs. struct choices, visibility, idiomatic trait usage, import grouping, test structure/naming, and doc comments.
---

# Schreiber's Rust Style

A house style for Rust. It sits **on top of** `rustfmt` and `clippy` defaults.

## How to use this skill

**Writing or editing Rust:** read the rule tables below, then open the reference
file for whichever section your change touches. The tables are the complete rule
set; the references carry the DO/DON'T pairs that disambiguate them.

**Reviewing Rust:** work the tables top to bottom as a checklist. Cite the rule
ID (e.g. `SC-3`) in review comments.

**Conflict resolution, in order:**

1. Compiler correctness beats every rule here.
2. A narrower, more specific rule beats a broader one.
3. This guide beats generic Rust convention.
4. When still tied, prefer the choice that puts a name closer to its only use.

## Scoping (`references/scoping.md`)

| ID | Rule |
| --- | --- |
| SC-1 | Bind a variable in the smallest block that contains its full lifetime. If a local exists only to compute one later value, wrap it in a block expression so it never reaches the function's top level. |
| SC-2 | A local may sit at function top level only when it is genuinely read from two or more later scopes. |
| SC-3 | Use a bare `{}` block, even one producing no value, to separate unrelated units of work inside one function. Give each block a one-line comment naming what it does. |
| SC-4 | Nest a helper `fn` inside its caller when that caller is its only call site. |
| SC-5 | Prefer a closure over a nested `fn` when the logic wants to capture from the enclosing scope. Use a nested `fn` when everything arrives as explicit parameters. Never re-pass a value as a closure parameter that the closure could have captured. |
| SC-6 | Declare closures at the top of the function body, before first use. This is the one deliberate exception to SC-1. |
| SC-7 | Never write divider comments (`// ------`, `// ==== FOO ====`). A divider means scoping failed: use a `{}` block inside a function, or a `mod` / separate file / separate crate outside one. |

## Modules and structs (`references/modules-and-structs.md`)

| ID | Rule |
| --- | --- |
| MS-1 | Group related items in a `mod` instead of prefixing free functions with a shared name (`mod util { fn foo }`, not `fn util_foo`). |
| MS-2 | Use the narrowest visibility that compiles: private → `pub(super)` → `pub(crate)` → `pub`. Reach for `pub` only when something outside the crate uses it. Speculative `pub` is API surface someone must now keep stable. |
| MS-3 | When an item is widened in visibility only so tests can reach it, say so in a comment directly above it: `// Visible for tests.` Don't repeat the visibility keyword in the comment; the signature below already states it. |
| MS-4 | Prefer a `mod` of free functions over a zero-field struct that exists only to namespace an `impl`. |
| MS-5 | Prefer a struct with a real `impl` over a `mod` when the same set of non-trivial parameters is threaded through several functions. Store them once instead of repeating them in every signature. |
| MS-6 | Split any `impl` block or `mod` that exceeds ~400 lines, cutting along a seam (responsibility, sub-resource, lifecycle stage) that leaves each piece cohesive. |

## Idioms (`references/idioms.md`)

| ID | Rule |
| --- | --- |
| ID-1 | Prefer an enum to a `bool` parameter or field. A `bool` forces the reader to decode what `true` means at the call site; a variant names it. |
| ID-2 | Never call `.unwrap()` or `.expect()` outside test code. Handle the error path. If a panic is genuinely unreachable, justify it with a `// PANIC:` comment stating why. Reserve `// SAFETY:` for `unsafe` blocks. |
| ID-3 | Implement the standard traits rather than hand-rolling their contract: `Default`, `From`, `TryFrom`, `Display`, `FromStr`, `AsRef`, `Iterator`. Implement `From`, never `Into`. |
| ID-4 | Exit early instead of nesting the happy path. Lean on `let ... else`, `if let`, and `?`. |
| ID-5 | Use `matches!` when you only need a `bool` out of a pattern test. |
| ID-6 | Name a local after the field it will fill, then use field-init shorthand. |
| ID-7 | Prefer pattern matching to manual field access plus conditionals wherever it makes control flow more explicit. |
| ID-8 | Group `use` statements into three blank-line-separated blocks in this order: **`std`/`core`/`alloc`, external crates, internal (`crate`, `super`, `self`).** Keep each block alphabetized. This is `rustfmt`'s `group_imports = "StdExternalCrate"`, so let the formatter enforce it. |
| ID-9 | Use `std::ops::Not` (`x.not()`) instead of a prefix `!` when the negation wraps a call, a macro, or a parenthesized expression. Keep prefix `!` for a bare identifier in a plain `if`. |
| ID-10 | Keep every list in `Cargo.toml` alphabetized: dependencies, dev-dependencies, build-dependencies, features, and workspace members. |
| ID-11 | Put a type hint on the RHS via turbofish (`.collect::<Vec<_>>()`), never on the LHS binding (`let x: Vec<_> = ...`). |
| ID-12 | Use `_` for any generic parameter the compiler can infer from context, rather than spelling out the concrete type. |

## Tests (`references/tests.md`)

| ID | Rule |
| --- | --- |
| TS-1 | Unit tests go in a `#[cfg(test)] mod tests` in the file under test; integration tests go in the crate's `tests/` directory. |
| TS-2 | Name tests `namespace_input_expectation`: subject, the input condition, then the expected outcome. Drop `namespace` when the module or file name already supplies it. |
| TS-3 | Label a test body with `// Arrange`, `// Act`, `// Assert` once the three phases are not obvious at a glance. Skip the labels for one-line-per-phase tests. Never extend a label with narrative; the label stands alone. |
| TS-4 | Declare every value a test uses as a named `const` in `Arrange`. No literal may appear inside `Act` or `Assert`; a value used by both a call and an assertion must come from one binding so the two cannot drift. |
| TS-5 | Give a custom panic message to any `assert*` whose failure reason is not obvious from the surrounding code. The explanation belongs in the message, not in a comment above it. |
| TS-6 | Coalesce tests that share most of their Arrange into one function with a `// Case: <name>` block per case. Hoist only the shared Arrange; each case keeps its own Act and Assert. |
| TS-7 | Once a `namespace_*` group exceeds three test functions, move it into its own `mod` or file so the prefix can be dropped (TS-2). TS-7 splits by subject; TS-6 merges by fixture: apply TS-6 within a group, TS-7 across groups. |
| TS-8 | Use Vox Machina names for genuinely arbitrary dummy data (`vex`, `vax`, `percy`, `keyleth`, `grog`, `pike`, `scanlan`), never `foo`/`bar`/`alice`/`bob`. When a value carries meaning the test depends on, name it for that meaning instead. |

## Documentation (`references/documentation.md`)

| ID | Rule |
| --- | --- |
| DOC-1 | Use markdown and line breaks liberally in doc comments: a one-line summary, a blank `///` line, then paragraphs, `#` headings, and bullet lists. It should read well as source and as rendered rustdoc. |
| DOC-2 | Continuation lines of a bullet are indented to align under the bullet's text. |
| DOC-3 | Separate a documented field from the undocumented fields that follow it with a blank line, so the doc comment's scope is unambiguous. |
| DOC-4 | Give every crate and non-obvious module a `//!` header at the top of the file, above the imports, covering purpose, public API, and design quirks. |
| DOC-5 | Do not document what the name already says. If a name needs a comment to be understood, rename it. |
| DOC-6 | Use rustdoc's conventional headings for their conventional meanings only: `# Safety` for `unsafe` contracts, `# Panics`, `# Errors`, `# Examples`. Anything else gets a heading of your own wording. |
| DOC-7 | Never write an em dash, in doc comments, comments, or commit messages. Use a period, comma, colon, semicolon, or parentheses instead, whichever the sentence calls for. |
