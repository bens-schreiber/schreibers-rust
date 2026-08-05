# Scoping

Rules `SC-1` … `SC-7`. The through-line: **a name should be visible only where it
is used, and the shape of the code should say where that is.**

## SC-1: Bind in the smallest block that contains the full lifetime

If a local exists only to compute one later value, wrap its computation in a
block expression rather than letting it leak to the function's top level.

Why: top-level bindings that are dead after one use make it impossible to tell,
at a glance, which names still matter for the rest of the function.

```rust
// DO: a and b cannot be confused for state the rest of the function reads
let total = {
    let base = order.subtotal();
    let tax = base * TAX_RATE;
    base + tax
};

// DON'T: base and tax stay in scope for the whole function despite being dead
let base = order.subtotal();
let tax = base * TAX_RATE;
let total = base + tax;
```

## SC-2: Top level is for values read from multiple scopes

The exception to SC-1. A local earns its place at the function's top level when
two or more later scopes genuinely read it. Note that the *computation* still
gets a block; only the result escapes.

```rust
// DO: total is read twice, far apart, so it belongs at the top level
let total = {
    let base = order.subtotal();
    let tax = base * TAX_RATE;
    base + tax
};

// ...many lines later...
let rounded = total.round();
log_receipt(total);
```

Read from exactly one later scope? That is SC-1: push it in.

## SC-3: Bare blocks separate unrelated work

Use a bare `{}` block, even one that produces no value, to fence off a unit of
work from the code around it. Each block gets a one-line comment naming what it
does.

Why: a blank line does not tell the reader "the locals above are irrelevant
below." A block does, and the compiler enforces it.

```rust
// DO
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

Both blocks bind `total` and neither leaks. Without the blocks you would need
two names for the same concept.

## SC-4: Nest a helper at its only call site

If `bar` is only ever called from inside `foo`, it lives inside `foo`.

```rust
// DO: parse_row exists for load_manifest and nowhere else
fn load_manifest(src: &str) -> Result<Vec<Row>, ParseError> {
    fn parse_row(line: &str) -> Result<Row, ParseError> {
        // ...
    }

    src.lines().map(parse_row).collect()
}

// DON'T: module scope implies "several callers" and there is only one
fn parse_row(line: &str) -> Result<Row, ParseError> {
    // ...
}

fn load_manifest(src: &str) -> Result<Vec<Row>, ParseError> {
    src.lines().map(parse_row).collect()
}
```

## SC-5: Closure when it captures, nested `fn` when it does not

Reach for a closure when the logic wants values from the enclosing scope. Reach
for a nested `fn` when everything arrives as explicit parameters: a nested `fn`
cannot capture, which is exactly the guarantee you want there.

Never re-pass a value as a closure parameter that the closure could have
captured; the parameter list is then pure noise.

```rust
// DO: the closure captures the fixtures it needs
fn score(weights: &Weights, bonus: u32) -> u32 {
    let weighted = |raw: u32| raw * weights.factor + bonus;

    weighted(base_score())
}

// DON'T: weights and bonus are in scope; threading them through is ceremony
fn score(weights: &Weights, bonus: u32) -> u32 {
    let weighted = |raw: u32, weights: &Weights, bonus: u32| raw * weights.factor + bonus;

    weighted(base_score(), weights, bonus)
}
```

## SC-6: Closures are declared at the top of the body

The one deliberate exception to SC-1: a closure goes at the top of the
function body, before the code that uses it, even when SC-1 would otherwise
push it down to just above its first use. The reader should meet the
vocabulary before the prose.

```rust
// DO
fn render(cfg: &Config, rows: &[Row]) -> String {
    let indent = |s: &str| format!("{}{s}", " ".repeat(cfg.indent));

    let header = indent("id, name");
    let body = rows.iter().map(|r| indent(&r.to_string()));

    // ...
}
```

## SC-7: No divider comments

A divider comment is a request for structure the language already offers.

Inside a function, that structure is a `{}` block (SC-3). Outside one, it is a
`mod`, a separate file, or a separate crate.

```rust
// DO
mod parsing {
    // ...
}

mod rendering {
    // ...
    fn render() {
        // Build the header.
        {
            // ...
        }

        // Build the body.
        {
            // ...
        }
    }
}
```

```rust
// DON'T
// ---------- PARSING ----------

// ...

// ========== RENDERING ==========

// ...
```
