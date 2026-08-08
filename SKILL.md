---
name: schreibers-rust
description: Ben Schreiber's Rust style guide. Use whenever writing, editing, or reviewing Rust (.rs) code, or when asked to fix nits, clean up style, or make code idiomatic. Covers scoping and blocks, module vs. struct, visibility, idioms, imports, test structure and naming, and doc comments. Not for generated or vendored Rust.
---

# Schreiber's Rust Style

House style on top of `rustfmt` and `clippy` defaults. `references/rules.md` has
worked examples.

**Judgement applies to the fix, not to the trigger.** Every trigger here is
countable: a read count, a per-phase line count, a shared fixture. Count it.
Judgement picks the remedy once a rule fires; it never excuses you from noticing
that it fired.

Apply only to code you are writing or editing. Skip generated, vendored, and
macro-heavy files. When rules collide: correctness, then the file's existing
conventions, then the more specific rule. Reviewing: cite the ID, and raise a
rule only where following it measurably helps the reader.

## The shape rules

Read these against the code you write, not once at the start.

### SC-1: bind at the read count

Count the later scopes that read a local. **Two or more:** top level. **Exactly
one:** inside the thing that reads it, never at top level. The trigger is the
read count, not the line count: a one-line binding read once is the most common
instance, not an exception.

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

Two exceptions: a short prelude to the function's tail expression, and a binding
whose name is what keeps a nearby `.expect()` or error message readable (ID-2).

### SC-2 and TS-6: "block" means `{}`, not a paragraph

**SC-2** fences unrelated units of work inside a function, each of several
lines, under a one-line comment naming it. **TS-6** coalesces tests sharing most
of their Arrange into one function, one case per block, hoisting only the shared
Arrange and leaving each case its own Act and Assert. A naming comment above
loose statements is a divider comment (SC-6), and it lets neighboring units
shadow each other's bindings.

```rust
// DO
// Case: object
{
    let value = parse("{}")?;
    assert_eq!(value, Value::Object(Vec::new()));
}

// DON'T: no braces, so the next `value` silently shadows this one
// Case: object
let value = parse("{}")?;
assert_eq!(value, Value::Object(Vec::new()));
```

**No shared Arrange means no TS-6.** Captioning independent assertions is not
coalescing; it is DOC-4 noise around tests that should have stayed separate.

```rust
// DON'T: nothing is hoisted, and each label restates the line under it
// Case: null
assert_eq!(parse("null"), Ok(Value::Null));

// Case: true
assert_eq!(parse("true"), Ok(Value::Bool(true)));
```

### TS-3: label all three phases when any one exceeds a line

Count the phases separately. Skip the labels only when all three are one-liners;
a one-line Act against a ten-line Assert still gets all three. An Arrange living
outside the body (a module-level `const`, a fixture `fn`) still counts as a
phase: label the line that pulls it in, or where there is none, label the Act
and Assert alone. Never extend a label with narrative.

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
```

## Scoping

| ID | Rule |
| --- | --- |
| SC-1 | Shape rule, above. |
| SC-2 | Shape rule, above. |
| SC-3 | Nest a helper `fn` in its only caller when short. Module scope if unit-tested, doc-tested, or long enough to displace the caller's body. |
| SC-4 | Closure when it captures from the enclosing scope, nested `fn` when everything arrives as parameters. Never re-pass what a closure could capture. |
| SC-5 | Closures go at the top of the function body: the one exception to SC-1. Not when hoisting would extend a mutable capture across code needing the same borrow. |
| SC-6 | No divider comments (`// ------`, `// ==== FOO ====`). Use a `{}` block inside a function, a `mod` or separate file outside one. |

## Modules and structs

| ID | Rule |
| --- | --- |
| MS-1 | A `mod`, not a shared prefix on free functions: `mod util { fn parse }`, not `fn util_parse`. |
| MS-2 | Narrowest visibility that compiles: private → `pub(super)` → `pub(crate)` → `pub`. `pub` only for something used outside the crate. |
| MS-3 | Visibility widened only for tests gets `// Visible for tests.` above it, without repeating the keyword. |
| MS-4 | A `mod` of free functions, not a zero-field struct that only namespaces an `impl`. |
| MS-5 | A struct with a real `impl`, not a `mod`, once the same non-trivial parameters thread through several functions. |
| MS-6 | Split an `impl` or `mod` past ~400 lines along a seam (responsibility, sub-resource, lifecycle stage) leaving each piece cohesive. No honest seam is itself the finding, and outranks the line count. |

## Idioms

| ID | Rule |
| --- | --- |
| ID-1 | An enum, not a `bool` parameter or field. Two bools with an impossible combination are one enum; independent flags stay separate. |
| ID-2 | Outside tests, handle the error path. An unreachable panic is `.expect("<why it cannot fail>")`, never bare `.unwrap()`, never a `// PANIC:` comment. `// SAFETY:` is for `unsafe` only. |
| ID-3 | Implement the standard traits instead of hand-rolling their contract: `Default`, `From`, `TryFrom`, `Display`, `FromStr`, `AsRef`, `Iterator`. `From`, never `Into`; `Into` as a bound is fine. |
| ID-4 | Exit early instead of nesting the happy path: `let ... else`, `if let`, `?`. |
| ID-5 | Name a local after the field it fills, then use field-init shorthand. |
| ID-6 | Pattern matching over field access plus conditionals. `matches!` when all you need is a `bool`. |
| ID-7 | Three blank-line-separated `use` blocks, alphabetized within each: `std`/`core`/`alloc`, external crates, internal (`crate`, `super`, `self`). Stable `rustfmt` will not group them; do it by hand. |
| ID-8 | `.not()` when negating a chain you continue (`x.is_empty().not().then(...)`) or a `matches!`. Prefix `!` stays in `if`/`while` conditions. |
| ID-9 | Every list in `Cargo.toml` alphabetized: dependencies, dev-dependencies, build-dependencies, features, workspace members. |
| ID-10 | Compound generics off the LHS: `.collect::<Vec<_>>()`, not `let names: Vec<_> =`. Scalars, `const`/`static`, signatures, and expressions with no turbofish stay LHS-annotated. |
| ID-11 | `_` for any generic parameter the compiler can infer. |

## Tests

| ID | Rule |
| --- | --- |
| TS-1 | Unit tests in a `#[cfg(test)] mod tests` in the file under test; integration tests in the crate's `tests/`. |
| TS-2 | `namespace_input_expectation`: subject, input condition, expected outcome. Drop `namespace` when the module or file supplies it. No `test_` prefix. |
| TS-3 | Shape rule, above. |
| TS-4 | Bind a value in Arrange when used more than once, so a call and its assertion cannot drift. Used once, it stays inline. |
| TS-5 | Custom panic message on any `assert*` whose failure reason is not obvious. It goes in the message, not a comment above it. |
| TS-6 | Shape rule, above. |
| TS-7 | Three or more tests sharing a `namespace_*` prefix become a sibling `#[cfg(test)] mod namespace_tests`, and the prefix comes off the test names (TS-2). **Count the group as it will stand when you finish, not as it stands mid-write.** TS-6 merges by fixture, TS-7 splits by subject. |
| TS-8 | Test data that models a real domain thing (a site, host, region, tenant, SKU, path) gets a plausible value, not a placeholder. `"us-east-2b"`, not `"foo"`. Codebase convention wins. |

## Documentation

| ID | Rule |
| --- | --- |
| DOC-1 | Markdown and line breaks in doc comments: one-line summary, blank `///`, then paragraphs, `#` headings, bullets. Indent bullet continuations under the bullet's *text* or rustdoc drops them from the list. |
| DOC-2 | Blank line between a documented field and the undocumented fields below it. |
| DOC-3 | `//!` header on every crate and non-obvious module, above the imports: purpose, public API, design quirks. |
| DOC-4 | Do not document what the name says. If a name needs a comment to be understood, rename it. |
| DOC-5 | `# Safety`, `# Panics`, `# Errors`, `# Examples` for their conventional meanings only. Anything else gets your own heading. |
| DOC-6 | Never an em dash, in doc comments, comments, or commit messages. Use a period, comma, colon, semicolon, or parentheses. |

## Before you call it done

Not done until these have been run over the diff. Report what each caught, by
ID, including the ones that caught nothing.

```sh
rg -n '^\s{4,8}let ' src/ tests/   # SC-1: exactly one later reader means push it in
rg -n -A1 '// Case:' src/ tests/   # TS-6: next line is `{`, shared Arrange hoisted above
rg -n -A6 '#\[test\]' src/ tests/  # TS-3: any phase over one line means all three labels
rg -n '\.unwrap\(\)' src/          # ID-2: every hit (tests are exempt, so src/ only)
rg -n 'let \w+: (Vec|HashMap|HashSet|BTreeMap)<' src/ tests/  # ID-10: every hit
rg -o --no-filename '^\s+fn ([a-z]+)_' -r '$1' src/ tests/ | sort | uniq -c | sort -rn | head
                                   # TS-7: a count of three or more needs its own mod
```
