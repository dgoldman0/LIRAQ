# 04 — LEL: LIRAQ Expression Language

**LIRAQ v1.0**

---

## 1 Purpose

LEL is the expression language used throughout LIRAQ for data bindings,
conditional logic, computed state values, and behavior conditions. It is
designed to be:

- **Pure.** No side effects. An expression's value depends only on its inputs.
- **Total.** Every well-formed expression produces a value. No expression
  diverges or throws.
- **Deterministic.** The same inputs always produce the same output.
- **Compact.** Maximum expression length is 2048 characters.

LEL is intentionally not a programming language. It has no variables, no
assignment, no loops, no user-defined functions. It is a formula language.

---

## 2 Syntax

### 2.1 Literals

| Type | Syntax | Examples |
|------|--------|---------|
| String | Single quotes | `'hello'`, `'it''s'` (escaped single quote) |
| Integer | Digits, optional leading `-` | `42`, `-7`, `0` |
| Float | Digits with `.` | `3.14`, `-0.5`, `0.0` |
| Boolean | Keywords | `true`, `false` |
| Null | Keyword | `null` |

### 2.2 State References

Bare dotted identifiers resolve against the state tree:

```
user.name                → value at path user.name
ship.systems.warp.status → value at path ship.systems.warp.status
_surfaces.active         → runtime surface list
```

References to non-existent paths evaluate to `null`.

### 2.3 Infix Operators

Arithmetic, comparison, and logic operators use conventional infix syntax.
Infix operators are syntactic sugar over the built-in functions (§3) — they
follow the same totality rules (division by zero → 0, type mismatch →
0/false/'').

| Category | Operators | Precedence (high → low) |
|----------|-----------|------------------------|
| Unary | `not`, `-` (negation) | 1 |
| Multiplicative | `*`, `/`, `%` | 2 |
| Additive | `+`, `-` | 3 |
| Comparison | `>`, `>=`, `<`, `<=` | 4 |
| Equality | `==`, `!=` | 5 |
| Logical AND | `and` | 6 |
| Logical OR | `or` | 7 |
| Conditional | `... ? ... : ...` | 8 (lowest) |

```
ship.power > 80 ? 'nominal' : ship.power > 40 ? 'reduced' : 'critical'
sensors.pressure * 1.5 + offset
user.role == 'admin' and feature.enabled
```

**String concatenation via `+`.** When either operand is a string, `+`
concatenates (coercing the other operand via `to-string`). This matches the
`concat()` function for the common two-operand case. Use `concat()` for
variadic concatenation (three or more segments).

**Parenthesized grouping.** `(expr)` overrides default precedence:

```
(ship.power + ship.reserve) * efficiency
```

### 2.4 Function Calls

```
function-name(arg1, arg2, ...)
```

All built-in functions (§3) can also be called in prefix form. The function-
call syntax is equivalent to the corresponding infix operator and remains
valid — it is not deprecated.

```
add(ship.power, 10)          — equivalent to: ship.power + 10
gt(sensors.pressure, 90)     — equivalent to: sensors.pressure > 90
concat('Alert: ', msg)       — equivalent to: 'Alert: ' + msg
```

The function-call form is preferred when deeply nesting or when variadic
(e.g., `concat(a, b, c, d)` is cleaner than `a + b + c + d`).

### 2.5 Nesting

Expressions can be nested to arbitrary depth (within the 2048-character limit):

```
ship.power > 80 ? 'nominal' : ship.power > 40 ? 'reduced' : 'critical'
```

The equivalent function-call form is also valid:

```
if(gt(ship.power, 80), 'nominal', if(gt(ship.power, 40), 'reduced', 'critical'))
```

### 2.6 Context Variables

Within collection templates, special variables are available:

| Variable | Meaning |
|----------|---------|
| `_item` | Current collection item |
| `_index` | Zero-based index of current item |

```
concat(_item.name, ' (', _item.role, ')')
```

---

## 3 Built-in Functions

### 3.1 Arithmetic

| Function | Signature | Returns | Notes |
|----------|-----------|---------|-------|
| `add(a, b)` | number, number → number | Sum | |
| `sub(a, b)` | number, number → number | Difference | |
| `mul(a, b)` | number, number → number | Product | |
| `div(a, b)` | number, number → number | Quotient | Returns `0` if `b` is `0` |
| `mod(a, b)` | number, number → number | Remainder | Returns `0` if `b` is `0` |
| `neg(a)` | number → number | Negation | |
| `abs(a)` | number → number | Absolute value | |
| `round(a)` | number → integer | Round to nearest | |
| `floor(a)` | number → integer | Round down | |
| `ceil(a)` | number → integer | Round up | |
| `min(a, b)` | number, number → number | Smaller value | |
| `max(a, b)` | number, number → number | Larger value | |
| `clamp(v, lo, hi)` | number × 3 → number | Clamped value | |

### 3.2 Comparison

| Function | Signature | Returns |
|----------|-----------|---------|
| `eq(a, b)` | any, any → boolean | `true` if equal |
| `neq(a, b)` | any, any → boolean | `true` if not equal |
| `gt(a, b)` | number, number → boolean | `true` if `a > b` |
| `gte(a, b)` | number, number → boolean | `true` if `a >= b` |
| `lt(a, b)` | number, number → boolean | `true` if `a < b` |
| `lte(a, b)` | number, number → boolean | `true` if `a <= b` |

Comparing incompatible types returns `false` (never fails).

### 3.3 Logic

| Function | Signature | Returns |
|----------|-----------|---------|
| `and(a, b)` | boolean, boolean → boolean | Logical AND |
| `or(a, b)` | boolean, boolean → boolean | Logical OR |
| `not(a)` | boolean → boolean | Logical NOT |
| `if(cond, then, else)` | boolean, any, any → any | Conditional |
| `coalesce(a, b)` | any, any → any | First non-null |

### 3.4 String

| Function | Signature | Returns |
|----------|-----------|---------|
| `concat(a, ...)` | any... → string | Concatenation (variadic) |
| `upper(s)` | string → string | Uppercase |
| `lower(s)` | string → string | Lowercase |
| `trim(s)` | string → string | Whitespace trimmed |
| `length(s)` | string → integer | Character count |
| `substring(s, start, end)` | string, int, int → string | Substring (0-based, exclusive end) |
| `contains(s, sub)` | string, string → boolean | Substring test |
| `starts-with(s, prefix)` | string, string → boolean | Prefix test |
| `ends-with(s, suffix)` | string, string → boolean | Suffix test |
| `replace(s, old, new)` | string × 3 → string | First occurrence replacement |
| `split(s, delimiter)` | string, string → array | Split to array |
| `join(arr, delimiter)` | array, string → string | Join array to string |
| `format(n, pattern)` | number, string → string | Number formatting |

`format` patterns: `'0'` = integer, `'0.0'` = one decimal, `'0.00'` = two
decimals, `'0%'` = percentage (multiplied by 100).

### 3.5 Array

| Function | Signature | Returns |
|----------|-----------|---------|
| `length(arr)` | array → integer | Item count |
| `at(arr, index)` | array, integer → any | Item at index (null if out of bounds) |
| `first(arr)` | array → any | First item (null if empty) |
| `last(arr)` | array → any | Last item (null if empty) |
| `includes(arr, val)` | array, any → boolean | Membership test |
| `slice(arr, start, end)` | array, int, int → array | Sub-array |
| `count(arr, field, value)` | array, string, any → integer | Count items where field equals value |
| `filter(arr, field, value)` | array, string, any → array | Items where field equals value |
| `map-field(arr, field)` | array, string → array | Extract one field from each item |
| `sort-by(arr, field)` | array, string → array | Sort by field (ascending) |
| `reverse(arr)` | array → array | Reverse order |

### 3.6 Type

| Function | Signature | Returns |
|----------|-----------|---------|
| `type-of(v)` | any → string | Type name: `'string'`, `'integer'`, `'float'`, `'boolean'`, `'null'`, `'array'`, `'object'` |
| `to-string(v)` | any → string | String coercion |
| `to-number(v)` | any → number | Numeric coercion (0 if not parseable) |
| `to-boolean(v)` | any → boolean | Truthy coercion |
| `is-null(v)` | any → boolean | Null test |
| `literal(v)` | any → any | Returns argument verbatim (prevents path resolution) |

---

## 4 Evaluation Rules

### 4.1 Totality

Every expression MUST produce a value:

- Division by zero returns `0`.
- Out-of-bounds array access returns `null`.
- Non-existent state paths return `null`.
- Type mismatches in arithmetic return `0`.
- Type mismatches in string operations return `''`.
- Type mismatches in comparisons return `false`.

### 4.2 Short-circuit Evaluation

- `if(cond, then, else)` evaluates only the selected branch.
- `and(a, b)` does not evaluate `b` if `a` is `false`.
- `or(a, b)` does not evaluate `b` if `a` is `true`.
- `coalesce(a, b)` does not evaluate `b` if `a` is non-null.

### 4.3 Binding Prefix

In UIDL attributes, LEL expressions are prefixed with `=` to distinguish them
from static values:

```xml
<label id="x" text="Hello" />          <!-- static string -->
<label id="y" bind="=user.name" /> <!-- LEL expression -->
```

The `=` prefix is **not** part of the LEL syntax; it is a UIDL convention.

### 4.4 Expression Limits

| Limit | Value |
|-------|-------|
| Maximum length | 2048 characters |
| Maximum nesting depth | 32 levels |
| Maximum function arguments | 16 |
| Maximum `concat` arguments | 32 |

---

## 5 Grammar

```ebnf
expression     = ternary

ternary        = or-expr ( '?' expression ':' expression )?
or-expr        = and-expr ( 'or' and-expr )*
and-expr       = equality ( 'and' equality )*
equality       = comparison ( ( '==' | '!=' ) comparison )?
comparison     = additive ( ( '>' | '>=' | '<' | '<=' ) additive )?
additive       = multiplicative ( ( '+' | '-' ) multiplicative )*
multiplicative = unary ( ( '*' | '/' | '%' ) unary )*
unary          = ( 'not' | '-' ) unary | postfix
postfix        = primary ( '(' arg-list? ')' | '.' path-segment )*
primary        = '(' expression ')' | literal-value | context-var | identifier

arg-list       = expression ( ',' expression )*

state-ref      = identifier ( '.' path-segment )*
path-segment   = identifier | integer-literal

context-var    = '_item' ( '.' path-segment )* | '_index'

literal-value  = string-lit | integer-lit | float-lit | bool-lit | null-lit

string-lit     = "'" ( [^'] | "''" )* "'"
integer-lit    = '-'? [0-9]+
float-lit      = '-'? [0-9]+ '.' [0-9]+
bool-lit       = 'true' | 'false'
null-lit       = 'null'

identifier     = [a-z] [a-z0-9-]*
```

### 5.1 Ambiguity Resolution

A bare identifier followed by `(` is a function call. A bare identifier
followed by `.` or end-of-expression is a state reference. This is unambiguous
because function names and state path segments share the same character set but
function calls always require parentheses.

Infix operators bind by precedence (§2.3). Parenthesized grouping `(expr)`
overrides precedence. The parser distinguishes grouping parentheses from
function-call parentheses by context: `(` following an identifier is a
function call; `(` at expression-start or after an operator is grouping.

### 5.2 Infix / Prefix Equivalence

Every infix expression has an equivalent function-call form:

| Infix | Function-call equivalent |
|-------|-------------------------|
| `a + b` | `add(a, b)` |
| `a - b` | `sub(a, b)` |
| `a * b` | `mul(a, b)` |
| `a / b` | `div(a, b)` |
| `a % b` | `mod(a, b)` |
| `-a` | `neg(a)` |
| `a > b` | `gt(a, b)` |
| `a >= b` | `gte(a, b)` |
| `a < b` | `lt(a, b)` |
| `a <= b` | `lte(a, b)` |
| `a == b` | `eq(a, b)` |
| `a != b` | `neq(a, b)` |
| `a and b` | `and(a, b)` |
| `a or b` | `or(a, b)` |
| `not a` | `not(a)` |
| `c ? t : e` | `if(c, t, e)` |

A conforming implementation MUST accept both forms and produce identical
results.

---

## 6 Extension Points

Implementations MAY register additional built-in functions. Custom functions:

- MUST be pure (no side effects).
- MUST be total (always return a value).
- MUST use `kebab-case` names.
- SHOULD be documented in the runtime's capability declaration.
- MUST NOT shadow standard built-in names.
