---
name: schreibers-rust
description: Ben Schreiber's Rust style guide. Use whenever writing, editing, or reviewing Rust (.rs) code, or when asked to fix nits, clean up style, or make code idiomatic. Governs scoping and block structure, module vs. struct choices, visibility, idiomatic trait usage, import grouping, test structure/naming, and doc comments. Not for generated or vendored Rust.
---

# Schreiber's Rust Style

A house style for Rust, on top of `rustfmt` and `clippy` defaults.

## How to use this skill

These are **preferences, not a linter.** Every rule names the problem it solves;
serve the reason, not the letter. A rule applied against its own reasoning is a
bug, not compliance.

Apply this only to code you are already writing or editing. Do not restyle
untouched code, and skip generated, vendored, and macro-heavy files.

Reviewing: raise a rule only where following it would measurably help the
reader. Cite the ID (e.g. `SC-3`).

**When rules collide:** correctness first, then the conventions already in the
file, then the more specific rule.

The tables below are the rules. Open a reference file only when a table row
leaves you genuinely unsure how it applies; each section says when that is.

## Scoping (`references/scoping.md`)

Open it when restructuring a function body or deciding between a block, a
closure, and a helper `fn`.

| ID | Rule |
| --- | --- |
| SC-1 | If a local exists only to compute one later value, wrap it in a block expression so it never reaches the function's top level. For multi-line intermediates that are genuinely dead afterward; a short prelude to a return is fine as-is, and a method chain or an extracted `fn` often beats a block. |
| SC-2 | A local belongs at function top level when two or more later scopes read it. |
| SC-3 | Use a bare `{}` block to separate unrelated units of work inside one function, each with a one-line comment naming what it does. For units of several lines each, not for every pair of statements. |
| SC-4 | Nest a helper `fn` inside its only caller when it is short and needs no test of its own. Keep it at module scope if it is unit-tested, doc-tested, or long enough to displace the caller's own body. |
| SC-5 | Prefer a closure over a nested `fn` when the logic wants to capture from the enclosing scope. Use a nested `fn` when everything arrives as explicit parameters. Never re-pass a value as a closure parameter that the closure could have captured. |
| SC-6 | Declare closures at the top of the function body, before first use: the one deliberate exception to SC-1. Not when hoisting would extend a mutable capture across code that needs the same borrow. |
| SC-7 | Never write divider comments (`// ------`, `// ==== FOO ====`). A divider means scoping failed: use a `{}` block inside a function, or a `mod` / separate file / separate crate outside one. |

## Modules and structs (`references/modules-and-structs.md`)

Open it when adding a module or type, or changing visibility.

| ID | Rule |
| --- | --- |
| MS-1 | Group related items in a `mod` instead of prefixing free functions with a shared name (`mod util { fn foo }`, not `fn util_foo`). |
| MS-2 | Use the narrowest visibility that compiles: private → `pub(super)` → `pub(crate)` → `pub`. Reach for `pub` only when something outside the crate uses it. Speculative `pub` is API surface someone must now keep stable. |
| MS-3 | When an item is widened in visibility only so tests can reach it, say so in a comment directly above it: `// Visible for tests.` Don't repeat the visibility keyword; the signature below already states it. |
| MS-4 | Prefer a `mod` of free functions over a zero-field struct that exists only to namespace an `impl`. |
| MS-5 | Prefer a struct with a real `impl` over a `mod` when the same set of non-trivial parameters is threaded through several functions. Store them once instead of repeating them in every signature. |
| MS-6 | Split an `impl` block or `mod` past ~400 lines along a seam (responsibility, sub-resource, lifecycle stage) that leaves each piece cohesive. If no honest seam exists, that is the finding, and it outranks the line count. |

## Idioms (`references/idioms.md`)

Open it when unsure whether ID-8 or ID-10 applies to a specific expression.

| ID | Rule |
| --- | --- |
| ID-1 | Prefer an enum to a `bool` parameter or field: a `bool` makes the reader decode what `true` means at the call site. Two bools whose combination has an impossible state are one enum; independent flags stay separate bools. |
| ID-2 | Outside tests, handle the error path instead of panicking. When a panic is genuinely unreachable, use `.expect("<why it cannot fail>")`, never a bare `.unwrap()`: the reason belongs in the panic message, where a log will show it. Reserve `// SAFETY:` for `unsafe` blocks. |
| ID-3 | Implement the standard traits rather than hand-rolling their contract: `Default`, `From`, `TryFrom`, `Display`, `FromStr`, `AsRef`, `Iterator`. Implement `From`, not `Into` (the blanket impl gives you `Into` free). `Into` as a generic bound is fine. |
| ID-4 | Exit early instead of nesting the happy path. Lean on `let ... else`, `if let`, and `?`. |
| ID-5 | Name a local after the field it will fill, then use field-init shorthand. |
| ID-6 | Prefer pattern matching to manual field access plus conditionals wherever it makes control flow more explicit. Use `matches!` when all you need out of the pattern is a `bool`. |
| ID-7 | Group `use` statements into three blank-line-separated blocks: **`std`/`core`/`alloc`, external crates, internal (`crate`, `super`, `self`)**, alphabetized within each. `rustfmt`'s `group_imports` does this but is nightly-only, so on stable it is by hand. |
| ID-8 | Use `.not()` when negating a chain you are continuing (`x.is_empty().not().then(...)`) or a `matches!`. Keep prefix `!` in `if`/`while` condition position. |
| ID-9 | Keep every list in `Cargo.toml` alphabetized: dependencies, dev-dependencies, build-dependencies, features, and workspace members. |
| ID-10 | Keep compound generic types (`Vec<_>`, `HashMap<_, _>`) off the LHS: hint on the RHS with turbofish (`.collect::<Vec<_>>()`). Plain scalars, `const`/`static`, signatures, and expressions with no turbofish to hang a hint on are fine annotated. Judgement call, not a ban. |
| ID-11 | Use `_` for any generic parameter the compiler can infer, rather than spelling out the concrete type. |

## Tests (`references/tests.md`)

Open it when writing a new test module or restructuring existing tests; the
table covers single-test edits.

| ID | Rule |
| --- | --- |
| TS-1 | Unit tests go in a `#[cfg(test)] mod tests` in the file under test; integration tests go in the crate's `tests/` directory. |
| TS-2 | Name tests `namespace_input_expectation`: subject, the input condition, then the expected outcome. Drop `namespace` when the module or file name already supplies it. |
| TS-3 | Label a test body with `// Arrange`, `// Act`, `// Assert` once the three phases are not obvious at a glance. Skip the labels when each phase is a single line. Never extend a label with narrative. |
| TS-4 | Bind a value in `Arrange` when it is used more than once, so a call and its assertion cannot drift apart. A literal used exactly once stays inline at its use site: a `const` per literal is noise, not clarity. |
| TS-5 | Give a custom panic message to any `assert*` whose failure reason is not obvious from the surrounding code. The explanation belongs in the message, not in a comment above it. |
| TS-6 | Coalesce tests that share most of their Arrange into one function with a `// Case: <name>` block per case. Hoist only the shared Arrange; each case keeps its own Act and Assert. |
| TS-7 | Once a `namespace_*` group passes about four test functions, move it into its own `mod` or file so the prefix can be dropped (TS-2). Only when you are already restructuring that group, not as a side effect of adding one test. TS-7 splits by subject; TS-6 merges by fixture. |
| TS-8 | Where you would otherwise write `foo`/`bar`/`alice`/`bob`, use a Vox Machina name (`vex`, `vax`, `percy`, `keyleth`, `grog`, `pike`, `scanlan`). Placeholders only. A value modeling a real domain thing (a site, host, region, customer) gets a plausible domain name, and existing codebase convention wins over both. |

## Documentation (`references/documentation.md`)

Open it when writing a `//!` header or a doc comment with structure.

| ID | Rule |
| --- | --- |
| DOC-1 | Use markdown and line breaks liberally in doc comments: a one-line summary, a blank `///` line, then paragraphs, `#` headings, and bullet lists. Indent bullet continuations under the bullet's text, or rustdoc drops them out of the list. It should read well as source and as rendered rustdoc. |
| DOC-2 | Separate a documented field from the undocumented fields that follow it with a blank line, so the doc comment's scope is unambiguous. |
| DOC-3 | Give every crate and non-obvious module a `//!` header at the top of the file, above the imports, covering purpose, public API, and design quirks. |
| DOC-4 | Do not document what the name already says. If a name needs a comment to be understood, rename it. |
| DOC-5 | Use rustdoc's conventional headings for their conventional meanings only: `# Safety` for `unsafe` contracts, `# Panics`, `# Errors`, `# Examples`. Anything else gets a heading of your own wording. |
| DOC-6 | Never write an em dash, in doc comments, comments, or commit messages. Use a period, comma, colon, semicolon, or parentheses instead, whichever the sentence calls for. |
