# Proposal: commands as values

2026-10-28 · Anton · Draft

## Summary

Split running a program into two steps, as Julia does: a `Cmd` value says *what* to run, and a verb
says *how*. This replaces `Os.run`, `Os.output`, `Os.exec` and `Os.start`, four functions whose
names don't tell them apart. `Cmd.pipe` joins commands into a pipeline that every verb accepts.

```
var head = try Cmd.read(Cmd.new("git", ["rev-parse", "--short", "HEAD"]))
```

The work is in the API only. The semantics of [process.md](process.md) stay as they are: the
foreground and Ctrl-C relay, process groups, `128 + n`, pipes and contexts.

## Motivation

`run`, `exec`, `output` and `start` are all reasonable verbs for starting a program, so the only way
to tell which captures output, which gives the child the terminal and which returns at once is to
read the docs. In a script, `Os.exec(...)` versus `Os.run(...)` doesn't show which one drops the
child's output into a record.

All four also repeat the same four parameters (`program`, `args`, `dir`, `env`). As a result, a
command can't be built in one place and run in another, logged before it runs, or passed to a
helper.

Joining two children takes an `Os.openPipe`, two `Os.start` calls, closing the right ends, and two
waits. A shell does the same with `a | b`.

## Prior art

- **Julia:** `` `git log` `` makes a `Cmd`. Then `run(cmd)`, `read(cmd, String)`, `readchomp`,
  `success`, `open`, and `pipeline(a, b)` to join commands.
- **Go:** `exec.Command` builds an `*exec.Cmd`, and `Run`, `Output`, `CombinedOutput` or `Start`
  runs it. Its verbs are as vague as ours, but the command is still a value. It has no pipelines.
- **Rust:** `Command::new("git").arg("log")`, then `.status()`, `.output()` or `.spawn()`.
- **bash:** `set -o pipefail` and `PIPESTATUS`.

## API

```
type Cmd = {
  program: string,
  args: Array[string] = [],
  dir: string | null = null,
  env: Map[string, string] | null = null,
}

func Cmd.new(program: string, args: Array[string] = [], dir: string | null = null,
        env: Map[string, string] | null = null): Cmd
func Cmd.string(c: Cmd | Cmd.Pipeline): string
```

`Cmd.new` is shorthand for the literal `Cmd{ program: "git", args: [...] }`, which has to spell out
`program:`.

`Cmd.string` renders a command as a line for a log: the program and its arguments, quoted as POSIX
`sh` would need them (`git commit -m 'a b'`), and the stages of a pipeline joined by ` | `. It leaves
out `env` and `dir`. A `Cmd` printed as a record shows `env` too, and with it any secrets the
environment holds.

The verbs take `c: Cmd | Cmd.Pipeline`. `wait`, `kill`, `stop` and `pid` take the `Cmd.Process` that
`start` returns.

| Verb | Streams | Result |
| --- | --- | --- |
| `Cmd.run(c, ctx:)` | the terminal, with the Ctrl-C relay | `Result[int, Error]`: any status |
| `Cmd.read(c, input:, ctx:)` | stdout captured, stderr in the Err | `Result[string, Error]`: nonzero is Err, one final newline dropped |
| `Cmd.readBytes(c, input:, ctx:)` | as `Cmd.read` | `Result[Bytes, Error]`: nothing trimmed |
| `Cmd.capture(c, input:, ctx:)` | all captured | `Result[Cmd.Output, Error]`: nonzero is Ok |
| `Cmd.start(c, stdin:, stdout:, stderr:, ctx:)` | any handle | `Result[Cmd.Process, Error]` |
| `Cmd.wait(p, ctx:)`, `Cmd.kill(p, signal)`, `Cmd.stop(p, grace:)`, `Cmd.pid(p)` | — | as today's `Os.*` |

`input` is `string | Bytes = ""`, as `stdin` is on `Os.exec` today. It has a different name from
`start`'s `stdin`, which is a handle. `Cmd.Output` is today's `Os.Output` plus `statuses` (see
Pipelines). `Cmd.Process` is today's `Os.Process`.

```
var build = Cmd.new("go", ["build"], dir: root)
if verbose { Array.push(build.args, "-v") }
Array.push(build.args, "./...")
println(`running ${Cmd.string(build)}`)
var status = try Cmd.run(build)

var server = try Cmd.start(Cmd.new("./server"), stdout: log)
defer { Cmd.stop(server) }
```

## Migration

`c` stands for `Cmd.new(program, args, dir:, env:)`.

| Now | After |
| --- | --- |
| `Os.run(program, args, dir:, env:, ctx:)` | `Cmd.run(c, ctx:)` |
| `Os.output(program, args, dir:, env:, stdin:, ctx:)` | `Cmd.read(c, input:, ctx:)` |
| `Os.exec(program, args, dir:, env:, stdin:, ctx:)` | `Cmd.capture(c, input:, ctx:)` |
| `Os.start(program, args, dir:, env:, stdin:, stdout:, stderr:, ctx:)` | `Cmd.start(c, stdin:, stdout:, stderr:, ctx:)` |
| `Os.wait`, `Os.kill`, `Os.stop`, `Os.pid` | `Cmd.wait`, `Cmd.kill`, `Cmd.stop`, `Cmd.pid` |
| `Os.Process`, `Os.Output` | `Cmd.Process`, `Cmd.Output` (which gains `statuses`) |
| `Os.openPipe`, `Os.Pipe`, `Os.which` | unchanged |

## Pipelines

```
type Cmd.Pipeline = { cmds: Array[Cmd] }
func Cmd.pipe(cmds: Array[Cmd]): Cmd.Pipeline   // panics when cmds is empty

var fixes = try Cmd.read(Cmd.pipe([
  Cmd.new("git", ["log", "--oneline"]),
  Cmd.new("grep", ["fix"]),
  Cmd.new("wc", ["-l"]),
]))
```

- **Wiring.** The first stage reads the pipeline's standard input: `input` for `read`, `readBytes`
  and `capture`, the `stdin` handle for `start`, and the terminal for `run`. The last stage writes
  the pipeline's standard output. Under `run`, only the first stage reads the terminal; every stage
  can still write to it. Stages are joined by OS pipes from one descriptor to the next, with no copy
  through the runtime. Each stage keeps its own `dir` and `env`.
- **Standard error.** Every stage's stderr goes where the verb sends stderr: to the terminal for
  `run`, to the `stderr` handle for `start`, and into one capture for `read`, `readBytes` and
  `capture`. Stages write it at the same time, so in a capture their lines interleave in the order
  they arrived, and no order between stages is promised.
- **Status.** The pipeline's status is that of the rightmost stage that failed, or 0 when every
  stage succeeded, as under bash's `pipefail`, so `false | cat` fails. One deliberate departure: a
  stage that died of SIGPIPE after a later stage ended doesn't count as failed, so `yes | head -n 1`
  succeeds, where bash gives 141. The excuse covers death by the signal only. A stage that ignores
  SIGPIPE and exits 1 on the write error, as Python does, counts as failed. Windows has no SIGPIPE,
  so the excuse never applies there.
- **Statuses.** `Cmd.Output` gains `statuses: Array[int]`, each stage's own status as bash's
  `PIPESTATUS` gives it, an excused SIGPIPE stage included as 141. A single command has one entry.
  Without it, there would be no way to tell which stage failed after `capture`.
- **Errors.** `Cmd.read`'s Err is today's message for the stage that set the status:
  `<program>: exit status <n>: <stderr>`, where `<stderr>` is the whole capture. When two stages run
  the same program, the stage's place is added: `grep (stage 3): exit status 1`.
- **One job.** `Cmd.start` puts every stage in one process group and returns one `Cmd.Process`.
  `wait` gives the pipeline's status, `kill` and `stop` reach every stage with one group signal, and
  `pid` is the last stage's, as `$!` is.
- **Ctrl-C.** Under `run`, Ctrl-C reaches every stage. `ant` passes the held interrupt on when the
  *last* stage died of SIGINT, not when any stage did. In `git log | less`, Ctrl-C kills `git` while
  `less` handles it and goes on, and the script must go on with it. The last stage is the one whose
  output the user sees, so its choice is the one that counts.
- **Failure.** When a stage can't be started, the stages already started are killed and the call is
  Err. When `ctx` is done, every stage is killed. Every verb panics on an empty pipeline, as
  `Cmd.pipe` does, since `Cmd.Pipeline{ cmds: [] }` can be written by hand.

## Decisions

- **Streams belong to the verb, not to `Cmd`.** Each verb wants different defaults:
  - `run` gives the child the terminal;
  - `read` and `capture` take its output;
  - `start` gives it the null device as stdin and the program's own stdout and stderr.

  A field can't default to "whatever the verb wants", because `null` already means the null device.
- **A top-level `Cmd`, not `Os.Cmd`.** Both work, since a namespace can hold types with functions
  (`ant-lang.md` §6: `Geometry.Point.norm` is a field read). `Cmd` is chosen because it is shorter:
  `Cmd.read(Cmd.new(...))` against `Os.Cmd.read(Os.Cmd.new(...))`. `Html` and `Panic` are the
  precedent for a prelude type with types of its own (`Html.Node`, `Panic.Frame`), which is what
  `Cmd.Pipeline`, `Cmd.Output` and `Cmd.Process` are. The cost: `Cmd` becomes a prelude name, no
  declaration may shadow one (`ant-lang.md` §3), and so a program that declares its own `Cmd` stops
  compiling and has to rename it.
- **`Cmd.read` trims, unlike `File.read`.** `File.read` returns a file exactly. `Cmd.read` drops one
  final newline, as `Os.output` does and as `$(...)` does in a shell, because a script almost always
  wants `git rev-parse HEAD` without it. That is the job of Julia's `readchomp`, under the shorter
  name, because it is the common case. `Cmd.readBytes` is there for exact output.
- **`Cmd.pipe` takes an array and returns a named type.** The array is there because Ant has no
  variadic functions, so a pair `Cmd.pipe(a, b)` would nest at three stages. The named type is there
  because a bare `Array[Cmd]` passed to a verb doesn't say it means a pipeline. A `pipe` field on
  `Cmd` was rejected: a stage list beside its own `program` contradicts itself. The union
  `Cmd | Cmd.Pipeline` is cheap, since both are named records with their own tags.
- **`capture`, not `collect`.** `collect` reads as an iterator's verb.
- **`Os.openPipe` stays, under that name.** `Cmd.pipe` covers joining commands, but some jobs still
  need a pipe made by hand:

  | Job | Without `Os.openPipe` |
  | --- | --- |
  | Join commands | `Cmd.pipe` covers it |
  | Read a child's output as it comes (logs, progress, `tail -f`) | needs a pipe |
  | Write to a child's stdin a piece at a time (driving `sqlite3`, a REPL) | needs a pipe |
  | Watch stdout and stderr live and separately | needs two pipes |
  | Merge stderr into stdout (`2>&1`) | needs one pipe given as both `stdout` and `stderr` |
  | Stream bytes between goroutines | `Channel` covers it |

  Go, Python and Rust set a stream to "piped" on `start` instead. In Ant that needs a marker type
  next to `File.Handle | null`, accessors on `Cmd.Process` that are null most of the time, and it
  carries the deadlock where nobody reads a piped stream and its buffer fills. The explicit form is
  short and covers every row:

  ```
  var out = try Os.openPipe()
  var p = try Cmd.start(Cmd.new("make"), stdout: out.write)
  var line = try File.readLine(out.read)   // each line as make writes it, null at the end
  ```

  The function keeps its name. `pipe` is already a verb meaning "make this flow into that", in
  `File.pipe` (which copies a handle) and `Cmd.pipe`, so `Os.pipe()` would read as one more join.
  The `open` prefix matches `File.open` and `File.create`, whose results the program has to close.
  The function stays in `Os` because it needs no command, and `File.openPipe` would sit next to
  `File.pipe`. A `Cmd.open`-style shortcut can follow if scripts show the three lines above are
  common.
- **The old calls go.** `Os.run`, `output`, `start`, `wait`, `kill`, `stop` and `pid` are a day old.
  `Os.exec` predates them, but leaving it in would keep two ways to do one thing. `tests/run` and the
  examples move to the new calls.

## Spec changes

- `stdlib.md`:
  - A new `Cmd` chapter takes the Processes, Interrupts and Streams sections from `Os` and adds
    Pipelines.
  - The `Os` listing keeps `openPipe` and `which` and loses the other process calls.
  - `Os.which`'s comment names `Cmd`'s verbs as the calls that start what it finds.
  - The intro says the file holds the `Cmd` type with its functions, as well as namespaces.
- `ant-lang.md`:
  - §11's list of I/O namespaces gains `Cmd`, with a line saying it is a type as well.
  - `Os.Process` becomes `Cmd.Process` in all three places: the Kinds table, the Unions paragraph
    that lists types without a discriminator, and the rendering rules.
- `process.md`: point to this proposal from its Decisions.

## Non-goals

- A command literal such as Julia's backticks. `` `...` `` is already a template string.
- A pipe operator such as Julia's `|>`, which needs new syntax.
- `&&` and `||`, which `if` covers.
- Tee and fan-out, which need `Os.openPipe` plus a goroutine.

## Follow-ups

- `Cmd.succeeds` returning `bool`.
- A helper such as `Cmd.withEnv(c, vars)` that merges `vars` onto `Os.environ()`. `env` is the
  child's whole environment, so adding one variable today takes building the merged map by hand.
