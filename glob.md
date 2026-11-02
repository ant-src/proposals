# Proposal: globbing

2026-11-02 · Anton · Accepted, in spec 92675aa

## Summary

Two functions, one pure and one that reads the disk:

```
Path.match(pattern: string, path: string, hidden: bool = false): Result[bool, Error]   // Err on a malformed pattern
File.glob(pattern: string, dir: string = ".", hidden: bool = false): Result[Array[string], Error]   // the paths below dir that match, in lexical order
```

- **The syntax is Go's `filepath.Match`** (`*`, `?`, `[a-z]`, `[^a-z]`, `\` to escape on Unix) plus
  `**` as a whole path element, which crosses directories, and `[!a-z]` as the shell writes it. No
  `{a,b}`.
- **Wildcards skip dot files**, as the shell does. `hidden: true` lets them match.
- **`**` does not descend through symbolic links**, as `File.walk` does not.
- **No match is an empty array.** A malformed pattern and an unreadable directory are `Err`.
- **`File.glob` is `File.walk` filtered by `Path.match`**, plus directories, without reading what
  the pattern rules out.

```ant
for path in try File.glob("*.txt") {          // a.txt, b.txt; not .notes.txt
  println(path)
}

var tests = try File.glob("**/*_test.ant", dir: root)   // paths relative to root, as walk's
var dirs = try File.glob("*/")                          // directories only: "src/", "tests/"

if try Path.match("*.log", Path.base(name)) { ... }
```

## Motivation

Ant is meant to replace shell scripts, and the first line of many is a glob:

| Shell                          | Ant today                                                           |
| ------------------------------ | ------------------------------------------------------------------- |
| `for f in *.txt`               | `File.list(".")`, then `String.endsWith(x, ".txt")` and a dot check |
| `rm -f build/*.o`              | the same, plus `Path.join` on every name                            |
| `shopt -s globstar; **/*.ant`  | `File.walk(".")` and `Path.ext`; reads every directory, `.git` too  |
| `for d in */`                  | `File.list`, then `File.stat` on each name                          |
| `[[ $name == *.@(jpg\|png) ]]` | `Path.ext(name)` against a list                                     |
| `ls src/*/main.ant`            | `File.list("src")`, then `File.exists` of each `Path.join`; by hand |

Each hand-written filter forgets something: the dot files, the directories, the order, or `walk`
descending into a 100 000-file `node_modules` when the pattern needed one directory.

## Prior art

### Syntax

| System                                | `*` `?` `[..]` | `[!..]` | `**`                     | `{a,b}`        | Escape                       |
| ------------------------------------- | -------------- | ------- | ------------------------ | -------------- | ---------------------------- |
| Go `filepath.Match`, `Glob`           | yes            | `^`     | no                       | no             | `\` on Unix; none on Windows |
| Go `doublestar` (package)             | yes            | `^` `!` | whole element            | yes, nesting   | `\`                          |
| POSIX `glob(3)`, `fnmatch(3)`         | yes            | `!`     | no                       | GNU option     | `\`, unless `FNM_NOESCAPE`   |
| bash                                  | yes            | `!` `^` | `globstar` (4.0)         | text, not glob | `\`, quotes                  |
| zsh                                   | yes            | `!` `^` | yes; `***` follows links | text, not glob | `\`, quotes                  |
| Python `glob.glob`                    | yes            | `!`     | `recursive=True` (3.5)   | no             | `[*]`; `glob.escape`         |
| Python `pathlib` `glob`, `full_match` | yes            | `!`     | yes                      | no             | `[*]`                        |
| Node `fs.glob` (22, stable 24)        | yes            | `!` `^` | yes                      | yes            | none on Windows              |
| Bun `Bun.Glob`                        | yes            | `!` `^` | yes                      | yes            | `\`                          |
| Java `getPathMatcher("glob:…")`       | yes            | `!`     | yes                      | yes            | `\`                          |
| Rust `glob` crate                     | yes            | `!`     | whole element            | no             | `[*]`                        |
| Ruby `Dir.glob`                       | yes            | `^` `!` | yes                      | yes            | `\`                          |
| `.gitignore`                          | yes            | `!`     | yes                      | no             | `\`                          |

`*` and `?` never cross a separator anywhere except Python's `fnmatch`, which knows nothing of
paths. `**` is in everything newer than POSIX except Go's standard library. Braces split the field:
in the shell they are not a glob at all but text expansion, which makes `{a,b}` words whether files
exist or not.

### Dot files, links, order, errors

| System                | `*` matches `.x`                   | `**` through a link to a directory | Order        | No match                     | I/O error below the root      |
| --------------------- | ---------------------------------- | ---------------------------------- | ------------ | ---------------------------- | ----------------------------- |
| Go `filepath.Glob`    | yes                                | —                                  | lexical      | empty                        | ignored                       |
| Go `doublestar`       | yes; `WithNoHidden`                | yes; `WithNoFollow`                | —            | empty                        | ignored; `WithFailOnIOErrors` |
| bash, zsh             | no; `dotglob`, `(D)`               | no (bash 4.3+); zsh `***`          | sorted       | the pattern back; `nullglob` | ignored                       |
| Python `glob.glob`    | no; `include_hidden` (3.11)        | yes                                | none         | empty                        | ignored                       |
| Python `pathlib.glob` | yes                                | no; `recurse_symlinks` (3.13)      | none         | empty                        | ignored                       |
| Node `fs.glob`        | no                                 | no; `followSymlinks` (24.16)       | none         | empty                        | —                             |
| Bun `Bun.Glob.scan`   | no; `dot`                          | no; `followSymlinks`               | none         | empty                        | —                             |
| Rust `glob` crate     | yes; `require_literal_leading_dot` | —                                  | alphabetical | empty                        | yielded per entry             |

The scripting languages that came after the shell skip dot files by default, and pathlib and Go do
not. Following links under `**` was the old default (bash before 4.3, Python's `glob`) and is now
opt-in (bash 4.3, zsh, pathlib 3.13, Node, Bun), because a link to `..` loops and a link to `/`
walks the machine. Only Go and the shell sort; only the shell gives the pattern back. Nearly
everyone ignores I/O errors, Go included.

### Case

Go matches case-sensitively on every system: `filepath.Glob("*.TXT")` does not find `a.txt` on
Windows or macOS. Python's `glob` folds case on Windows, `pathlib` takes `case_sensitive` (3.12) and
folds on Windows by default, and Node folds on Windows and macOS. The shell folds only with
`nocaseglob`.

## Design

### Syntax

A pattern is split into elements at the separator, and each element is matched against one element
of the path:

| Pattern  | Matches                                                               |
| -------- | --------------------------------------------------------------------- |
| `*`      | any run of characters, the empty one included, within one element     |
| `?`      | one code point                                                        |
| `[abc]`  | one code point of the set; `[a-z]` a range, by code point             |
| `[^a-z]` | one code point not in the set; `[!a-z]` is the same                   |
| `\*`     | `*` itself, on Unix; on Windows `\` is a separator, and `[*]` escapes |
| `**`     | as a whole element, zero or more elements; elsewhere, as `*`          |
| other    | itself                                                                |

This is Go's `filepath.Match`, so the matcher can be Go's for every element but `**`, with three
additions:

- **`**` crosses directories.** `**/*.ant` matches `a.ant` and `x/y/a.ant`. `src/**` matches
  everything below `src`, not `src` itself. `**` inside an element, as in `a**b`, is `*`, as in bash
  and `doublestar`. Go's standard library lacks it, and `File.walk` plus a filter is the Go answer,
  but every shell and scripting language has it, and a script that cannot write `**/*.ant` is the
  case this proposal is for.
- **`[!...]` negates**, as `[^...]` does in Go. Every shell and POSIX write `!`. In Go, `[!a]` is
  the set of `!` and `a`, a reading no script means.
- **A trailing separator** matches directories only, and the path that comes back keeps it, as in
  the shell and Python: `*/` gives `src/`.

`?` and the sets work on code points, as Go's on runes and as `String.length` counts. A name in a
decomposed form, `e` and a combining accent, is two code points to `?`.

On Windows both `/` and `\` separate, as they do for `Path.join` and Go's `filepath`, so a pattern
written with `/` works on every system. `\` cannot escape there, so a portable pattern escapes with
brackets: `[*]`, `[?]`, `[[]`.

### Dot files

A wildcard does not match a `.` at the start of an element: `*` matches `a.txt` and not `.env`, `**`
does not descend into `.git`, and `?ab` does not match `.ab`. A pattern element that begins with a
literal `.`, as `.*` or `.git`, matches them. `.` and `..` are matched only by themselves.
`hidden: true` lifts the rule for the wildcards.

This parts from Go, whose `*` matches `.env`, for the shell's rule. Scripts mean "the files" when
they write `*`. A `**` that walks `.git` is slower and lists files nobody wanted, and a script that
deletes `build/*` would take `.gitkeep` with it. Python's `glob`, Node and Bun made the same choice.
`hidden` is a parameter and not only `.*` in the pattern, because without braces no pattern says
"every file below, hidden ones too": `**/*` and `**/.*` together still never enter `.config/`.

### Links

`**` does not descend through a link to a directory, as `File.walk` does not, and the link itself is
matched by its name, as a file. A literal element or a single `*` element goes through a link, as
`ls link/*` does: `link/*.txt` lists the files where the link points. A dangling link is matched by
its name, as in the shell and Python. There is no `follow` parameter; it can be added later as Node
and pathlib did.

### `dir` and the results

`File.glob(pattern, dir)` matches a relative pattern from `dir` and returns paths relative to it, as
`File.walk(dir)` does, so `Path.join([dir, p])` reaches each. An absolute pattern ignores `dir` and
returns absolute paths. `dir` is a parameter and not a prefix of the pattern so that a directory
whose name holds `*` or `[`, from an argument or a temporary name, needs no escaping. This is
Python's `root_dir` and Node's `cwd`.

The results:

- **Files and directories both,** as the shell gives them, and only directories for a trailing
  separator. `File.walk` is for files only.
- **In lexical order** by element, the order `File.walk` uses, so `a/x` comes before `a-b/x`. Go and
  the shell sort, and a script that prints or deletes in a stable order needs it.
- **Each path once,** even when two `**` could reach it, which Python does not promise.
- **Cleaned,** as `Path.clean` gives them, except for the trailing separator of a directory pattern.
  `./*.txt` gives `a.txt`.
- **An empty array when nothing matches.** The shell's pattern-back is the bug behind `rm *.tmp`
  removing a file named `*.tmp`, and every language since returns nothing.

### Case

Wildcards and sets compare code points exactly, on every system, as Go does. A literal element is
found by the file system, so on macOS and Windows `README.MD` finds `README.md`, and the path that
comes back is spelled as the pattern spelled it, as Go's `Glob` returns it. `*.TXT` does not match
`a.txt` anywhere. So the wildcards of `Path.match`, which has no disk to ask, and of `File.glob`
agree on every system, and only a literal element, which the program spelled out, is the file
system's to resolve. A `caseSensitive` parameter, as pathlib has, can come later.

### Errors

`Path.match` is `Err` on a malformed pattern: a `[` without its `]`, a range such as `[z-a]`, or a
trailing `\` on Unix. `File.glob` is `Err` on a malformed pattern before it reads anything, so a bad
pattern fails on an empty directory too, as Go checks it.

`File.glob` is also `Err` when `dir` is not a directory it can read, and when it cannot read a
directory the pattern leads into, as `File.walk` is. Go, Python and the shell skip those silently.
Ant's rule is that what fails returns `Err`: a script that deletes `**/*.tmp` and is told it did so
should not have skipped a directory it could not open. A program that wants the shell's behaviour
catches the `Err` and falls back to `File.walk` and `Path.match`. A directory that does not exist
below `dir`, as `missing/*.txt` asks for, is no match, not `Err`.

### `Path.match`

`Path.match` matches the whole of `path`, element by element, with the same rules as `File.glob`, so
`File.glob(p, dir)` gives the files and directories below `dir` whose relative path `q` has
`Path.match(p, q) == Ok{ value: true }`, except for the links that `**` does not enter. It looks
only at the text: it does not clean `path`, so `Path.match("*.txt", "./a.txt")` is
`Ok{ value: false }`, and for a name alone a program passes `Path.base`. The argument order is Go's
and `Regex.test`'s, pattern first.

It returns a `Result`, as Go's `Match` returns an error, because the `stdlib.md` rule is that a
namespace there does not panic on bad input, and a pattern often comes from an argument. In a filter
that is `Path.match(p, x) == Ok{ value: true }`.

## Considered and rejected

- **`{a,b}` alternatives.** Go, Python, Rust's `glob` and `.gitignore` have none, and in the shell
  they are text expansion, not matching. They make `,` and `{` special in every pattern, so a name
  such as `{draft}.txt` needs escaping, and on Windows `\` cannot escape it. `*.{jpg,png}` is
  `Path.ext` against a list, or two globs.
- **Following links under `**`,** see Links.
- **The pattern back when nothing matches,** see the results.
- **`Path.match` returning `bool`,** false on a malformed pattern, as Go's `Match` does alongside
  its error. A typo in a pattern would then match nothing, silently, which is what `rm` with the
  wrong pattern does in the shell.
- **A compiled pattern,** `Glob.compile` returning a value as `Regex.compile` does. Matching is
  cheap and patterns are short, and Go compiles nothing.
- **A lazy `Seq` result,** as Python's `iglob` and Node's async iterator. `File.walk` returns an
  array, and sorting needs all of it.
- **`Path.escapeGlob`,** Python's `glob.escape`. `dir` covers the case that needs it, a directory
  from outside, and a name inside a pattern can bracket its specials.
- **Extended globs,** bash's `@(a|b)`, `!(x)`, zsh's qualifiers such as `(.)` and `(om)`. A program
  filters with `File.stat` and `Array.filter`.
- **Exclusion patterns,** Node's `exclude`, `.gitignore` files. `Array.filter` with `Path.match`.

## Spec changes

- `stdlib.md`, File: `File.glob` and a paragraph on its results, links and errors. A row in the
  links table: `glob` does not descend through a link under `**`, and goes through one in a literal
  element.
- `stdlib.md`, Path: `Path.match` and the pattern syntax table, the dot rule and the case rule.
- `ant-lang.md`: none.

Implementation, beyond the spec: `internal/lib` gets a matcher per element on top of
`filepath.Match`, with `**` and `[!` handled before it; `File.glob` reads only the directories the
pattern's elements lead into, and `os.ReadDir` per directory; `prelude.d.ant` docs; tests in
`tests/run`, with a link loop and an unreadable directory on Unix.

## Non-goals

- `{a,b}`, extended globs and exclusion, see Considered and rejected.
- Case folding.
- Globbing in `Cmd`: `Cmd.new("rm", "*.o")` passes `*.o` as it is, as Go's `exec.Command` does, and
  a program expands it with `File.glob` first.
- A glob in `import` paths or `ant test` arguments.

## Decisions

Accepted 2026-11-02 as proposed:

1. `**` as a whole element, zero or more elements, one or more as the last; elsewhere it is `*`.
2. Wildcards skip names that begin with a dot; `hidden: true` lifts it.
3. A directory the pattern leads into that cannot be read is `Err`, as for `File.walk`.
4. `Path.match` returns `Result[bool, Error]`.
5. No braces.
6. A trailing separator matches directories only, and the result keeps it.

Settled in writing the spec: in a set, `-` and `]` are escaped, as in Go, and a range whose end
comes before its start is malformed, which Go does not check.

## Open questions

None.
