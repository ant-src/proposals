# Flags

**Draft · 10 November 2026**

Add a small typed flags library to Ant. Use one set of declarations for parsing, help, and shell
completion.

## Example

```ant
let cli = Flags.new("tool", description: "Process files.")

let jobs = cli.option(
  "jobs", Flags.integer, default: 4,
  short: "j", env: "JOBS", help: "Number of workers.",
)
let force = cli.flag("force", short: "f", help: "Overwrite existing files.")
let files = cli.arguments("FILE", Flags.text, min: 1)

let args = cli.parseOrExit()

let workers = args.get(jobs)       // int
let overwrite = args.get(force)    // bool
let paths = args.get(files)        // Array[string]
```

A declaration returns a `Flags.Key[T]`; `get` returns its `T`. Keys hold no mutable “current value.”
No new language syntax or reflection is required.

## Declarations

For the generic constructors, `convert` is an ordinary `func(string): Result[T, Error]`.

| Call on `cli`                                 | Value from `args.get(key)` | When omitted                  |
| --------------------------------------------- | -------------------------- | ----------------------------- |
| `option(name, convert, default: value)`       | `T`                        | Explicit default              |
| `optional(name, convert)`                     | `T \| null`                | `null`                        |
| `required(name, convert)`                     | `T`                        | Error                         |
| `repeated(name, convert)`                     | `Array[T]`                 | Empty array                   |
| `flag(name, default: false, negatable: true)` | `bool`                     | Boolean default               |
| `count(name)`                                 | `int`                      | Zero                          |
| `argument(name, convert)`                     | `T`                        | Error                         |
| `optionalArgument(name, convert)`             | `T \| null`                | `null`                        |
| `arguments(name, convert, min: 0)`            | `Array[T]`                 | Empty array, unless `min > 0` |

Named options accept `short` and `help`. Value-taking options also accept `valueName` (default
`"VALUE"`). Scalar options and flags accept `env`. Other optional metadata defaults to absent or
empty.

A short name is one ASCII letter. Long names are nonempty, exact, case-sensitive strings without a
leading hyphen, whitespace, controls, or `=`. Prefer kebab-case.

Required positional arguments come first. At most one optional argument or argument list may follow,
and it must be last. Extra operands are errors.

Built-in converters are `Flags.text`, `Flags.integer`, `Flags.float`, and `Flags.boolean`. They
preserve text, parse signed base-10 integers, parse finite decimal floats, and accept exactly
`true`/`false`, respectively. Numeric converters reject surrounding whitespace and out-of-range
values.

Custom conversion and validation use ordinary functions:

```ant
func positive(s: string): Result[int, Error] {
  let n = try Flags.integer(s)
  if n < 1 {
    return Err{ error: Error{ message: "expected a positive integer" } }
  }
  Ok{ value: n }
}
```

The parser adds the option name to conversion errors. Typed defaults bypass conversion; the
programmer supplies valid defaults. Validate relationships between options in ordinary Ant code
after parsing.

## Parsing and results

```ant
Flags.new(
  name: string, description: string = "",
  version: string | null = null, help: bool = true,
): Flags

Flags.parse(
  flags: Flags, args: Array[string],
  env: Map[string, string] = Map{}, prefix: bool = false,
): Result[Flags.Outcome, Error]

Flags.parseOrExit(
  flags: Flags, args: Array[string] = Os.args(),
  env: Map[string, string] = Os.environ(), prefix: bool = false,
): Flags.Values
```

`Flags.Outcome` is `Flags.Parsed{ values }`, `Flags.Help{ message }`, or `Flags.Version{ message }`.
The explicit parser returns these outcomes without printing, exiting, or reading process state. Its
argument array excludes the program path, as `Os.args()` does.

`parseOrExit` returns values, prints help/version to stdout and exits `0`, or prints an input error
to stderr and exits `2`. It adds a help hint when help is enabled. Use `parse` when the caller must
control output or cleanup: `Os.exit` skips deferred blocks.

Results provide:

- `get(key)` — the typed value, without rerunning its converter.
- `provided(key)` — whether command-line input supplied it, including explicit `false`, zero, or
  empty text.
- `source(key)` — `CommandLine`, `Environment{ name }`, `Default`, or `Absent`, all under `Flags`.
- `rest()` — the untouched suffix in prefix mode; an empty array in full mode.

Each parse produces independent bindings and fresh parser-owned arrays. Custom values and mutable
defaults retain normal Ant reference semantics. Failed parsing exposes no partial result; effects
performed by user converters are not rolled back.

Invalid definitions and access with a foreign or later-registered key panic. Invalid user input
returns `Err`. Parsing does not freeze the builder; later declarations affect later parses.

### Defaults and environment

Precedence is **CLI → explicit environment variable → default → absence/error**.

Only an option's declared `env` name is used. Empty environment values are present. Invalid selected
input errors; it never falls through to a default. An environment fallback can satisfy `required`.
Repeated options, counters, and positionals have no environment fallback in v1.

An unused environment value is not converted:

```text
JOBS=bad tool --jobs=4       succeeds
tool --jobs=bad --jobs=4    errors
```

Missing optional values, empty repeated lists, and absent counters have source `Absent`; declared
defaults have source `Default`.

## Command-line rules

| Case                                          | Rule                                                                      |
| --------------------------------------------- | ------------------------------------------------------------------------- |
| `--jobs 4`, `--jobs=4`, `-j 4`, `-j4`, `-j=4` | Accepted                                                                  |
| `--offset -5`                                 | A value-taking option consumes the next token, even if it begins with `-` |
| `--output --help`                             | `"--help"` is the output value                                            |
| `--output --`                                 | Missing-value error; use `--output=--` for the literal value              |
| `--name=`                                     | Present empty string                                                      |
| `--`                                          | End option recognition; later tokens are operands                         |
| `-`                                           | Ordinary data                                                             |
| `-- -5`                                       | A negative positional number                                              |
| `--jobs=2 --jobs=4`                           | Last scalar occurrence wins                                               |
| `--include a --include b`                     | Collect in order; no automatic comma splitting                            |
| `-vvv`                                        | Three occurrences of a counter with short name `v`                        |
| Unknown or abbreviated option                 | Error                                                                     |

Full parsing accepts options between operands. Each named option consumes zero or one value;
optional or greedy multi-value option arguments are excluded.

Short clusters scan left to right. A value-taking option consumes the remaining suffix, stripping
one leading `=`, or the next token when no suffix is present. A boolean followed by `=` consumes the
suffix as its value and ends the cluster.

A negatable flag `cache` accepts `--cache`, `--no-cache`, and `--cache=false`. Boolean values must
be attached with `=`; `--cache false` leaves `false` as an operand. Only the positive spelling
accepts a value. Repeated booleans use the last value. Positive names of negatable flags must not
start with `no-`. Negation names and built-in names participate in collision checks.

On a successful value parse, every explicit occurrence is converted once, including overridden
occurrences. Stop at the first conversion error.

### Help

Generate `-h`/`--help` by default and `--version` when a version is supplied. These controls take no
values. Reserve no short version alias.

Scan token ownership first. Lexical errors still fail; genuine help/version requests then bypass
conversion, environment resolution, requiredness, and operand counts. Help wins when both are
requested. Thus `--jobs=bad --help` shows help, while `--jobs --help` supplies a value to `jobs`.

`cli.help()` and `cli.usage()` return plain text. Help uses declaration order, descriptions, names,
value labels, defaults, and environment names. It does not display current environment values or
evaluate defaults again.

### Wrappers and subcommands

`prefix: true` stops before the first operand. For `tool --verbose child --verbose`, the parent
parses the first flag and returns `["child", "--verbose"]` from `rest()`.

Every token after that boundary stays untouched, including a child's `--`. A delimiter reached
before the boundary is consumed. Prefix mode requires no positional declarations; violating this is
programmer misuse.

The application selects a child parser and passes it the remaining arguments. No command framework
is required.

## Integrated shell completion

**Goal:** both forms work with Ant's existing shell setup:

```sh
ant script.ant --f<TAB>        # --force
ant run script.ant --f<TAB>    # --force
```

### Existing implementation

At [Ant commit 41e69e2][ant-completion], `ant --comp bash|zsh|fish` prints shell adapters that ask
Ant for candidates once installed. `main` handles those requests before normal execution. Both
script invocation forms currently fall back to filesystem completion after the script path.

The backend already represents candidates as `word`, `file`, or `dir`. Flag names fit its existing
replies. This is shell completion; the REPL's expression completer is separate.
[Source][ant-completion], [routing][ant-main], [tests][ant-tests].

### Integration

Add an internal `completeScript(path, precedingArgs, currentWord)` route for both invocation forms.
Preserve the existing shell installation and reply mechanism; no per-script registration is needed.

1. Read the script's AST without running it.
2. Identify the directly declared parser used by an unambiguous top-level `parseOrExit` or `parse`
   call.
3. Extract its direct, straight-line registrations preceding that call: literal names, short names,
   negation settings, positional shapes, and help/version settings.
4. Use the Flags scanner's token-ownership rules, allowing unfinished input, to suggest names where
   an option may begin.

Use only calls that parse the unmodified script arguments (omitted argv or direct `Os.args()`) with
a statically known `prefix` mode; otherwise fall back. Recognize namespace and dot calls as
equivalent. Do not merge unrelated parser instances. No script, imported initializer, default
expression, converter, or application handler executes during completion.

For the first version, complete long names, short names, generated negations, and enabled
help/version names. Keep filesystem completion for values and operands. Respect `--` and prefix
boundaries:

| Cursor position               | Behavior                                  |
| ----------------------------- | ----------------------------------------- |
| `ant script.ant --f`          | Suggest `--force`                         |
| `ant script.ant --no-c`       | Suggest `--no-cache` if declared          |
| `ant script.ant --output --f` | Complete a value; do not suggest flags    |
| `ant script.ant -- --f`       | Complete an operand; do not suggest flags |

**Static discovery is partial.** Flags built through helpers, loops, imports, or computed names
remain valid at runtime but may not complete. Ambiguous parser selection or unknown earlier option
arity falls back to filesystem completion. Do not guess or run the script to fill the gap.

Completing choices, custom values, and child command schemas is follow-up work. Attached-value
completion such as `--format=j` also needs shell insertion tests: the current backend does not model
replacement ranges or spacing directives, and Bash treats `=` specially.

Verify both invocation forms in Bash, Zsh, and Fish, including quoted paths, unfinished words, value
ownership, and `--`. Include a script with an observable top-level side effect to confirm that Tab
never executes it.

## Rationale and scope

The design combines typed declarations from libraries such as [clap][clap], the explicit parse/exit
split of [Swift ArgumentParser][swift], and the small extensible core of [Node's parser][node].
These sources support the design principles; Ant's exact policies above are deliberate choices.

Typed keys fit Ant's current generics and avoid a second record-metadata schema. Native opaque
storage needs no generic runtime type reconstruction.

Ship parsing, help, scalar environment fallback, prefix composition, and static flag-name completion
together. Defer reflection shortcuts, command trees, automatic configuration discovery, prompts,
response files, constraint languages, and dynamic completion callbacks.

[ant-completion]:
  https://github.com/ant-src/ant/blob/41e69e2a58adcc932e087216f6b1b6e6f1c01513/cmd/ant/complete.go
[ant-main]:
  https://github.com/ant-src/ant/blob/41e69e2a58adcc932e087216f6b1b6e6f1c01513/cmd/ant/main.go
[ant-tests]:
  https://github.com/ant-src/ant/blob/41e69e2a58adcc932e087216f6b1b6e6f1c01513/cmd/ant/complete_test.go
[clap]: https://docs.rs/clap/latest/clap/_faq/index.html
[swift]:
  https://github.com/apple/swift-argument-parser/blob/main/Sources/ArgumentParser/Documentation.docc/Articles/ManualParsing.md
[node]: https://nodejs.org/api/util.html#utilparseargsconfig
