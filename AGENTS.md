# AGENTS.md — Kit Lang Trading Bot Project

> **Purpose:** Concise instructions for AI agents (Cursor, Claude Code, etc.) working on this Kit Lang trading system.  
> **Rule of thumb:** If a directive says ALWAYS or NEVER, violating it is a bug.

---

## 1. Project Overview

This is a quantitative trading system written in **Kit Lang** (functional language compiling via Zig). The codebase prioritizes **financial correctness** over convenience. Invalid states must be unrepresentable at the type level, not caught at runtime.

**Architecture layers (strictly separated):**

| Module | Effect | Responsibility |
|--------|--------|----------------|
| `Trading.Signals` | `Pure` | Signal generation, backtesting logic |
| `Trading.Risk` | `Pure` | Risk checks, margin calculations |
| `Trading.Types` | `Pure` | Refinement types (`Price`, `Quantity`, etc.) |
| `Trading.Execution` | `IO` | Order submission, exchange communication |
| `Trading.Boundaries` | `IO` | Parsing external data (exchange JSON, configs) |

**Golden rule:** `Trading.Signals` and `Trading.Risk` must never import `Trading.Execution` or any IO/network module.

---

## 2. Critical Safety Rules

### 2.1 Refinement Types — ALWAYS Use for Financial Quantities

Define domain invariants as types. Every price, quantity, and percentage in the system must use them.

```kit
type Price       = {p: Decimal | p > 0.0M}
type Quantity    = {q: Int     | q > 0}
type Confidence  = {c: Float   | c >= 0.0 && c <= 1.0}
type FillPercent = {f: Float   | f >= 0.0 && f <= 1.0}
```

### 2.2 Safe Construction — NEVER Use `Type!` in Trading Code

`Type!(val)` panics if the predicate fails. A panic in a live trading bot is unacceptable.

| Operator | Use in trading? | Why |
|----------|----------------|-----|
| `Type!(val)` | **NEVER** | Panics on invalid input |
| `Type?(val)` | **Preferred** | Returns `Option` — safe |
| `Type ?! (val)` | **Preferred** | Returns `Result` — safe |

**Correct pattern at every external boundary:**

```kit
sanitize-price = fn(raw: Decimal) =>
  match Price?(raw)
  | Some p -> Ok p
  | None -> Err f"Rejecting invalid price: ${raw}"
```

### 2.3 `??` (Null Coalesce) — NEVER Use for Market Data

`??` silently substitutes a default when data is missing. In trading, this causes bad trades.

```kit
# CATASTROPHIC — never do this
price = fetch-price(symbol) ?? 0.0M

# CORRECT — propagate the failure
price = fetch-price(symbol) ?!
```

`??` is acceptable **only** for configuration values (`port`, `timeout`) where a default is architecturally safe.

### 2.4 Linear Types — ALWAYS Use for Execution Resources

Prevent double-submission, duplicate cancellations, and resource leaks.

| Annotation | Meaning | Trading use case |
|------------|---------|-----------------|
| `@linear` | Must use exactly once | Order handles, exchange sockets |
| `@affine` | Use at most once | Cancel tokens, idempotency keys |
| `@relevant` | Use at least once | Audit loggers (must flush) |

```kit
# Order can only be submitted once — compiler enforces this
submit-once = fn(order @linear, exchange: Exchange) =>
  ack = Exchange.submit exchange order    # consumes order
  track-ack ack

# Cancel token can be used once or dropped, never twice
cancel-if-needed = fn(token @affine, should-cancel: Bool) =>
  if should-cancel then Exchange.cancel token else :kept
```

### 2.5 Result Handling — NEVER Silently Discard

A `Result` from an order submission, price fetch, or exchange call must always be handled. Propagate with `?!` or match explicitly.

```kit
# CORRECT — propagate
response = Exchange.submit exchange order ?!

# WRONG — silent discard (logic bug where bot thinks order is live)
Exchange.submit exchange order
```

### 2.6 Runtime Guards — NEVER Check What Refinements Already Prove

If a parameter already has a refinement type, a runtime check is redundant and noisy.

```kit
# BAD — Quantity already proves q > 0
fn submit(q: Quantity) =>
  guard true = q > 0 else Err "Quantity must be positive"

# GOOD — trust the type; only check cross-parameter rules
fn submit(q: Quantity, limit: Int) =>
  guard true = q <= limit else Err "Exceeds position limit"
```

Use `guard` **only** for cross-parameter business rules (margin limits, position caps).

### 2.7 Effect Discipline — Keep Backtesting Pure

Kit's effect system is inferred, not enforced. You must enforce separation manually.

- **ALWAYS** suffix pure functions with `-pure`
- **ALWAYS** keep pure functions in `Trading.Signals` or `Trading.Risk`
- **NEVER** import an IO module into a `-pure` module

```kit
# Trading.Signals — no network, no exchange imports
sma-pure = fn(period: Int, prices: List Decimal) => ...

# Trading.Execution — calls pure logic, then performs IO
run-live = fn(bars: List Bar) =>
  signals = sma-pure 20 (close-prices bars)     # pure
  List.each signals (fn(s) => execute s)         # IO
```

---

## 3. Naming Conventions

| Construct | Convention | Example |
|-----------|-----------|---------|
| Variables / functions | `kebab-case` | `user-name`, `max-position` |
| Types / constructors | `PascalCase` | `OrderStatus`, `TradeEvent` |
| Pure functions | `kebab-case` + `-pure` suffix | `sma-pure`, `generate-signals-pure` |
| Booleans | MUST start with `is-` or `has-` and end with `?` | `is-filled?`, `has-position?` |
| Modules | `PascalCase` segments | `module Trading.Core` |
| Test files | `<name>.test.kit` | `risk-tests.test.kit` |

**NEVER use `SCREAMING_SNAKE_CASE` or `camelCase`.**

---

## 4. Error Handling Decision Tree

| Situation | Use |
|-----------|-----|
| Linear pipeline, uniform error handling | `Result.and-then` / `Option.and-then` with pipes |
| Fallible operation inside a function, propagate up | `?!` operator |
| Need a default for **config only** | `??` operator |
| Different branches need different logic | Explicit `match` |
| Extract value or fail early | `guard` macro |

**Preferred pipeline pattern:**

```kit
parse-config(text)
  |> Result.and-then validate
  |> Result.and-then transform
  |> Result.and-then save
```

---

## 5. File Organization

```
src/
├── Trading/
│   ├── Types.kit           # Refinement types, ADTs
│   ├── Signals.kit         # Pure signal generation
│   ├── Risk.kit            # Risk checks, margin, position limits
│   ├── Execution.kit       # Order submission, exchange IO
│   └── Boundaries.kit      # External data parsing (JSON, CSV)
├── main.kit                # Entry point
kit.toml
```

**Import rules:**
- `Trading.Signals` may import `Trading.Types`, `Trading.Risk`
- `Trading.Risk` may import `Trading.Types`
- `Trading.Execution` may import all of the above
- **No upstream module may import `Trading.Execution`**

---

## 6. Build, Check & Test Commands

```bash
# Type check (run before every commit)
kit check --no-spinner src/*.kit

# Run tests
kit test --no-spinner

# Compile to native binary
kit build --no-spinner src/main.kit -o bot

# Format check (CI)
kit format --check src/*.kit

# Development workflow
kit dev
```

**Before committing checklist:**
1. `kit format --check src/*.kit` — clean formatting
2. `kit check src/*.kit` — zero type errors and warnings
3. `kit test` — all tests pass
4. Verify `main` binding exists in entry file

---

## 7. Common Anti-Patterns

### Using `Type!` with market data
```kit
# BAD — panics on invalid exchange payload
price = Price!(exchange-price)

# GOOD — return Err, never panic
price = match Price?(exchange-price)
| Some p -> p
| None -> return Err "Invalid exchange price"
```

### Using `??` for prices or quantities
```kit
# BAD — fabricates a default, causing bad trades
price = fetch-price(symbol) ?? 0.0M

# GOOD — force caller to handle absence
price = fetch-price(symbol) ?!
```

### Runtime-checking refinement invariants
```kit
# BAD — redundant; Quantity already proves q > 0
guard true = q > 0 else Err "Quantity must be positive"

# GOOD — only cross-parameter rules need runtime checks
guard true = q <= max-position else Err "Position limit exceeded"
```

### Forgetting `@linear` on order handles
```kit
# BAD — duplicate execution risk
fn submit(order: OrderType, exchange: Exchange) => ...

# GOOD — compiler enforces single use
fn submit(order @linear, exchange: Exchange) => ...
```

### Deeply nested match for linear flow
```kit
# BAD — verbose nesting
match a
| Ok x -> match b x
  | Ok y -> match c y
    | Ok z -> Ok z
    | Err e -> Err e
  | Err e -> Err e
| Err e -> Err e

# GOOD — pipeline
a |> Result.and-then b |> Result.and-then c
```

---

## 8. Lint & Warning Codes to Respect

| Code | Meaning | Action |
|------|---------|--------|
| `W001` | Missing defer for resource cleanup | Fix |
| `W002` | Binding defined but never used | Remove or prefix with `-` |
| `W014` | Non-exhaustive pattern match | Add missing cases |
| `T013` | Use after move (linear types) | Refactor to avoid reuse |
| `T014` | Linearity violation | Ensure resource consumed correctly |

**Do not ignore warnings.** Suppress `W002` only by prefixing unused bindings with `-`.

---

## 9. Agent FAQ & Reference Index

> **How to use this project:** `AGENTS.md` (this file) is your primary context — it is always loaded. `SYNTAX.md` is a deep reference manual. **Do not load the full `SYNTAX.md` into context.** If you need syntax details not covered below, request the specific line range from `SYNTAX.md` using your file-read tool.

### 9.1 Quick FAQ

| Question | Answer | Deep Dive |
|----------|--------|-----------|
| How do I define a refinement type? | `type Price = {p: Decimal \| p > 0.0M}` | SYNTAX.md 334–374 |
| How do I construct a refinement type safely? | `Type?(val)` → Option, `Type ?! (val)` → Result. **NEVER** `Type!(val)`. | SYNTAX.md 344–358 |
| What's the difference between `??` and `?!`? | `??` provides a default (dangerous for market data). `?!` propagates Err/None early (safe). | SYNTAX.md 726–738 |
| When do I use `@linear` vs `@affine`? | `@linear` = exactly once (order handles). `@affine` = at most once (cancel tokens). | SYNTAX.md 375–420 |
| How do I handle errors in a pipeline? | Use `Result.and-then`: `a \|> Result.and-then b \|> Result.and-then c` | SYNTAX.md 929–934 |
| How do I extract a value or fail early? | Use `guard`: `guard Some v = opt else Err "missing"` | SYNTAX.md 850–858 |
| How are generics written? | Space-separated, **never** angle brackets: `Option Int`, NOT `Option<Int>` | SYNTAX.md 57–64 |
| How do I pattern match on a list? | `\| [h \| t] -> ...` or `\| [first, ..rest] -> ...` | SYNTAX.md 804–812 |
| How do I send a message to an actor? | `actor <- msg` or `Actor.send actor msg` | SYNTAX.md 1029–1031 |
| How do I create a pure function? | Ensure no IO/network/state; suffix with `-pure`; keep in `Trading.Signals` | SYNTAX.md 1377–1395 |
| What numeric types are available? | `Int`, `Float`, `Decimal` (use `M` suffix: `19.99M`), `Int64` (`L`), `BigInt` (`I`) | SYNTAX.md 151–166 |
| How do I import selected names? | `import Module.{name, other as renamed}` | SYNTAX.md 1099–1100 |
| What's the `\|>` vs `\|>>` difference? | `\|>` = thread-first (insert as first arg). `\|>>` = thread-last (insert as last arg). | SYNTAX.md 659–665 |
| How do I check if a value matches a pattern? | `if value is Some _ then ... else ...` | SYNTAX.md 829–836 |
| When should I use `match` vs `guard`? | `match` for full branching. `guard` for "extract or return early" logic. | SYNTAX.md 850–858 |

### 9.2 Section Index for SYNTAX.md

Use this table to request specific line ranges when you need deeper reference material.

| Section | Lines | Keywords |
|---------|-------|----------|
| Refinement Types | 334–374 | `Price`, `Quantity`, `ValidPort`, `?`, `?!`, `!` |
| Linear Types | 375–420 | `@linear`, `@affine`, `@relevant`, order handles |
| Error Handling Operators | 726–738 | `??`, `?!`, `and-then` |
| Pattern Matching | 797–875 | `match`, `guard`, `is`, `as` |
| Trading Bot: Risk Management | 1493–1510 | `guard`, cross-parameter rules |
| Trading Bot: Refinement Types | 1511–1537 | `Price`, `Quantity`, safe construction |
| Trading Bot: Pure Signal Generation | 1528–1537 | `sma-pure`, deterministic logic |
| Trading Bot: Safety Checklist | 1586–1630 | `@linear`, `?`, `-pure`, `??` ban |
| Anti-Patterns | 1470–1571 | `BAD`, `WRONG`, common mistakes |
| Concurrency | 992–1074 | `Channel`, `Actor`, `<-` |
| Modules & Imports | 1078–1125 | `import`, `export`, `module` |

### 9.3 Requesting Deep Reference

**If the FAQ above does not resolve your ambiguity:**

1. Identify the relevant keyword from the FAQ or index.
2. Use your `read_file` tool to fetch **only** the specific line range from `SYNTAX.md`.
3. **Never** read the full `SYNTAX.md` (2,040 lines). Target 20–60 lines at a time.

**Example agent workflow:**

```
Agent: I need to verify the exact syntax for safe refinement construction.
→ Check FAQ §9.1: "How do I construct a refinement type safely?"
→ FAQ points to SYNTAX.md 344–358.
→ read_file("SYNTAX.md", offset=344, limit=15)
→ Get precise syntax: `match Price?(raw) | Some p -> Ok p | None -> Err ...`
```

---

## 10. When in Doubt

1. **Prefer compile-time proof over runtime check.**
2. **Prefer `Err` propagation over panic or default.**
3. **Prefer `@linear` over careful coding.**
4. **Prefer `-pure` suffix + module isolation over documentation.**
5. **Ask before adding new language features or dependencies.**
