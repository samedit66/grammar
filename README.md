# grammar - tiny parsing DSL

A small, pleasant-to-use declarative parsing mini-library with a compact pattern DSL, per-alternative precedence, limited left-recursive operator support, and convenient repetition/quantifier syntax. Intended for small DSLs, examples, and teaching - not a production-grade parser generator.

---

## Features

A tiny parser you actually want to write grammars in - readable, pragmatic, and a little clever.

- **Friendly DSL.** Patterns are compact and clear: named captures, quoted literals, and bare punctuation all work together so rules read like the grammar in your head.
- **Straightforward repetition.** Built-in quantifiers let you write lists and repetitions without awkward recursion: `<item*:xs>`, `<item+:xs>`, `[ ... ]:list` and `{ ... }:list` give you easy access to lists of captured values.
- **Optionals with defaults.** Want a value when something’s missing? Use `<field?:name=42>` or just `<field?:name>` (gets `None` when absent).
- **Real operator support.** Left-recursive operator forms with per-alternative `prec` and `assoc` make arithmetic and expression grammars concise and correct. Prefix/postfix unary forms supported too.
- **Lightweight hooks.** `@rule(..., transform=..., validate=...)` lets you normalize or reject captures before the semantic action runs - great for tidy semantics.
- **Smarter lexing.** `token()` auto-decides whether to wrap patterns with `\b` and won’t accidentally break tokens such as negative numbers or exponent notation.
- **Helpful errors and decent speed.** Error messages include line/column and friendly hints for failed group matches. A conservative per-parse memo cache speeds up heavier grammars without changing semantics.

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

- `'<lit>'` or `"<lit>"` - literal text. Unquoted punctuation such as `,` is also accepted.
- `<NAME:var>` - reference to a token or rule `NAME`; when `:var` is present the matched value is passed as an argument to the semantic function.
- `<NAME?:var>` - optional reference; if missing and `:var` provided the var receives `None` (or a default if specified - see below).
- `<NAME?:var=DEFAULT>` - optional with default value; `DEFAULT` is parsed with Python literal rules where possible (numbers, quoted strings); otherwise it is treated as a raw string.
- `<NAME*:xs>` - zero-or-more repetition of a **single** reference; when named (here `xs`) yields a `list` (possibly empty) of matched values.
- `<NAME+:xs>` - one-or-more repetition of a single reference; named capture yields a non-empty `list`.
- `[ ... ]:name` - group repetition (zero-or-more): the group inner pattern is attempted repeatedly; if `:name` is provided you get a `list` of items where each item is either the single captured value (if the inner sequence had exactly one named capture) or a `tuple` of captures.
- `{ ... }:name` - group repetition (one-or-more): fails if zero occurrences were found.

Examples:

- `@rule("<expr:x> [',' <expr:y>]:rest")` → `x` is first `expr`, `rest` is a list of following exprs.
- `@rule("<id?:name='anon'>")` → `name` will be the matched id, or `'anon'` if missing.
- `@rule("<num*:ns>")` → `ns` becomes a list of numbers (maybe empty).

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

`ParseError` shows line, column, a snippet of the failing line and a caret pointing to the position. When group repetitions fail (for example a required `{ ... }` didn't match any occurrences) the error now includes a friendly expected hint such as `one or more '<item>'` when possible.
