# Proposal: a lightweight `Flag` namespace

2026-11-11 · draft

## Summary

Add Go's `flag` package to the standard library, nearly as it is: a script declares its flags as an
array of `Flag` records, `Flag.parse` reads `Os.args()` by Go's rules, and typed getters return each
flag's value, or its default when the command line did not give it. The type of a flag is the type
of its default, as `flag.Int("port", 8080, "...")` makes an `int` flag in Go. `Flag.defaults` prints
the usage lines Go's `PrintDefaults` prints. There is nothing else: no subcommand tree, no
completion, no reflection, no new syntax.

```ant
#!/usr/bin/env ant
//! Serves a directory over HTTP.

let flags = [
  Flag{ name: "port", alias: "p", value: 8080, usage: "listen on `port`" },
  Flag{ name: "root", value: ".", usage: "serve files from `dir`" },
  Flag{ name: "verbose", alias: "v", usage: "log every request" },
]

var set = try Flag.parse(flags)
if set.help {
  print(Flag.defaults(flags))
  return
}

var port = set.int("port")          // 8080 unless -port or -p gave one
var root = set.string("root")
var verbose = set.bool("verbose")
```

```
$ ./serve.ant -h
  -p, -port port
    	listen on port (default 8080)
  -root dir
    	serve files from dir (default ".")
  -v, -verbose
    	log every request
$ ./serve.ant -port=x
invalid value "x" for flag -port: parse error
```

## Motivation

Every script that takes an option today walks `Os.args()` by hand: a `while` over an index, a
`String.startsWith(arg, "-")`, a `String.cut` on `=`, a `String.toInt` and its `null`, an error
message for each case, and a usage string kept in step with all of it by hand. The loop is twenty
lines before the script does anything, it differs from script to script, and each one gets a corner
wrong: `-o` as the last argument, `--` before a file named `-x`, a value that starts with `-`.

The common case is small. A script has a handful of flags, each a switch or a single value with a
default, and some positional arguments after them. It wants a usage line on `-h` and a clear error
on a typo. Go's `flag` package covers exactly that in a few hundred lines, and a generation of Go
tools shows that its rules are enough for real programs. Ant's standard library already mirrors Go
where it can (`File`, `Os`, `Cmd`), so a script author who knows `flag` knows this, and the spec can
point at Go's behavior instead of inventing its own.

## Prior art

- **Go `flag`.** Typed flags whose type comes from the constructor, `-name` and `--name` alike,
  `-name=value` or `-name value`, booleans only as `-name` or `-name=false`, parsing stops at the
  first argument that is not a flag, `-h` and `-help` reported as `ErrHelp`, `PrintDefaults` for
  usage. Values reach the program through pointers or `flag.Value`, which Ant does not have.
- **Node `util.parseArgs`.** A declarative options object in, a `{ values, positionals }` object
  out, with `type: "string" | "boolean"`, `short`, `multiple` and `default`. Close in spirit: data
  in, data out. Untyped beyond strings and booleans, and GNU-style.
- **Deno `std/cli/parseArgs`, minimist.** Parse anything into a loose map with no declaration;
  unknown flags pass silently, and a typo becomes a value nobody reads.
- **POSIX `getopt`, GNU `getopt_long`.** Short-option clusters (`-la`), interleaved options and
  operands, `--long` versus `-s`. The convention most Unix tools follow, and the one with the most
  rules to state.
- **Python `argparse`, Rust `clap`, Swift ArgumentParser.** Full frameworks: subcommands, required
  arguments, choices, generated help, completion. Far past what a script needs, and a large surface
  to specify.

A typed, declared, Go-shaped parser is the smallest design that still catches typos, converts
numbers, and prints usage.

## The design

### Declarations

```
type Flag = { name: string, value: bool | int | float64 | string = false, usage: string = "", alias: string = "" }
type Flag.Set = { flags: Array[Flag], values: Map[string, Array[bool | int | float64 | string]],
                  args: Array[string], help: bool }
```

`Flag` is a record type and, like `Cmd`, a namespace whose functions are declared on it, so `Flag`
names the record in a literal and an annotation and the functions in a call. `Flag.Set` is a type in
its namespace, as `Panic.Frame` is in `Panic`'s, and a set reaches the getters through `.`:
`set.int("port")` is `Flag.int(set, "port")`.

- **`name`** is the flag as the command line writes it, without dashes: `name: "port"` is `-port`
  and `--port`.
- **`value`** is the default, and its type is the flag's type: `bool`, `int`, `float64` or `string`.
  A flag that leaves it out is a `bool` that defaults to `false`, the common switch. The literal
  checks against the union, so `8080` is an `int` and `8080.0` a `float64`.
- **`usage`** is one line of help. A word in backquotes names the flag's argument in
  `Flag.defaults`, as in Go: ``"listen on `port`"`` prints `-port port` and `listen on port`.
- **`alias`** is a second name for the flag, usually one letter: `alias: "p"` makes `-p` the same
  flag as `-port`. Go has no alias and programs declare the flag twice over one variable; a field is
  the smaller form of that idiom where there are no pointers. Its values are recorded under `name`.

A name or an alias is not empty, does not start with `-`, and contains no `=`, and no two flags of
one array share a name or an alias. Those are facts about the program, not the input, so
`Flag.parse` and `Flag.defaults` panic on a declaration that breaks them, before reading any
argument, as `Csv.parse` panics on a bad separator and Go panics on `flag redefined`.

### Functions

```
Flag.parse(flags: Array[Flag], args: Array[string] = Os.args()): Result[Flag.Set, Error]   // Go's rules, below; panics on a bad declaration
Flag.defaults(flags: Array[Flag]): string                  // Go's PrintDefaults text, in declaration order; panics on a bad declaration

Flag.bool(set: Flag.Set, name: string): bool                // the last value given, or the default
Flag.int(set: Flag.Set, name: string): int                  // likewise
Flag.float(set: Flag.Set, name: string): float64            // likewise
Flag.string(set: Flag.Set, name: string): string            // likewise
Flag.strings(set: Flag.Set, name: string): Array[string]    // every value given, in order; [] when none, the default not included
Flag.isSet(set: Flag.Set, name: string): bool               // given at least once, under its name or its alias
```

Each getter takes the flag's `name`, not its alias, and panics when no flag of the set has that name
or when the flag has another type, with a message that names both, such as
`Flag.int: flag -root is a string`. A misspelled name in a getter is a bug in the script, and a
panic on its first run says so; returning a default would hide it.

`Flag.strings` is the one thing Go's `flag` does not have built in: a flag given several times, such
as `-I dir -I dir2`, keeps every value, and the scalar getters read the last, which is Go's rule for
a repeated flag. A script that wants a list declares a string flag and reads it with `strings`;
there is no separate list type to declare.

`Set.values` and `Set.flags` are fields because a record's fields are visible, and a test or a
generic tool may read them; a script reads values through the getters, which apply the default and
check the type.

### Command-line rules

These are Go's, from `flag.FlagSet.Parse`:

- An argument that starts with `-` or `--` followed by at least one other character is a flag;
  `-port` and `--port` are the same. Three dashes, or a dash followed by `=`, is `Err`
  `bad flag syntax: ---port`.
- `-name=value` gives the value after the first `=`. A flag that is not a `bool` and has no `=`
  takes the next argument as its value, whatever it is, so `-o -x` sets `o` to `-x`; with no next
  argument it is `Err` `flag needs an argument: -o`.
- A `bool` flag is `true` as `-name`, and takes a value only after `=`: `1`, `t`, `T`, `TRUE`,
  `true`, `True` and `0`, `f`, `F`, `FALSE`, `false`, `False`, as Go's `strconv.ParseBool`. So
  `-v file` is the switch and then a positional argument.
- An `int` value is read by `String.toInt` and a `float64` by `String.toFloat`; one they reject is
  `Err` `invalid value "x" for flag -port: parse error`. Go reads an `int` with base prefixes, where
  `010` is eight; a script's `-count 010` meaning ten is the safer reading, and `0x` is rare enough
  on a command line to leave out. A `string` takes any value, `""` included.
- An unknown name is `Err` `flag provided but not defined: -prot`.
- Parsing stops at the first argument that is not a flag, at `-` alone, or after `--`, which is
  dropped. That argument and every one after it are `args`, unread. So `tool -v build -o out` parses
  `-v` and leaves `["build", "-o", "out"]`, which is how a script with subcommands hands the rest to
  a second `Flag.parse`, and a file named `-x` is reached as `tool -- -x`.
- `-h`, `-help` and `--help`, when no flag declares that name, stop parsing with `help` set to
  `true` and the arguments after it in `args`. Go returns `ErrHelp` here; a field is the same signal
  without a second kind of error, and it lets `try` keep its one meaning. A script that never reads
  `help` ignores the request, which is visible on its first `-h`.

An error message names the argument as the user wrote it, with its dashes, so `--prot` reports
`--prot`. The messages are Go's, which already follow Ant's convention of lower case without closing
punctuation (`ant-lang.md` section 8, Wrapping errors). `try Flag.parse(flags)` ends a script with
the message and status 1. A script that wants the conventional status 2 and the usage after the
error writes it out:

```ant
var parsed = Flag.parse(flags)
if parsed is Err {
  eprintln(parsed.error.message)
  eprint(Flag.defaults(flags))
  Os.exit(2)
}
var set = parsed.value
```

### Usage text

`Flag.defaults` writes Go's `PrintDefaults` format, one entry per flag, each line ending in `\n`:
two spaces, the alias and the name each with one dash and joined by `, `, the argument's name, and
the usage on the next line after four spaces and a tab, with `(default …)` after it unless the
default is `false`, `0`, `0.0` or `""`. A string default is quoted. The argument's name is the word
in backquotes, with the backquotes removed from the usage, or else the type: `int`, `float`,
`string`; a `bool` has none. When the whole first part is one dash and one letter, the usage follows
on the same line after a tab, as Go does for `-v`.

Go sorts the entries by name, because its flags live in a map. Here they are an array, and the
author's order, most important first, is the better one, so `defaults` keeps it.

The text holds only the flags. A script prints its own first line, with `Path.base(Os.program())`
and its positional arguments, since only it knows them:

```ant
if set.help {
  println("usage: ${Path.base(Os.program())} [flags] <file>...")
  print(Flag.defaults(flags))
  return
}
```

### Subcommands

Since parsing stops at the first positional argument, a subcommand is two calls and no new API:

```ant
let global = [Flag{ name: "verbose", alias: "v", usage: "say more" }]
let buildFlags = [Flag{ name: "out", alias: "o", value: "a.out", usage: "write to `file`" }]

var top = try Flag.parse(global)
var command = Array.first(top.args) ?? "help"
var rest = Array.slice(top.args, Math.min(1, len(top.args)), len(top.args))
if command == "build" {
  var set = try Flag.parse(buildFlags, rest)
  if top.bool("verbose") { eprintln("building ${set.string("out")}") }
  // set.args holds the files to build
}
```

## Spec changes

- `stdlib.md`: a `Flag` section after `Os`, with the declarations, the functions, the command-line
  rules and the usage format above. The intro's list of namespaces gains `Flag`, and its sentence on
  panics names `Flag` beside `Csv` for a bad declaration and a getter's wrong name or type.
- `ant-lang.md` section 11: `Flag` joins the list of prelude names that are types with functions
  declared on them (`Result`, `Error`, `Cmd`, ...), and the list of I/O namespaces that live in
  `stdlib.md`.
- `prelude.d.ant`: the two types and eight functions with doc comments; `ant doc Flag` shows them.
- Section 10, Scripts: the example that reads `Os.args()` may stay; a second line points at `Flag`
  for scripts that take options.

The implementation is a native namespace over the same rules, about the size of `Csv`, and the docs
get a `lib/Flag` page whose examples run under `ant test`.

## Non-goals

- **GNU syntax.** No clusters (`-la`), no interleaving of flags after positional arguments, no
  distinction between `-x` and `--long`. Go's rules are fewer to state, a script's users meet them
  in every Go tool, and a program that must look like `ls` can still walk `Os.args()` itself.
- **Required flags, choices, ranges.** A flag has a default; a script that needs one given checks
  `isSet` and says so, and a range is an `if`. Each would be a field with rules for one case.
- **Environment variables, config files.** `Os.env(name) ?? default` as the `value` of a flag covers
  the common fallback, since the array is built at run time.
- **Typed records through `Reflect`.** Declaring a record type and filling it from the command line
  is library code over reflection (`typed-data.md`) and a larger design (`flags.md`); this proposal
  is the floor under both and keeps working without them.
- **Shell completion and generated man pages.**

## Alternatives considered

- **Untyped values.** Return `Map[string, Array[string]]` and let the script convert, as Node does.
  Smaller still, but every script then writes the `String.toInt` and its error message, which is
  most of the hand-written loop this replaces.
- **A builder, `Flag.int(fs, "port", 8080, "...")`, as Go writes it.** Go's constructors return
  pointers that `Parse` fills later; Ant has no pointers, and a constructor that returned a getter
  closure would make each flag a function to call. Declaring the flags as data, and reading them
  after the parse, gives the same one line per flag without that.
- **One field per type, `int: int | null`, `string: string | null`.** Says the type twice, and
  allows a flag with two defaults. The union on `value` makes the default the type, as Go's
  constructor name does.
- **Panicking on a parse error.** A typo on the command line is input, not a bug, so it is `Err`.

## Open questions

1. **Durations.** Go has `flag.Duration`, read as `1h30m`. `Time.parseDuration` reads ISO 8601
   (`PT1H30M`), which nobody types on a command line. Add `Time.Duration` to the union with Go's
   syntax for flags only, or leave it out and let a script read a string?
2. **The help names.** Go answers `-h` and `-help`; this adds `--help`, which Go also accepts
   because it treats `--` and `-` alike. Should a declared flag named `h` keep `-help` for help? Go
   says no: it answers only names not defined.
3. **Status 2 by default.** A `Flag.parseOrExit` that prints the error and the usage and exits 2, as
   Go's `ExitOnError`, would save the four-line `if` above. It is one more function, and the `try`
   form is already short; leave it until scripts show they want it.
4. **Relation to `flags.md`.** If both are accepted, `Flags` should be built on `Flag.parse`'s rules
   so the two read a command line the same way.
