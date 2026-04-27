# Kit Language Grammar Specification

This document provides a formal grammar specification for the Kit programming language in Extended Backus-Naur Form (EBNF).

## Notation

- `|` - alternation
- `[ ]` - optional
- `{ }` - zero or more repetitions
- `( )` - grouping
- `" "` - terminal string
- `UPPER_CASE` - token/terminal
- `lower_case` - non-terminal

---

## 1. Program Structure

```ebnf
program         = { declaration } EOF ;

declaration     = type_def
                | import_stmt
                | export_stmt
                | module_decl
                | test_block
                | extern_decl
                | trait_def
                | trait_impl
                | binding ;
```

---

## 2. Type Definitions

```ebnf
type_def        = "type" TYPE_NAME { TYPE_VAR } "=" type_body ;

type_body       = constructor { "|" constructor }
                | "|" constructor { "|" constructor } ;

constructor     = CONSTRUCTOR_NAME [ constructor_args ] ;

constructor_args = "(" field_list ")"
                 | type { type } ;

field_list      = field { "," field } ;

field           = IDENT { attribute } ;

TYPE_NAME       = UPPER_IDENT ;
TYPE_VAR        = LOWER_IDENT ;
CONSTRUCTOR_NAME = UPPER_IDENT ;
```

### Attributes

Attributes provide metadata annotations on constructor fields. They are used for
ORM mappings, serialization hints, validation rules, and other metadata.

```ebnf
attribute       = "@" attr_name [ "(" attr_args ")" ] ;

attr_name       = IDENT { "-" IDENT } ;     (* supports hyphenated names *)

attr_args       = attr_arg { "," attr_arg } ;

attr_arg        = STRING
                | INT
                | FLOAT
                | "true" | "false"
                | IDENT ;
```

---

## 3. Import/Export

```ebnf
import_stmt     = "import" import_path [ "as" IDENT ]
                | "import" import_path "." "{" import_list "}" ;

import_path     = STRING
                | module_path ;

module_path     = IDENT { "." IDENT } ;

import_list     = import_item { "," import_item } ;
import_item     = IDENT [ "as" IDENT ] ;

export_stmt     = "export" ( binding | type_def ) ;

module_decl     = "module" module_path ;
```

---

## 4. Test Blocks

```ebnf
test_block      = [ "@skip" [ STRING ] ] "test" [ test_name ] indented_block ;

test_name       = STRING | KEYWORD_LITERAL ;
```

The `@skip` attribute marks a test to be skipped during execution, with an optional reason string.

### Test Assertions

Kit provides a comprehensive set of assertion functions for use within test blocks:

**Basic Assertions:**
- `assert! cond` - Assert condition is true
- `assert-eq! expected actual` - Assert two values are equal
- `assert-ne! a b` - Assert two values are not equal
- `assert-true! value` - Assert value is true
- `assert-false! value` - Assert value is false

**Option/Result Assertions:**
- `assert-some! value` - Assert value is Some variant
- `assert-none! value` - Assert value is None variant
- `assert-ok! value` - Assert value is Ok variant
- `assert-err! value` - Assert value is Err variant

**Comparison Assertions (integers only):**
- `assert-lt! a b` - Assert a < b
- `assert-gt! a b` - Assert a > b
- `assert-lte! a b` - Assert a <= b
- `assert-gte! a b` - Assert a >= b

**Float Assertions:**
- `assert-approx! expected actual tolerance` - Assert floats are approximately equal

**Failure Assertions:**
- `assert-fail! message` - Unconditionally fail with message

### Test Examples

```kit
test "basic assertions"
  assert!(2 + 2 == 4)
  assert-eq!(1 + 1, 2)
  assert-ne!("hello", "world")
  assert-true!(true)
  assert-false!(false)

test "option assertions"
  assert-some!(Some 42)
  assert-none!(None)

test "result assertions"
  assert-ok!(Ok 42)
  assert-err!(Err "error")

test "comparison assertions"
  assert-lt!(3, 5)
  assert-gt!(10, 5)
  assert-lte!(3, 3)
  assert-gte!(5, 5)

test "float approximation"
  assert-approx!(3.14159, 3.14, 0.01)
```

**Note:** Assertions only work in `test` blocks and in the Interpreter. The Compiler does not compile tests.

---

## 5. Match Macros

Kit provides three concise pattern matching macros for common operations:

```ebnf
is_expr         = expression "is" pattern ;

as_expr         = expression "as" pattern "->" expression "else" expression ;

guard_stmt      = "guard" pattern "=" expression "else" expression ;
```

- **`is`**: Check if a value matches a pattern, returns `Bool`
- **`as`**: Extract value if pattern matches, return alternative otherwise
- **`guard`**: Bind pattern variables if match succeeds, return early otherwise

These macros provide concise alternatives to verbose match expressions.

---

## 6. Compile-Time Macros

Kit supports user-defined compile-time macros for code generation:

```ebnf
macro_def       = "macro" IDENT { IDENT } "=" quasiquote ;

quasiquote      = "`" "(" expression ")" ;

unquote         = "$" IDENT ;
```

Macros are expanded at compile time. The quasiquote syntax (`` ` ``) creates a code
template, and `$param` unquotes (substitutes) parameter values into the template.

```kit
# Define a macro
macro double x = ` ($x + $x)

# Usage - expands to (5 + 5) at compile time
result = double(5)  # 10
```

---

## 7. Extern Declarations

```ebnf
extern_decl     = extern_c_decl | extern_zig_decl ;

extern_c_decl   = "extern-c" IDENT "(" [ param_list ] ")" "->" type
                  "from" STRING "link" STRING ;

extern_zig_decl = "extern-zig" IDENT "(" [ param_list ] ")" "->" type
                  "from" STRING "link" STRING ;

param_list      = param { "," param } ;
param           = IDENT ":" type ;
```

- `extern-c` declares a C function binding
- `extern-zig` declares a Zig function binding

---

## 8. Traits

```ebnf
trait_def       = "trait" IDENT TYPE_VAR [ "requires" constraint_list ]
                  trait_body ;

trait_body      = { trait_method } ;

trait_method    = IDENT ":" type
                | IDENT "=" expression ;

trait_impl      = "extend" type "with" IDENT [ "where" constraint_list ]
                  impl_body ;

impl_body       = { method_def } ;

method_def      = IDENT "=" expression ;

constraint_list = constraint { "," constraint } ;
constraint      = IDENT TYPE_VAR ;
```

---

## 9. Bindings

```ebnf
binding         = { contract } pattern [ ":" type ] [ linearity ] "=" expression
                | qualified_binding ;

linearity       = " @linear"      (* must use exactly once *)
                | " @affine"      (* can use at most once *)
                | " @relevant"      (* must be used at least once *)
                | " @unrestricted"  (* no constraints - default *) ;

qualified_binding = TYPE_NAME "." IDENT "=" expression ;

contract        = "@pre" "(" expression [ "," STRING ] ")"
                | "@post" "(" expression [ "," STRING ] ")" ;
```

Contracts (`@pre` and `@post`) provide precondition and postcondition assertions
for functions. The optional string argument provides a custom error message.

### Destructuring in Bindings

Bindings support pattern matching with tuple and record patterns, allowing you to
extract values directly:

```kit
# Tuple destructuring
(x, y) = (10, 20)
(a, b, c) = get-triple()

# Record destructuring with shorthand (field name becomes binding name)
{name, age} = person

# Record destructuring with explicit rename
{name: user-name, age: user-age} = person

# Partial record destructuring (extract only some fields)
{x, z} = {x: 1, y: 2, z: 3}

# Nested destructuring
{outer: {inner}} = nested-record
((a, b),  (c, d)) = nested-tuples

# Wildcards to ignore values
{keep, ignore: _} = data
(_, important, _) = triple
```

---

## 10. Expressions

### Precedence (lowest to highest)

```ebnf
expression      = defer_expr ;

defer_expr      = "defer" expression
                | null_coalesce ;

null_coalesce   = pipe_expr { "??" pipe_expr } ;

pipe_expr       = send_expr { ( "|>" | "|>>" ) send_expr } ;

send_expr       = or_expr [ "<-" or_expr ] ;

or_expr         = and_expr { ( "||" | "or" ) and_expr } ;

and_expr        = is_as_expr { ( "&&" | "and" ) is_as_expr } ;

is_as_expr      = equality [ "is" pattern ]
                | equality "as" pattern "->" expression "else" expression
                | equality ;

equality        = comparison { ( "==" | "!=" ) comparison } ;

comparison      = concat { ( "<" | "<=" | ">" | ">=" ) concat } ;

concat          = cons { "++" cons } ;

cons            = term { ( "::" | "@" ) term } ;

term            = factor { ( "+" | "-" ) factor } ;

factor          = unary { ( "*" | "/" | "%" ) unary } ;

unary           = ( "-" | "!" ) unary
                | call_expr ;

call_expr       = primary { call_suffix } ;

call_suffix     = "(" [ arg_list ] ")"           (* C-style call *)
                | "." IDENT                       (* field access *)
                | "?!"                            (* error propagation *)
                | prefix_arg ;                    (* Haskell-style *)

prefix_arg      = primary ;                       (* excluding operators *)

arg_list        = expression { "," expression } ;
```

### Primary Expressions

```ebnf
primary         = INT
                | FLOAT
                | STRING
                | "true" | "false"
                | "()"                            (* unit *)
                | KEYWORD_LITERAL                 (* :symbol *)
                | IDENT
                | field_accessor
                | lambda
                | if_expr
                | match_expr
                | for_expr
                | sql_expr
                | group_or_tuple
                | list_literal
                | set_literal
                | record_literal
                | interpolated_string
                | heredoc ;

field_accessor  = "." IDENT ;                     (* shorthand for fn(x) => x.IDENT *)

lambda          = "fn" [ "(" [ param_patterns ] ")" ] "=>" lambda_body ;

param_patterns  = param_pattern { "," param_pattern } ;

param_pattern   = [ "~" ] pattern ;          (* ~ marks lazy-accepting parameter *)

lambda_body     = expression
                | indented_block ;
```

Note: Zero-arity functions can omit parentheses: `fn => expr` is equivalent to `fn() => expr`.

```ebnf

if_expr         = "if" expression "then" if_branch "else" if_branch ;

if_branch       = expression
                | indented_block ;

match_expr      = "match" expression match_arms ;

match_arms      = { "|" match_arm } ;

match_arm       = or_pattern [ guard ] "->" expression
                | or_pattern [ guard ] "=>" expression ;

or_pattern      = pattern { "|" pattern } ;

guard           = "when" expression
                | "if" expression ;

for_expr        = "for" pattern "in" expression "=>" for_body ;

for_body        = expression
                | indented_block ;

sql_expr        = "sql" [ expression [ "as" type_name ] ] "{" sql_content "}" ;

sql_content     = { SQL_CHAR | "${" expression [ ":" FORMAT_SPEC ] "}" } ;

group_or_tuple  = "(" expression [ "," expression { "," expression } ] ")" ;

list_literal    = "[" [ list_elements ] "]" ;

list_elements   = expression { "," expression } [ "|" expression ] ;

set_literal     = "#{" [ expression { "," expression } ] "}" ;

record_literal  = "{" [ record_fields ] "}" ;

record_fields   = record_field { "," record_field } ;

record_field    = "..." expression                (* spread *)
                | IDENT ":" expression
                | IDENT ;                         (* shorthand *)

interpolated_string = STRING_START { string_part } STRING_END ;

string_part     = STRING_CONTENT
                | "${" expression [ ":" FORMAT_SPEC ] "}" ;

heredoc         = "<<~" IDENT NEWLINE heredoc_content IDENT ;

heredoc_content = { HEREDOC_CHAR | "${" expression [ ":" FORMAT_SPEC ] "}" } ;
```

Heredocs use squiggly syntax (`<<~`) which strips common leading indentation.
They support the same string interpolation as regular strings.

---

## 11. Patterns

```ebnf
pattern         = "_"                             (* wildcard *)
                | INT
                | FLOAT
                | STRING
                | "true" | "false"
                | KEYWORD_LITERAL
                | IDENT                           (* binding or constructor *)
                | constructor_pattern
                | tuple_pattern
                | list_pattern
                | record_pattern ;

constructor_pattern = CONSTRUCTOR_NAME [ constructor_payload ] ;

constructor_payload = "(" pattern { "," pattern } ")"
                    | "{" record_pat_fields [ ".." ] "}"
                    | pattern { pattern } ;           (* space-separated *)

tuple_pattern   = "(" pattern "," pattern { "," pattern } ")" ;

list_pattern    = "[" [ list_pat_elements ] "]" ;

list_pat_elements = ".." pattern                  (* rest at start *)
                  | pattern { "," pattern } [ ( "|" | ".." ) pattern ] ;

record_pattern  = "{" [ record_pat_fields ] "}" ;

record_pat_fields = record_pat_field { "," record_pat_field } ;

record_pat_field = IDENT [ ":" pattern ] ;
```

---

## 12. Types

```ebnf
type            = simple_type [ "->" type ] ;     (* function type *)

simple_type     = "Int" | "Float" | "String" | "Bool" | "Unit"
                | TYPE_NAME [ type_args ]
                | tuple_type
                | record_type
                | refinement_type
                | "(" type ")" ;

type_args       = simple_type { simple_type } ;

tuple_type      = "(" type "," type { "," type } ")" ;

record_type     = "{" [ record_type_fields ] [ "..." ] "}" ;

record_type_fields = record_type_field { "," record_type_field } ;

record_type_field = IDENT ":" type ;

refinement_type = "{" IDENT ":" type "|" expression "}" ;
```

### Refinement Types

Refinement types constrain values beyond what the base type expresses. The syntax
follows Liquid Haskell: `{binding: BaseType | predicate}`.

```ebnf
refinement_type = "{" IDENT ":" type "|" expression "}" ;
```

The binding introduces a variable scoped to the predicate expression. The predicate
must evaluate to a boolean.

### Refinement Construction

Refined values are constructed using special operators:

```ebnf
refinement_construct = TYPE_NAME "!" "(" expression ")"    (* assert *)
                     | TYPE_NAME "?" "(" expression ")"    (* Option *)
                     | TYPE_NAME "?!" "(" expression ")" ; (* Result *)
```

- `T!(expr)` - Assert construction; panics if predicate fails
- `T?(expr)` - Safe construction; returns `Option T`
- `T?!(expr)` - Safe construction; returns `Result T String`

### Row Polymorphism

Kit supports opt-in row polymorphism for record types using `...` syntax. By default,
record types are closed (require exact fields). Adding `...` makes them open (accept
extra fields).

```ebnf
record_type     = "{" [ record_type_fields ] [ "..." ] "}" ;
```

- **Closed record (default)**: `{name: String}` - requires exactly the specified fields
- **Open record (opt-in)**: `{name: String, ...}` - requires specified fields, allows extras
- **Empty open record**: `{...}` - accepts any record

---

## 13. Indented Blocks

```ebnf
indented_block  = NEWLINE INDENT { block_item } DEDENT ;

block_item      = binding
                | expression ;
```

Note: Indentation is significant. An indented block begins when the indentation
level increases and ends when it returns to the previous level.

---

## 14. Lexical Elements

### Identifiers

```ebnf
IDENT           = LOWER_IDENT | UPPER_IDENT ;

LOWER_IDENT     = ( LETTER_LOWER | "_" ) { LETTER | DIGIT | "_" | "-" } ;

UPPER_IDENT     = LETTER_UPPER { LETTER | DIGIT | "_" | "-" } ;

LETTER          = LETTER_LOWER | LETTER_UPPER ;
LETTER_LOWER    = "a" ... "z" ;
LETTER_UPPER    = "A" ... "Z" ;
DIGIT           = "0" ... "9" ;
```

### Numeric Literals

```ebnf
INT             = DECIMAL_INT | HEX_INT | BINARY_INT | OCTAL_INT ;

DECIMAL_INT     = [ "-" ] DIGIT { DIGIT | "_" } [ INT_SUFFIX ] ;

HEX_INT         = [ "-" ] "0" ( "x" | "X" ) HEX_DIGIT { HEX_DIGIT | "_" } ;

BINARY_INT      = [ "-" ] "0" ( "b" | "B" ) BINARY_DIGIT { BINARY_DIGIT | "_" } ;

OCTAL_INT       = [ "-" ] "0" ( "o" | "O" ) OCTAL_DIGIT { OCTAL_DIGIT | "_" } ;

FLOAT           = [ "-" ] DIGIT { DIGIT } "." DIGIT { DIGIT } [ FLOAT_SUFFIX ] ;

HEX_DIGIT       = DIGIT | "a" ... "f" | "A" ... "F" ;
BINARY_DIGIT    = "0" | "1" ;
OCTAL_DIGIT     = "0" ... "7" ;

INT_SUFFIX      = "L"                             (* int64 *)
                | "u" | "ul"                      (* uint32 *)
                | "I" ;                           (* bigint *)

FLOAT_SUFFIX    = "f" | "F"                       (* float32 *)
                | "m" | "M" ;                     (* decimal *)
```

Examples: `255`, `0xFF`, `0b1111_1111`, `0o377`, `1_000_000`

### String Literals

```ebnf
STRING          = '"' { STRING_CHAR | ESCAPE_SEQ } '"' ;

STRING_CHAR     = any character except '"', '\', or '${' ;

ESCAPE_SEQ      = "\" ( "n" | "r" | "t" | "\" | '"' | "$" ) ;
```

### Keyword Literals

```ebnf
KEYWORD_LITERAL = ":" IDENT ;
```

### Comments

```ebnf
COMMENT         = "#" { any character except newline } NEWLINE ;

DOC_COMMENT     = "##" { any character except newline } NEWLINE ;
```

Doc comments (`##`) are associated with the following declaration and can be
used for documentation generation.

---

## 15. Operator Precedence Table

| Precedence | Operators          | Associativity | Description           |
|------------|--------------------|--------------|-----------------------|
| 1 (lowest) | `defer`            | prefix       | Deferred execution    |
| 2          | `??`               | right        | Null coalesce         |
| 3          | `\|>` `\|>>`       | left         | Pipe operators        |
| 4          | `<-`               | -            | Actor send            |
| 5          | `\|\|` `or`        | left         | Logical OR            |
| 6          | `&&` `and`         | left         | Logical AND           |
| 7          | `is` `as`          | -            | Pattern test/extract  |
| 8          | `==` `!=`          | left         | Equality              |
| 9          | `<` `<=` `>` `>=`  | left         | Comparison            |
| 10         | `++`               | left         | String concatenation  |
| 11         | `::` `@`           | left         | List cons/append      |
| 12         | `+` `-`            | left         | Addition/subtraction  |
| 13         | `*` `/` `%`        | left         | Multiplication/etc    |
| 14         | `-` `!`            | right        | Unary negation/not    |
| 15         | calls, `.`, `?!`   | left         | Application/access    |
| 16 (high)  | literals           | -            | Primary expressions   |

---

## 16. Unicode Operator Aliases

Kit supports Unicode characters as aliases for common operators:

| Unicode | ASCII  | Description         |
|---------|--------|---------------------|
| `×`     | `*`    | Multiplication      |
| `÷`     | `/`    | Division            |
| `⇒`     | `=>`   | Fat arrow           |
| `→`     | `->`   | Right arrow         |
| `−`     | `-`    | Minus sign          |
| `≠`     | `!=`   | Not equal           |
| `≤`     | `<=`   | Less than or equal  |
| `≥`     | `>=`   | Greater than or equal |
| `…`     | `...`  | Spread operator     |
| `𝑓`     | `fn`   | Lambda keyword      |

---

## 17. Reserved Words

```
as        defer     else      export    extend    extern-c  extern-zig
false     fn        for       from      guard     if        import
in        is        link      macro     match     module    requires
sql       test      then      trait     true      type      when
where     with
```

---

## 18. Pre-registered Constructors

The following constructors are pre-registered and can be used in patterns
without a type definition:

```
# Result and Option types
Ok    Err    Some    None

# Backoff strategies
NoBackoff    Constant    Linear    Exponential

# Jitter strategies
NoJitter    FullJitter    EqualJitter    ProportionalJitter
```

---

## 19. Examples

### Type Definition
```kit
type Option a = Some a | None

type Result a e =
  | Ok a
  | Err e

type Point = Point(x, y)
```

### Type Definition with Attributes
```kit
# Single-line with attributes
type User = User(id@primary-key@auto-increment, name, email@unique)

# Multi-line with attributes
type Product = Product(
  id@primary-key, 
  price@default(0.0), 
  category_id@fkey(Category, id), 
  tags@json("tags", omitempty)
)

# ADT with attributes
type Status =
  | Pending
  | Active(user_id@fkey(User, id))
  | Completed(timestamp)
```

### Pattern Matching
```kit
match value
  | Some x -> x
  | None -> default

match point
  | Point(0, 0) -> "origin"
  | Point(x, 0) -> "on x-axis"
  | Point(0, y) -> "on y-axis"
  | Point(x, y) -> "at (${x}, ${y})"
```

### Lambda Functions
```kit
add = fn(a, b) => a + b

# Zero-arity function (parentheses optional)
get-time = fn => now()

process = fn(items) =>
  result = items |> map square
  result

# Lazy-accepting parameter (~ sigil)
maybe-compute = fn(~value, should-force?) =>
  if should-force? then force(value) else 0
```

### For Expressions
```kit
# Simple for loop
for x in [1, 2, 3] => print(x)

# With tuple destructuring
for (k, v) in map-entries(m) => print("${k}: ${v}")

# Multi-line body
for item in items =>
  processed = transform(item)
  save(processed)
```

### SQL Expressions
```kit
# Simple SQL
sql {SELECT * FROM users}

# With connection
sql db {SELECT * FROM users WHERE id = ${user_id}}

# With type casting
sql db as User {SELECT * FROM users WHERE id = 1}
```

### Heredocs
```kit
html = <<~HTML
  <div>
    <h1>${title}</h1>
    <p>${content}</p>
  </div>
HTML
```

### Contracts
```kit
@pre(n >= 0, "n must be non-negative")
@post(result >= 0)
factorial = fn(n) =>
  if n <= 1 then 1 else n * factorial(n - 1)
```

### Test Blocks
```kit
test "addition works"
  assert-eq!(1 + 1, 2)

@skip "not implemented yet"
test "future feature"
  todo()
```

### Numeric Literals
```kit
decimal = 1 _000_000
hex = 0xFF
binary = 0b1010_1010
octal = 0o755
int64 = 100L
bigint = 999999999999999999I
float32 = 3.14f
```

### List Patterns
```kit
# Head and tail
match list
  | [head | tail] -> process(head, tail)
  | [] -> empty()

# Rest pattern
match args
  | [first, second, ..rest] -> handle(first, second, rest)
  | [..all] -> handle-all(all)
```

### Traits
```kit
trait Eq a
  eq: (a, a) -> Bool
  ne = fn(a, b) => ! (eq a b)

extend Int with Eq
  eq = fn(a, b) => a == b
```

### Pipe Operators
```kit
data |> transform |> output       # thread-first
list |>> map double |>> filter    # thread-last
```

### Refinement Types
```kit
# Type declarations
type PositiveInt = {n: Int | n > 0}
type NonZero = {n: Int | n != 0}
type Percentage = {p: Float | p >= 0.0 && p <= 100.0}
type NonEmptyList a = {xs: List a | not(empty?(xs))}

# Assert construction (panics on failure)
port = ValidPort!(8080)

# Safe construction (returns Option)
port = ValidPort?(user_input)
match port
  | Some p -> use-port(p)
  | None -> print("Invalid port")

# Safe construction (returns Result)
port = ValidPort ?! (user_input)
match port
  | Ok p -> use-port(p)
  | Err msg -> print("Error: ${msg}")

# Using in function signatures
divide = fn(a: Int, b: NonZero) => a / b
```

### Linear Types
```kit
# Linear: must use exactly once (prevents use-after-free)
process-file = fn(path) =>
  handle @linear = File.open path
  content = File.read-all handle
  File.close handle  # Consumes the handle
  content

# Affine: can use at most once (optional cleanup)
maybe-process = fn(path) =>
  resource @affine = allocate-memory()
  if should-process path then
    result = process resource
    free resource  # Optional cleanup
    Some result
  else
    None  # Can skip cleanup

# Relevant: must use at least once (must flush)
log-batch = fn(logger @relevant, messages) =>
  List.each messages (fn(msg) => Logger.log logger msg)
  Logger.flush logger  # Must use at least once

# Parameter linearity
process-stream = fn(stream @linear) =>
  data = Stream.read stream
  Stream.close stream
  data
```

### Match Macros
```kit
# is: Check if value matches pattern (returns Bool)
opt = Some 42
if opt is Some _ then "has value" else "empty"

# as: Extract value with default
name = user as Some u -> u.name else "anonymous"

# guard: Bind or return early
process = fn(opt) =>
  guard Some value = opt else Err "was None"
  guard true = value > 0 else Err "not positive"
  Ok (value * 2)
```

### Compile-Time Macros
```kit
# Define macros with quasiquote syntax
macro double x = ` ($x + $x)
macro square x = ` ($x * $x)
macro unless cond then-val else-val =
  ` (if not($cond) then $then-val else $else-val)

# Use macros (expanded at compile time)
result = double(5)         # expands to (5 + 5)
squared = square(4)        # expands to (4 * 4)
msg = unless((x > 0), "negative", "positive")
```

### Row Polymorphism
```kit
# Closed record (default) - requires exact fields
greet-closed = fn(person: {name: String}) =>
  "Hello, " ++ person.name

greet-closed({name: "Kit"})           # OK
# greet-closed({name: "Kit", age: 1}) # Type error: extra field

# Open record (opt-in with ...) - accepts extra fields
greet-open = fn(person: {name: String, ...}) =>
  "Hello, " ++ person.name

greet-open({name: "Kit"})             # OK
greet-open({name: "Kit", age: 1})     # OK - extra field allowed

# Empty open record - accepts any record
log-record = fn(r: {...}) => println(r)
```

### Field Accessor Shorthand
```kit
# .field is shorthand for fn(x) => x.field
users = [{name: "Alice", age: 30}, {name: "Bob", age: 25}]

# These are equivalent:
names1 = users |> map(fn(u) => u.name)
names2 = users |> map .name

# Useful in pipelines
users |> filter(fn(u) => u.age > 21) |> map .name
```

### Destructuring in Bindings
```kit
# Tuple destructuring
point = (10, 20)
(x, y) = point
println "x = ${x}, y = ${y}"

# Record destructuring with shorthand syntax
person = {name: "Alice", age: 30}
{name, age} = person
println "Name: ${name}, Age: ${age}"

# Record destructuring with explicit rename
{name: user-name, age: user-age} = person
println "User: ${user-name}"

# Partial destructuring (only extract needed fields)
config = {host: "localhost", port: 8080, debug: true}
{port} = config

# Nested destructuring
user = {info: {name: "Bob", email: "bob@example.com"}}
{info: {name, email}} = user

# Mixed tuple and record destructuring
data = ({x: 1, y: 2}, {x: 3, y: 4})
({x: x1, y: y1}, {x: x2, y: y2}) = data
```

---

## Notes

1. **Indentation**: Kit uses significant whitespace for multi-line constructs.
   Indented blocks are used in lambda bodies, if/else branches, match arms,
   and test blocks.

2. **Kebab-case**: Identifiers can contain hyphens (`my-function`), which is
   idiomatic for function names.

3. **Constructor Recognition**: Uppercase identifiers are treated as
   constructors in patterns.

4. **Negative Numbers in Application**: When using Haskell-style function
   application, negative numbers must be parenthesized: `func (-1)` not
   `func -1`.

5. **Attributes**: Attributes provide metadata annotations on constructor fields
   using the `@` prefix. Attribute names can be hyphenated (`@primary-key`) and
   can take arguments (`@default(0)`, `@fkey(User, id)`). Common uses include:
   - **ORM**: `@primary-key`, `@auto-increment`, `@unique`, `@index`, `@fkey`
   - **Serialization**: `@json("fieldName")`, `@json-ignore`
   - **Validation**: `@min(0)`, `@max(100)`, `@pattern("[a-z]+")`
   - **Documentation**: `@doc("description")`, `@deprecated`

6. **Refinement Types**: Refinement types use the syntax `{binding: Type | predicate}`
   following Liquid Haskell. Construction uses `T!(expr)` for assert, `T?(expr)` for
   Option, or `T?!(expr)` for Result. See `docs/design/refinement-types.md`.

7. **For Expressions**: For loops (`for x in list => body`) desugar to `each`.
   They support pattern destructuring in the binding.

8. **SQL Expressions**: SQL blocks support string interpolation with `${expr}`.
   The optional connection expression allows specifying a database connection,
   and `as Type` enables automatic result type casting.

9. **Heredocs**: Squiggly heredocs (`<<~DELIM`) strip common leading indentation
   from the content, making embedded code and markup easier to read.

10. **Contracts**: `@pre` and `@post` attributes on bindings define preconditions
    and postconditions that are checked at runtime. Use with functions to enforce
    invariants.

11. **Zero-arity Functions**: Lambda expressions without parameters can omit
    the parentheses: `fn => expr` is equivalent to `fn() => expr`.

12. **Lazy-accepting Parameters**: The `~` sigil before a parameter indicates it
    accepts lazy values without auto-forcing. Regular parameters auto-force any
    `Lazy` or `Memo` values passed to them. Example: `fn(~value) => ...`.

13. **Match Macros**: The `is`, `as`, and `guard` macros provide concise pattern
    matching. `is` returns a boolean, `as` extracts with a default, and `guard`
    binds or returns early. These expand to equivalent match expressions.

14. **Compile-Time Macros**: User-defined macros use quasiquote syntax: `` `(expr) ``
    creates a code template, and `$param` substitutes parameters. Macros are
    expanded at compile time before type checking.

15. **Row Polymorphism**: Record types are closed by default (require exact fields).
    Add `...` to make them open: `{name: String, ...}` accepts records with
    at least a `name` field. Use `{...}` to accept any record.

16. **Field Accessor Shorthand**: The syntax `.field` is shorthand for
    `fn(x) => x.field`, useful in pipelines: `users |> map .name`.

17. **Destructuring in Bindings**: Tuple patterns `(a, b) = tuple` and record
    patterns `{x, y} = record` can be used in bindings to extract values. Record
    shorthand `{x}` means `{x: x}` (bind field `x` to variable `x`). Use
    `{field: name}` to rename, and `{field: _}` to ignore fields.

18. **Error Propagation**: The postfix `?!` operator unwraps `Ok`/`Some` values
    or returns early from the enclosing function with `Err`/`None`. It is parsed
    at call-expression level (same precedence as `.` and function application).

19. **Actor Send**: The `<-` operator sends a message to an actor:
    `actor <- message` desugars to `Actor.send actor message`. Its precedence
    is between pipe operators and logical OR.

20. **Keyword Aliases**: `or` can be used in place of `||`, and `and` can be
    used in place of `&&`.
