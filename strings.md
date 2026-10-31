# Proposal: one interpolating string, Go raw strings

2026-10-27 · Anton · Accepted

## Summary

`"..."` becomes the only interpolating string: escapes, `${expr}` interpolation, and line breaks.
Backtick template strings go away. A backtick string becomes a raw string, as in Go: the text
between the backticks is kept as written, with no escapes and no interpolation, and may span lines.
Single quotes stay unused.

```ant
var greeting = "hello, ${name}"

var query = `SELECT id, name
FROM users
WHERE email LIKE '%@example.com'`
```

## Motivation

- Plain and template strings do one job. Adding `${x}` to a string should not mean changing quotes.
- Ant has no raw strings, so regexes, Windows paths, shell and JSON need `\\` and `\"`.
- Go's raw string is familiar, delimited on both sides, and fits inline in calls: ``f(`a`, b)``.

## Rules

**`"..."`**

- Escapes: `\n`, `\t`, `\r`, `\\`, `\"`, `\$`, `\u{...}`. `` \` `` is dropped.
- `${expr}` renders with `String.from`; braces nest inside. `\$` is a literal `$`; a `$` not
  followed by `{` needs no escape.
- A line break inside the quotes is one `\n` in the value, whatever the file uses. Indentation is
  kept as written.

**`` `...` ``**

- Everything between the backticks is the value: `\`, `"`, `$` and `${` have no meaning.
- A raw string may span lines. Line breaks are `\n` (carriage returns are dropped, as in Go), and
  indentation is kept as written.
- A raw string cannot contain a backtick. Text with one is `"..."` or raw strings joined with `+`.

```ant
var origin = Regex.compile(`^https?://[^/"]+/`)

var script = `#!/bin/sh
echo "deploying ${APP}"
`
```

`script` keeps `${APP}` for the shell and ends with `\n`.

## Edge cases

| Source                                | Result                                                  |
| ------------------------------------- | ------------------------------------------------------- |
| `` `C:\temp\new` ``                   | backslashes kept: `C:\temp\new`                         |
| `` `a ${x} b` `` (old template habit) | text `a ${x} b`; no interpolation                       |
| `"${"}"}"`                            | `"}"`: strings nest inside interpolation                |
| `"${2}@$1"` for `Regex.replace`       | now interpolates; write `"\${2}@$1"` or `` `${2}@$1` `` |
| `name="${x}"` in HTML markup          | unchanged: attribute quotes are markup, not Ant strings |
| unterminated `` ` ``                  | syntax error at the opening backtick                    |
| CRLF source file                      | both forms produce `\n` only                            |

## `ant fmt`

Text inside `"..."` and `` `...` `` is kept byte for byte. fmt never converts between the two forms
and never reindents the continuation lines of a multi-line string.

## Trade-offs

- Multi-line raw strings carry their indentation into the value, so they sit at column 0 or the
  value gets leading spaces. Go has the same cost.
- No interpolation in raw text: indented text with values is `"..."` with indentation as written, or
  raw strings joined with `+`.
- No backtick inside a raw string. Markdown code spans and shell command substitution need `"..."`.
- `${` gains meaning in `"..."`, which breaks the regex replacement strings in the corpus.
- Old template strings still parse, as raw strings, so a missed migration gives literal `${x}` text
  instead of an error.

## Spec changes

- **ant-lang.md §2 Literals:** new examples, escapes paragraph, raw string rules; remove "There are
  no raw strings".
- **§2 Line endings:** line breaks in both forms are `\n`.
- **§5:** HTML `{expr}` text refers to `${...}` of strings.
- **§11 String, Regex; §12 Literals; stdlib.md Time:** "template string" becomes string
  interpolation; rewrite the `"$1"` note.
- **fmt.md 3, 4.1, 4.9, 4.10:** 4.9 becomes Strings; drop the CRLF exception for strings.

Migration: Ant is unreleased, so a script rewrites the ~100 corpus files with templates in one step
(backtick → `"`, `"` → `\"`, `` \` `` → `` ` ``) and adds `\$` to plain strings that contain `${`.

## Decisions

1. No interpolating raw strings (`` $`...` ``) for now.
2. Multi-line strings keep their indentation as written, as in Go; nothing is stripped.
3. A raw string that contains `${` does not warn: shell scripts and regex replacements use it on
   purpose, and the migration rewrote every template in the corpus.
4. `${...}` over `{...}`: braces are common in string text (JSON, CSS, code), `$` is rare, and the
   form is the one JS, Kotlin and shell readers know. Markup keeps `{...}`, as JSX does.
5. `String.from` writes a `$` before `{` as `\$` inside quotes, so a rendering reads back.

Accepted 2026-10-30: spec 10efa5d, ant 376d736.
