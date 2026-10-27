# Proposal: one string syntax, raw lines

2026-10-27 · Anton

## Summary

`"..."` becomes the only quoted string. It takes escapes, interpolates `${expr}`, and may span lines.
Backtick template strings go away. The backtick instead starts a raw string line, as Zig's `\\`
does: everything after `` ` `` up to the end of the line is text, taken as written, with no escapes
and no interpolation. Raw lines on consecutive source lines form one string, joined with `\n`.
Spaces and tabs before each backtick are not part of the string. Single quotes stay unused.

```ant
var greeting = "hello, ${name}"

var query =
  `SELECT id, name
  `FROM users
  `WHERE email LIKE '%@example.com'
```

`query` is `"SELECT id, name\nFROM users\nWHERE email LIKE '%@example.com'"`.

## Motivation

1. **Two quoted forms do one job.** A plain string is a template string without interpolation and
   without line breaks. Choosing between them is a decision with no payoff, and adding `${x}` to a
   plain string means changing its quotes. JavaScript's own guides now say to prefer backticks
   everywhere; Ant can skip the split.
2. **Multi-line text fights indentation.** Today a multi-line template keeps every byte, so a SQL
   query inside a function either starts its continuation lines at column 0 or carries the
   function's indentation into the value. `ant fmt` cannot reindent it (fmt 4.9), so the code
   around it cannot move either.
3. **There are no raw strings.** Regex sources for `Regex.compile`, Windows paths, shell snippets,
   JSON samples and test fixtures all need `\\` and `\"` today. Section 2 says outright "There are
   no raw strings".
4. **Raw lines have no delimiter to collide with.** The text ends at the end of the line, so it can
   hold `"`, `` ` ``, `${`, `\` and anything else without an escape, and no fence length or hash
   count is ever needed.

## Prior art

| Language   | Interpolating               | Raw                         | Indentation of multi-line text                   |
| ---------- | --------------------------- | --------------------------- | ------------------------------------------------ |
| Zig        | none                        | `\\` line prefix            | before `\\` ignored, after kept                  |
| JavaScript | `` `${x}` ``                | `String.raw` tag            | kept                                             |
| Go         | none                        | `` `...` ``                 | kept                                             |
| Swift      | `"\(x)"`, `"""` multi-line  | `#"..."#`                   | stripped up to the closing `"""`                 |
| Kotlin     | `"$x"`, `"""` multi-line    | `"""` (no escapes, but `$`) | kept; `trimIndent()` by convention               |
| Rust       | `format!` only              | `r#"..."#`                  | kept; `\` at line end skips the break and indent |
| Java       | none (`formatted`)          | none                        | text blocks strip common indent                  |

Zig's form is the one taken here for raw text. It needs no closing delimiter, never needs escaping,
and lets the formatter reindent freely. Its cost is that every line carries a marker, which Zig
users accept for the reasons in the trade-offs below.

## Proposal

### `"..."` strings

- Escapes as today: `\n`, `\t`, `\r`, `\\`, `\"`, `\$` and `\u{...}`. `` \` `` is dropped, since a
  backtick needs no escape inside `"..."`; it becomes an unknown escape, a syntax error.
- `${expr}` interpolates any expression, rendered with `String.from`, with braces nesting inside, as
  template strings do now. `\$` writes a literal `$`; `$` not followed by `{` needs no escape, so
  `"$5"` and `"$1"` are fine.
- A line break inside the quotes is part of the string. It is always one `\n` in the value, whichever
  of `\n`, `\r\n` or `\r` the file used; a carriage return is written `\r`. The rest of each line,
  leading spaces included, is kept as written.

```ant
var report = "user ${user.name} has ${len(orders)} orders
total: ${Array.sum(Array.map(orders, { x.amount }))}"

var price = "$5, not \${price}"   // "$5, not ${price}"
```

### Raw strings

A raw line is a backtick followed by the rest of the source line. Its text is every character after
the backtick up to the line break, excluding the line break.

- **First line.** A raw string may start anywhere a string literal may, including after other tokens
  on the same line: ``var s = `abc``.
- **Continuation.** If the next source line, after spaces and tabs, starts with a backtick, it adds
  `\n` and its text to the string. The run ends at the first line that does not: a blank line, a
  comment line, or any other token.
- **Indentation.** Spaces and tabs before a backtick are not part of the string. Everything after it
  is, including leading and trailing spaces and tabs.
- **No escapes, no interpolation.** `\`, `"`, `` ` `` and `${` are ordinary characters.
- **Ends.** The string has no trailing line break unless a final empty `` ` `` line adds one. A
  single `` ` `` alone is `""`.
- **What follows it.** A raw string ends its line, so a token after it goes on the next line. Inside
  parentheses, brackets and braces that is already allowed, since newlines there are ignored. A line
  break after a raw string ends the expression as any line break does, so `.`, `?.` and `|` on the
  next line continue it under the existing rule (section 2, Line endings).

```ant
var origin = Regex.compile(
  `^https?://[^/"]+/
)

var script =
  `#!/bin/sh
  `set -eu
  `echo "deploying ${APP}"
  `

var path = `C:\Users\anton\notes.txt

var banner =
  `   Ant   
  | String.trim()
```

`origin` needs no `\/` and no `\"`. `script` keeps `${APP}` for the shell and ends with `\n`, from
the empty last line. `banner` shows a pipeline continuing after a raw string.

### Edge cases

| Source                                         | Value or result                                                 |
| ---------------------------------------------- | --------------------------------------------------------------- |
| `` ` ``                                        | `""`                                                            |
| `` `a `` then `` ` `` on the next line         | `"a\n"`                                                         |
| `` `a `` then `` `b `` with a blank line between | two strings; the second is a discarded statement: syntax error |
| `` `a `` then `// note` then `` `b ``          | same as the blank line                                          |
| `` `  x  `` (spaces after the backtick)        | `"  x  "`; trailing spaces kept                                 |
| tab before the backtick, tab after it          | first ignored, second kept                                      |
| `` `abc` `` (old template habit)               | syntax error, see below                                         |
| `"${"}"}"`                                     | `"}"`: a string inside an interpolation nests                   |
| `"\${x}"`                                      | `"${x}"`                                                        |
| `"$"`, `"a$b"`, `"${"`                         | `"$"`, `"a$b"`; `"${"` is an unterminated interpolation         |
| file saved with CRLF                           | both forms produce `\n` only                                    |
| raw string as the last line of a file          | text ends at end of file, no line break added                   |

**Discarded raw strings.** A raw string that is a statement whose value is discarded is a syntax
error, as an HTML expression is (section 5). This is what makes a blank or comment line inside a
run fail loudly instead of silently cutting the string short. The message says that raw lines must
be consecutive and suggests an empty `` ` `` line for a blank line.

**Old backtick strings.** ``println(`hi ${name}`)`` would read as the raw text ``hi ${name}`)`` and
then fail on the missing `)`, with a confusing message. Two rules make the habit, from JavaScript
and Go as well as old Ant, fail clearly:

1. A one-line raw string whose text ends in `` ` `` is a syntax error whose message suggests
   `"..."`. Text that truly ends in a backtick is written as a `"..."` string.
2. When parsing fails after a raw string whose first line holds another backtick, the error notes
   that a raw string runs to the end of its line.

### Regex replacement strings

`Regex.replace` expands `${n}` and `${name}` in its replacement (section 11, Regex). Those strings
must now escape the `$` or be raw: `"\${2}@$1"`, or `` `${2}@$1 `` on its own line. `$1` alone is
unaffected. The spec text "A plain string does not interpolate `$`, so `"$1"` needs no escape"
becomes a note that `$n` needs no escape but `${n}` does, with the raw form as the usual spelling.

### HTML attributes

Attribute values in markup, `name="..."`, are markup, not Ant strings, and stay without
interpolation; `{` there is still a syntax error pointing at `name={...}` (section 5). A raw string
can appear in markup only inside `{...}`, on lines of its own.

### `ant fmt`

- Raw lines are reindented like code. A run that starts its own line puts every backtick at that
  line's indentation; a run that starts after other tokens puts its later backticks one level deeper
  than the statement. The text after each backtick is kept byte for byte, trailing spaces included.
- `"..."` text is kept byte for byte, as template text is today (4.9), and so is not reindented.
- Line breaks inside strings no longer need the CRLF exception of 4.1, since their value is `\n`
  either way; `ant fmt` writes `\n`.
- fmt does not convert between the two forms.

## Trade-offs

- **A marker on every line.** Pasting a block of text needs a backtick added to each line. Editors
  do this with column selection; Zig users report it is rarely a problem. In exchange the formatter
  owns indentation and no content ever needs escaping.
- **No interpolation in raw text.** An indented multi-line message with values in it is either a
  `"..."` string with its indentation as written, or raw lines joined to values with `+` or
  `String.join`. This is the weakest case of the design; see open questions.
- **`$` gains meaning in `"..."`.** Any existing plain string containing `${` changes meaning. In
  the current corpus that is the regex replacement strings in tests and docs, and each needs one
  `\`.
- **One token per line.** A raw string must be followed by a line break, so ``f(`a`, b)`` cannot be
  written on one line. Short raw text that needs to sit inline is better as `"..."` anyway.
- **Backtick breaks familiarity.** JavaScript and Go readers expect `` `...` `` to be delimited.
  The two diagnostics above exist because of that.

## Spec changes

`spec/ant-lang.md`:

- **Section 2, Literals.** Replace the template string example with a multi-line `"..."` and a raw
  string. Rewrite the escapes paragraph: drop `` \` ``, give interpolation and line breaks to
  `"..."`, and replace "There are no raw strings" with the raw string rules above, including the
  two backtick diagnostics.
- **Section 2, Line endings.** A line break inside `"..."` is `\n` in the value. A raw string ends
  at the line break and continues onto a following line that starts with a backtick; add that to
  the list of things that are not inferred, since it is a lexical rule and needs no lookahead past
  the next line's first token. A regex literal still cannot hold a line break.
- **Section 2, regex `/` rule.** "the start of a `${...}` interpolation" now means one inside
  `"..."`.
- **Section 5, Statements.** Extend the discarded-value syntax error from HTML expressions to raw
  strings. Interpolation braces in `{expr}` of HTML now refer to `${...}` of a string.
- **Section 11, String.** "`print`, `println` and template strings" and "only inside template
  strings" become string interpolation.
- **Section 11, Regex.** Rewrite the replacement note as above.
- **Section 12, Literals.** "A template string evaluates each `${...}`" becomes a string literal.

`spec/stdlib.md`: "template strings" in the Time text section becomes string interpolation.

`spec/fmt.md`:

- **3** (what is kept): string literals and raw text keep their spelling.
- **4.1**: drop the CRLF exception for strings; the trailing-space and tab exceptions cover the text
  of `"..."` strings and of raw lines; raw lines are no longer an indentation exception.
- **4.9**: rename to Strings; cover `"..."` interpolations as now, and add raw line indentation.
- **4.10**: "formatted as a template string is" becomes as a `"..."` string is.

## Migration

Ant has no release yet, so this is a one-step change with no deprecation period. A script rewrites
every template string in the corpus (`ant/tests`, `ant/examples`, `internal/format/testdata`,
`internal/prelude`, about a hundred files) to `"..."`: `"` becomes `\"`, `` \` `` becomes
`` ` ``, and the text is otherwise unchanged. Plain strings containing `${` get `\$`. Templates
that were multi-line only to hold a block of text are then rewritten by hand to raw lines where the
result reads better.

## Non-goals

- Single-quoted strings or a character literal.
- Indentation stripping for multi-line `"..."` (Swift, Java text blocks).
- A line-continuation escape, `\` at end of line (Rust).
- Tagged templates or custom interpolation handlers.
- Interpolation in raw lines through a second prefix.

## Open questions

1. **Interpolating raw lines.** Should a second marker, say `` $` ``, start a line that interpolates
   `${...}` but keeps the raw lines' indentation rule? It would cover indented SQL and HTML-like
   text with values. Recommended: not now; wait for real programs that need it.
2. **Dedent for `"..."`.** Instead of the above, a multi-line `"..."` could strip the indentation of
   its closing quote's line, as Swift does. That makes `"..."` change meaning when reindented,
   which is why this proposal leaves indentation to raw lines.
3. **The one-line trailing backtick rule.** It costs a real case (raw text ending in `` ` ``) to
   catch a habit. The alternative is only the second, softer diagnostic.
