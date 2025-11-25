# grammar - tiny parsing DSL

A small, pleasant-to-use declarative parsing mini-library with a compact pattern DSL, per-alternative precedence, limited left-recursive operator support, and convenient repetition/quantifier syntax. Intended for small DSLs, examples, and teaching - not a production-grade parser generator.

---

## Features

A tiny parser you actually want to write grammars in - readable, pragmatic, and a little clever.

- **Rule names inside patterns (`::=`).** You can now specify the rule name on the left-hand side of a pattern: `"<expr> ::= <expr:left> '+' <expr:right>"`. When present this rule-name **takes precedence** over the `name=` decorator argument and over the function-name heuristic used previously.
- **Anonymous (positional) captures.** Use `capture_anon=True` on `@rule(...)` to request that references *without* explicit `:var` names be captured and passed to the semantic action in positional order. This makes short rules like `@rule("<assignment> ::= <varname> '=' <expr>", capture_anon=True)` work with `def assignment(self, x1, x2)` as you expected.
- **Robust literal quoting & escaping.** Literals inside single or double quotes are first-class and can contain spaces and punctuation. Single-quoted literals use SQL-style doubling for an embedded single quote: `'''` (actually `''`) becomes `'` in the literal value; double-quoted literals support backslash escapes. This makes operators like `'<`' and `'>'` safe as literals inside patterns.
- **Named captures first-class.** If you specify `<NAME:var>` it continues to be the canonical way to capture a submatch; anonymous capturing is opt-in and does not change existing behavior unless you enable it.
- **Smarter tokens.** `token()` auto-decides word boundaries but with improved heuristics so operators and numeric/exponent forms aren't accidentally wrapped with `\b` when that would break matching.
- **Transform / validate hooks.** `@rule(..., transform=..., validate=...)` lets you normalize, post-process, or reject an alternative before the semantic action runs. Returning `None` from `transform` or `False` from `validate` makes the alternative behave as if it didn't match.
- **Improved error diagnostics.** Parse errors include line/column, a code snippet and caret, *and* a short lookahead showing what the parser actually found; expected hints for failed group/quantifier matches are more informative.
- **Left-recursive operator support.** Per-alternative `prec` and `assoc` are supported as before, plus prefix/postfix unary forms.
- **Conservative memoization.** A per-parse memo cache speeds repeated work while preserving the left-recursive growth algorithm semantics.


---

# Installation

I'm not planning on uploading `grammar` to **PyPi**, so if you want to install, run the following:

```bash
pip install git+https://github.com/samedit66/grammar.git
```

---

## Quick example - calculator

```py
from math import prod
from grammar import Grammar, token, rule

class Calculator(Grammar):
    number = token(r"\d+", int)

    @rule("<expr:x> '!'", prec=40, unary="postfix")
    def expr_postfix_inc(self, x):
        return prod(range(1, x + 1))

    @rule("<expr:x> '+' <expr:y>", prec=10, assoc="left")
    def expr_add(self, x, y):
        return x + y

    @rule("<expr:x> '-' <expr:y>", prec=10, assoc="left")
    def expr_sub(self, x, y):
        return x - y

    @rule("<expr:x> '*' <expr:y>", prec=20, assoc="left")
    def expr_mul(self, x, y):
        return x * y

    @rule("<expr:x> '/' <expr:y>", prec=20, assoc="left")
    def expr_div(self, x, y):
        return x / y

    @rule("'-' <expr:x>", prec=30, unary="prefix")
    def expr_neg(self, x):
        return -x

    @rule("<number:n>")
    def expr_number(self, n):
        return n

    @rule("'(' <expr:val> ')'")
    def expr_paren(self, val):
        return val

    @rule("<expr:res>")
    def top(self, res):
        return res

if __name__ == "__main__":
    calc = Calculator()
    tests = [
        "1 + 2 * 3", # 7
        "-1 + 4", # 3
        "2 * 3 + 4", # 10
        "2 * (3 + 4)", # 14
        "3! + 1", # 7
        "5! / 5", # 24.0
    ]
    for t in tests:
        print(t, "->", calc.parse(t))
```

See [examples](./examples) folder for other grammars like [JSON](./examples/json_grammar.py) or [TinyBasic](./examples/tiny_basic.py).

---

## Pattern syntax

- `'<lit>'` or `"<lit>"` — literal text. Spaces, punctuation and operators are allowed in literals. Single-quoted literals use **doubling** to include a single quote inside (e.g. `"hello ''world''!"` → `hello 'world'!`). Double-quoted strings support backslash escapes.
- `<NAME:var>` — reference to a token or rule `NAME`; when `:var` is present the matched value is passed as an argument to the semantic function.
- `<NAME?:var>` — optional reference; if missing and `:var` provided the var receives `None` (or a `DEFAULT` if specified).
- `<NAME?:var=DEFAULT>` — optional with default value; `DEFAULT` is parsed with Python literal rules when possible (numbers, quoted strings) — otherwise treated as raw string.
- `<NAME*:xs>` — zero-or-more repetition of a **single** reference; when named yields a `list` (possibly empty).
- `<NAME+:xs>` — one-or-more repetition of a single reference; named capture yields a non-empty `list`.
- `[ ... ]:name` — group repetition (zero-or-more): the inner sequence is attempted repeatedly; if `:name` provided you get a `list` of items where each item is either a single value (if the inner sequence captured one thing) or a `tuple` of captures.
- `{ ... }:name` — group repetition (one-or-more): fails if zero occurrences were found.
- **Rule name on the left-hand side:** patterns may now include an explicit rule name before `::=`. Example: `"<expr> ::= <expr:left> '+' <expr:right>"`. This left-hand name, when present, overrides a `name=` kwarg passed to `@rule()` and the function-name-based fallback.
- **Anonymous captures:** If you set `capture_anon=True` in the `@rule` decorator, submatches that do not have an explicit `:var` will be captured and passed to the semantic action in positional order. This applies also to repetitions and groups (you will receive lists/tuples accordingly).

Examples:

- `@rule("<expr:x> [',' <expr:y>]:rest")` → `x` is first `expr`, `rest` is a list of following exprs.
- `@rule("<id?:name='anon'>")` → `name` will be the matched id, or `'anon'` if missing.
- `@rule("<num*:ns>")` → `ns` becomes a list of numbers (maybe empty).
- `@rule("<assignment> ::= <varname> '=' <expr>", capture_anon=True)` → function `def assignment(self, x1, x2)` receives `x1=varname`, `x2=expr`.

---

## `token()` and word-boundary heuristics

`token(pattern, convert=None, with_word_boundaries=None, ignore_case=False)` defines a lexical token. When `with_word_boundaries` is not supplied the library attempts to automatically decide whether to wrap the regex with `\b...\b`.

The heuristics were improved so that patterns which contain letters but do **not** start with an alphanumeric character (for example, a numeric token beginning with `-` that contains `e` for exponent) **are not** wrapped with `\b` - this prevents spurious mismatches like `-12.34e+2` failing to match.

If you want to force `\b` behavior, pass `with_word_boundaries=True` explicitly.

---

## `@rule` options: `transform` and `validate`

You can now pass two convenient hooks to `@rule`:

- `transform: Callable[[List[Any]], Any]` - receives the *raw captured argument list* and should return either:
  - an iterable (list/tuple) which will be used as the argument list for the semantic function; or
  - a single value (it will be passed as one argument); or
  - `None` to indicate this alternative should be treated as NOT matched.

- `validate: Callable[[List[Any]], bool]` - receives the raw captured args and returns `True` to accept or `False` to reject this alternative (rejection behaves like no match).

- `capture_anon: bool` — when `True`, submatches without explicit `:var` names are captured and supplied positionally to the semantic action. Use this when you prefer short patterns and positional args in your semantic functions.

These hooks allow you to do lightweight checks and normalization right next to the grammar instead of inside semantic actions.

Example:

```py
@rule('<pair:first> [<COMMA> <pair:p>]:rest', name='object',
      validate=lambda args: all(isinstance(k, str) for k, _ in ([args[0]] + args[1])),
      transform=lambda args: args[0] if len(args) == 1 else args)
def object_nonempty(self, first, rest):
    ...
```

---

## Memoization

The engine uses a conservative per-parse memo cache keyed by `(rule_name, position, min_prec)` to reduce repeated work. This speeds many grammars, but the cache is intentionally conservative to remain compatible with the left-recursive growth algorithm. You don’t need to enable anything - the `parse()` entrypoint initializes the cache automatically.

---

## Error messages

`ParseError` shows line, column, a snippet and caret. Newer messages include a short `Found: '...'` lookahead revealing what characters were present at the failure point and better "expected" hints for failed group/quantifier matches. This should make diagnosing syntax mistakes faster.
