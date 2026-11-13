# Proposal: flags, typed keys over Go's rules

2026-11-11 · draft

## Summary

A `Flags` namespace in the standard library that sits between the two drafts before it. From
`flags.md` it takes typed keys: each declaration returns a `Flags.Key[T]`, and reading a value with
it is checked by the compiler, so a misspelled flag or a wrong type does not compile. It also takes
converters, which are ordinary functions, and declared positional arguments. From `flag.md` it takes
Go's command-line rules and Go's usage text, so the spec states them by pointing at Go, and
`parseOrExit` exits as Go's `ExitOnError` does. It leaves out shell completion, environment
fallbacks, value sources, counters, and GNU syntax. Each of those can come later without changing
what is here.

```ant
#!/usr/bin/env ant

let cli = Flags.new(description: "Resize images.")
let width = cli.option("width", Flags.int, 800, alias: "w", help: "target width in `px`")
let quality = cli.option("quality", Flags.int, 85, help: "JPEG quality, 1 to 100")
let force = cli.flag("force", alias: "f", help: "overwrite existing files")
let files = cli.arguments("file", Flags.string, min: 1)

var args = cli.parseOrExit()

for path in args.get(files) {           // Array[string]
  resize(path, args.get(width), overwrite: args.get(force))   // int, bool
}
```

```
$ ./resize.ant -h
usage: resize.ant [flags] file...
Resize images.

  -f, -force
    	overwrite existing files
  -quality int
    	JPEG quality, 1 to 100 (default 85)
  -w, -width px
    	target width in px (default 800)
$ ./resize.ant -widht 10 a.png
flag provided but not defined: -widht
usage: resize.ant [flags] file...
...
$ echo $?
2
```

## What it keeps, and from where

| Decision                                          | Taken from | Why                                                                                              |
| ------------------------------------------------- | ---------- | ------------------------------------------------------------------------------------------------ |
| `Flags.Key[T]`, `values.get(key)`                 | `flags.md` | The compiler checks a flag's name and type, where `set.int("port")` checks them only at run time |
| Converters `func(string): Result[T, Error]`       | `flags.md` | Any type, and validation is an ordinary function, with no list of built-in kinds                 |
| Declared positional arguments, `min`              | `flags.md` | "At least one file" is the check every script writes by hand                                     |
| `parseOrExit`, exit 2                             | both       | The common path is one line, with the status a shell expects for bad usage                       |
| Go's syntax: `-x` is `--x`, stop at first operand | `flag.md`  | Fewer rules, stated by reference; subcommands without a prefix mode                              |
| Go's usage format, declaration order              | `flag.md`  | No new format to specify; the author's order is better than Go's sorted one                      |
| `alias` instead of short names                    | `flag.md`  | Under Go's rules `-j` and `--j` are one flag, so a second name is all a short name is            |
| Completion, `env`, `source`, `count`, `required`  | neither    | Each is its own design; see Non-goals                                                            |

## Declarations

```
type Flags                                    // a parser: its declarations, in order
type Flags.Key[T]                             // one declaration's handle; get returns its T
type Flags.Values                             // the result of one successful parse
type Flags.Help = { text: string }            // a parse that asked for help, with the text to print
type Flags.Outcome = Flags.Values | Flags.Help

Flags.new(name: string = Path.base(Os.program()), description: string = ""): Flags

Flags.flag(f: Flags, name: string, default: bool = false, alias: string = "", help: string = ""): Flags.Key[bool]
Flags.option[T](f: Flags, name: string, convert: func(string): Result[T, Error], default: T, alias: string = "", help: string = ""): Flags.Key[T]
Flags.optional[T](f: Flags, name: string, convert: func(string): Result[T, Error], alias: string = "", help: string = ""): Flags.Key[T | null]
Flags.repeated[T](f: Flags, name: string, convert: func(string): Result[T, Error], alias: string = "", help: string = ""): Flags.Key[Array[T]]
Flags.argument[T](f: Flags, name: string, convert: func(string): Result[T, Error]): Flags.Key[T]
Flags.arguments[T](f: Flags, name: string, convert: func(string): Result[T, Error], min: int = 0, max: int | null = null): Flags.Key[Array[T]]

Flags.string(s: string): Result[string, Error]   // always Ok
Flags.int(s: string): Result[int, Error]         // String.toInt; Err "parse error"
Flags.float(s: string): Result[float64, Error]   // String.toFloat; Err "parse error"
Flags.bool(s: string): Result[bool, Error]       // Go's strconv.ParseBool; Err "parse error"
```

`Flags`, `Flags.Key[T]` and `Flags.Values` are opaque references, as a `Mutex` or a `Random` is:
they have no fields, `==` compares identity, and `Json.stringify` panics on them. A declaration is a
call on the parser, `cli.option(...)` for `Flags.option(cli, ...)`, which adds the flag and returns
its key. `T` is solved from the converter, so `cli.option("jobs", Flags.int, 4)` is a
`Flags.Key[int]`, and a default of another type is a compile error.

- **`flag`** is a switch: `false` unless given, or `default`. It is a `bool` flag in Go's sense, so
  `-force=false` turns off one that defaults to `true`.
- **`option`** takes a value and has a default, the `T` that `get` returns when the command line
  does not give it. The default is a value, not text: it is not converted, and the programmer makes
  it valid.
- **`optional`** takes a value and has none when absent: `get` returns `null`. A required flag is an
  `optional` and an `if`, which names the flag in its own words.
- **`repeated`** keeps every value given, in order: `-I a -I b` is `["a", "b"]`, and absent is `[]`.
  Values are not split on commas.
- **`argument`** is a required positional argument. **`arguments`** takes the rest, at least `min`
  and at most `max` of them; `min: 0, max: 1` is an optional one. Positional declarations are taken
  in order, and `arguments`, if present, is the last.

`name` and `alias` are a flag's names without dashes: `alias: "j"` makes `-j` the same flag as
`-jobs`. `help` is one line; a word in backquotes names the flag's value in the usage text, as in
Go. A positional `name` is the word the usage line and its errors show.

Declaration errors are bugs in the program, so they panic at the declaration, naming it: an empty
name, a name or alias that starts with `-` or contains `=` or white space, a name or alias used
twice in one parser, an `argument` after an `arguments`, a second `arguments`, `min < 0`, or
`max < min`. A declaration on a parser that has already parsed panics too, since its key could never
hold a value; a parser parses as many times as it is called.

## Parsing and values

```
Flags.parse(f: Flags, args: Array[string] = Os.args()): Result[Flags.Outcome, Error]
Flags.parseOrExit(f: Flags, args: Array[string] = Os.args()): Flags.Values
Flags.help(f: Flags): string                       // the usage text, below

Flags.get[T](v: Flags.Values, key: Flags.Key[T]): T
Flags.isSet[T](v: Flags.Values, key: Flags.Key[T]): bool    // the command line gave it, under its name or alias
```

`parse` prints nothing, exits nowhere, and reads no process state but its default argument. It
returns `Ok` of `Flags.Values`, `Ok` of `Flags.Help` when the arguments asked for help, or `Err` on
input the declarations reject. Each converter runs once per value given, left to right, and parsing
stops at the first `Err`; a failed parse gives no partial values.

`parseOrExit` is Go's `ExitOnError`: on `Help` it prints the text to standard output and exits 0; on
`Err` it prints the message and the usage text to standard error and exits 2; otherwise it returns
the values. It calls `Os.exit`, which skips deferred blocks, so a script that must clean up calls
`parse` and decides.

`get` panics when the key belongs to another parser, which is the one misuse the types allow. Keys
and values are not shared between parses: each call returns fresh values, and a `repeated` or
`arguments` array is the caller's to change.

## Command-line rules

Go's `flag.FlagSet.Parse`, with two additions, aliases and positional declarations:

- An argument that starts with `-` or `--` and has a name after it is a flag; `-jobs`, `--jobs`, and
  with `alias: "j"`, `-j` and `--j`, are one flag. Three dashes, or a dash and then `=`, is `Err`
  `bad flag syntax: ---jobs`. There are no clusters: `-fv` is the flag named `fv`.
- `-name=value` gives the text after the first `=`, and `-name=` gives `""`. A flag that is not a
  `flag` takes the next argument when it has no `=`, whatever that argument is, so `-o -x` gives
  `-x` and `-o --help` gives `--help`. With no next argument it is `Err`
  `flag needs an argument: -o`.
- A `flag` is `true` as `-name` and takes a value only after `=`, read by `Flags.bool`: `1`, `t`,
  `T`, `TRUE`, `true`, `True`, `0`, `f`, `F`, `FALSE`, `false`, `False`. So `-force file` is the
  switch and then an operand.
- A scalar flag given twice keeps the last value; both are converted.
- A converter's `Err` becomes `invalid value "x" for flag -jobs: ` and its message, naming the flag
  as the user wrote it.
- An unknown name is `Err` `flag provided but not defined: -widht`, with its dashes as written.
- Parsing stops at the first argument that is not a flag, at `-` alone, or after `--`, which is
  dropped. That argument and every one after it are operands, as in Go, so `tool a.png -force` has
  two operands. The operands fill the positional declarations in order: each `argument` takes one,
  converted, and `arguments` takes the rest. Too few is `Err` `missing argument file`; more than the
  declarations take is `Err` `unexpected argument "x"`; a converter's `Err` is
  `invalid value "x" for argument file: ` and its message.
- `-h`, `-help` and `--help`, when no flag of the parser has that name or alias, end parsing with
  `Flags.Help` before any converter, positional count, or value after them is checked. A flag's
  value is not a request for help: in `-o -h`, `-h` is the value of `o`.

The messages are Go's where Go has one, and all are lower case without closing punctuation
(`ant-lang.md` section 8, Wrapping errors).

## Usage text

`Flags.help(f)` and `Flags.Help.text` are the same string:

1. `usage: `, the parser's name, ` [flags]` when it declares any, and each positional name in order:
   `file`, `[file]` for one that may be absent, `file...` for `arguments` with `min: 1` or more,
   `[file...]` for `min: 0`.
2. The description, when there is one.
3. An empty line, and then each flag in declaration order, in Go's `PrintDefaults` form: two spaces,
   the alias and the name each with one dash and joined by `, `, a space and the value's name, then
   the help on the next line after four spaces and a tab. The value's name is the word in
   backquotes, with the backquotes removed from the help; or `int`, `float` or `string` for the
   built-in converters; or `value`. A `flag` has none. `(default …)` follows the help, the value
   written by `String.from` and a string in quotes, unless it is `false`, `0`, `0.0`, `""` or
   `null`. When the first part is one dash and one letter, the help follows it on the same line
   after a tab, as Go does for `-v`.

Every line ends in `\n`. The help flag itself is not listed, as Go does not list it.

## Subcommands

Since parsing stops at the first operand, a subcommand is an `argument` and an `arguments` that keep
the rest as text, and a second parser:

```ant
let cli = Flags.new()
let verbose = cli.flag("verbose", alias: "v", help: "say more")
let command = cli.argument("command", Flags.string)
let rest = cli.arguments("arg", Flags.string)
var top = cli.parseOrExit()

if top.get(command) == "build" {
  let build = Flags.new(name: "tool build")
  let out = build.option("out", Flags.string, "a.out", alias: "o", help: "write to `file`")
  let files = build.arguments("file", Flags.string, min: 1)
  var args = build.parseOrExit(top.get(rest))
  for file in args.get(files) { compile(file, args.get(out), verbose: top.get(verbose)) }
}
```

`tool -v build -o x a.ant` gives `-v` to the first parser and `-o x a.ant` to the second, and
`tool build -h` prints the second parser's help. No prefix mode and no `rest()` are needed.

## Custom values

A converter is any function from the text to a `Result`, so validation lives in it:

```ant
func percent(s: string): Result[int, Error] {
  var n = try Flags.int(s)
  if n < 1 || n > 100 { return Result.err(Error{ message: "must be 1 to 100" }) }
  Result.ok(n)
}

let quality = cli.option("quality", percent, 85, help: "JPEG `quality`")
```

`-quality 0` then reports `invalid value "0" for flag -quality: must be 1 to 100`. A check that ties
two flags together is ordinary code after the parse.

An environment fallback is a default computed before declaring, so the precedence of command line,
then environment, then a fixed value falls out of evaluation order:

```ant
let jobs = cli.option("jobs", Flags.int, String.toInt(Os.env("JOBS") ?? "") ?? 4, alias: "j")
```

## Spec changes

- `stdlib.md`: a `Flags` section after `Os`, with the declarations, functions, rules and usage
  format above. The intro's list of namespaces gains `Flags`.
- `ant-lang.md` section 4, Kinds: `Flags`, `Flags.Key` and `Flags.Values` join the opaque reference
  kinds compared by identity. Section 11 lists `Flags` among the prelude names that are types with
  functions declared on them, and among the namespaces in `stdlib.md`.
- `prelude.d.ant`: the types and functions with doc comments.
- `ant/docs`: a `lib/Flags` page whose examples run under `ant test`.

## Non-goals

- **Shell completion.** `flags.md`'s design, reading literal declarations from the script's syntax
  tree without running it, applies here unchanged, since declarations are calls with literal names.
  It touches the `ant` command and the shells, not the library, so it is a proposal of its own.
- **`env:` on a flag, and `source(key)`.** A default expression covers the common fallback (above).
  Telling the user which variable a value came from is help text the script can write.
- **GNU syntax.** No clusters, no `--no-x` negation, no options after operands, no `-vvv` counter;
  verbosity levels are an `option("v", Flags.int, 0)` given as `-v=2`.
- **`required`.** `optional` and a check in the script, which can say what the flag is for.
- **`--version`.** A `flag` named `version` and two lines of code.
- **Typed records through `Reflect`.** A library over `typed-data.md` can build a parser from a
  record type later, on top of these functions.

## Alternatives considered

- **`flag.md` as written.** Smaller, but `set.int("port")` turns a typo into a run-time panic, and
  every script checks its positional arguments by hand.
- **`flags.md` as written.** Its rules table has about twenty cases of GNU syntax that the spec
  would state one by one, its `env`/`source` machinery adds two unions for a fallback a default
  already gives, and completion ties the `ant` command to one library's call shape. Its `Key[T]`
  design is kept.
- **A record per flag, as `flag.md` does, but returning keys.** `Flags.Key[T]` needs `T` solved from
  something; an array of `Flag` records of different `T` has one element type. One call per
  declaration is what gives each key its own `T`.

## Open questions

1. **Durations.** Go has `flag.Duration` with `1h30m`; `Time.parseDuration` reads ISO 8601. A
   `Flags.duration` converter with Go's syntax, or nothing until a script needs it?
2. **Help for an absent declaration.** Should `-h` be answered even when a parser declares a flag
   named `h`? Go says no; this follows Go.
3. **Freezing.** A declaration after a parse panics here. `flags.md` let later declarations affect
   later parses; no use for that is known.
