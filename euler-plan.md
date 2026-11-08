# Euler — Design and Implementation Plan

**Status:** Draft for review  
**Date:** 2026-11-04  
**Project:** Euler, a unit-aware symbolic calculator with a WYSIWYG LaTeX editor  
**Working directory:** `/home/kaarlo/Projects/euler` (currently empty)

---

## 1. Summary

Euler is a browser-based, unit-aware symbolic calculator. The user writes mathematics in a WYSIWYG LaTeX editor (MathQuill), and the result is computed locally in a Rust core compiled to WebAssembly. The defining feature is first-class physical units and physical constants: expressions such as

```latex
12\,\mathrm{V}\cdot 0.5\,\mathrm{A}
```

evaluate to `6\,\mathrm{W}`, while ordinary symbolic algebra still works:

```latex
x^2 - 5x + 6 = 0
```

evaluates to `x = 2` or `x = 3`.

The architecture is deliberately small:

- **One Rust core** (`euler-core`) containing the LaTeX parser, unit engine, constants registry, symbolic engine, and formatter.
- **One WASM wrapper** (`euler-wasm`) exposing a JSON-based API to the browser.
- **One static frontend** (Vite + TypeScript + Svelte) with MathQuill for input and KaTeX for output.
- **No server, no accounts, no network calls at runtime.**

The first version uses a focused, built-in Rust CAS rather than an external CAS. This avoids licensing complications, keeps the WASM bundle small, and gives the unit layer complete control over symbolic values. A narrow `CasEngine` boundary is kept so a larger CAS can be added later without rewriting the parser, units, or UI.

---

## 2. Goals and non-goals

### 2.1 Goals

1. **WYSIWYG math input.** The user types standard math notation, not a programming language.
2. **Correct unit arithmetic.** Dimensions are tracked through every operation; incompatible operations are rejected with a clear explanation.
3. **Automatic derived units.** `A·V` becomes `W`, `V/A` becomes `Ω`, `J/s` becomes `W`, etc., with sensible handling of ambiguous cases such as `N·m`.
4. **A large physical-constants library.** Common CODATA constants are available by short aliases, with values, units, and uncertainties.
5. **A useful symbolic calculator.** Exact arithmetic, simplification, expansion, factoring, equation solving, differentiation, basic integration, and small matrices.
6. **A TI-Nspire-like worksheet.** Multiple lines, persistent variables, editable history, exact/approximate toggle.
7. **Fast, local, private.** Everything runs in the browser; typical expressions evaluate in milliseconds.
8. **A codebase that can grow.** Clear module boundaries, data-driven units/constants, and a replaceable CAS.

### 2.2 Non-goals for v1

These are explicitly out of scope for the first usable version. They are listed so that the implementation does not accidentally grow into a multi-year CAS project.

- General-purpose computer algebra (full Risch integration, limits, series, Gröbner bases, number theory).
- Complex numbers.
- Plots and geometry.
- Uncertainty propagation through calculations (constants store uncertainty, but v1 does not propagate it).
- Unit-aware matrices (see §10.7).
- User-defined functions and programming constructs.
- Symbolic vectors/tensors.
- Currency and information units.
- Server-side computation, accounts, collaboration, or cloud storage.
- A mobile-native app.

### 2.3 Product principles

- **Exact first, decimal on demand.** Prefer exact rational and symbolic results; show an approximate form when useful.
- **Never guess silently.** If a unit is ambiguous or a CAS operation is unsupported, say so and show the unevaluated expression. Do not return a plausible but wrong answer.
- **Units are explicit.** Units are written inside `\mathrm{...}`. Plain italic identifiers are variables. This removes the usual ambiguity between `m` (mass/variable) and `\mathrm{m}` (metre).
- **Display both forms when useful.** For ambiguous unit simplifications, show the primary result and a secondary SI-equivalent hint (e.g. `6 N·m` with `= 6 J`).
- **Small core, rich data.** Most of the breadth comes from generated unit/constant tables, not from hand-written special cases.
---

## 3. What the user sees

### 3.1 Worksheet model

The main screen is a vertical worksheet, similar in spirit to TI-Nspire CAS:

```
┌───────────────────────────────────────────────────────────────┐
│  m := 2\,\mathrm{kg}                              = 2 kg      │
│  v := 3\,\mathrm{m/s}                             = 3 m/s     │
│  \frac{1}{2} m v^2                                = 9 J       │
│  \frac{1}{2} m v^2 \to \mathrm{kJ}                = 0.009 kJ  │
│                                                               │
│  x^2 - 5x + 6 = 0                                 x = 2 ∨ 3  │
│                                                               │
│  ▎ 12\,\mathrm{V}\cdot 0.5\,\mathrm{A}            = 6 W       │
└───────────────────────────────────────────────────────────────┘
```

- Each line is an independent math field.
- `Enter` evaluates the current line and creates/focuses the next line.
- `Shift+Enter` inserts a newline? (MathQuill is single-line; in v1, `Shift+Enter` always creates a new line without evaluating.)
- `Up`/`Down` move between lines when the cursor is at the edge of the field.
- `Escape` leaves the editor and returns to the worksheet.
- Results appear to the right of the input on wide screens and below it on narrow screens.
- Clicking a result expands a detail panel: exact form, approximate form, unit breakdown, constant uncertainty, and a copy button.
- A settings panel controls exact/approximate default, significant digits, angle mode, and unit-display preference.
- A searchable **Units & Constants** palette inserts the correct LaTeX into the active line. This is important because typing `\mathrm{...}` by hand is tedious.
- Variables persist top-to-bottom. Editing an earlier line invalidates downstream results; the worksheet replays the affected lines.

### 3.2 Evaluation semantics

- **Assignment** uses `:=` (or `\coloneqq`): `m := 2\,\mathrm{kg}`.
- **Equation** uses `=`: `x^2 - 5x + 6 = 0`. A line that is an equation with one free variable is solved automatically.
- **Unit conversion** uses `\to` (also accept `->` and `\rightarrow`): `2\,\mathrm{km} \to \mathrm{m}`.
- **Function call** uses standard LaTeX: `\sin(x)`, `\ln(x)`, `\det(A)`, `\operatorname{solve}(..., x)`.
- **Implicit multiplication** is supported: `2x`, `2\,\mathrm{m}`, `\frac{1}{2} m v^2`.
- **Mixed units** are normalized internally to SI base units; the display unit is derived from the input and from a preference table.
- **Exact/approximate:** exact rationals and symbolic constants stay exact; floats and measured constants are marked approximate. A toggle shows the decimal form.

### 3.3 Syntax contract

Euler accepts a deliberately small, well-defined subset of LaTeX. The parser is not a TeX engine.

| Construct | Accepted forms | Notes |
|---|---|---|
| Numbers | `2`, `3.14`, `1.2\times10^{-3}`, `\frac{1}{2}` | Decimal input becomes an exact rational, not a binary float. |
| Variables | `x`, `y`, `x_1`, `\alpha`, `\theta` | Plain identifiers and Greek letters. |
| Units | `\mathrm{m}`, `\mathrm{km}`, `\mathrm{A}`, `\mathrm{V}`, `\mathrm{kg\cdot m/s^2}` | Units live inside `\mathrm{...}`. |
| Unit expressions | `\mathrm{m/s}`, `\mathrm{kg\cdot m^2/s^3}`, `\mathrm{N\cdot m}` | Prefixes are resolved inside the unit expression. |
| Constants | `\mathrm{c}`, `\mathrm{c_0}`, `\mathrm{N_A}`, `\mathrm{k_B}`, `\mathrm{h_P}` | Same `\mathrm{}` namespace, with aliases to avoid unit collisions. |
| Operators | `+ - \cdot \times \div / ^ ! \%` | `\pm` is not supported in v1. |
| Relations | `= \ne < \le > \ge` | Equations and inequalities; v1 solves equations, not inequalities. |
| Functions | `\sin \cos \tan \ln \log \exp \sqrt \abs \min \max \det \operatorname{...}` | Function names are case-insensitive except where noted. |
| Grouping | `( )`, `\left( \right)`, `[ ]`, `\left[ \right]`, `\{ \}` | `\left`/`\right` are ignored semantically. |
| Roots | `\sqrt{x}`, `\sqrt[3]{x}` | `\sqrt[3]{x}` becomes `x^{1/3}`. |
| Fractions | `\frac{a}{b}`, `a/b` | `\frac` is preferred for display. |
| Matrices | `\begin{pmatrix}1&2\\3&4\end{pmatrix}` | Also `bmatrix`; matrix support is gated by an editor spike. |
| Text/spacing | `\, \; \! \quad \qquad` | Ignored for semantics. |
| Unicode | `° µ Ω · × −` | Normalized to LaTeX equivalents before parsing. |

Not supported in v1: `\sum`, `\prod`, `\int` with general limits (basic integrals only), `\lim`, `\begin{cases}`, `\begin{aligned}`, chemistry (`\ce{}`), and plotting commands.

### 3.4 Motivating examples

```latex
2\,\mathrm{km} + 300\,\mathrm{m} \to \mathrm{km}
% 2.3 km

12\,\mathrm{V}\cdot 0.5\,\mathrm{A}
% 6 W

\frac{100\,\mathrm{km}}{2\,\mathrm{h}} \to \mathrm{m/s}
% 13.888... m/s

\frac{1}{2} m v^2 \quad\text{with } m=2\,\mathrm{kg},\ v=3\,\mathrm{m/s}
% 9 J

\mathrm{c}^2 \cdot 1\,\mathrm{kg}
% exact: c^2 kg; approximate: 8.9875517873681764e16 J

\sin(30^\circ)
% 1/2

20\,\mathrm{^\circ C} \to \mathrm{K}
% 293.15 K

1\,\mathrm{in} \to \mathrm{mm}
% 25.4 mm

x^2 - 5x + 6 = 0
% x = 2 or x = 3

\frac{d}{dx}\left(x^3\,\mathrm{m}\right)
% 3x^2 m

\int 3x^2\,dx
% x^3 + C

\det\begin{pmatrix}1&2\\3&4\end{pmatrix}
% -2
```

### 3.5 Error experience

Errors are part of the product, not an afterthought.

- Parse errors show the offending LaTeX fragment and a column.
- Unknown units offer suggestions (`\mathrm{metre}` → “did you mean `\mathrm{m}`?”).
- Dimension errors explain the mismatch in human terms:
  `Cannot add 2 m (length) and 3 s (time).`
- Unsupported CAS operations return the unevaluated expression plus a reason:
  `∫ sin(x^2) dx — integration of this form is not implemented in v1.`
- A fuel/time limit prevents the UI from hanging on a pathological simplification or matrix.
---

## 4. Architecture

### 4.1 One core, one boundary

```
┌─────────────────────────────────────────────────────────────────────┐
│ Browser                                                             │
│                                                                     │
│  MathQuill input ──► LaTeX normalizer ──► WASM bridge               │
│                                              │                      │
│                                              ▼                      │
│                                  ┌─────────────────────────┐        │
│                                  │ euler-core (Rust/WASM)  │        │
│                                  │                         │        │
│                                  │  lexer / parser         │        │
│                                  │  AST + resolver         │        │
│                                  │  unit engine            │        │
│                                  │  constants registry     │        │
│                                  │  evaluator + CAS        │        │
│                                  │  formatter              │        │
│                                  └───────────┬─────────────┘        │
│                                              │ JSON result          │
│                                              ▼                      │
│  KaTeX result ◄── worksheet state ◄── result normalizer             │
│                                                                     │
│  localStorage autosave (source lines only)                          │
└─────────────────────────────────────────────────────────────────────┘
```

There is deliberately **no second CAS process and no Web Worker in v1**. The Rust/WASM call is synchronous and fast for the supported operations. If profiling later shows that a long symbolic operation blocks the UI, the same JSON API can be moved into a worker without changing the core or the UI contract.

### 4.2 Crate responsibilities

| Crate/module | Responsibility | Must not depend on |
|---|---|---|
| `euler-core::expr` | Symbolic expression tree, numbers, ordering, canonicalization | Units, parser, WASM |
| `euler-core::latex` | Lexer, parser, AST construction, LaTeX normalization | Evaluation, units |
| `euler-core::units` | Dimensions, unit definitions, prefixes, conversion, display simplification | Parser, CAS |
| `euler-core::constants` | Constant definitions, aliases, lookup, uncertainty metadata | Parser, CAS |
| `euler-core::value` | `Quantity`, `Matrix`, `Value`, numeric formatting | Parser |
| `euler-core::cas` | Simplify, expand, factor, solve, diff, integrate, matrix algebra | Parser, units |
| `euler-core::eval` | Name resolution, environment, operator semantics, unit-aware evaluation | WASM |
| `euler-core::format` | Value → LaTeX/plain text | WASM |
| `euler-wasm` | `wasm-bindgen` API, JSON serialization, panic handling | Frontend |
| `web` | Worksheet UI, MathQuill adapter, KaTeX rendering, persistence | Rust internals |

The dependency direction is strictly downward:

```
euler-wasm → euler-core::{eval, format}
eval → {latex, units, constants, value, cas}
cas → {expr, value}
units → {expr, value}
constants → {expr, value}
latex → expr
format → {expr, value, units}
```

### 4.3 Evaluation pipeline

For a line such as

```latex
12\,\mathrm{V}\cdot 0.5\,\mathrm{A}
```

the pipeline is:

1. **Normalize** the MathQuill LaTeX (remove `\left`/`\right`, spacing, unify operators, normalize Unicode).
2. **Lex** into tokens: number, identifier, unit command, operator, delimiter.
3. **Parse** into an AST with source spans.
4. **Resolve** names:
   - plain identifiers → variables or function names;
   - `\mathrm{...}` → unit, constant, or function (in that order, with aliases).
5. **Evaluate** with dimension tracking:
   - `12 V` → quantity `12`, dimension `M L² T⁻³ I⁻¹`, display `V`;
   - `0.5 A` → quantity `0.5`, dimension `I`, display `A`;
   - multiplication → magnitude `6`, dimension `M L² T⁻³`, display `V·A`;
   - display simplification → `W`.
6. **Format** to LaTeX: `6\,\mathrm{W}`.
7. **Return** a structured JSON result with exact/approximate forms, warnings, and optional unit breakdown.

### 4.4 Source spans and errors

- The lexer records byte spans for every token.
- The parser builds `Spanned<T>` nodes for expressions.
- Every error carries a `Span`, a machine-readable `ErrorCode`, a human message, and an optional hint.
- The frontend shows the error under the line and can display the exact offending substring. Inline underlining inside MathQuill is not attempted in v1; it is brittle and not worth the complexity.

---

## 5. LaTeX parsing

### 5.1 Why a custom parser

A general TeX parser is unnecessary and would accept syntax the evaluator cannot handle. A hand-written recursive-descent parser for a documented subset is smaller, faster, and gives precise errors. It also lets us define precedence and implicit multiplication explicitly.

### 5.2 Lexer

Token kinds:

```rust
enum TokenKind {
    Number(String),
    Ident(String),          // x, y, alpha, x_1
    Command(String),        // \frac, \sqrt, \sin, \mathrm, \cdot
    UnitBlock(String),      // contents of \mathrm{...}
    Op(Op),                 // + - * / ^ _ ! = < <= > >=
    LParen, RParen, LBracket, RBracket,
    Comma, Semicolon, Amp,
    End,
}
```

Lexer rules:

- Commands are `\` followed by ASCII letters, or a single non-letter (`\{`, `\}`, `\%`, `\,`).
- Numbers include decimal points and exponent notation (`1.2e-3`, `1.2\times10^{-3}` after parsing).
- Unicode input is normalized first: `°` → `^\circ`, `µ` → `\mu`, `Ω` → `\Omega`, `·` → `\cdot`, `×` → `\times`, `−` → `-`.
- Whitespace and spacing commands (`\,`, `\;`, `\!`, `\quad`, `\qquad`) are insignificant, except inside `\mathrm{...}` where they separate unit factors.
- `\left` and `\right` are consumed and ignored; the following delimiter is tokenized normally.
- Braces are structural. Unbalanced braces produce a parse error with the opening position.

### 5.3 Grammar

Precedence, from lowest to highest:

```
line        := assignment | conversion | equation | expr
assignment  := ident (':=' | '\coloneqq') expr
conversion  := expr '\to' unit_expr
equation    := expr ('=' | '\ne' | '<' | '\le' | '>' | '\ge') expr
expr        := term (('+' | '-') term)*
term        := unary (('*' | '\cdot' | '\times' | '\div' | '/') unary | implicit unary)*
unary       := ('+' | '-') unary | power
power       := postfix ('^' unary)?          // right-associative
postfix     := atom ('!')*
atom        := number | ident | unit_block | constant | function_call
             | '(' expr ')' | '[' expr ']'
             | '\frac' '{' expr '}' '{' expr '}'
             | '\sqrt' ('[' expr ']')? '{' expr '}'
             | matrix
```

Matrix environments are parsed at the atom level: `\begin{pmatrix}` / `\begin{bmatrix}`, `&` separates columns, `\\` separates rows, and `\end{...}` closes the matrix.

Design decisions:

- **Implicit multiplication** is at the same precedence as explicit multiplication and is left-associative. Therefore `1/2x` parses as `(1/2)·x`. Users who mean `1/(2x)` should write `\frac{1}{2x}`. This is documented in the syntax guide.
- **Power is right-associative:** `2^{3^2}` = `2^9`.
- **Function application** accepts both `\sin(x)` and `\sin x`. The parser treats a known function command followed by an atom as a call. `\sin^2 x` is parsed as `(\sin x)^2`.
- **Unary minus** binds looser than power: `-x^2` = `-(x^2)`, matching standard notation.
- **Factorial** is postfix and only valid on dimensionless values.
- **Conversion** binds looser than arithmetic but tighter than assignment: `2\,\mathrm{km} + 300\,\mathrm{m} \to \mathrm{km}` converts the whole sum.
- **Equations** are not chained in v1 (`a = b = c` is an error).

### 5.4 The `\mathrm{...}` namespace

`\mathrm{...}` is the explicit “named thing” namespace. Its contents are parsed by a small unit-expression grammar:

```
unit_expr  := unit_factor (('*' | '\cdot' | '/' | space) unit_factor)*
unit_factor := unit_atom ('^' exponent)?
unit_atom  := prefix? unit_symbol | '\circ' | '°' | ...
```

Resolution order inside `\mathrm{...}`:

1. Exact unit symbol (case-sensitive).
2. Exact constant alias (case-sensitive).
3. Known function name if the block is immediately followed by `(` (e.g. `\mathrm{det}(A)`).
4. If the block contains unit operators (`/`, `\cdot`, `*`, `^`), parse it as a unit expression and resolve each atom.
5. Prefixed unit: split a leading SI prefix and look up the remainder as a prefixable unit.
6. Error with suggestions.

Affine temperature units (`\mathrm{^\circ C}`, `\mathrm{^\circ F}`) and degree symbols are matched as whole aliases before the generic unit-expression parser.

This order makes `\mathrm{min}` a minute (exact unit), not milli-inch; `\mathrm{Pa}` a pascal, not peta-year; `\mathrm{cd}` candela, not centi-day. Collisions between units and constants are resolved in favor of units, and the constants table provides non-colliding aliases.

### 5.5 MathQuill normalization

MathQuill's `latex()` output is close to our subset but not identical. The adapter runs a small, tested normalization pass before the Rust parser:

- Strip `\left` and `\right`.
- Strip spacing commands outside `\mathrm{}`.
- Convert `\cdot` / `\times` to canonical `\cdot` tokens (the parser accepts all three).
- Convert `\operatorname{name}` to a known function token when possible.
- Convert `\degree` / `^\circ` to a canonical degree token.
- Remove redundant braces around single tokens only when safe.
- Preserve `\mathrm{...}` exactly; it is semantically meaningful.
- Reject or flag constructs MathQuill can emit but Euler does not support, with a clear message.

A corpus of real MathQuill outputs is captured with Playwright and committed as a test fixture. This prevents editor upgrades from silently breaking the parser.
---

## 6. Values, numbers, and exactness

### 6.1 Number representation

```rust
pub enum Number {
    Rational(BigRational),   // integers, fractions, exact terminating decimals
    Float(f64),              // transcendental results and measured values
}
```

- Integer and rational arithmetic uses `num-bigint` + `num-rational` (pure Rust, WASM-friendly). Do not use `rug`/GMP in v1.
- Decimal input such as `0.1` is parsed as the exact rational `1/10`, not as a binary float. This makes `0.1 + 0.2 = 0.3` exact.
- Transcendental evaluations (`\sin(1)`, `\ln(2)`, `\sqrt{2}` when not a perfect square) produce `Float`. The result is marked approximate.
- Symbolic constants (`\pi`, `e`, physical constants) are expression atoms, not floats, until numeric evaluation is requested. This allows `\sin(\pi/6)` to simplify to `1/2` exactly.
- No arbitrary-precision floats and no complex numbers in v1.

### 6.2 Expression representation

```rust
pub enum Expr {
    Num(Number),
    Sym(Symbol),          // variable name
    Const(ConstantId),    // \pi, e, c, N_A, ...
    Add(Vec<Expr>),
    Mul(Vec<Expr>),
    Pow(Box<Expr>, Box<Expr>),
    Func(FuncId, Vec<Expr>),
    Matrix(MatrixExpr),
}
```

- `Expr` is a tree in v1. Hash-consing or an arena is an optimization that can be added later without changing the public model.
- `Add` and `Mul` are flattened and canonicalized after every mutation.
- A deterministic total order (`Ord`) is implemented for terms and factors so output is stable and testable.
- `Expr::Const` keeps physical constants symbolic. Numeric substitution happens in `eval_numeric` or when the user requests an approximate result.

### 6.3 Quantity

A quantity is a magnitude plus a dimension plus the unit the user wants to see:

```rust
pub struct Quantity {
    pub mag: Expr,              // magnitude in SI base units (may be symbolic)
    pub dim: Dimension,         // seven SI base dimensions
    pub display: Option<UnitExpr>, // preferred display unit
    pub temp: TempRole,         // None | Absolute | Delta (temperature only)
}
```

Why store the magnitude in SI base units?

- Addition, multiplication, and conversion become simple.
- Symbolic magnitudes compose naturally: `x m + y m` → `(x + y) m`.
- The display unit is a presentation choice, not part of the mathematics.
- Affine units (temperature) are handled in one place: convert to base on input, convert from base on output.

Example:

```text
2 km        → mag = 2000, dim = L, display = km
300 m       → mag = 300,  dim = L, display = m
sum         → mag = 2300, dim = L, display = km  → display as 2.3 km
```

### 6.4 Unit scales are expressions

Not every conversion factor is rational. `30^\circ` is exactly `π/6` radians, and measured constants are decimal floats. Unit scales therefore use an expression, not just a rational:

```rust
pub enum Scale {
    Exact(BigRational),  // 25.4 mm per inch, 1000 m per km
    Symbolic(Expr),      // π/180 for degrees, π for revolutions
    Float(f64),          // measured conversion factors, if any
}
```

This lets `30^\circ \to \mathrm{rad}` produce the exact `\pi/6`, and lets `\sin(30^\circ)` simplify exactly to `1/2` when the CAS recognizes `\sin(\pi/6)`.

### 6.5 Exact vs approximate

- A result is **exact** if it contains only rationals and symbolic constants, and no float was introduced.
- A result is **approximate** if it contains any `Float` (including measured constants when numerically substituted).
- The internal value is never rounded. Rounding happens only in the formatter.
- Significant digits default to 12 and are configurable from 3 to 15.
- The UI shows `=` for exact and `\approx` for approximate. The detail panel always offers the other form when one exists.

### 6.6 Variables and environment

```rust
pub struct Environment {
    vars: BTreeMap<Symbol, Value>,
}
```

- A variable can hold a scalar, a quantity, or a matrix.
- Assignment is explicit with `:=`. `=` is an equation.
- Unassigned identifiers are symbolic variables with no dimension.
- Built-in units and constants cannot be shadowed.
- The environment is derived by replaying worksheet lines top-to-bottom. This makes undo/redo and editing earlier lines simple and deterministic.
- The WASM engine keeps the current environment in Rust; the frontend stores only the source lines and a serialized environment snapshot for autosave.

### 6.7 Value type

```rust
pub enum Value {
    Quantity(Quantity),              // includes dimensionless scalars
    Matrix(Matrix<Quantity>),        // dimensionless entries in v1
    Boolean(bool),
    Equation { lhs: Expr, rhs: Expr }, // simplified but unsolved
}
```

A plain number is simply a `Quantity` with `dim = 0` and no display unit. This avoids a second parallel arithmetic system.

---

## 7. Unit system

### 7.1 Dimensions

Euler tracks the seven SI base dimensions:

| Index | Symbol | Dimension | Base unit |
|---|---|---|---|
| 0 | L | length | metre |
| 1 | M | mass | kilogram |
| 2 | T | time | second |
| 3 | I | electric current | ampere |
| 4 | Θ | thermodynamic temperature | kelvin |
| 5 | N | amount of substance | mole |
| 6 | J | luminous intensity | candela |

```rust
pub struct Dimension(pub [i16; 7]);
```

- Exponents are small integers. `i16` is ample.
- Angle and solid angle follow SI: `rad` and `sr` are dimensionless units with scale 1. This keeps photometry (`lm = cd·sr`) and standard dimensional analysis simple.
- Consequence: `Hz` and `rad/s` share dimensions. Euler does not silently insert a factor of `2π`. Documentation explains this; users can write `2\pi f` explicitly.
- Fractional dimensions are rejected in v1. `\sqrt{4\,\mathrm{m^2}}` → `2 m`; `\sqrt{1\,\mathrm{m}}` returns an unevaluated root with an explanatory warning instead of inventing `m^{1/2}`.
- Temperature is dimension Θ, but affine units get special handling (§7.5).
- Information, currency, and arbitrary user dimensions are out of scope for v1.

### 7.2 Unit definitions

Units are data, not code. A unit definition contains:

```rust
pub struct UnitDef {
    pub symbol: &'static str,       // "m", "W", "degC"
    pub latex: &'static str,        // "\\mathrm{m}", "\\mathrm{^\\circ C}"
    pub name: &'static str,         // "metre"
    pub dim: Dimension,
    pub scale: Scale,               // magnitude in SI base units
    pub offset: Option<Scale>,      // affine units only
    pub prefixable: bool,
    pub aliases: &'static [&'static str],
    pub preferred: bool,            // prefer this when simplifying a dimension
    pub ambiguous: bool,            // e.g. N·m vs J
}
```

Coverage for v1:

- **SI base units:** m, kg, s, A, K, mol, cd.
- **SI derived units:** rad, sr, Hz, N, Pa, J, W, C, V, F, Ω, S, Wb, T, H, °C, lm, lx, Bq, Gy, Sv, kat.
- **Accepted non-SI units:** min, h, d, °, ′, ″, L, t, au, pc, eV, u/Da, Å, barn, bar, atm, mmHg, Torr, psi, gal, cal, kcal, BTU, hp, Wh, kWh, kn, mph, ft, in, yd, mi, nmi, acre, oz, lb, st, ton, and common astronomy/particle units.
- **Prefixes:** y, z, a, f, p, n, µ, m, c, d, da, h, k, M, G, T, P, E, Z, Y.
- **Aliases:** LaTeX spellings and common names (`\Omega`, `ohm`, `Ohm`; `\mu m`, `µm`, `um`; `^\circ C`, `°C`, `degC`; `\Delta` differences are not v1).
- **Exactness:** conversion factors use exact rationals wherever the definition is exact (inch, pound, calorie, eV, atm, light-year, etc.). Measured conversions use floats and are marked approximate.
- **Ambiguous customary units are explicit.** Do not silently choose between US and imperial variants: use `gal_US`/`gal_imp`, `ton_US`/`ton_imp`, `floz_US`/`floz_imp`, `cal_th`/`cal_IT`, `BTU_IT`, `hp_mech`/`hp_metric`. `oz` means mass ounce; `floz` means fluid ounce.
- **Common compound symbols are explicit aliases.** `kWh`, `mph`, `rpm`, `psi`, `mmHg`, and similar symbols are registry entries with their own scale and dimension. The parser does not guess arbitrary concatenations such as `kWh` = `kW·h`.

### 7.3 Prefix resolution

Prefix resolution must be deterministic and must not turn a common unit into a bizarre prefixed one.

Algorithm:

1. If the symbol is an exact unit, use it.
2. Otherwise, try candidate splits `prefix + remainder`, longest prefix first.
3. Accept a split only if the remainder is a known, prefixable unit.
4. If multiple splits are valid, use an explicit exception table for collisions; otherwise prefer the longest prefix.
5. Apply the prefix factor to the unit scale and keep the prefix in the display unit (`km`, `mA`, `µF`).
6. Prefixes are not applied to affine units (`°C`, `°F`) or to already-prefixed units.

Examples:

- `min` → minute (exact unit wins, not milli-inch).
- `Pa` → pascal (exact unit wins, not peta-year).
- `cd` → candela (exact unit wins, not centi-day).
- `mm` → milli-metre.
- `dam` → deca-metre.
- `k\Omega` → kilo-ohm.
- `\mu m` → micro-metre.
- `kg` is the SI base unit for mass and is not prefixable; use `g` for prefixes (`mg`, `Mg`).

### 7.4 Unit expressions

```rust
pub enum UnitExpr {
    Atom { prefix: Option<PrefixId>, unit: UnitId },
    Mul(Vec<UnitExpr>),
    Div(Box<UnitExpr>, Box<UnitExpr>),
    Pow(Box<UnitExpr>, i16),
}
```

- `\mathrm{kg\cdot m/s^2}` parses as `kg · m / s²`.
- The evaluator computes a `UnitValue { dim, scale, offset, display }` from a `UnitExpr`.
- Display simplification cancels identical units in numerator and denominator before applying derived-unit preferences.
- A unit expression can contain constants? No. Constants may have units, but the unit expression itself is only units and prefixes.

### 7.5 Temperature (affine units)

Temperature is the one place where a simple multiplicative scale is wrong. `20 °C` is not `20 × K`.

Rules:

- Internally, absolute temperatures are stored in kelvin: `K = °C + 273.15`, `K = (°F + 459.67) × 5/9`.
- On output, convert back to the requested display unit.
- `TempRole` distinguishes absolute temperatures from temperature differences:
  - `20 °C` → `Absolute`.
  - `20 K` used in an additive context → treated as a difference by default.
  - `20 °C - 10 °C` → `Delta` in K (10 K), not `-263.15 °C`.
  - `20 °C + 5 K` → `Absolute` (25 °C), treating `5 K` as a difference.
  - `20 °C + 30 °C` → error: adding two absolute temperatures is meaningless.
- Multiplication/division of an absolute temperature is allowed only after conversion to K; the result is marked as a warning because it is usually physically meaningless.
- `\Delta` temperature units are not supported in v1; differences are displayed in K.

This is more rigorous than most calculators and avoids the classic `20 °C + 10 °C = 30 °C` mistake.

### 7.6 Unit arithmetic

| Operation | Rule |
|---|---|
| `a + b`, `a - b` | Dimensions must match. Convert `b` to `a`'s display unit, add/subtract magnitudes. Temperature rules above apply. |
| `a * b` | Dimensions add; scales multiply; display units combine. |
| `a / b` | Dimensions subtract; scales divide; display units combine. |
| `a ^ n` | Exponent must be dimensionless. Dimensions multiply by integer `n`. |
| `sqrt(a)` | If all dimension exponents are even, divide them by 2; otherwise return an unevaluated root with a warning. |
| `abs(a)` | Same dimension; magnitude absolute. |
| `min/max(a,b)` | Same dimension; result in the first operand's display unit. |
| `sin/cos/tan` | Argument must be dimensionless or an angle unit; result dimensionless. |
| `asin/acos/atan` | Argument dimensionless; result dimensionless with display `rad` or `deg` according to angle mode. |
| `exp/ln/log` | Argument dimensionless; result dimensionless. |
| `floor/ceil/round` | Argument dimensionless in v1. |
| `diff/ integrate` | See §10.6. |

### 7.7 Display-unit simplification

The result should look like physics, not like a dump of SI base units.

Primary display rules:

1. If the result dimension exactly matches a preferred derived unit and the operation's unit signature maps unambiguously to it, use that unit.
2. Otherwise preserve the natural product/quotient of the operand display units.
3. Cancel identical units first.
4. For ambiguous dimensions, keep the operand form as primary and show the SI equivalent as a secondary hint.

Preference table (excerpt):

| Signature | Primary display | Secondary | Notes |
|---|---|---|---|
| `A·V` | `W` | — | Unambiguous |
| `V/A` | `Ω` | — | Unambiguous |
| `C/s` | `A` | — | Unambiguous |
| `J/s` | `W` | — | Unambiguous |
| `N/m²` | `Pa` | — | Unambiguous |
| `kg·m/s²` | `N` | — | Unambiguous |
| `kg·m²/s²` | `J` | — | Unambiguous |
| `N·m` | `N·m` | `J` | Torque vs energy ambiguity |
| `J/kg` | `J/kg` | `Gy` / `Sv` | Context-dependent |
| `1/s` | `Hz` | `Bq` | Context-dependent |
| `cd·sr` | `lm` | — | `sr` is dimensionless |
| `lm/m²` | `lx` | — | — |

A toolbar command “Simplify units” forces the canonical SI derived/base form when the user wants it.
---

## 8. Physical constants

### 8.1 Data model

Constants are data-driven, like units:

```rust
pub struct ConstantDef {
    pub id: ConstantId,
    pub symbol: &'static str,          // "c", "N_A", "h_P"
    pub aliases: &'static [&'static str],
    pub name: &'static str,            // "speed of light in vacuum"
    pub value: Expr,                   // exact rational or float
    pub unit: &'static str,            // "m/s", "J/(mol*K)"
    pub uncertainty: Option<f64>,      // absolute, in the constant's unit
    pub exact: bool,
    pub codata_year: Option<u16>,      // e.g. 2022
    pub source: &'static str,
}
```

- Exact SI defining constants use exact rational/decimal values: `c`, `h`, `e`, `k_B`, `N_A`, `Δν_Cs`, `K_cd`.
- Derived exact constants use exact expressions where possible, e.g. `R = N_A k_B`.
- Measured constants carry a value and uncertainty from CODATA 2022. The UI shows the uncertainty in the detail panel; v1 does not propagate it.
- Constants can be used symbolically (`Expr::Const`) or numerically. The exact/approximate toggle controls substitution in the result.

### 8.2 Unified `\mathrm{}` namespace and aliases

The resolver checks units first, then constants. To avoid collisions, constants have explicit, documented aliases. Common collisions:

| Desired constant | Recommended alias | Collision / reason |
|---|---|---|
| speed of light | `\mathrm{c}` or `\mathrm{c_0}` | `c` is not a unit |
| Planck constant | `\mathrm{h_P}` | `\mathrm{h}` is hour |
| reduced Planck | `\mathrm{hbar}` or `\hbar` | dedicated alias |
| elementary charge | `\mathrm{e}` or `\mathrm{e_c}` | plain `e` is Euler's number |
| Boltzmann constant | `\mathrm{k_B}` | no unit collision |
| Avogadro constant | `\mathrm{N_A}` | `N` is newton; `N_A` is distinct |
| gas constant | `\mathrm{R_g}` | `R` may be roentgen |
| gravitational constant | `\mathrm{G_0}` | `G` is gauss if included |
| standard gravity | `\mathrm{g_n}` or `\mathrm{g_0}` | `g` is gram |
| vacuum permittivity | `\mathrm{epsilon_0}` | `\varepsilon_0` also accepted |
| vacuum permeability | `\mathrm{mu_0}` | `\mu_0` also accepted |
| electron mass | `\mathrm{m_e}` | `m` is metre/variable |
| proton mass | `\mathrm{m_p}` | — |
| neutron mass | `\mathrm{m_n}` | — |
| fine-structure constant | `\mathrm{alpha}` | no unit collision |

Rules:

- Case is significant. `\mathrm{M}` is the mega prefix (not a unit by itself); `\mathrm{m}` is metre.
- Autocomplete shows both units and constants, with name, symbol, unit, and category. This is the primary discovery mechanism.
- If a user types a colliding bare name, the error message explains the alias, e.g. `\mathrm{h}` is hour; use `\mathrm{h_P}` for the Planck constant.
- Long readable aliases are also provided for difficult symbols: `\mathrm{speed_of_light}`, `\mathrm{planck_constant}`, `\mathrm{avogadro}`, `\mathrm{boltzmann}`.

### 8.3 Dataset scope

Aim for roughly 200–400 constants, generated from CODATA 2022 and supplemented with useful exact definitions:

- Defining SI constants and derived constants.
- Electromagnetic constants.
- Atomic and nuclear constants.
- Particle masses and widths.
- Physico-chemical constants.
- Astrophysical constants.
- Standard gravity, standard atmosphere, and common reference values.
- Mathematical constants: `\pi`, `e`, `\tau`, `\varphi` (golden ratio).

The data file is committed. A `scripts/fetch_codata.py` tool can regenerate the raw table, but builds never require network access. Each entry records its source and CODATA year.

### 8.4 Numeric substitution policy

- In exact mode, physical constants remain symbolic atoms. `E = m c^2` stays `E = m c^2` until `m` is known or the user requests a decimal.
- In approximate mode, constants are replaced by their numeric values.
- A standalone constant always has a numeric detail line: `c = 299792458 m/s (exact)`.
- Unit conversion of a symbolic constant forces numeric substitution for the converted magnitude; the exact symbolic form remains available in the detail panel.
- Measured constants show `value ± uncertainty` in the detail panel, with the CODATA year.

This preserves formula manipulation while still making physical calculations easy.

---

## 9. Symbolic engine (built-in CAS)

### 9.1 Scope and honesty

The built-in CAS is a **focused symbolic calculator**, not a general-purpose CAS. The v1 feature matrix is:

| Capability | v1 | Later |
|---|---|---|
| Exact rational arithmetic | ✅ | — |
| Canonical simplify / collect terms | ✅ | — |
| Expand products and powers | ✅ | — |
| Factor univariate polynomials | ✅ | multivariate, algebraic extensions |
| Solve linear/quadratic equations | ✅ | higher degree, transcendental |
| Solve linear systems via matrices | ✅ | nonlinear systems |
| Differentiate | ✅ | partial derivatives, implicit |
| Integrate standard forms | ✅ | Risch, special functions |
| Definite integrals of standard forms | ✅ | numeric quadrature, improper integrals |
| Matrices (dimensionless) | ✅ | unit-aware matrices, eigenvalues |
| Limits / series | ❌ | later |
| Complex numbers | ❌ | later |
| Assumptions (x > 0, x real) | ❌ | later |

Unsupported operations return an unevaluated expression plus a clear message. This is a feature: the calculator never pretends to have solved something it did not.

### 9.2 Canonical form

`normalize(expr)` produces a deterministic canonical form:

- Flatten nested `Add` and `Mul`.
- Combine numeric coefficients.
- Combine like terms in sums.
- Combine powers with the same base in products.
- Sort terms and factors by a stable total order.
- Apply arithmetic identities: `x + 0 = x`, `x * 1 = x`, `x * 0 = 0`, `x^0 = 1`, `x^1 = x`.
- Keep `0`, `1`, and `-1` in a canonical position.

This canonical form is used for both simplification and display, so equal expressions format identically.

### 9.3 Simplify

`simplify` applies rewrite rules to a fixpoint within a fuel budget:

- Arithmetic identities and rational normalization.
- Collect like terms and factors.
- Combine fractions (`a/b + c/d`).
- Reduce rational expressions by canceling common polynomial factors when easy.
- Basic trig identities: `sin^2 + cos^2 = 1`, `sin(0) = 0`, `cos(0) = 1`, `tan = sin/cos`.
- Exact special angles: `sin(π/6) = 1/2`, `cos(π/3) = 1/2`, etc.
- Log/exp identities only when safe for real positive arguments or when the argument is numeric.
- Do not “simplify” `sqrt(x^2)` to `x` without assumptions; keep it unevaluated or return `|x|` only when `x` is known real.

### 9.4 Expand and factor

- `expand` distributes multiplication over addition and expands integer powers of sums.
- `factor`:
  - Extracts a common monomial factor.
  - Factors univariate polynomials over the rationals using the rational-root theorem and quadratic factoring.
  - Recognizes difference of squares and simple cubics.
  - Leaves irreducible factors intact.
- Multivariate factorization is out of scope; common-factor extraction is still useful.

### 9.5 Solve

Input forms:

- `x^2 - 5x + 6 = 0` → automatically solve for `x`.
- `\operatorname{solve}(eq, x)` → explicit variable.
- `\operatorname{linsolve}(A, b)` → linear system via matrices.

Algorithm:

1. Move everything to one side: `lhs - rhs = 0`.
2. Collect the equation as a polynomial in the target variable.
3. Degree 1: exact solution.
4. Degree 2: exact quadratic formula; simplify surds where possible.
5. Degree > 2: rational roots and simple factoring; otherwise return unsupported.
6. Equations with quantity coefficients: support **linear** equations by treating coefficients as quantities and dividing. Example: `v = d/t`, solve for `t` → `t = d/v`, with dimensions checked.
7. Quadratic equations with unit coefficients: out of scope in v1 (the discriminant’s dimensions need careful handling); return a clear message.
8. If the equation is numeric and true/false, return a boolean.

### 9.6 Differentiation

Rules for `+`, `-`, `*`, `/`, powers, and the standard functions:

- `sin`, `cos`, `tan`, `asin`, `acos`, `atan`
- `sinh`, `cosh`, `tanh`
- `exp`, `ln`, `log`, `sqrt`, `abs`
- constants and variables
- product rule, quotient rule, chain rule
- integer and symbolic powers

Units:

- If the expression is a quantity, differentiate its magnitude and keep the dimension.
- If the variable itself has units (assigned quantity), the result dimension is `expr_dim / var_dim`.
- In v1, symbolic variables are dimensionless unless assigned. Document this.

### 9.7 Integration

Supported in v1:

- Linearity: `∫(af + bg) = a∫f + b∫g`.
- Polynomials: `∫x^n dx = x^{n+1}/(n+1)` for `n ≠ -1`.
- `∫1/x dx = ln|x|`.
- `∫e^{ax} dx`, `∫sin(ax) dx`, `∫cos(ax) dx`.
- `∫1/(ax+b) dx`.
- Simple linear substitution: `∫f(ax+b) dx`.
- Definite integrals of the above by evaluating the antiderivative at the limits.

Not supported: general Risch integration, products of trig functions beyond simple cases, special functions, improper integrals. Return the unevaluated integral with a message.

Units:

- Indefinite integration of a constant quantity multiplies by the integration variable symbol: `∫3 m/s dt = 3 m/s · t`. When `t` is later assigned `2 s`, this becomes `6 m`.
- Definite integrals require limits with compatible units; the result dimension is `integrand_dim × limit_dim`.
- If the integral cannot be dimension-checked, return an error rather than a guess.

### 9.8 Matrices

Types and operations:

```rust
pub struct Matrix<T> {
    rows: usize,
    cols: usize,
    data: Vec<Vec<T>>,
}
```

v1 operations:

- Construction from `\begin{pmatrix}...\end{pmatrix}`.
- Addition, subtraction, scalar multiplication.
- Matrix multiplication.
- Transpose (`A^T` or `\operatorname{transpose}(A)`).
- Determinant via Bareiss elimination for exact entries; Gaussian elimination for symbolic entries.
- Inverse via Gauss-Jordan; error if singular.
- Identity matrix.
- `\operatorname{linsolve}(A, b)` for linear systems.
- Size limit: 20×20, with a fuel budget.

Matrix entries are dimensionless in v1. If a unit-valued entry is supplied, return `unit-aware matrices are not supported in v1`; this avoids incorrect determinant/inverse dimensions. Unit-aware matrices are a later extension.

### 9.9 Fuel and limits

Every CAS operation runs with a `Fuel` counter (default 100,000 rewrite steps) and a wall-clock guard in the WASM wrapper (e.g. 2 seconds for interactive evaluation). On exhaustion:

- Return `ErrorCode::CasFuel`.
- Include the partially simplified expression if available.
- Never panic, never block the UI indefinitely.

This is essential in a browser: symbolic simplification and matrix determinants can explode combinatorially.
---

## 10. Evaluation semantics

### 10.1 Resolution

Evaluation walks the AST and produces a `Value`.

- A plain identifier `x` resolves to:
  1. a local variable from the environment;
  2. a known function if followed by `(`;
  3. otherwise an unassigned symbolic variable.
- A `\mathrm{...}` block resolves to:
  1. an exact unit;
  2. an exact constant;
  3. a prefixed unit;
  4. a known function if followed by `(`;
  5. an error with suggestions.
- Unknown identifiers are not errors; they are symbolic variables. This is what makes `x^2 + 1` useful.
- Unknown `\mathrm{...}` names are errors, because the user explicitly asked for a named unit/constant.

### 10.2 Arithmetic with quantities

The evaluator implements the unit rules from §7.6. Key points:

- Addition/subtraction require matching dimensions.
- Multiplication/division combine dimensions and magnitudes.
- Powers require a dimensionless exponent.
- Functions have dimension contracts (dimensionless arguments for `exp`/`ln`, angle-or-dimensionless for trig).
- Comparisons require compatible dimensions.
- Assignment stores the value and its display unit.

### 10.3 Equations

A line that parses as an equation is handled as follows:

- If both sides evaluate to numeric values, return `true` or `false`.
- If there is exactly one free variable, solve for it automatically.
- If there are multiple free variables, return the simplified equation and a message suggesting `\operatorname{solve}(eq, var)`.
- If the equation contains quantities, attempt the linear quantity solver; otherwise return unsupported.

### 10.4 Assignment

- `x := expr` evaluates `expr` and stores it.
- Assigning to a unit or constant name is an error.
- Reassignment is allowed and affects subsequent lines.
- The worksheet derives the environment by replaying lines; there is no hidden mutable state across reloads.

### 10.5 Conversion

`expr \to unit_expr`:

1. Evaluate `expr` to a quantity.
2. Evaluate `unit_expr` to a `UnitValue`.
3. Check dimensions.
4. For affine units, convert through the base unit.
5. Set the display unit to the target and format.

Examples:

```latex
2\,\mathrm{km} + 300\,\mathrm{m} \to \mathrm{m}   % 2300 m
\frac{100\,\mathrm{km}}{2\,\mathrm{h}} \to \mathrm{m/s} % 13.888... m/s
20\,\mathrm{^\circ C} \to \mathrm{K}              % 293.15 K
```

### 10.6 Calculus and units

- Differentiation keeps the quantity dimension. If the variable is a quantity, divide the dimension by the variable's dimension.
- Indefinite integration multiplies the dimension by the integration variable's dimension. A symbolic integration variable is dimensionless, so the result may keep a symbolic factor (e.g. `3 m/s · t`); substitution of a quantity later yields the correct dimension.
- Definite integration requires compatible limit dimensions and multiplies by the limit dimension.
- If the units cannot be handled correctly, return an error or an unevaluated expression. Do not silently drop units.

### 10.7 Unit-aware matrices (out of scope)

Matrices are dimensionless in v1. The reason is that determinant and inverse change the unit of every entry in a way that depends on row/column scaling; doing this correctly requires a more sophisticated representation (e.g. a unit factor per row/column). Rather than implement it halfway, v1 rejects unit-valued matrix entries. This is documented and tested.

### 10.8 Angle mode

- `rad` and `deg` are dimensionless units with scales 1 and `π/180`.
- A bare number passed to a trig function is interpreted according to the angle-mode setting:
  - radians (default): `sin(1)` means `sin(1 rad)`;
  - degrees: `sin(30)` means `sin(30°)`.
- An explicit angle unit always wins: `sin(30^\circ)` works in either mode.
- Inverse trig returns a dimensionless value displayed in `rad` or `deg` according to the mode.
- This is a display/interpretation setting, not a dimension. Document the `Hz` vs `rad/s` nuance from §7.1.

---

## 11. Formatting

### 11.1 Output contract

`format(value, options) -> Formatted`:

```rust
pub struct Formatted {
    pub latex: String,          // primary result
    pub plain: String,          // copy-friendly text
    pub exact: bool,
    pub approx_latex: Option<String>,
    pub unit_breakdown: Option<String>,
    pub warnings: Vec<Warning>,
}
```

### 11.2 Numbers

- Exact integers print as integers.
- Exact rationals print as `\frac{a}{b}`; if the denominator is 1, as an integer.
- Floats print with the configured significant digits (default 12).
- Scientific notation is used when `|x| >= 10^6` or `0 < |x| < 10^{-4}`:
  `1.23456789012 \times 10^{8}`.
- Negative zero is normalized to zero.
- The formatter never rounds the internal value; it only rounds the displayed digits.

### 11.3 Units

- Unit symbols are upright: `\mathrm{m}`, `\mathrm{kg}`, `\mathrm{^\circ C}`.
- Products use `\cdot`.
- Simple quotients use a slash: `\mathrm{m/s}`.
- Complex quotients use a fraction: `\frac{\mathrm{kg\cdot m}}{\mathrm{s^2}}`.
- Exponents apply to the unit: `\mathrm{m^2}`, `\mathrm{s^{-1}}`.
- A space separates magnitude and unit: `2.3\,\mathrm{km}`.
- Base units use a canonical order: kg, m, s, A, K, mol, cd. Named derived and non-SI units preserve the order in which they were written, so torque stays `N·m` rather than `m·N`.
- The detail panel shows the SI base decomposition: `W = kg·m²/s³`.

### 11.4 Expressions

- Use standard LaTeX precedence and parentheses.
- Prefer `\frac` for fractions.
- Use `\cdot` for explicit multiplication only where needed; implicit multiplication is also acceptable in output.
- Functions use `\sin`, `\cos`, `\ln`, etc.
- Matrices use `\begin{pmatrix}...\end{pmatrix}`.
- Equation solutions use `\lor` for alternatives: `x = 2 \lor x = 3`.

### 11.5 Exact and approximate pairing

When a result has both forms, the primary is chosen by the user's setting:

- Exact mode: `\frac{1}{3}` with a secondary `\approx 0.333333333333`.
- Approximate mode: `0.333333333333` with a secondary `= \frac{1}{3}` if exact.

Measured constants always mark the result approximate when numerically substituted.

---

## 12. WASM API

### 12.1 Design

The WASM boundary is intentionally small and JSON-based. JSON is easy to log, test, and evolve; the performance overhead is negligible for a calculator.

```rust
#[wasm_bindgen]
pub struct Engine {
    env: Environment,
    settings: Settings,
}

#[wasm_bindgen]
impl Engine {
    #[wasm_bindgen(constructor)]
    pub fn new() -> Engine;

    /// Parse, resolve, evaluate, and format one line.
    pub fn eval(&mut self, latex: &str) -> String; // JSON EvalResult

    /// Parse and dimension-check without running the CAS.
    pub fn preview(&self, latex: &str) -> String; // JSON PreviewResult

    /// Parse only.
    pub fn parse(&self, latex: &str) -> String; // JSON ParseResult

    /// Search units, constants, and functions.
    pub fn completions(&self, prefix: &str, limit: usize) -> String;

    /// Convert a value expression to a target unit.
    pub fn convert(&self, value_latex: &str, unit_latex: &str) -> String;

    /// Environment snapshot for autosave/undo.
    pub fn snapshot(&self) -> String;
    pub fn restore(&mut self, snapshot: &str) -> Result<(), JsValue>;

    pub fn reset(&mut self);
    pub fn set_settings(&mut self, settings_json: &str) -> Result<(), JsValue>;
}
```

### 12.2 Result shape

```json
{
  "status": "ok",
  "inputLatex": "12\\,\\mathrm{V}\\cdot 0.5\\,\\mathrm{A}",
  "resultLatex": "6\\,\\mathrm{W}",
  "resultPlain": "6 W",
  "exact": true,
  "approxLatex": null,
  "unitBreakdown": "W = kg·m²/s³",
  "warnings": [],
  "error": null,
  "assignment": null,
  "timingMs": 0.42
}
```

Error shape:

```json
{
  "status": "error",
  "error": {
    "code": "unit/dimension-mismatch",
    "message": "Cannot add a length (m) and a time (s).",
    "hint": "Convert one operand to the other's unit first.",
    "span": { "start": 8, "end": 12 },
    "snippet": "\\mathrm{s}"
  }
}
```

Error codes are stable strings for UI logic and tests:

- `parse/*`
- `unit/unknown`
- `unit/dimension-mismatch`
- `unit/affine`
- `eval/domain`
- `cas/unsupported`
- `cas/fuel`
- `matrix/singular`
- `matrix/too-large`
- `matrix/units-unsupported`
- `internal/*`

### 12.3 State and replay

- The engine owns the environment.
- The frontend stores only worksheet source lines and settings.
- On reload, the frontend replays the lines through `eval` to rebuild the environment. This is deterministic and avoids serializing complex values.
- For undo/redo of a line, the frontend replays from the changed line. The engine caches nothing across replays except the unit/constant registries.
- `snapshot`/`restore` are available for performance and for exporting a session, but replay remains the source of truth.

### 12.4 WASM build

- `wasm-pack build crates/euler-wasm --target web --release`.
- `wasm-opt -Oz` in the release build.
- `panic = "abort"`, `lto = true`, `codegen-units = 1`, `opt-level = "s"`.
- `console_error_panic_hook` only in debug builds.
- No TOML parsing at runtime: unit and constant tables are generated Rust statics. JSON serialization uses a small, explicit serde layer at the WASM boundary only.
- Target size: under 1.5 MB gzipped for the core + data. Track this in CI.
---

## 13. Frontend

### 13.1 Stack

- **Vite** for development and bundling.
- **TypeScript** for the UI and WASM boundary.
- **Svelte 5** for the worksheet UI. Svelte compiles to a small runtime, is excellent for reactive lists, and keeps the component code short.
- **MathQuill** for WYSIWYG input, behind an adapter interface.
- **KaTeX** for rendering results. KaTeX is fast, static, and has a strict `trust: false` default.
- **No UI component library.** A small set of CSS tokens and components keeps the bundle small.

### 13.2 Editor adapter

MathQuill is isolated behind an interface so it can be replaced by MathLive later without touching the worksheet or the parser.

```ts
export interface MathEditor {
  getLatex(): string;
  setLatex(latex: string): void;
  focus(): void;
  blur(): void;
  insert(latex: string): void;
  clear(): void;
  onEdit(cb: (latex: string) => void): () => void;
  onEnter(cb: () => void): () => void;
  destroy(): void;
}
```

The MathQuill implementation:

- Creates `MQ.MathField` on a `<span>`.
- Pins jQuery **3.7.1**. MathQuill uses jQuery APIs removed in jQuery 4.
- Uses a pinned MathQuill build from the active GitHub repository/commit (the npm release is old). Vendor the built CSS/JS or use a Git dependency; document the exact commit.
- Configures `autoCommands` for `\pi`, `\theta`, `\sqrt`, `\infty`, and common Greek letters.
- Configures `autoOperatorNames` for `sin cos tan ln log exp det`.
- Configures `handlers.edit` to emit normalized LaTeX.
- Imports MathQuill's fonts through Vite's asset pipeline.
- Provides a plain LaTeX `<textarea>` fallback for accessibility and for debugging.

The adapter is the only file that imports MathQuill or jQuery. This is the key to the “MathQuill now, MathLive later” decision.

### 13.3 Worksheet state

```ts
type LineStatus = 'idle' | 'preview' | 'ok' | 'warning' | 'error';

interface WorksheetLine {
  id: string;
  latex: string;
  status: LineStatus;
  result?: EvalResult;
  preview?: PreviewResult;
  dirty: boolean;
}
```

- The worksheet is an ordered list of lines.
- The environment is derived by replaying lines through the engine.
- On edit:
  - run `preview` after a 150–250 ms debounce for syntax and dimension feedback;
  - mark the line and all downstream lines dirty;
  - do not run expensive CAS work until `Enter` or a short idle timeout.
- On `Enter`:
  - evaluate the line;
  - if it is an assignment, update the environment and mark downstream lines dirty;
  - re-evaluate downstream lines only when the user runs them or after a debounce, to avoid surprise cascades.
- A “Run all” button replays the entire worksheet.
- Results are cached per line; replaying starts from the first dirty line.

### 13.4 Components

| Component | Responsibility |
|---|---|
| `App.svelte` | Layout, settings, engine initialization |
| `Worksheet.svelte` | Line list, replay orchestration, keyboard navigation |
| `MathLine.svelte` | Mounts the editor adapter, handles Enter/Up/Down, shows status |
| `ResultView.svelte` | KaTeX render, exact/approx toggle, copy, detail panel |
| `UnitPalette.svelte` | Searchable units/constants/functions, inserts LaTeX |
| `VariablesPanel.svelte` | Current variables, insert/delete |
| `SettingsDialog.svelte` | Exact/approx, sig figs, angle mode, unit display |
| `HelpDialog.svelte` | Syntax cheat sheet and examples |
| `ErrorBanner.svelte` | Parse/eval errors with span snippet |

### 13.5 MathQuill input UX

- Typing `\mathrm{m}` works directly.
- For reliable MathQuill rendering, prefer separate blocks: `\mathrm{kg}\cdot\mathrm{m}/\mathrm{s}^2`. The parser also accepts a single `\mathrm{kg\cdot m/s^2}` block.
- The Units & Constants palette is the primary way to insert units and constants. It has categories and search over symbol, name, and aliases.
- A “unit mode” shortcut (`Ctrl+U` or a toolbar button) inserts `\mathrm{}` with the cursor inside, so the user can type `km`.
- Typing `\` opens a lightweight command palette with unit/constant/function suggestions. This is a palette panel, not a caret-anchored autocomplete, because MathQuill does not expose reliable caret coordinates for all layouts.
- Common constants get toolbar buttons (`c`, `g`, `N_A`, `k_B`, `h`).
- A matrix button opens a small grid dialog and inserts `\begin{pmatrix}...\end{pmatrix}`. This avoids relying on MathQuill’s matrix editing (which must be verified in the first spike).
- A conversion button inserts `\to \mathrm{}`.
- An assignment button inserts `:=`.
- A “plain LaTeX” toggle replaces the active MathQuill field with a textarea containing the same LaTeX.

### 13.6 Results

- Results are rendered with KaTeX, not MathQuill. Input is editable; output is static and fast.
- The primary result is large; the secondary exact/approx form is smaller.
- The detail panel shows:
  - exact form;
  - approximate form;
  - unit breakdown (SI base dimensions);
  - constant uncertainties;
  - warnings.
- A copy button copies the LaTeX; a second copy button copies plain text.
- Clicking a unit in the detail panel inserts it into the active line.
- For ambiguous unit simplifications, show the primary and secondary forms side by side: `6 N·m` with `= 6 J`.

### 13.7 Persistence and sessions

- Autosave worksheet source lines and settings to `localStorage` under a versioned key.
- Export/import a session as JSON:
  ```json
  {
    "version": 1,
    "settings": { "exact": true, "sigFigs": 12, "angleMode": "rad" },
    "lines": [{ "latex": "...", "id": "..." }]
  }
  ```
- No server, no accounts. Share by exporting a file or a URL fragment later.
- On load, validate the JSON shape before replaying. Treat localStorage as untrusted input.
- Provide a “Clear worksheet” action with confirmation.

### 13.8 Accessibility

- MathQuill’s accessibility is limited; provide the plain LaTeX textarea and an `aria-label` on each math field.
- Every result has a plain-text `aria-label` and a copyable plain form.
- Keyboard-only operation: Tab between fields/panels, Enter to evaluate, Up/Down to navigate, Escape to leave.
- Respect `prefers-reduced-motion`.
- Color is never the only signal for errors; use text and icons.
- KaTeX output is given `aria-hidden="true"` with a plain-text sibling for screen readers.

### 13.9 Styling

- Minimal, high-contrast, calculator-like, zero border radius.
- System font stack for UI; KaTeX fonts for math.
- Light and dark themes via CSS custom properties.
- Responsive layout: result to the right on wide screens, below on narrow screens.
- No animations that delay interaction.

### 13.10 Frontend testing

- **Unit tests** for the LaTeX normalizer and formatters.
- **Playwright** tests:
  - mount MathQuill and type expressions;
  - verify the emitted LaTeX corpus;
  - evaluate a line and assert the rendered result;
  - test keyboard navigation and persistence;
  - smoke-test matrices through the dialog.
- **Visual snapshots** for a few representative worksheets.
- A test that the WASM module initializes and an end-to-end calculation works in a production build.
---

## 14. Data files and generation

### 14.1 Source of truth

Units and constants are maintained as TOML files in `data/`:

```
data/
├─ units.toml
├─ prefixes.toml
├─ constants.toml
└─ constant_aliases.toml
```

The Rust build does **not** parse TOML at runtime. A generator produces static Rust tables:

```
crates/euler-core/src/data/
├─ units_generated.rs
├─ constants_generated.rs
└─ mod.rs
```

The generated files are committed. CI runs the generator and fails if the output differs, so data changes are reviewable.

Why not parse TOML at runtime? It adds `serde` + `toml` code to the WASM bundle, slows startup, and turns data errors into runtime errors. Generated statics are smaller and faster.

### 14.2 Unit schema

```toml
[[unit]]
symbol = "m"
latex = "\\mathrm{m}"
name = "metre"
dimension = { L = 1 }
scale = "1"
prefixable = true
aliases = ["meter", "metre"]
preferred = false

[[unit]]
symbol = "W"
latex = "\\mathrm{W}"
name = "watt"
dimension = { L = 2, M = 1, T = -3 }
scale = "1"
prefixable = true
preferred = true
aliases = ["watt", "watts"]

[[unit]]
symbol = "degC"
latex = "\\mathrm{^\\circ C}"
name = "degree Celsius"
dimension = { Theta = 1 }
scale = "1"
offset = "273.15"
prefixable = false
affine = true
aliases = ["°C", "degC", "celsius"]

[[unit]]
symbol = "in"
latex = "\\mathrm{in}"
name = "inch"
dimension = { L = 1 }
scale = "127/5000"        # 0.0254 m exactly
prefixable = false
aliases = ["inch", "inches"]
```

Rules:

- `scale` is a rational string, a decimal string, or a symbolic expression string such as `"pi/180"`.
- `offset` is present only for affine units.
- `dimension` keys are `L, M, T, I, Theta, N, J`.
- `preferred = true` marks a unit as a candidate for automatic display simplification.
- `ambiguous = true` marks cases like `N·m` where a secondary equivalent should be shown.
- Every unit has at least one LaTeX form and one plain alias.
- Exact conversion factors are written as rational strings, not floats.

### 14.3 Constants schema

```toml
[[constant]]
id = "c"
symbol = "c"
aliases = ["c_0", "speed_of_light"]
name = "speed of light in vacuum"
value = "299792458"
unit = "m/s"
exact = true
codata = 2022
source = "SI defining constant"

[[constant]]
id = "G"
symbol = "G_0"
aliases = ["G", "gravitational_constant"]
name = "Newtonian constant of gravitation"
value = "6.67430e-11"
unit = "m^3/(kg*s^2)"
uncertainty = "1.5e-15"
exact = false
codata = 2022
source = "CODATA 2022"

[[constant]]
id = "N_A"
symbol = "N_A"
aliases = ["avogadro", "avogadro_constant"]
name = "Avogadro constant"
value = "6.02214076e23"
unit = "1/mol"
exact = true
codata = 2022
source = "SI defining constant"
```

The generator:

1. Validates dimensions and units.
2. Resolves constant units through the unit registry.
3. Converts values to SI base units at generation time.
4. Emits static Rust definitions.
5. Emits a machine-readable index for the frontend palette.

### 14.4 CODATA import

`scripts/fetch_codata.py`:

- Downloads the NIST CODATA 2022 ASCII table.
- Parses name, symbol, value, uncertainty, and unit.
- Maps NIST unit strings to Euler unit expressions.
- Emits a draft `constants.toml`.
- Leaves aliases and categories for manual review.

The generated TOML is committed. The script is a maintenance tool, not part of the build. Cite NIST/CODATA in the README and in the constants detail panel.

---

## 15. Repository layout

```
euler/
├─ Cargo.toml                     # workspace
├─ rust-toolchain.toml
├─ crates/
│  ├─ euler-core/
│  │  ├─ Cargo.toml
│  │  ├─ build.rs                 # optional; runs data generator check
│  │  ├─ src/
│  │  │  ├─ lib.rs
│  │  │  ├─ expr/
│  │  │  │  ├─ mod.rs
│  │  │  │  ├─ number.rs
│  │  │  │  ├─ expr.rs
│  │  │  │  └─ normalize.rs
│  │  │  ├─ latex/
│  │  │  │  ├─ mod.rs
│  │  │  │  ├─ lexer.rs
│  │  │  │  ├─ parser.rs
│  │  │  │  ├─ ast.rs
│  │  │  │  └─ normalize.rs
│  │  │  ├─ units/
│  │  │  │  ├─ mod.rs
│  │  │  │  ├─ dimension.rs
│  │  │  │  ├─ registry.rs
│  │  │  │  ├─ unit_expr.rs
│  │  │  │  ├─ convert.rs
│  │  │  │  └─ display.rs
│  │  │  ├─ constants/
│  │  │  │  ├─ mod.rs
│  │  │  │  └─ registry.rs
│  │  │  ├─ value/
│  │  │  │  ├─ mod.rs
│  │  │  │  ├─ quantity.rs
│  │  │  │  ├─ matrix.rs
│  │  │  │  └─ value.rs
│  │  │  ├─ cas/
│  │  │  │  ├─ mod.rs
│  │  │  │  ├─ simplify.rs
│  │  │  │  ├─ expand.rs
│  │  │  │  ├─ factor.rs
│  │  │  │  ├─ solve.rs
│  │  │  │  ├─ diff.rs
│  │  │  │  ├─ integrate.rs
│  │  │  │  └─ matrix.rs
│  │  │  ├─ eval/
│  │  │  │  ├─ mod.rs
│  │  │  │  ├─ env.rs
│  │  │  │  ├─ evaluator.rs
│  │  │  │  ├─ functions.rs
│  │  │  │  └─ error.rs
│  │  │  ├─ format/
│  │  │  │  ├─ mod.rs
│  │  │  │  ├─ latex.rs
│  │  │  │  ├─ number.rs
│  │  │  │  └─ units.rs
│  │  │  └─ data/
│  │  │     ├─ mod.rs
│  │  │     ├─ units_generated.rs
│  │  │     └─ constants_generated.rs
│  │  └─ tests/
│  │     ├─ parser_tests.rs
│  │     ├─ unit_tests.rs
│  │     ├─ cas_tests.rs
│  │     ├─ golden_tests.rs
│  │     └─ mathquill_corpus.rs
│  └─ euler-wasm/
│     ├─ Cargo.toml
│     └─ src/lib.rs
├─ data/
│  ├─ units.toml
│  ├─ prefixes.toml
│  ├─ constants.toml
│  └─ constant_aliases.toml
├─ web/
│  ├─ package.json
│  ├─ vite.config.ts
│  ├─ tsconfig.json
│  ├─ index.html
│  ├─ src/
│  │  ├─ main.ts
│  │  ├─ App.svelte
│  │  ├─ components/
│  │  │  ├─ Worksheet.svelte
│  │  │  ├─ MathLine.svelte
│  │  │  ├─ ResultView.svelte
│  │  │  ├─ UnitPalette.svelte
│  │  │  ├─ VariablesPanel.svelte
│  │  │  ├─ SettingsDialog.svelte
│  │  │  ├─ HelpDialog.svelte
│  │  │  └─ ErrorBanner.svelte
│  │  ├─ editor/
│  │  │  ├─ adapter.ts
│  │  │  ├─ mathquill.ts
│  │  │  └─ normalize.ts
│  │  ├─ engine/
│  │  │  ├─ wasm.ts
│  │  │  └─ types.ts
│  │  ├─ stores/
│  │  │  ├─ worksheet.ts
│  │  │  └─ settings.ts
│  │  └─ styles/
│  │     ├─ tokens.css
│  │     └─ app.css
│  ├─ tests/
│  │  ├─ editor.spec.ts
│  │  ├─ worksheet.spec.ts
│  │  └─ fixtures/mathquill-corpus.json
│  └─ vendor/
│     └─ mathquill/             # pinned build, if not using a git dep
├─ xtask/
│  ├─ Cargo.toml
│  └─ src/main.rs                 # data generator / CI checks
├─ scripts/
│  └─ fetch_codata.py
├─ docs/
│  ├─ syntax.md
│  ├─ units.md
│  ├─ constants.md
│  ├─ cas.md
│  └─ architecture.md
├─ .github/workflows/ci.yml
├─ package.json                  # workspace scripts (pnpm)
├─ pnpm-workspace.yaml
├─ flake.nix
├─ flake.lock
├─ README.md
└─ LICENSE
```

### 15.1 Workspace scripts

```json
{
  "scripts": {
    "dev": "pnpm --filter web dev",
    "build:wasm": "wasm-pack build crates/euler-wasm --target web --release",
    "build:web": "pnpm --filter web build",
    "build": "pnpm build:wasm && pnpm build:web",
    "test:rust": "cargo test --workspace",
    "test:web": "pnpm --filter web test",
    "test:e2e": "pnpm --filter web playwright test",
    "lint": "cargo clippy --workspace --all-targets -- -D warnings && pnpm --filter web lint",
    "fmt": "cargo fmt --all && pnpm --filter web fmt",
    "gen:data": "cargo run -p euler-xtask -- gen-data"
  }
}
```

Use `pnpm` for reproducibility and speed. Commit `pnpm-lock.yaml` and `Cargo.lock`.

---

## 16. Build, CI, and deployment

### 16.1 Toolchain

- Rust stable (pin in `rust-toolchain.toml`).
- `wasm-pack`, `wasm-bindgen-cli`, `wasm-opt` (Binaryen).
- Node 20+ and pnpm.
- Playwright browsers in CI.

### 16.2 CI pipeline

On every pull request:

1. `cargo fmt --check`
2. `cargo clippy -- -D warnings`
3. `cargo test --workspace`
4. `cargo run -p euler-xtask -- gen-data --check` (generated data is current)
5. `wasm-pack build ... --release`
6. Check WASM size budget.
7. `pnpm install --frozen-lockfile`
8. `pnpm build`
9. Playwright tests against the production build.
10. Optional: compare a CAS corpus against SymPy for supported operations (development aid, not a runtime dependency).

### 16.3 Deployment

- Static hosting: GitHub Pages, Cloudflare Pages, or Netlify.
- Serve `.wasm` with `application/wasm`.
- Set long cache headers for hashed assets and the WASM file.
- No server-side code.
- Add a service worker only if offline support becomes a requirement; not in v1.

### 16.4 Licensing

- Project code: MIT OR Apache-2.0 (choose one; dual is common in Rust).
- MathQuill: MPL-2.0. Using the unmodified library is fine; modifications to MathQuill files must be published under MPL-2.0.
- jQuery: MIT.
- KaTeX: MIT.
- Svelte/Vite: MIT.
- CODATA/NIST data: factual values; include attribution and source notes. Do not copy large explanatory texts.
- Unit definitions: derived from SI Brochure / NIST SP 811; include attribution.
- No GPL dependencies in v1. This is a direct benefit of the built-in CAS decision.
---

## 17. Testing and quality

### 17.1 Test layers

| Layer | What it covers | Tool |
|---|---|---|
| Lexer/parser unit tests | tokens, precedence, implicit multiplication, errors, spans | `cargo test` |
| MathQuill corpus | real `latex()` outputs from the editor | fixture + Playwright capture |
| Unit tests | dimensions, prefixes, conversions, affine temperature, display simplification | `cargo test` |
| Constants tests | aliases, collisions, exactness, units, uncertainty metadata | `cargo test` |
| CAS tests | canonical form, simplify, expand, factor, solve, diff, integrate, matrices | `cargo test` |
| Golden tests | input LaTeX → expected result LaTeX/plain/exact flag | `insta` or JSON fixtures |
| Property tests | conversion round-trips, dimension arithmetic, parser no-panic | `proptest` |
| WASM tests | JSON boundary, error mapping, state/replay | `wasm-bindgen-test` + browser |
| E2E tests | editor, worksheet, persistence, results | Playwright |
| Fuzz tests | malformed LaTeX never panics | `cargo-fuzz` (nightly, optional) |

### 17.2 Golden test format

```toml
[[case]]
input = "12\\,\\mathrm{V}\\cdot 0.5\\,\\mathrm{A}"
result_latex = "6\\,\\mathrm{W}"
exact = true
warnings = []

[[case]]
input = "2\\,\\mathrm{km} + 300\\,\\mathrm{m} \\to \\mathrm{km}"
result_latex = "2.3\\,\\mathrm{km}"
exact = true

[[case]]
input = "\\sin(30^\\circ)"
result_latex = "\\frac{1}{2}"
exact = true

[[case]]
input = "x^2 - 5x + 6 = 0"
result_latex = "x = 2 \\lor x = 3"
exact = true
```

Golden tests are the primary regression net for the parser/unit/formatter pipeline. Keep them in `crates/euler-core/tests/golden/` and run them in CI.

### 17.3 Unit test matrix

At minimum:

- Exact conversions: `1 in → 25.4 mm`, `1 ft → 0.3048 m`, `1 mi → 1609.344 m`, `1 lb → 0.45359237 kg`, `1 atm → 101325 Pa`, `1 eV → 1.602176634e-19 J`.
- Prefixes: `1 km → 1000 m`, `1 mm → 0.001 m`, `1 µF → 1e-6 F`, `1 kΩ → 1000 Ω`.
- Derived units: `A·V → W`, `V/A → Ω`, `J/s → W`, `N/m² → Pa`, `C/s → A`.
- Ambiguity: `N·m` primary `N·m`, secondary `J`; `J/kg` primary `J/kg`, secondary `Gy/Sv`.
- Addition: `1 km + 1 m → 1.001 km`, `1 m + 1 s` error.
- Multiplication/division: `10 m / 2 s → 5 m/s`.
- Powers: `(2 m)^2 → 4 m²`, `sqrt(4 m²) → 2 m`, `sqrt(1 m)` warning.
- Temperature: `20 °C → 293.15 K`, `68 °F → 20 °C`, `20 °C - 10 °C → 10 K`, `20 °C + 30 °C` error.
- Angles: `30° → π/6 rad`, `sin(30°) → 1/2`, `sin(30)` depends on angle mode.
- Constants: `c → 299792458 m/s`, `N_A → 6.02214076e23 1/mol`, `R_g` exact value, aliases resolve.
- Unit/constant collisions: `\mathrm{h}` = hour, `\mathrm{h_P}` = Planck; `\mathrm{m}` = metre, `\mathrm{m_e}` = electron mass.

### 17.4 CAS test matrix

- Normalization: `x + x → 2x`, `x * x → x^2`, `(x+1)^2` expand/factor round-trip.
- Solve: linear, quadratic with rational roots, quadratic with surds, no solution, infinite solutions.
- Differentiation: polynomials, product rule, quotient rule, chain rule, trig, exp/log.
- Integration: polynomials, `1/x`, `e^{ax}`, `sin(ax)`, `cos(ax)`, `1/(ax+b)`, linear substitution.
- Definite integrals: polynomial and trig with numeric limits.
- Matrices: 2×2/3×3 determinant, inverse, singular error, linear solve, symbolic entries, size limit.
- Units in calculus: `d/dx(x^2 m) → 2x m`, `∫3 m/s dt → 3 m/s·t`, substitution.
- Fuel: pathological simplification returns `cas/fuel`, not a hang.
- Unsupported: `∫sin(x^2) dx` returns an unevaluated integral with `cas/unsupported`.

### 17.5 Cross-checking with SymPy (development only)

For the supported subset, generate a corpus of expressions and compare numerical values at random points with SymPy. This catches sign errors and simplification bugs. It is a development script, not a CI runtime dependency. The results are committed as golden values.

### 17.6 Fuzzing and robustness

- The parser must never panic on arbitrary input. Fuzz with random LaTeX and arbitrary Unicode.
- The formatter must never produce invalid LaTeX for internal values.
- All public Rust APIs return `Result`, not panic.
- The WASM boundary catches panics in debug and returns `internal/panic` in release if a panic occurs (with `panic = "abort"`, this is a hard trap; keep panic-free code paths and test them).

---

## 18. Performance and size budgets

### 18.1 Targets

| Metric | Target |
|---|---|
| Initial JS+CSS+WASM (gzipped) | < 2.5 MB total |
| Rust WASM (gzipped) | < 1.5 MB |
| Parse + evaluate typical expression | < 1 ms |
| Full worksheet replay (50 lines) | < 50 ms |
| Symbolic simplify/solve typical | < 50 ms |
| Hard interactive limit | 2 s, then fuel error |
| Time to first interaction (warm cache) | < 1 s |
| Time to first interaction (cold) | < 3 s on a mid-range laptop |

### 18.2 Techniques

- Generate units/constants as static Rust arrays; no runtime TOML/serde.
- Avoid `regex` and heavy parser combinators.
- Use `num-bigint`/`num-rational`; avoid GMP.
- `wasm-opt -Oz`, LTO, `opt-level = "s"`, `codegen-units = 1`, `panic = "abort"`.
- Lazy-load the WASM module after the first paint; show a skeleton.
- Cache parse results for unchanged lines.
- Debounce preview evaluation.
- Use a fuel budget to bound CAS work.
- Measure bundle size and benchmark parse/eval in CI.

### 18.3 When to move to a Web Worker

If a symbolic operation can exceed ~50 ms, or if the UI stutters during worksheet replay, move the engine to a Web Worker. The JSON API is designed for this: the worker owns the `Engine`, and the main thread sends `{id, method, args}` messages. Do not add a worker until profiling shows it is needed; the extra complexity is not justified for v1.

---

## 19. Security and privacy

- **No server, no telemetry, no network requests at runtime.** The app can be served as static files and works offline after loading.
- **No user data leaves the browser.** Worksheet content is stored only in `localStorage` and exported files.
- **XSS:** never inject user LaTeX as raw HTML. MathQuill and KaTeX render into DOM elements; KaTeX uses `trust: false` and `strict` defaults. Error messages are rendered as text.
- **localStorage is untrusted.** Validate the session JSON schema and bounds before replaying. Reject unknown versions.
- **WASM sandbox:** the Rust code runs in the browser sandbox. No filesystem, no network.
- **Resource limits:** fuel, matrix size limits, and a wall-clock guard prevent denial-of-service via a crafted worksheet.
- **Supply chain:** pin dependencies, commit lockfiles, use `cargo audit` and `pnpm audit` in CI.

---

## 20. Risks and mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Building a CAS is larger than expected | Schedule slip, incomplete features | Strict v1 scope; unsupported operations return unevaluated; `CasEngine` seam for a later external CAS |
| MathQuill output quirks or breakage | Parser failures | Adapter + normalization corpus captured with Playwright; pin a known commit; plain LaTeX fallback |
| MathQuill matrix editing is inadequate | Matrix feature blocked | Spike in M0; modal grid editor that inserts LaTeX; fallback to a matrix chip or MathLive for those lines |
| MathQuill npm package is stale | Build/security issues | Use a pinned GitHub commit or the Desmos fork; vendor the build; pin jQuery 3.7.1 |
| Unit/constant name collisions | Wrong results | Units-first resolution; explicit aliases; collision tests; autocomplete shows categories |
| Affine temperature errors | Wrong physics | `TempRole` model; dedicated tests; clear errors |
| Measurement constants are wrong or stale | Wrong results | Generate from CODATA 2022; record year/source/uncertainty; do not hand-edit values |
| Unit display ambiguity (`N·m` vs `J`) | Confusing output | Primary operand form + secondary SI equivalent; “Simplify units” command |
| CAS hangs on pathological input | UI freeze | Fuel budget, matrix size limit, wall-clock guard |
| WASM bundle too large | Slow load | Static data, no runtime serde, `wasm-opt`, size budget in CI |
| Exact arithmetic becomes slow | UI stutter | Fuel, size limits, optional worker, benchmark corpus |
| Data licensing | Legal risk | CODATA/NIST attribution; factual data; no copied prose; project license MIT/Apache |
| Scope creep | Never ships | Milestones and explicit non-goals; defer limits/series/complex/plots |

### 20.1 The biggest risk

The biggest risk is not units or MathQuill. It is **CAS scope**. The plan therefore treats the CAS as a focused calculator engine with a hard feature matrix, not as a general-purpose algebra system. Every unsupported case must fail clearly and quickly. The `CasEngine` boundary is a deliberate escape hatch: if the project later needs Risch integration or advanced solving, an external engine can be integrated behind the same interface without changing the parser, unit system, or worksheet.
---

## 21. Milestones

Each milestone ends with a demoable result and tests. The order is chosen so that the riskiest integration points are validated early.

### M0 — Spikes and scaffold

- Create the Rust workspace, Vite/Svelte app, and CI.
- Build a “hello world” WASM function and call it from the frontend.
- Mount MathQuill in a Svelte component behind the `MathEditor` adapter.
- Capture a MathQuill output corpus for fractions, roots, powers, units, Greek letters, degrees, and matrices.
- **Spike:** verify MathQuill matrix rendering and MathQuill’s current build/install story. If matrices are not usable, decide on the modal grid fallback.
- **Exit criteria:** `pnpm dev` shows a MathQuill field; `getLatex()` returns expected strings; WASM round-trip works; CI is green.

### M1 — Parser and formatter core

- Implement the lexer, parser, AST, spans, and error codes for the v1 LaTeX subset.
- Implement exact rational numbers and the expression tree.
- Implement the formatter for numbers and expressions.
- Add parser/formatter golden tests and fuzz smoke tests.
- **Exit criteria:** `2 + 3 * 4` parses and formats; `\frac{1}{2}` is exact; errors include spans; no panics on malformed input.

### M2 — Units, constants, and the unit-aware evaluator

This is the core differentiator and should not be rushed.

- Implement dimensions, unit registry, prefixes, unit expressions, conversion, and display simplification.
- Add the generated unit data and the initial CODATA constants table.
- Implement `Quantity` arithmetic, affine temperature handling, angle mode, and dimension errors.
- Implement `\mathrm{}` resolution and collision aliases.
- Add the unit test matrix from §17.3.
- **Exit criteria:** `A·V → W`, `2 km + 300 m → 2.3 km`, `20 °C → 293.15 K`, `1 in → 25.4 mm`, `sin(30°) → 1/2`; unit/constant collisions are tested.

### M3 — Worksheet UI and end-to-end evaluation

- Implement the multi-line worksheet, replay, autosave, undo/redo, variables panel, settings, and error UI.
- Implement the Units & Constants palette and the unit/assignment/conversion buttons.
- Render results with KaTeX; add exact/approx toggle and detail panel.
- Add Playwright tests for typing, evaluation, persistence, and keyboard navigation.
- **Exit criteria:** A user can build a small physics worksheet, edit an earlier line, and see downstream results update; results are correct and readable.

### M4 — Symbolic engine I: simplify, expand, factor, solve, differentiate

- Implement canonicalization, simplify rules, expand, factor (univariate), solve (linear/quadratic), and differentiation.
- Implement the fuel budget and unsupported-operation results.
- Add CAS golden tests and a SymPy cross-check corpus.
- **Exit criteria:** `x^2 - 5x + 6 = 0` solves; `(x+1)^2` expands and factors; `d/dx(x^3 m)` works; unsupported forms fail clearly.

### M5 — Symbolic engine II: integration and matrices

- Implement standard-form integration, definite integrals, and unit-aware limits.
- Implement matrix construction, arithmetic, determinant, inverse, transpose, and linear solve.
- Add matrix UI (dialog or editor support) and tests.
- **Exit criteria:** `∫3x^2 dx → x^3 + C`; `∫_0^2 3x^2 dx → 8`; 2×2/3×3 determinant and inverse work; singular and oversized matrices fail clearly.

### M6 — Polish and release

- Accessibility pass, responsive layout, dark/light theme.
- Performance profiling, WASM size check, worker decision.
- Documentation: syntax guide, units, constants, CAS support, architecture.
- Deployment to static hosting; release `v0.1.0`.
- **Exit criteria:** Production build passes all tests; bundle within budget; docs cover the supported subset; a new user can complete the examples in §3.4 without help.

### Post-v1 roadmap

- MathLive adapter (if MathQuill becomes a liability).
- Advanced CAS behind `CasEngine` (limits, series, Risch, complex numbers).
- Unit-aware matrices and uncertainty propagation.
- User-defined functions and multi-line programs.
- Plotting and geometry.
- Session sharing via URL/file.
- Optional PWA/offline support.
- Optional desktop wrapper (Tauri) if a native app is desired.

---

## 22. Key architectural decisions (ADR summary)

| # | Decision | Rationale | Revisit when |
|---|---|---|---|
| 1 | Built-in Rust CAS for v1 | No licensing/size risk; full control of units; one WASM module | Advanced CAS features become essential |
| 2 | `CasEngine` boundary | Allows replacing the CAS without touching parser/units/UI | An external CAS is adopted |
| 3 | MathQuill behind `MathEditor` adapter | Meets the requested UX; isolates jQuery and LaTeX quirks | MathQuill blocks matrices/accessibility or becomes unmaintained |
| 4 | Canonical SI base magnitude + display unit | Simple, correct arithmetic and conversion | Unit-aware matrices or affine-heavy workflows demand more |
| 5 | Exact rational first, f64 fallback | Predictable results; `0.1 + 0.2 = 0.3` | Performance of big rationals becomes a problem |
| 6 | Units/constants as generated static data | Small WASM, fast startup, reviewable data | Data size or update cadence demands runtime loading |
| 7 | Static site, no server | Privacy, simplicity, zero operating cost | Collaboration or cloud sync is required |
| 8 | JSON WASM boundary | Easy to test/log/evolve; worker-ready | Profiling shows JSON overhead matters |

---

## 23. Open decisions to confirm

These have recommended defaults in the plan. They should be confirmed before or during M0.

1. **Project license:** MIT OR Apache-2.0 for Euler code. Recommended: dual MIT/Apache-2.0, with third-party notices.
Answer: dual MIT/Apache-2.0, can change later
2. **Matrix scope:** dimensionless matrices in v1. Recommended: yes; unit-aware matrices later.
Answer: dimensionless
3. **Exactness default:** exact mode by default with an approximate toggle. Recommended: yes.
Answer: yes
4. **Constants in symbolic results:** keep constants symbolic in exact mode and substitute in approximate mode. Recommended: yes.
Answer: yes
5. **Angle mode default:** radians, with an explicit `30°` always working. Recommended: yes.
Answer: yes
6. **Temperature arithmetic:** absolute temperatures add/subtract with the `TempRole` rules in §7.5. Recommended: yes.
Answer: yes
7. **Ambiguous unit display:** primary operand form + secondary SI equivalent (`N·m` + `J`). Recommended: yes.
Answer: yes
8. **MathQuill build source:** official repo pinned commit vs Desmos fork vs vendored build. Decide in M0 after the matrix/rendering spike.
Answer: desmos fork
9. **Deployment target:** GitHub Pages vs Cloudflare Pages vs Netlify. Recommended: GitHub Pages for simplicity, Cloudflare Pages if custom headers are needed.
Answer: Github Pages at the start, Cloudflare Pages later
10. **Do you want a public web app, a personal tool, or both?** This affects accessibility/polish priorities and the license choice.
Answer: public web app, but mainly for personal use
11. **Frontend framework:** Svelte 5 is the recommended default. If you prefer fewer dependencies, the same component boundaries work with vanilla TypeScript or Preact; confirm before M3.
Answer: Svelte 5

---

## 24. Appendix A — Worked examples

### A.1 Electrical power

```latex
12\,\mathrm{V}\cdot 0.5\,\mathrm{A}
```

1. `V` → dimension `M L² T⁻³ I⁻¹`, scale 1.
2. `A` → dimension `I`, scale 1.
3. Multiply magnitudes: `12 × 0.5 = 6`.
4. Add dimensions: `M L² T⁻³ I⁻¹ + I = M L² T⁻³`.
5. Display signature `V·A` maps unambiguously to `W`.
6. Result: `6\,\mathrm{W}`.

### A.2 Speed conversion

```latex
\frac{100\,\mathrm{km}}{2\,\mathrm{h}} \to \mathrm{m/s}
```

1. `100 km` → `100000 m`, display `km`.
2. `2 h` → `7200 s`, display `h`.
3. Divide: `100000 / 7200 = 125/9 m/s`, display `km/h`.
4. Convert to `m/s`: display `m/s`.
5. Result: `\frac{125}{9}\,\mathrm{m/s}` exact, with `≈ 13.8888888889 m/s` approximate.

### A.3 Kinetic energy with units

```latex
m := 2\,\mathrm{kg}
v := 3\,\mathrm{m/s}
\frac{1}{2} m v^2
```

1. `m` is a quantity `2 kg`.
2. `v` is a quantity `3 m/s`.
3. `v^2` → `9 m²/s²`.
4. `m v^2` → `18 kg·m²/s²`.
5. `1/2` → `9 kg·m²/s²`.
6. Display signature `kg·m²/s²` maps unambiguously to `J`.
7. Result: `9\,\mathrm{J}`.

### A.4 Speed of light

```latex
\mathrm{c}^2 \cdot 1\,\mathrm{kg}
```

1. `c` resolves to the exact constant `299792458 m/s`.
2. In exact mode, the constant stays symbolic: `c² · 1 kg` → `c² kg` (dimension energy).
3. In approximate/numeric mode, `c²` → `8.9875517873681764e16 m²/s²`.
4. Multiply by `1 kg` → `8.9875517873681764e16 kg·m²/s²`.
5. Display as `J`.
6. Approximate result: `8.9875517873681764 \times 10^{16}\,\mathrm{J}`; exact result: `c^2\,\mathrm{kg}`.

### A.5 Temperature

```latex
20\,\mathrm{^\circ C} \to \mathrm{K}
20\,\mathrm{^\circ C} - 10\,\mathrm{^\circ C}
20\,\mathrm{^\circ C} + 5\,\mathrm{K}
20\,\mathrm{^\circ C} + 30\,\mathrm{^\circ C}
```

- `293.15 K`
- `10 K` (temperature difference)
- `25 °C` (5 K treated as a difference)
- error: cannot add two absolute temperatures

### A.6 Equation solving

```latex
x^2 - 5x + 6 = 0
```

- Move to `x² - 5x + 6 = 0`.
- Discriminant `25 - 24 = 1`.
- Roots `(5 ± 1)/2` → `2` and `3`.
- Result: `x = 2 \lor x = 3`.

### A.7 Matrix

```latex
A := \begin{pmatrix}1&2\\3&4\end{pmatrix}
\det(A)
A^{-1}
```

- `det(A) = -2`.
- `A^{-1} = \begin{pmatrix}-2&1\\1.5&-0.5\end{pmatrix}` (exact: `\begin{pmatrix}-2&1\\3/2&-1/2\end{pmatrix}`).
- If `A` is singular, return `matrix/singular` with a clear message.

---

## 25. Appendix B — Unit/constant collision policy

The resolver is intentionally simple and predictable:

1. Exact unit symbol.
2. Exact constant alias.
3. Prefixed unit.
4. Function name if followed by `(`.
5. Error with suggestions.

Consequences:

- `\mathrm{h}` is always hour.
- `\mathrm{h_P}` is the Planck constant.
- `\mathrm{m}` is metre; `\mathrm{m_e}` is electron mass.
- `\mathrm{N}` is newton; `\mathrm{N_A}` is Avogadro.
- `\mathrm{G}` is gauss if gauss is included; `\mathrm{G_0}` is the gravitational constant.
- `\mathrm{R}` may be roentgen; `\mathrm{R_g}` is the gas constant.
- `\mathrm{g}` is gram; `\mathrm{g_n}` is standard gravity.
- `\mathrm{e}` is elementary charge; plain `e` is Euler's number.

The autocomplete palette displays the namespace and category so users do not have to memorize these rules. Error messages for collisions include the correct alias.

---

## 26. Appendix C — Glossary

- **Dimension:** the physical nature of a quantity, represented by exponents of the seven SI base dimensions.
- **Unit:** a named scale for a dimension (metre, foot, hour).
- **Quantity:** a magnitude plus a dimension plus a preferred display unit.
- **Affine unit:** a unit with an offset, such as °C or °F.
- **Canonical form:** a deterministic representation of an expression used for simplification and display.
- **Fuel:** a counter that bounds symbolic computation.
- **Display unit:** the unit the user wants to see; the internal magnitude is always in SI base units.
- **Exact:** no float introduced; rationals and symbolic constants only.
- **Approximate:** contains a float or a numerically substituted measured constant.
- **`\mathrm{}` namespace:** the explicit namespace for units, constants, and named functions.
- **CAS:** computer algebra system; in v1, the focused built-in symbolic engine.
- **WASM:** WebAssembly; the Rust core compiled for the browser.

---

## 27. Appendix D — What “comprehensive” means here

This plan covers:

- the product behavior and syntax contract;
- the parser grammar and MathQuill normalization;
- exact arithmetic and the value model;
- the dimension/unit system, including prefixes, derived units, affine temperature, and display simplification;
- the physical-constants data model and collision policy;
- the built-in CAS scope and algorithms;
- evaluation, calculus-with-units, and matrix rules;
- formatting and the WASM API;
- the frontend architecture and MathQuill adapter;
- data generation, repository layout, build, CI, deployment, and licensing;
- testing, performance, security, risks, milestones, and open decisions.

The next step is to confirm the open decisions in §23, then start M0: scaffold the workspace, mount MathQuill, and validate the matrix/editor assumptions before writing the parser.
