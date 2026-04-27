# Kit Lang — Complete Syntax & LLM Source of Truth

> **Version:** 2026.4.24  
> **Purpose:** Authoritative reference for generating correct, idiomatic Kit Lang code.  
> **Audience:** LLMs and developers. Read this before writing Kit.  
> **Companion:** See `GRAMMAR.md` for the formal EBNF specification.

---

## Table of Contents

1. [Critical Rules for LLMs (Read First)](#1-critical-rules-for-llms-read-first)
2. [Lexical Structure](#2-lexical-structure)
3. [Program Structure](#3-program-structure)
4. [Type System](#4-type-system)
5. [Type Definitions](#5-type-definitions)
6. [Bindings](#6-bindings)
7. [Expressions](#7-expressions)
8. [Pattern Matching](#8-pattern-matching)
9. [Control Flow](#9-control-flow)
10. [Error Handling](#10-error-handling)
11. [Concurrency](#11-concurrency)
12. [Modules & Imports](#12-modules--imports)
13. [Traits](#13-traits)
14. [FFI](#14-ffi)
15. [Macros](#15-macros)
16. [Testing](#16-testing)
17. [Operator Reference](#17-operator-reference)
18. [Effect System](#18-effect-system)
19. [Best Practices & Conventions](#19-best-practices--conventions)
20. [Anti-Patterns](#20-anti-patterns)
21. [Common Idioms by Domain](#21-common-idioms-by-domain)
22. [Lint & Warnings](#22-lint--warnings)
23. [Tooling & Workflow](#23-tooling--workflow)
24. [Appendices](#24-appendices)

---

## 1. Critical Rules for LLMs (Read First)

### 1.1 Naming Conventions (NON-NEGOTIABLE)

| Construct | Convention | Example |
|-----------|-----------|---------|
| Variables | `kebab-case` | `user-name`, `max-position` |
| Functions | `kebab-case` | `calculate-pnl`, `is-valid?` |
| Types | `PascalCase` | `OrderStatus`, `TradeEvent` |
| Constructors | `PascalCase` | `Buy`, `Limit`, `PartiallyFilled` |
| Modules | `PascalCase` segments | `module Trading.Core`, `module Trading.Risk` |
| Booleans | MUST start with `is-` or `has-` and end with `?` | `is-filled?`, `has-position?` |
| Test files | `<name>.test.kit` | `order-tests.test.kit` |

**NEVER use `SCREAMING_SNAKE_CASE` or `camelCase` for identifiers.**

### 1.2 Type Arguments Are Space-Separated

```kit
Option Int              # CORRECT
Result String Error     # CORRECT
List (List Int)         # CORRECT

Option<Int>             # WRONG — angle brackets are NOT Kit syntax
Result<String, Error>   # WRONG
```

### 1.3 Zero-Arity Functions

Zero-arity functions can omit parentheses in definition AND are **auto-invoked** when referenced:

```kit
now = fn => Time.now          # definition without parens
x = now                       # CALLS the function; x is the result, not the function

# To pass a zero-arity function as a value, you must wrap it:
delayed = fn => now           # returns a function that calls now when invoked
```

### 1.4 Every Kit File Needs a `main` Binding to Execute

```kit
main = fn =>
  println "Hello, World!"
```

Without `main`, the file type-checks but produces no output when run.

### 1.5 Indentation Is Significant

Multi-line constructs (lambda bodies, `if`/`else` branches, `match` arms, test blocks) use indentation. Do not mix tabs and spaces.

### 1.6 Negative Numbers in Prefix Calls

When using Haskell-style function application, negative numbers must be parenthesized:

```kit
func (-1)     # CORRECT
func -1       # WRONG — parsed as func minus 1
```

### 1.7 `print` vs `println`

Use `println` for line output. `print` does not append a newline.

### 1.8 Record Spread Syntax

```kit
old = {name: "Alice", age: 30}
new = {old | age: 31}         # CORRECT — update age
new = {old | email: "a@b"}    # CORRECT — add field
```

---

## 2. Lexical Structure

### 2.1 Comments

```kit
# Single-line comment

## Doc comment — associated with the next declaration

x = 42  # inline comment
```

### 2.2 Identifiers

```ebnf
LOWER_IDENT = (a-z | "_") {a-z | A-Z | 0-9 | "_" | "-"}
UPPER_IDENT = A-Z {a-z | A-Z | 0-9 | "_" | "-"}
```

Kit idiomatically uses **kebab-case** (hyphens allowed).

```kit
my-function     # valid function name
_my-var         # valid (underscore prefix)
MyType          # valid type/constructor name
```

### 2.3 Reserved Words

```
as        defer     else      export    extend    extern-c  extern-zig
false     fn        for       from      guard     if        import
in        is        link      macro     match     module    requires
sql       test      then      trait     true      type      when
where     with
```

### 2.4 Numeric Literals

| Literal | Type | Example |
|---------|------|---------|
| `42` | `Int` (32-bit signed) | `1_000_000` |
| `0xFF` | `Int` (hex) | `0xFF0000` |
| `0b1010` | `Int` (binary) | `0b1111_0000` |
| `0o755` | `Int` (octal) | `0o644` |
| `42L` | `Int64` | `9000000000L` |
| `42u` | `UInt32` | `255u` |
| `42ul` | `UInt64` | `1000ul` |
| `999I` | `BigInt` | `123456789012345I` |
| `3.14` | `Float` (64-bit) | `19.99` |
| `3.14f` | `Float32` | `3.14f` |
| `3.14M` / `3.14m` | `Decimal` | `19.99M` |

### 2.5 String Literals

```kit
# Double-quoted with interpolation
name = "Kit"
greeting = "Hello, ${name}!"

# With format specifiers
"Value: ${n:.2f}"

# Multiline heredoc — indentation stripped
html = <<~HTML
  <div>
    <h1>${title}</h1>
  </div>
HTML
```

### 2.6 Keyword Literals

```kit
:ok
:error
:pending
:inc
```

Syntax: `:` followed by an identifier. Commonly used as actor messages and lightweight enums.

### 2.7 Boolean & Unit

```kit
true
false
()        # unit value
```

---

## 3. Program Structure

A Kit file is a sequence of top-level declarations evaluated in order:

```ebnf
declaration = type_def
            | import_stmt
            | export_stmt
            | module_decl
            | test_block
            | extern_decl
            | trait_def
            | trait_impl
            | binding
```

**Recommended file order:**

```kit
## Module documentation

module MyProject.SubModule

import Other.Module as Alias
import Other.Module.{name, other as renamed}
import "./local-file.kit"

# Types
type Color = Red | Green | Blue

# Traits
trait Eq a
  eq : a -> a -> Bool

# FFI
extern-c printf(fmt: String, ...) -> Int from "libc" link "libc.so"

# Bindings
greet = fn(name) => "Hello, ${name}!"

# Tests
test "greeting works"
  assert-eq! (greet "Kit") "Hello, Kit!"
```

---

## 4. Type System

Kit uses **Hindley-Milner type inference**. Types are inferred automatically; annotations are optional but recommended at module boundaries.

### 4.1 Primitive Types

| Type | Description | Example |
|------|-------------|---------|
| `Int` | 32-bit signed integer | `42` |
| `Float` | 64-bit IEEE-754 float | `3.14` |
| `String` | UTF-8 text | `"hello"` |
| `Bool` | Boolean | `true` |
| `Unit` | Singleton type | `()` |

### 4.2 Function Types

```kit
# Int -> Int
increment = fn(x) => x + 1

# Int -> Int -> Int (curried)
add = fn(a, b) => a + b

# Explicit type synonym
type BinOp = Int -> Int -> Int

# Higher-order
map : (a -> b) -> List a -> List b
```

Functions are **curried** automatically. `fn(a, b) => ...` is sugar for `fn(a) => fn(b) => ...`.

### 4.3 Generic Application

**Space-separated**, never angle brackets:

```kit
Option Int
Result String Error
List (List Int)
Map String Int
```

### 4.4 Tuple Types

```kit
# (String, Int)
person : (String, Int) = ("Alice", 30)

# (Int, Int, Int)
rgb : (Int, Int, Int) = (255, 128, 0)
```

### 4.5 Record Types

**Closed (default):** requires exactly the specified fields.

```kit
# {name: String, age: Int}
user : {name: String, age: Int} = {name: "Alice", age: 30}
```

**Open (opt-in with `...`):** accepts records with at least the specified fields.

```kit
# Accepts any record with a `name` field
greet = fn(person: {name: String, ...}) =>
  "Hello, ${person.name}"

# Empty open record — accepts any record
accept-any = fn(r: {...}) => "got it"
```

### 4.6 List Types

```kit
numbers : List Int = [1, 2, 3]
names : List String = ["Alice", "Bob"]
```

### 4.7 Refinement Types

Refinement types constrain values beyond what the base type expresses. Syntax follows Liquid Haskell: `{binding: BaseType | predicate}`.

```kit
# Type declarations
type PositiveInt = {n: Int | n > 0}
type NonZero = {n: Int | n != 0}
type Percentage = {p: Float | p >= 0.0 && p <= 100.0}
type NonEmptyList a = {xs: List a | not(empty?(xs))}

# Assert construction (panics if predicate fails)
port = ValidPort!(8080)

# Safe construction — returns Option
port = ValidPort?(user_input)
match port
| Some p -> use-port(p)
| None -> println "Invalid port"

# Safe construction — returns Result
port = ValidPort ?! (user_input)
match port
| Ok p -> use-port(p)
| Err msg -> println "Error: ${msg}"

# In function signatures
divide = fn(a: Int, b: NonZero) => a / b
```

**Trading bot example:**

```kit
type Price = {p: Decimal | p > 0.0M}
type Quantity = {q: Int | q > 0}
type Confidence = {c: Float | c >= 0.0 && c <= 1.0}

# Construction at boundaries
price = Price!(50.0M)           # panic on invalid input
qty   = Quantity?(userInput)    # safe Option
```

### 4.8 Linear Types

Four levels of linearity tracked at compile time:

```kit
# @linear — must use exactly once (prevents use-after-free)
process-file = fn(path) =>
  handle @linear = File.open path
  content = File.read-all handle
  File.close handle              # consumes the handle
  content

# @affine — can use at most once (optional cleanup)
resource @affine = allocate-memory()
if should-process path then
  result = process resource
  free resource                  # optional cleanup
  Some result
else
  None                           # can skip cleanup

# @relevant — must use at least once (must flush)
log-batch = fn(logger @relevant, messages) =>
  List.each messages (fn(msg) => Logger.log logger msg)
  Logger.flush logger            # must use at least once

# @unrestricted — no constraints (default)

## Trading-specific linear type usage

# @linear — order handles must be submitted exactly once
# Prevents double-submission, which can cause duplicate executions
submit-once = fn(order @linear, exchange: Exchange) =>
  ack = Exchange.submit exchange order    # consumes order
  track-ack ack

# @affine — cancel tokens may be used at most once
# Prevents duplicate cancellations of the same order
cancel-if-needed = fn(token @affine, should-cancel: Bool) =>
  if should-cancel then
    Exchange.cancel token                   # consumes token
    :cancelled
  else
    :kept                                    # token dropped, no double-cancel

# @relevant — audit loggers must flush at least once before shutdown
flush-audit = fn(logger @relevant, events: List TradeEvent) =>
  List.each events (fn(e) => Audit.log logger e)
  Audit.flush logger                        # mandatory flush
```

---

## 5. Type Definitions

### 5.1 Sum Types (ADTs)

```kit
# Simple enumeration
type Color = Red | Green | Blue

# Variants with data
type Shape
  = Circle Float
  | Rectangle Float Float
  | Point

# Variants with named fields
type Order =
  | Market(side: Side, quantity: Quantity)
  | Limit(side: Side, price: Price, quantity: Quantity)
  | Stop(side: Side, trigger: Price, quantity: Quantity)

# Recursive type
type Tree a = Leaf a | Branch (Tree a) (Tree a)
```

### 5.2 Record Types (via Constructor)

Kit defines record-like types through **constructors with named fields**, not inline record syntax:

```kit
type Person = Person(name: String, age: Int)

type Address = Address(
  street: String,
  city: String,
  zip: String
)
```

Record **literals** (values) use `{...}` syntax, but record **types** must be declared via constructors.

### 5.3 Attributes on Constructor Fields

Attributes provide metadata on constructor fields using `@` prefix. Names can be hyphenated and take arguments.

```kit
type User = User(
  id @primary-key @auto-increment,
  name,
  email @unique,
  price @default(0.0)
)
```

Common uses:
- **ORM:** `@primary-key`, `@auto-increment`, `@unique`, `@index`, `@fkey(Table, col)`
- **Serialization:** `@json("fieldName")`, `@json-ignore`, `@json("field", omitempty)`
- **Validation:** `@min(0)`, `@max(100)`, `@pattern("[a-z]+")`
- **Docs:** `@doc("description")`, `@deprecated`

### 5.4 Built-in ADTs

These are pre-registered; do not redefine them:

```kit
type Option a = Some a | None
type Result a e = Ok a | Err e
```

Also pre-registered constructors for patterns:
```
Ok    Err    Some    None
NoBackoff    Constant    Linear    Exponential
NoJitter    FullJitter    EqualJitter    ProportionalJitter
```

---

## 6. Bindings

Bindings are **immutable** by default. A binding is a name, an optional type annotation, and a value.

### 6.1 Value Bindings

```kit
# Simple binding
x = 42

# With type annotation
name : String = "Kit"

# Pattern binding — tuple destructuring
(x, y) = (10, 20)

# Pattern binding — record destructuring
{name, age} = person

# Record destructuring with rename
{name: user-name, age: user-age} = person

# Partial / nested destructuring
{port} = config
{info: {name, email}} = user
((a, b), (c, d)) = nested-tuples

# Wildcards to ignore values
{keep, ignore: _} = data
(_, important, _) = triple
```

### 6.2 Function Bindings

```kit
# Anonymous function
square = fn(x) => x * x

# Multi-parameter (curried)
add = fn(a, b) => a + b

# Multi-line body (indentation significant)
factorial = fn(n) =>
  if n <= 1 then
    1
  else
    n * factorial (n - 1)

# Zero-arity — parentheses optional
get-time = fn => now()
get-time = fn() => now()   # equivalent

# Lazy-accepting parameter (~ sigil)
maybe-compute = fn(~value, should-force?) =>
  if should-force? then force(value) else 0
```

### 6.3 Qualified Bindings

```kit
# Bind a value to a type-qualified name
Person.name = "default"
```

### 6.4 Contracts

```kit
@pre(n >= 0, "n must be non-negative")
@post(result >= 0)
factorial = fn(n) =>
  if n <= 1 then 1 else n * factorial(n - 1)
```

Contracts are checked at runtime. The optional string argument provides a custom error message.

**Trading-specific contracts:**

```kit
@pre(portfolio.available-margin >= 0.0M)
@post(result >= 0.0M)
@post(result <= portfolio.margin-limit)
calculate-margin = fn(order: OrderType, portfolio: Portfolio) =>
  order.price * order.quantity * margin-rate

@pre(total-risk >= 0.0M)
@pre(max-drawdown >= 0.0M)
@post(result == (total-risk <= max-drawdown))
is-within-risk-limits = fn(total-risk: Decimal, max-drawdown: Decimal) =>
  total-risk <= max-drawdown
```

### 6.5 Linearity Annotations

```kit
# Annotate binding with linearity
handle @linear = File.open "file.txt"
resource @affine = allocate()
logger @relevant = create-logger()
```

---

## 7. Expressions

### 7.1 Precedence Table (lowest to highest)

| Prec | Operator | Assoc | Description |
|------|----------|-------|-------------|
| 1 | `defer` | prefix | Deferred execution |
| 2 | `??` | right | Null coalesce |
| 3 | `\|>`, `\|>>` | left | Pipe operators |
| 4 | `<-` | — | Actor send |
| 5 | `\|\|`, `or` | left | Logical OR |
| 6 | `&&`, `and` | left | Logical AND |
| 7 | `is`, `as` | — | Pattern test/extract |
| 8 | `==`, `!=` | left | Equality |
| 9 | `<`, `<=`, `>`, `>=` | left | Comparison |
| 10 | `++` | left | String concatenation |
| 11 | `::`, `@` | left | List cons/append |
| 12 | `+`, `-` | left | Addition/subtraction |
| 13 | `*`, `/`, `%` | left | Multiplication/etc |
| 14 | `-`, `!` | right | Unary negation/not |
| 15 | calls, `.`, `?!` | left | Application/access |
| 16 | literals | — | Primary expressions |

### 7.2 `if` Expressions

```kit
absolute = fn(n) =>
  if n < 0 then -n else n

grade = fn(score) =>
  if score >= 90 then "A"
  else if score >= 80 then "B"
  else if score >= 70 then "C"
  else "F"
```

`if` is an **expression** — it always produces a value. Both branches must have the same type. `else` is mandatory.

### 7.3 `match` Expressions

```kit
match value
| pattern1 -> expression1
| pattern2 -> expression2
| _ -> default-expression

# Match arms can also use => for multi-line bodies
match value
| Some x =>
    doubled = x * 2
    doubled
| None -> 0
```

Match arms are checked **top-to-bottom**. Kit requires **exhaustiveness** — every possible case must be handled. The compiler/linter warns for non-exhaustive matches (W014).

```kit
describe = fn(n) =>
  match n
  | 0 -> "zero"
  | 1 -> "one"
  | x -> "other: ${x}"

sum-list = fn(lst) =>
  match lst
  | [] -> 0
  | [x | rest] -> x + sum-list rest
```

### 7.4 `for` Expressions

```kit
# Basic iteration
for x in [1, 2, 3] => println x

# With pattern destructuring
for (k, v) in map-entries(m) => println "${k}: ${v}"

# Multi-line body
for item in items =>
  processed = transform(item)
  save(processed)
```

`for` desugars to `each`.

### 7.5 Pipes

```kit
# |> — thread-first (inserts as first argument)
result = data |> transform |> output
# equivalent to: output (transform data)

# |>> — thread-last (inserts as last argument)
result = list |>> map double |>> filter even?
# equivalent to: filter even? (map double list)
```

### 7.6 List Literals

```kit
numbers = [1, 2, 3, 4, 5]
empty = []

# Cons
with-zero = cons 0 numbers

# Prepend with ::
new-list = 0 :: numbers

# Literal with tail
more = [1, 2 | rest]   # equivalent to cons 1 (cons 2 rest)

# Rest pattern extraction
match args
| [first, second, ..rest] -> handle(first, second, rest)
| [..all] -> handle-all(all)
```

### 7.7 Set Literals

```kit
unique = #{1, 2, 3, 2, 1}   # => #{1, 2, 3}
```

### 7.8 Record Literals

```kit
# Full syntax
person = {name: "Alice", age: 30}

# Shorthand (field name becomes variable name)
name = "Alice"
age = 30
person = {name, age}

# Spread / update
older = {person | age: 31}
with-email = {person | email: "alice@example.com"}
```

### 7.9 Field Access

```kit
person.name                     # direct access
Record.get "name" person        # dynamic access (returns Option)

# Field accessor shorthand — .field is fn(x) => x.field
get-name = .name
users |>> map (.name)

# Equivalent to:
users |>> map (fn(u) => u.name)
```

### 7.10 Error Handling Operators

```kit
# ?? — null coalesce (Option or Result)
value = maybe-value ?? default
name = get-user-name(id) ?? "Anonymous"
port = parse-int(input) ?? 8080

# ?! — error propagation (unwrap Ok/Some or return early with Err/None)
process-file = fn(path) =>
  content = File.read(path) ?!        # Returns Err early if read fails
  parsed = JSON.parse(content) ?!     # Returns Err early if parse fails
  Ok (transform parsed)
```

The `?!` operator is parsed at call-expression level (same precedence as `.` and function application).

### 7.11 `defer`

```kit
defer cleanup()
```

Defers execution of the expression. Useful for resource cleanup.

### 7.12 Function Application

Kit supports two calling conventions:

```kit
# C-style call
add(3, 7)

# Haskell-style prefix call (space-separated)
add 3 7

# Mixed — data-first pipe
result = 10 |> add 5   # add 10 5
```

### 7.13 `sql` Expressions

```kit
# Simple
sql {SELECT * FROM users}

# With connection
sql db {SELECT * FROM users WHERE id = ${user_id}}

# With type casting
sql db as User {SELECT * FROM users WHERE id = 1}
```

SQL blocks support string interpolation with `${expr}`. The optional `as Type` enables automatic result type casting.

### 7.14 Interpolated Strings & Heredocs

```kit
# Regular interpolation
"Hello ${name}!"
"Value: ${n:x}"           # with format specifier

# Heredoc — squiggly syntax strips common leading indentation
html = <<~HTML
  <div>
    <h1>${title}</h1>
    <p>${content}</p>
  </div>
HTML
```

---

## 8. Pattern Matching

### 8.1 Patterns

| Pattern | Matches | Example |
|---------|---------|---------|
| `_` | Any value, no binding | `\| _ -> "catch-all"` |
| *literal* | Exact value | `\| 0 -> "zero"` |
| *identifier* | Any value, binds to name | `\| x -> x * 2` |
| `Ctor args` | Constructor with payload | `\| Some x -> x` |
| `Ctor(a, b)` | Constructor with tuple payload | `\| Point(x, y) -> ...` |
| `Ctor{a, b}` | Constructor with record payload | `\| User{name, email} -> ...` |
| `(a, b)` | Tuple destructuring | `\| (x, y) -> x + y` |
| `[a \| b]` | List cons | `\| [h \| t] -> h` |
| `[a, b, ..rest]` | List with rest | `\| [first, second, ..rest] -> ...` |
| `{a, b}` | Record destructuring | `\| {name, age} -> ...` |
| `A \| B` | Or-pattern | `\| Ok x \| Some x -> x` |

### 8.2 Guards

```kit
classify = fn(n) =>
  match n
  | x if x < 0 -> "negative"
  | 0 -> "zero"
  | x when x > 0 -> "positive"
```

Guards may use `if` or `when` before the condition.

### 8.3 Match Macros

#### `is` — pattern test (returns `Bool`)

```kit
if status is Passed then "ok" else "fail"

# Expands to:
# match status | Passed -> true | _ -> false
```

#### `as` — extract with default

```kit
name = user as Some {name, ..} -> name else "Anonymous"

# Expands to:
# match user | Some {name, ..} -> name | _ -> "Anonymous"
```

#### `guard` — bind or return early

```kit
process = fn(id) =>
  guard Some user = get-user(id) else Err "Not found"
  guard Ok validated = validate(user) else Err "Invalid"
  Ok (transform validated)
```

**Syntax:** `guard pattern = expression else return-expression`

The `guard` macro is the preferred way to express "extract or fail early" logic.

### 8.4 Active Patterns

```kit
# Define
pattern positive = fn(n) =>
  if n > 0 then Some n else None

pattern even = fn(n) =>
  if n % 2 == 0 then Some n else None

# Use
result = match 42
| positive x -> x * 2
| _ -> 0
```

---

## 9. Control Flow

### 9.1 Conditionals

```kit
if condition then expr else expr
```

Always requires `else`. Every branch must produce the same type.

### 9.2 Pattern Matching

See [Pattern Matching](#8-pattern-matching).

### 9.3 Iteration

```kit
# for loop (desugars to each)
for x in collection => body

# fold for accumulation
total = fold (fn(acc, x) => acc + x) 0 numbers
```

### 9.4 Deferred Execution

```kit
defer cleanup()
```

---

## 10. Error Handling

Kit provides **four** mechanisms for error handling. Choosing the right one is critical for readable code.

### 10.1 Decision Tree

| Situation | Use |
|-----------|-----|
| Linear pipeline, uniform error handling | `Result.and-then` / `Option.and-then` with pipes |
| Fallible operation in a function, propagate up | `?!` operator |
| Need a default value if missing | `??` operator |
| Different branches need different logic | Explicit `match` |
| Need values from outer scopes in inner branches | Nested `match` (not pipes) |

### 10.2 Pipeline with `and-then`

**Preferred for linear sequences where each step transforms the previous result:**

```kit
# Good: linear pipeline with uniform error handling
parse-config(text)
  |> Result.and-then validate
  |> Result.and-then transform
  |> Result.and-then save
```

### 10.3 Error Propagation with `?!`

**Preferred when errors should propagate up to the caller:**

```kit
read-and-parse = fn(path) =>
  text = File.read(path) ?!
  parsed = JSON.parse(text) ?!
  Ok (transform parsed)
```

### 10.4 Null Coalesce with `??`

**Preferred when you have a sensible default:**

```kit
name = lookup-user(id) ?? "Anonymous"
port = config.port ?? 8080
```

### 10.5 Explicit Match

**Preferred when different error cases need different handling:**

```kit
# Good: nested match when outer values are needed in inner branches
match get-user(id)
| Some user ->
    match validate(user)
    | Ok validated -> "User ${user.name} is valid: ${validated.status}"
    | Err e -> "User ${user.name} failed: ${e}"
| None -> "Not found"
```

### 10.6 Anti-Pattern: Deeply Nested Match for Linear Flow

```kit
# BAD
match a
| Ok x ->
    match b x
    | Ok y ->
        match c y
        | Ok z -> Ok z
        | Err e -> Err e
    | Err e -> Err e
| Err e -> Err e

# GOOD
a |> Result.and-then b |> Result.and-then c
```

### 10.7 Trading Safety: Never Use `??` for Market Data

`??` (null coalesce) provides a default when a value is missing. In trading, a missing price or quantity must **never** be silently replaced with a default — this causes bad trades.

```kit
# CATASTROPHIC — do not use ?? for market data
price = fetch-price(symbol) ?? 0.0M          # Could trigger a market order at $0
qty = fetch-position(symbol) ?? 100          # Could open an unwanted 100-lot position

# CORRECT — propagate the error, forcing the caller to handle absence
price = fetch-price(symbol) ?!
qty = fetch-position(symbol) ?!

# CORRECT — if a default is truly acceptable, make it explicit via match
price = match fetch-price(symbol)
| Some p -> p
| None ->
    # Log, alert, and halt — never fabricate a price
    Logger.error "Price missing for ${symbol}"
    halt-strategy()
```

**Rule:** `??` is acceptable only for configuration values (ports, timeouts) where a sensible default is architecturally safe. Never use `??` for prices, quantities, positions, or market state.

---

## 11. Concurrency

Kit provides built-in concurrency primitives. No external packages needed.

### 11.1 Channels

```kit
# Create channels (SPSC, MPSC, or unbounded)
ch = Channel.spsc 100 |> Result.unwrap

# Send and receive
Channel.send ch "hello"
msg = Channel.recv ch |> Result.unwrap

# Non-blocking variants
if Channel.try-send ch value then println "sent"
match Channel.try-recv ch
| Some v -> use v
| None -> println "empty"

# With timeout
match Channel.recv-timeout ch 1000
| Some v -> use v
| None -> println "timed out"
```

### 11.2 Actors

```kit
# Create actor with handler and initial state
counter = Actor.new
  fn(msg, state) =>
    match msg
    | :inc -> state + 1
    | :dec -> state - 1
  0

# Send messages (two equivalent forms)
Actor.send counter :inc
counter <- :inc          # using <- operator

# Access state
println (Actor.state counter)
Actor.stop counter

# Note: Actor.spawn is also available and functionally equivalent
```

### 11.3 Supervisors

```kit
# Create supervisor with restart strategy
sup = Supervisor.start :isolate [
  {name: "worker1", handler: handler, state: 0},
  {name: "worker2", handler: handler, state: 0}
]

# Strategies: :isolate, :all, :cascade
Supervisor.stop sup
```

### 11.4 Parallel

Synchronous-looking parallel operations (no async/await):

```kit
# Run on another thread, block until done
result = Parallel.run (fn => expensive-computation())

# With timeout
match Parallel.run-timeout 5000 (fn => slow-op())
| Some (Ok v) -> use v
| Some (Err e) -> handle-error e
| None -> handle-timeout

# Parallel map
results = Parallel.map urls (fn(url) => Http.get url)

# Run multiple tasks
results = Parallel.all [fn => task1, fn => task2]
first = Parallel.first [fn => fast, fn => slow]   # First to complete
any = Parallel.any [fn => might-fail, fn => backup] # First success
```

---

## 12. Modules & Imports

### 12.1 Module Declaration

Every file should start with a module declaration. Test files must have one.

```kit
module MyProject.Core
module MyProject.Utils.Helpers
module Tests.Core              # for test files in kit-tests/core/
module Tests.Encoding.CsvDialect  # for nested test dirs
```

### 12.2 Imports

```kit
# Import entire module (PascalCase module name)
import String
import Data.Map as M

# Import selected names
import Data.Map.{get, insert, delete as remove}

# Wildcard import
import Data.Map.*

# Import from string path
import "./local-file.kit"

# Aliased import
import Encoding.JSON as J
```

### 12.3 Exports

`export` must be followed by a **complete binding** or **type definition**:

```kit
# Export a binding
export greet = fn(name) => "Hello, ${name}!"

# Export a type
export type Color = Red | Green | Blue
```

Only exported bindings and types are visible to importing modules.

**Note:** `export` does not take a bare identifier; provide the full definition.

---

## 13. Traits

### 13.1 Definition

```kit
trait Eq a
  eq : a -> a -> Bool
  ne = fn(a, b) => ! (eq a b)
```

### 13.2 Implementation

```kit
extend Int with Eq
  eq = fn(a, b) => a == b
```

### 13.3 With Constraints

```kit
trait Ord a requires Eq a
  lt : a -> a -> Bool
  gt : a -> a -> Bool
```

### 13.4 Traits for Strategy Abstraction

```kit
trait Strategy s
  generate : s -> List Tick -> List Signal
  name : s -> String

extend MovingAverageCrossover with Strategy
  generate = fn(s, ticks) => ...
  name = fn(_) => "MA-Cross"
```

---

## 14. FFI

Kit can bind to C and Zig libraries.

```kit
# C function binding
extern-c printf(fmt: String, ...) -> Int from "libc" link "libc.so"

# Zig function binding
extern-zig my-zig-fn(x: Int) -> Int from "mylib" link "libmylib.a"
```

**Best practice:** Wrap FFI calls in Kit functions with proper error handling. Do not expose raw FFI signatures to business logic.

---

## 15. Macros

Macros expand at compile time via quasiquotation.

### 15.1 Definition

```kit
macro double x = `($x + $x)
macro square x = `($x * $x)
macro unless cond then-val else-val =
  `(if (not $cond) then $then-val else $else-val)
```

### 15.2 Quasiquote Syntax

| Syntax | Meaning |
|--------|---------|
| `` `(...) `` | Quasiquote — create template |
| `$name` | Unquote — substitute parameter |
| `$(expr)` | Unquote expression — evaluate and substitute |

### 15.3 Usage

```kit
result = double(5)         # expands to (5 + 5)
squared = square(4)        # expands to (4 * 4)
msg = unless((x > 0), "negative", "positive")
```

### 15.4 When to Use Macros

- **Use macros** for repetitive syntactic patterns that cannot be captured by functions.
- **Do NOT use macros** when a regular function suffices. Macros bypass type checking at definition site.

---

## 16. Testing

### 16.1 Test Blocks

```kit
test "addition works"
  assert-eq! (add 2 3) 5

test "list operations"
  nums = [1, 2, 3]
  assert-eq! (length nums) 3

@skip "not implemented yet"
test "future feature"
  todo()
```

**Important:** Assertions only work in `test` blocks and in the Interpreter. The Compiler does not compile tests.

### 16.2 Assertions

| Assertion | Description |
|-----------|-------------|
| `assert! cond` | Assert true |
| `assert-eq! expected actual` | Assert equality |
| `assert-ne! a b` | Assert not equal |
| `assert-true! value` | Assert value is true |
| `assert-false! value` | Assert value is false |
| `assert-some! value` | Assert Some variant |
| `assert-none! value` | Assert None variant |
| `assert-ok! value` | Assert Ok variant |
| `assert-err! value` | Assert Err variant |
| `assert-lt! a b` | Assert a < b |
| `assert-gt! a b` | Assert a > b |
| `assert-lte! a b` | Assert a <= b |
| `assert-gte! a b` | Assert a >= b |
| `assert-approx! e a tol` | Assert floats approximately equal |
| `assert-fail! message` | Unconditionally fail |

### 16.3 Test File Conventions

- **Naming:** `<name>.test.kit`
- **Module:** Every test file must start with a `module` declaration
- **Location:** `kit-tests/` uses `module Tests.<Subdir>` based on subdirectory

```
kit-tests/
├── core/bool.test.kit          -> module Tests.Core
├── actor/actor-basic.test.kit  -> module Tests.Actor
├── encoding/csv-dialect/       -> module Tests.Encoding.CsvDialect
```

---

## 17. Operator Reference

### 17.1 ASCII Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `+` | Add | `1 + 2` |
| `-` | Subtract / negate | `5 - 3`, `-x` |
| `*` | Multiply | `2 * 3` |
| `/` | Divide | `10 / 3` |
| `%` | Modulo | `10 % 3` |
| `==` | Equal | `x == y` |
| `!=` | Not equal | `x != y` |
| `<`, `<=`, `>`, `>=` | Comparison | `a < b` |
| `&&`, `and` | Logical AND | `a && b` |
| `\|\|`, `or` | Logical OR | `a \|\| b` |
| `!` | Logical NOT | `!flag` |
| `++` | String concat | `"hi" ++ " " ++ "kit"` |
| `::` | Cons | `1 :: [2, 3]` |
| `@` | List append | `a @ b` |
| `\|>` | Pipe (thread-first) | `x \|> f y` |
| `\|>>` | Pipe (thread-last) | `x \|>> f y` |
| `<-` | Actor send | `actor <- msg` |
| `??` | Null coalesce | `opt ?? default` |
| `?!` | Error propagation | `result ?!` |

### 17.2 Unicode Aliases

| Unicode | ASCII | Description |
|---------|-------|-------------|
| `×` | `*` | Multiplication |
| `÷` | `/` | Division |
| `⇒` | `=>` | Fat arrow |
| `→` | `->` | Right arrow |
| `−` | `-` | Minus sign |
| `≠` | `!=` | Not equal |
| `≤` | `<=` | Less than or equal |
| `≥` | `>=` | Greater than or equal |
| `…` | `...` | Spread operator |
| `𝑓` | `fn` | Lambda keyword |

---

## 18. Effect System

Kit tracks effects at the type level. Effects are **inferred automatically** by the compiler, not annotated by the programmer.

```kit
# Pure function — no side effects (inferred as Pure)
pure-signal = fn(prices) => sma 20 prices

# IO function (inferred as IO because send-to-exchange performs I/O)
place-order = fn(o) => send-to-exchange o

# State mutation (inferred as State)
update-position = fn(pos, fill) => ...
```

**Best practice:** Keep backtesting and signal generation logic **pure**. Isolate IO (order placement, logging, network) in effectful functions. This ensures backtests are deterministic and reproducible. Effects are tracked automatically; you do not write effect annotations.

**Important limitation for trading systems:** Because effects are inferred, not enforced, the compiler will not reject a function that accidentally calls an IO operation inside what should be pure signal-generation logic. You must enforce this separation through code review and module boundaries. A recommended convention is to suffix pure functions with `-pure` and keep them in a separate module from IO functions:

```kit
# In Trading.Signals — no IO imports allowed
sma-pure = fn(period: Int, prices: List Decimal) => ...
generate-signals-pure = fn(bars: List Bar, strategy: Strategy) => ...

# In Trading.Execution — IO functions call pure logic, then execute
run-strategy = fn(bars: List Bar) =>
  signals = generate-signals-pure bars my-strategy    # pure
  List.each signals (fn(s) => execute s)              # IO
```

---

## 19. Best Practices & Conventions

### 19.1 General Style

- Use **kebab-case** for all identifiers except types and constructors.
- Booleans **must** start with `is-` or `has-` and end with `?`: `is-filled?`, `has-position?`.
- Prefix unused bindings with `-` to suppress W002 warnings: `-ignore = some-side-effect()`.
- Use `println` instead of `print` for line output.
- Do not ignore linter warnings.

### 19.2 Function Design

- Prefer **pure functions** for business logic. Suffix pure functions with `-pure` and keep them in a dedicated module.
- Use `guard` for cross-parameter precondition checks only; trust refinement types for single-parameter invariants.
- Use `?!` for propagating errors in IO functions.
- Use `Result.and-then` pipelines for linear transformation chains.
- Annotate order handles and connection resources with `@linear` to prevent duplicate execution.

### 19.3 Type Design

- Use **ADTs** (sum types) for state machines and order types.
- Use **refinement types** for domain invariants (prices > 0, percentages in [0,100]).
- Use **linear types** (`@linear`) for order handles, exchange connections, and any resource that must be consumed exactly once.
- Use **closed records** by default. Open records (`...`) only when generality is needed.
- Make invalid states **unrepresentable** via the type system.
- **Never use `Type!` (assert construction) in trading code.** Always use `Type?` or `Type?!` to propagate invalid data as `Err` rather than panicking.

### 19.4 Error Handling

- Use `Result` for recoverable errors.
- Use `Option` for optional values.
- Avoid exceptions for control flow.
- Provide contextual error messages in `Err` variants.

### 19.5 Module Organization

```kit
module Trading.Core

# 1. Imports
import Trading.Types as T
import Trading.Risk

# 2. Type definitions
type Signal = Buy Quantity | Sell Quantity | Hold

# 3. Traits

# 4. Private helpers

# 5. Public API
export generate-signal
export execute-strategy

# 6. Tests
test "signal generation"
  ...
```

---

## 20. Anti-Patterns

### 20.1 Using `any` / Dynamic Typing

Kit is statically typed. Do not try to bypass the type system.

### 20.2 Deeply Nested Matches for Linear Flow

```kit
# BAD
match a
| Ok x ->
    match b x
    | Ok y ->
        match c y
        | Ok z -> Ok z
        | Err e -> Err e
    | Err e -> Err e
| Err e -> Err e

# GOOD
a |> Result.and-then b |> Result.and-then c
```

### 20.3 Using Exceptions for Control Flow

Kit favors `Result` and `Option`. Use `match`, `?!`, `??`, and `guard` instead of throwing exceptions.

### 20.4 Wildcard Imports in Production Code

```kit
import Module.*    # Avoid — obscures where names come from

import Module.{foo, bar}   # Prefer — explicit dependencies
```

### 20.5 Forgetting `main`

A file without `main` will type-check but produce no output when run.

### 20.6 Mixing Tabs and Spaces

Indentation is significant. Use spaces consistently.

### 20.7 Angle Brackets for Generics

```kit
Option<Int>     # WRONG
Option Int      # CORRECT
```

### 20.8 Using `print` Instead of `println`

`print` does not add a newline. Use `println` for normal output.

### 20.9 Trading-Specific Anti-Patterns

**Using `Type!` (assert construction) with market data:**
```kit
# BAD — panics on invalid exchange payload, crashing the bot
price = Price!(exchange-price)

# GOOD — propagate failure via Result
price = match Price?(exchange-price)
| Some p -> p
| None -> return Err "Invalid exchange price"
```

**Using `??` with market data or order parameters:**
```kit
# BAD — fabricating a default price causes bad trades
price = fetch-price(symbol) ?? 0.0M

# GOOD — force explicit handling
price = fetch-price(symbol) ?!
```

**Runtime-checking invariants that refinements already guarantee:**
```kit
# BAD — redundant runtime check; Quantity type already proves q > 0
fn submit(q: Quantity) =>
  guard true = q > 0 else Err "Quantity must be positive"   # never needed

# GOOD — trust the type; check only cross-parameter business rules
fn submit(q: Quantity, limit: Int) =>
  guard true = q <= limit else Err "Exceeds limit"
```

**Forgetting `@linear` on order handles:**
```kit
# BAD — order handle can be passed to submit multiple times
duplicate-risk = fn(order: OrderType, exchange: Exchange) =>
  Exchange.submit exchange order
  Exchange.submit exchange order   # Duplicate execution!

# GOOD — linear type enforces single submission
submit-once = fn(order @linear, exchange: Exchange) =>
  Exchange.submit exchange order    # consumed exactly once
```

---

## 21. Common Idioms by Domain

### 21.1 Trading Bot: Domain Modeling with ADTs

```kit
type Side = Buy | Sell

type OrderType =
  | Market(side: Side, quantity: Quantity)
  | Limit(side: Side, price: Price, quantity: Quantity)
  | Stop(side: Side, trigger: Price, quantity: Quantity)
  | StopLimit(side: Side, stop: Price, limit: Price, quantity: Quantity)

type OrderStatus =
  | Created
  | Submitted(time: Time)
  | PartiallyFilled(filled: Int, remaining: Int)
  | Filled(avg-price: Decimal)
  | Rejected(reason: String)
  | Cancelled

type Signal =
  | Enter(direction: Side, quantity: Quantity, order-type: OrderType)
  | Exit(reason: String)
  | Hold
```

### 21.2 Trading Bot: State Machine with Exhaustive Matching

```kit
transition-order = fn(status: OrderStatus, event: TradeEvent) =>
  match (status, event)
  | (Created, Submit) -> Submitted(Time.now())
  | (Submitted(_), Ack) -> PartiallyFilled(0, order-quantity)
  | (Submitted(_), Fill(q)) -> PartiallyFilled(q, order-quantity - q)
  | (PartiallyFilled(f, r), Fill(q)) when f + q >= r -> Filled(calculate-avg())
  | (PartiallyFilled(f, r), Fill(q)) -> PartiallyFilled(f + q, r - q)
  | (Filled(_), _) -> status
  | (_, CancelRequest) -> Cancelled
  | (s, e) -> Rejected("Invalid transition from ${s} via ${e}")
```

### 21.3 Trading Bot: Risk Management with Refinement Types

When refinement types are used rigorously, runtime guards for basic domain invariants are redundant — the compiler has already proven them. Risk management should focus on cross-parameter constraints (e.g., margin limits) rather than re-checking what the type system guarantees.

```kit
# Refinement types from section 21.4 guarantee positivity
# Quantity = {q: Int | q > 0}
# Price = {p: Decimal | p > 0.0M}

validate-order = fn(
    order: OrderType,                          # order.quantity is already Quantity (> 0)
    portfolio: Portfolio
  ) =>
  # Only cross-parameter business rules need runtime checks
  guard true = order.quantity <= max-position-size
    else Err "Position limit exceeded"
  guard true = portfolio.margin-used < portfolio.margin-limit
    else Err "Margin exceeded"

  Ok order

# If you receive raw external data, validate at the boundary and return Result
parse-and-validate = fn(raw-qty: Int, raw-price: Decimal) =>
  qty = match Quantity?(raw-qty)
  | Some q -> q
  | None -> return Err "Invalid quantity"

  price = match Price?(raw-price)
  | Some p -> p
  | None -> return Err "Invalid price"

  Ok (Limit(Buy, price, qty))
```

### 21.4 Trading Bot: Refinement Types for Safety

```kit
type Price = {p: Decimal | p > 0.0M}
type Quantity = {q: Int | q > 0}
type DrawdownLimit = {d: Float | d >= 0.0 && d <= 100.0}

# Boundaries sanitize external input — NEVER use `!` (assert construction) in trading code.
# `Type!(val)` panics on invalid input, which can crash a live trading bot.
# Always use `?` (Option) or `?!` (Result) and propagate the failure.

safe-price = fn(raw: Decimal) =>
  match Price?(raw)
  | Some p -> Ok p
  | None -> Err "Invalid price: must be > 0"

safe-quantity = fn(raw: Int) =>
  match Quantity?(raw)
  | Some q -> Ok q
  | None -> Err "Invalid quantity: must be > 0"

# At the outer boundary, halt on invalid data rather than fabricate a value
parse-market-data = fn(raw: JsonValue) =>
  price = Json.get-decimal "price" raw ?!
  qty   = Json.get-int     "quantity" raw ?!

  Ok {
    price: safe-price(price) ?!,
    quantity: safe-quantity(qty) ?!
  }
```

### 21.5 Trading Bot: Pure Signal Generation

```kit
# Pure — no side effects, deterministic
generate-signals = fn(prices: List Decimal, strategy: Strategy) =>
  short = sma 20 prices
  long = sma 50 prices

  match (short, long)
  | (Some(s), Some(l)) when s > l * 1.01 -> [Enter(Buy, 100, Market(Buy, 100))]
  | (Some(s), Some(l)) when s < l * 0.99 -> [Enter(Sell, 100, Market(Sell, 100))]
  | _ -> []
```

### 21.6 Trading Bot: Backtesting Pipeline

```kit
run-backtest = fn(strategy: Strategy, data: List Bar) =>
  data
  |> generate-signals-for-bars strategy
  |> apply-slippage 0.001M
  |> simulate-executions
  |> calculate-pnl
  |> generate-report
```

### 21.7 Pipeline Chaining

```kit
result = data
  |> filter (fn(x) => x > 0)
  |> map (fn(x) => x * 2)
  |> fold (fn(a, x) => a + x) 0
```

### 21.8 Early Return with `guard`

```kit
process = fn(opt) =>
  guard Some value = opt else Err "was None"
  guard true = value > 0 else Err "not positive"
  Ok (value * 2)
```

### 21.9 Tail Recursion

```kit
sum-tail = fn(lst) =>
  helper = fn(items, acc) =>
    match items
    | [] -> acc
    | [x | rest] -> helper rest (acc + x)
  helper lst 0
```

### 21.10 Trading Bot: Rigorous Safety Checklist

This section ties together the advanced type-system features for production trading systems.

**1. Refinement types at every boundary**

```kit
type Price = {p: Decimal | p > 0.0M}
type Quantity = {q: Int | q > 0}
type Confidence = {c: Float | c >= 0.0 && c <= 1.0}
```

**2. Safe construction only — never `Type!`**

```kit
# At the system boundary (exchange API, user input, config files)
sanitize-price = fn(raw: Decimal) =>
  match Price?(raw)
  | Some p -> Ok p
  | None -> Err f"Rejecting invalid price: ${raw}"
```

**3. Linear types for execution resources**

```kit
submit-and-track = fn(order @linear, exchange: Exchange) =>
  ack = Exchange.submit exchange order
  track-ack ack

# Cannot accidentally call submit-and-track twice on the same order binding
```

**4. Contracts for cross-parameter invariants**

```kit
@pre(portfolio.available-margin >= 0.0M)
@post(result >= 0.0M)
@post(result <= portfolio.margin-limit)
calculate-margin = fn(order: OrderType, portfolio: Portfolio) => ...
```

**5. Pure signal generation isolated from IO**

```kit
# Trading.Signals module — no network, no exchange imports
generate-signals-pure = fn(bars: List Bar, strategy: Strategy) => ...

# Trading.Execution module — calls pure logic, then performs IO
run-live = fn(bars: List Bar) =>
  signals = generate-signals-pure bars my-strategy
  List.each signals (fn(s) => execute s)
```

**6. Explicit Result handling — no `??` for market data**

```kit
# CORRECT
price = fetch-price(symbol) ?!
qty   = fetch-quantity(symbol) ?!

# WRONG — never fabricate defaults for market data
# price = fetch-price(symbol) ?? 0.0M
```

---

## 22. Lint & Warnings

Kit's `kit check` command runs static analysis. Do not ignore warnings.

### 22.1 Warning Codes (W0xx)

| Code | Description |
|------|-------------|
| `W001` | Missing defer for resource cleanup |
| `W002` | Binding defined but never used |
| `W003` | Shadowed binding |
| `W004` | Import never used |
| `W005` | Missing else branch |
| `W006` | Boolean naming convention (should use `is-*?` or `has-*?`) |
| `W007` | Unreachable pattern |
| `W008` | Parameter never used |
| `W009-W012` | Suspicious code patterns |
| `W013` | Deprecated function usage |
| `W014` | Non-exhaustive pattern match |
| `W015` | Additional suspicious patterns |

### 22.2 Suggestion Codes (S0xx)

| Code | Description |
|------|-------------|
| `S001` | Wildcard import could be selective |

### 22.3 Type Errors (T0xx)

| Code | Description |
|------|-------------|
| `T013` | Use after move (linear types) |
| `T014` | Linearity violation |

**Fixing W002:** Remove the unused binding or prefix with `-`:
```kit
-ignore = some-side-effect()   # suppresses W002
```

---

## 23. Tooling & Workflow

### 23.1 Essential Commands

```bash
# Initialize project
kit init my-project

# Type check (always run before committing)
kit check src/*.kit
kit check --no-spinner src/*.kit

# Run interpreter
kit run src/main.kit
kit run --no-spinner src/main.kit

# Compile to native binary
kit build src/main.kit -o my-app
kit build --no-spinner src/main.kit -o my-app

# Run tests
kit test
kit test --coverage
kit test tests/ --format json --output results.json

# Format code
kit format src/*.kit
kit format --check src/*.kit   # CI validation

# Development workflow
kit dev
```

### 23.2 kit.toml Format

```toml
[package]
name = "my-package"
version = "0.1.0"
authors = ["Author Name"]
description = "Package description"

[dependencies]
some-dep = "1.0.0"
git-dep = { git = "https://github.com/user/repo", tag = "v1.0" }
local-dep = { path = "../local-package" }

[tasks]
test = "kit test tests/"
check = "kit check --no-spinner src/*.kit"
format = "kit format src/*.kit"
ci = ["format", "check", "test"]
```

### 23.3 Package Management

```bash
kit install              # Install deps from kit.toml
kit add <package>        # Add dependency
kit remove <package>     # Remove dependency
kit clean                # Clean kit_modules and lock file
kit list                 # List installed packages
```

### 23.4 Before Committing Checklist

1. Run `kit format --check src/*.kit` — ensure formatting is clean.
2. Run `kit check src/*.kit` — no type errors or warnings.
3. Run `kit test` — all tests pass.
4. Verify `main` binding exists in entry files.

---

## 24. Appendices

### Appendix A: Reserved but Not Yet Implemented

The following keywords are reserved for future use:

- **`with`** — Reserved for future scoped effect handlers.

### Appendix B: Type Inspection (Runtime)

```kit
type-of 42           # "Int"
type-sig double      # "Int -> Int"
```

### Appendix C: Full EBNF Grammar Summary

For the complete formal grammar in EBNF, see `GRAMMAR.md`. Key non-terminals:

```ebnf
program         = { declaration } EOF
declaration     = type_def | import_stmt | export_stmt | module_decl
                | test_block | extern_decl | trait_def | trait_impl | binding
type_def        = "type" TYPE_NAME { TYPE_VAR } "=" type_body
type_body       = constructor { "|" constructor }
                | "|" constructor { "|" constructor }
constructor     = CONSTRUCTOR_NAME [ constructor_args ]
constructor_args = "(" field_list ")" | type { type }
binding         = { contract } pattern [ ":" type ] [ linearity ] "=" expression
                | qualified_binding
contract        = "@pre" "(" expression [ "," STRING ] ")"
                | "@post" "(" expression [ "," STRING ] ")"
linearity       = " @linear" | " @affine" | " @relevant" | " @unrestricted"
expression      = defer_expr
defer_expr      = "defer" expression | null_coalesce
null_coalesce   = pipe_expr { "??" pipe_expr }
pipe_expr       = send_expr { ( "|>" | "|>>" ) send_expr }
send_expr       = or_expr [ "<-" or_expr ]
or_expr         = and_expr { ( "||" | "or" ) and_expr }
and_expr        = is_as_expr { ( "&&" | "and" ) is_as_expr }
is_as_expr      = equality [ "is" pattern ]
                | equality "as" pattern "->" expression "else" expression
                | equality
equality        = comparison { ( "==" | "!=" ) comparison }
comparison      = concat { ( "<" | "<=" | ">" | ">=" ) concat }
concat          = cons { "++" cons }
cons            = term { ( "::" | "@" ) term }
term            = factor { ( "+" | "-" ) factor }
factor          = unary { ( "*" | "/" | "%" ) unary }
unary           = ( "-" | "!" ) unary | call_expr
call_expr       = primary { call_suffix }
call_suffix     = "(" [ arg_list ")" | "." IDENT | "?!" | prefix_arg
primary         = INT | FLOAT | STRING | "true" | "false" | "()"
                | KEYWORD_LITERAL | IDENT | field_accessor | lambda
                | if_expr | match_expr | for_expr | sql_expr
                | group_or_tuple | list_literal | set_literal
                | record_literal | interpolated_string | heredoc
lambda          = "fn" [ "(" [ param_patterns ] ")" ] "=>" lambda_body
param_patterns  = param_pattern { "," param_pattern }
param_pattern   = [ "~" ] pattern
match_expr      = "match" expression match_arms
match_arms      = { "|" match_arm }
match_arm       = or_pattern [ guard ] "->" expression
                | or_pattern [ guard ] "=>" expression
or_pattern      = pattern { "|" pattern }
guard           = "when" expression | "if" expression
pattern         = "_" | INT | FLOAT | STRING | "true" | "false"
                | KEYWORD_LITERAL | IDENT | constructor_pattern
                | tuple_pattern | list_pattern | record_pattern
refinement_type = "{" IDENT ":" type "|" expression "}"
record_type     = "{" [ record_type_fields ] [ "..." ] "}"
```

### Appendix D: Kit for LLMs — Quick Cheat Sheet

```kit
# --- Types ---
type Color = Red | Green | Blue          # ADT
type Point = Point(x: Float, y: Float)   # Record constructor
type Opt a = Some a | None               # Generic

# --- Functions ---
add = fn(a, b) => a + b                  # Curried
apply = fn(f, x) => f x                  # Higher-order

# --- Pattern Match ---
match x
| Some n -> n
| None -> 0

# --- Error Handling ---
# Propagate:    val = fallible() ?!
# Default:      val = maybe ?? default
# Pipeline:     a |> Result.and-then b
# Extract:      guard Some v = opt else Err "missing"

# --- Pipes ---
data |> fn(x) => x * 2                   # thread-first
list |>> map double |>> filter even?     # thread-last

# --- Records ---
p = {name: "A", age: 30}
p.name                                   # access
{name, age} = p                          # destructure
{p | age: 31}                            # update

# --- Guards ---
guard true = x > 0 else Err "negative"

# --- Refinement ---
type Pos = {n: Int | n > 0}
x = Pos!(5)                              # assert
y = Pos?(input)                          # Option

# --- Concurrency ---
actor <- msg                              # send
ch = Channel.spsc 10 |> Result.unwrap

# --- Testing ---
test "name"
  assert-eq! (add 1 2) 3
```

---

*This document reflects Kit Lang as of version 2026.4.24. For the formal EBNF grammar, see `GRAMMAR.md`.*
