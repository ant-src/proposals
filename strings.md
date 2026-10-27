# Proposal: one string syntax, raw lines

2026-10-27 · Anton

## Summary

`"..."` becomes the only quoted string: escapes, `${expr}` interpolation, and line breaks. Backtick
template strings go away. A backtick instead starts a raw line, as Zig's `\\` does: the rest of the
source line is text, with no escapes and no interpolation. Consecutive raw lines join with `\n`.
Spaces and tabs before each backtick are ignored. Single quotes stay unused.

```ant
var greeting = "hello, ${name}"

var query =
  `SELECT id, name
  `FROM users
  `WHERE email LIKE '%@example.com'
```

## Motivation

- Plain and template strings do one job. Adding `${x}` to a string should not mean changing quotes.
- Multi-line templates keep every byte, so their indentation leaks into the value and `ant fmt`
  cannot reindent them.
- Ant has no raw strings, so regexes, Windows paths, shell and JSON need `\\` and `\"`.
- A raw line ends at the end of the line, so it never needs an escape or a longer delimiter.

## Rules

**`"..."`**

- Escapes: `\n`, `\t`, `\r`, `\\`, `\"`, `\$`, `\u{...}`. `` \` `` is dropped.
- `${expr}` renders with `String.from`; braces nest inside. `\$` is a literal `$`; a `$` not
  followed by `{` needs no escape.
- A line break inside the quotes is one `\n` in the value, whatever the file uses. Indentation is
  kept as written.

**Raw lines**

- A raw string starts anywhere a literal may: ``var s = `abc``.
- A following line whose first non-blank character is a backtick continues it, adding `\n`. A blank
  line, a comment, or any other token ends it.
- Text after the backtick is kept exactly, trailing spaces included. A lone `` ` `` is `""`; an empty
  last line adds a trailing `\n`.
- The string ends its line, so the next token goes on the next line. Inside `(`, `[` and `{` that
  already works; a leading `.`, `?.` or `|` continues the expression as usual.

```ant
var origin = Regex.compile(
  `^https?://[^/"]+/
)

var script =
  `#!/bin/sh
  `echo "deploying ${APP}"
  `
```

`script` keeps `${APP}` for the shell and ends with `\n`.

## Edge cases

| Source                                   | Result                                                   |
| ---------------------------------------- | -------------------------------------------------------- |
| blank or comment line between raw lines  | two strings; the discarded second one is a syntax error  |
| `` `abc` `` (JS, Go, old Ant habit)      | syntax error: one-line raw text may not end in `` ` ``   |
| ``f(`a`, b)`` on one line                | the raw text is ``a`, b)``; error notes raw runs to EOL  |
| `"${"}"}"`                               | `"}"`: strings nest inside interpolation                 |
| `"${2}@$1"` for `Regex.replace`          | now interpolates; write `"\${2}@$1"` or a raw line       |
| `name="${x}"` in HTML markup             | unchanged: attribute quotes are markup, not Ant strings  |
| CRLF source file                         | both forms produce `\n` only                             |

The discarded-statement rule is the one HTML expressions already have (section 5).

## `ant fmt`

Raw lines are reindented like code: backticks at the statement's indentation, or one level deeper
when the string starts after other tokens. Text after the backtick and text inside `"..."` are kept
byte for byte. fmt never converts between the two forms.

## Trade-offs

- Every raw line carries a backtick; pasted text needs column editing. In return the formatter owns
  indentation and nothing is ever escaped.
- No interpolation in raw text: indented text with values is `"..."` with indentation as written, or
  raw lines joined with `+`. This is the weakest case.
- `${` gains meaning in `"..."`, which breaks the regex replacement strings in the corpus.
- JS and Go readers expect `` `...` `` to be delimited; the two diagnostics above exist for that.

## Spec changes

- **ant-lang.md §2 Literals:** new examples, escapes paragraph, raw string rules and diagnostics;
  remove "There are no raw strings".
- **§2 Line endings:** line breaks in `"..."` are `\n`; raw continuation onto backtick lines.
- **§5:** discarded raw string is a syntax error; HTML `{expr}` text refers to `${...}` of strings.
- **§11 String, Regex; §12 Literals; stdlib.md Time:** "template string" becomes string
  interpolation; rewrite the `"$1"` note.
- **fmt.md 3, 4.1, 4.9, 4.10:** 4.9 becomes Strings with raw line indentation; drop the CRLF
  exception for strings.

Migration: Ant is unreleased, so a script rewrites the ~100 corpus files with templates in one step
(`"` → `\"`, `` \` `` → `` ` ``) and adds `\$` to plain strings that contain `${`.

## Open questions

1. Add interpolating raw lines with a second marker, such as `` $` ``? Recommended: not yet.
2. Strip indentation from multi-line `"..."` as Swift does? Recommended: no; raw lines cover it.
3. Is the one-line trailing-backtick error worth losing raw text that ends in `` ` ``?
