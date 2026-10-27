# Proposal: running programs

2026-10-27 · Anton

## Summary

Grow `Os.exec` into a small process API modelled on Go's `exec.Cmd`, spelled the Ant way: named
parameters instead of a struct to fill in, `File.Handle` for every stream, and `Result` for every
failure. Five additions and one change:

- `Os.run`: the child shares the program's terminal, and the call returns its exit status.
- `Os.output`: the child's standard output as text, and `Err` on a nonzero status.
- `Os.start`, `Os.wait`, `Os.kill`: a child that runs beside the program, with its streams wired to
  any handle.
- `Os.pipe`: a pair of connected handles, to read a child's output as it comes or to join two
  children.
- `Os.which`: where `PATH` finds a program.
- A child ended by a signal has the status `128 + n`, as a shell reports it, not `-1`.

`Os.exec` keeps its signature and meaning, except that `stdin` also takes a handle. No new syntax
and no shell. This closes two items of `ant/TODO.md`: "Running a command on the terminal" and
"Command output as text".

## Motivation

`Os.exec` is one shape of call: feed the child a fixed input, wait, capture both outputs whole. It
is the right shape for `go version` and wrong for most of what a script that drives other programs
does.

1. **A program that needs the terminal cannot be run.** `ssh`, `git commit`, `$EDITOR`, `less`,
   `make` with progress, `docker run -it`: each reads the keyboard, draws on the screen, or checks
   `isatty`. Under `Os.exec` the child reads an empty string and writes into a buffer the user sees
   only when it exits, if at all. A wrapper script that ends by starting such a program cannot be
   written in Ant.
2. **Output for a value is three steps every time.** `git rev-parse HEAD` is wanted as a string:
   check the `Err`, check `status`, read `stdout`, trim the newline. Every script grows a helper:

   ```ant
   func sh(program: string, args: Array[string]): Result[string, Error] {
     var out = try Os.exec(program, args)
     if out.status != 0 {
       return Result.err(Error{ message: `${program}: status ${out.status}: ${String.trim(out.stderr)}` })
     }
     Result.ok(String.trimSuffix(out.stdout, "\n"))
   }
   ```

3. **Nothing streams.** A build that runs ten minutes shows nothing until it ends. A log follower
   (`tail -f`, `kubectl logs -f`, `journalctl -f`) never ends, so `Os.exec` never returns. An output
   larger than memory cannot be processed.
4. **Nothing runs beside the program.** A test script that starts a server, runs requests against
   it, and stops it cannot be written: `Os.exec` blocks until the server exits.
5. **No redirection.** Writing a child's output straight to a file, feeding it a file, or piping one
   child into another (`git log | grep fix`) needs the whole stream in memory, or a shell, which
   brings back quoting.
6. **A killed child's status cannot be passed on.** `-1` is not an exit status a wrapper can exit
   with, and it hides which signal it was.

## Prior art

Every language surveyed has the same three layers, and differs in how many names it spends on them
and what the defaults are.

| Language             | Run, terminal shared                                 | Output as a value                                                 | Child beside the program                                         | Stream choices                                                            |
| -------------------- | ---------------------------------------------------- | ----------------------------------------------------------------- | ---------------------------------------------------------------- | ------------------------------------------------------------------------- |
| Go `os/exec`         | `cmd.Run()` with `Stdin/Stdout/Stderr = os.Std*`     | `cmd.Output()`; nonzero is `*ExitError` carrying captured stderr  | `Start`, `Wait`, `StdoutPipe`; `Process.Kill`, `Signal`          | any `io.Reader`/`io.Writer`; `*os.File` is passed directly; `nil` is null |
| Rust `std::process`  | `Command::status()`: **inherits by default**         | `Command::output()`: **pipes by default**; status is not checked  | `spawn()` → `Child`, `wait`, `kill`; `CommandExt::exec` replaces | `Stdio::inherit`, `piped`, `null`, or a `File`                            |
| Python `subprocess`  | `run(args)`                                          | `check_output`, `run(capture_output=True, text=True, check=True)` | `Popen`, `wait`, `terminate`, `kill`, `communicate`              | `None` (inherit), `PIPE`, `DEVNULL`, a file, `STDOUT` to merge            |
| Node `child_process` | `spawnSync(cmd, args, { stdio: "inherit" })`         | `execFileSync` returns stdout, **throws on nonzero** with stderr  | `spawn` → `ChildProcess`, events, `kill(signal = "SIGTERM")`     | `"pipe"` (default), `"inherit"`, `"ignore"`, a stream or fd, per stream   |
| Deno                 | `new Deno.Command(cmd, { args }).spawn()`            | `.output()`: piped by default, status not checked                 | `.spawn()`: **inherits by default**                              | `"piped"`, `"inherit"`, `"null"`                                          |
| Swift Subprocess     | `run(.name("vim"))`                                  | `run(.name("ls"), output: .string(limit: 4096))`: limit required  | `run` with a body that gets the execution and streams            | `.discarded`, `.string(limit:)`, `.bytes(limit:)`, a file descriptor      |
| Java                 | `ProcessBuilder.inheritIO().start().waitFor()`       | read `getInputStream()` by hand                                   | `start()` → `Process`, `destroy`, `onExit`                       | `Redirect.INHERIT`, `PIPE`, `DISCARD`, `to(file)`, `from(file)`           |
| Ruby                 | `system("git", "commit")` → true/false/nil           | `` `cmd` ``, `Open3.capture3`; status in `$?`                     | `spawn`, `Process.wait`, `Process.kill`                          | `out: :err`, `out: "file"`, `in: :close`                                  |
| Julia                | ``run(`vim $file`)``: throws on nonzero              | ``read(`git rev-parse HEAD`, String)``                            | `run(cmd; wait = false)`, `pipeline(a, b)`                       | `pipeline(cmd, stdout = "f")`, `devnull`; interpolation splices arguments |
| Scala `os-lib`       | `os.proc(...).call(stdout = os.Inherit)`             | `os.proc(...).call().out.text()`: **`check = true` by default**   | `spawn()` → `SubProcess`                                         | `os.Inherit`, `os.Pipe`, a path                                           |
| zx, Bun `$`          | `` $`git commit` ``, `.verbose()`                    | ``(await $`git rev-parse HEAD`).stdout``; **throws on nonzero**   | the unawaited promise, `.kill()`                                 | a shell; template interpolation is quoted                                 |
| Elixir               | `System.cmd(cmd, args, into: IO.stream())`           | `System.cmd/3` → `{output, status}`                               | `Port.open`                                                      | `stderr_to_stdout: true`                                                  |
| C `system(3)`        | `system("vim f")`; **ignores SIGINT while it waits** | `popen`                                                           | `fork`/`exec`/`waitpid`                                          | file descriptors                                                          |

What Ant takes:

- **Go's model of streams.** A stream is a handle, and a handle that is a real file is handed to the
  child as it is, so the terminal, a file and a socket are one case. Go needs `io.Writer` adapters
  and goroutines for everything else; Ant has `File.pipe` and `go` for that and needs nothing new.
- **Rust and Deno's defaults by call, not by flag.** Rust's `status()` inherits and `output()`
  captures; Deno did the same. The call names the intent, so no call takes a mode parameter that
  half its callers set. Node's `spawn` pipes by default, and "why does my child print nothing" is
  its most asked question.
- **Checked output.** Node's `execFileSync`, zx, Bun, Julia and Scala's `os-lib` fail on a nonzero
  status, and Python added `check=True` because unchecked calls hid failures. Go's `Output` reports
  it as an error that carries stderr. A call that returns only the text must check, or the text of a
  failed command flows on as data. `Os.exec` stays unchecked, since it returns the status.
- **The shell's status for signals.** Python reports `-n`, Go `-1`, Node `null` and a name. Only the
  shell's `128 + n` is a status a wrapper can exit with and the next shell will read the same way.
- **The shell's handling of Ctrl-C.** `system(3)` ignores SIGINT while its child runs; bash goes one
  step further and, when the child died of SIGINT, acts on it itself (Cracauer, "Proper handling of
  SIGINT/SIGQUIT", the "wait and cooperative exit" rule). Without that step a loop of `Os.run` calls
  cannot be stopped from the keyboard; without the first, `less` cannot be quit with Ctrl-C without
  killing the script.
- **No shell, no template.** zx, Bun and Julia make a command a quoted template. Ant has no
  backtick-command syntax and keeps arguments an `Array[string]`, as `Os.exec` already does.
- **No mandatory limit.** Swift Subprocess requires a size limit on every captured output. Ant's
  other reads of whole things, `File.read` and `Http` bodies, have none; a program that fears a
  large output streams it through `Os.pipe`.

## The API

```
type Output = { status: int, stdout: string, stderr: string, stdoutBytes: Bytes, stderrBytes: Bytes }   // unchanged
type Process                                    // opaque: a started child
type Pipe = { read: File.Handle, write: File.Handle }

Os.run(program: string, args: Array[string] = [], dir: string | null = null,
       env: Map[string, string] | null = null, ctx: Context | null = null): Result[int, Error]
                                                // the child shares the program's three streams; Ok of its status
Os.output(program: string, args: Array[string] = [], dir: string | null = null,
          env: Map[string, string] | null = null, stdin: string | Bytes | File.Handle = "",
          ctx: Context | null = null): Result[string, Error]
                                                // standard output without one final newline; Err on a nonzero status, with standard error in the message
Os.exec(program: string, args: Array[string] = [], dir: string | null = null,
        env: Map[string, string] | null = null, stdin: string | Bytes | File.Handle = "",
        ctx: Context | null = null): Result[Os.Output, Error]
                                                // unchanged, except that stdin may be a handle

Os.start(program: string, args: Array[string] = [], dir: string | null = null,
         env: Map[string, string] | null = null,
         stdin: File.Handle | null = Os.stdin(), stdout: File.Handle | null = Os.stdout(),
         stderr: File.Handle | null = Os.stderr(), ctx: Context | null = null): Result[Os.Process, Error]
                                                // returns once the child is running; null is the null device
Os.wait(p: Os.Process, ctx: Context | null = null): Result[int, Error]   // its status, once it has exited; every call gets the same status
Os.kill(p: Os.Process, signal: string = "SIGTERM"): Result[void, Error]  // Ok when it has exited already
Os.pid(p: Os.Process): int

Os.pipe(): Result[Os.Pipe, Error]               // what is written to write is read from read
Os.which(program: string): string | null        // the path PATH finds, as the calls above would; null when none
```

`program`, `args`, `dir` and `env` mean what they mean for `Os.exec` today in every call: no shell,
`PATH` lookup when `program` has no separator, `env` the whole environment.

### `Os.run`

Rules:

1. The child's standard input, output and error are the program's own. Before it starts, what the
   program has buffered for its standard output is flushed (see Buffering below).
2. The call blocks the calling goroutine until the child exits and is `Ok` of its status, nonzero
   included. It is `Err` only when the child could not be started or waited for, as for `Os.exec`.
3. While a goroutine waits in `Os.run`, an interrupt from the keyboard (SIGINT, and SIGQUIT where
   the host has it) does not end the run and does not make `Os.interrupt()` done. The terminal
   delivers it to the child as well, which is in the same process group, and the child decides.
4. When the child then exits **because of** SIGINT, the program receives the interrupt as if it had
   just arrived: a program that has not called `Os.interrupt()` ends with status 130, one that has
   sees the context done. When the child handled it and exited on its own, as `less`, `vim` and
   `python` do, the program goes on.
5. SIGTERM, and an interrupt that does not come from the keyboard, are handled as they are today:
   the run, or `ctx`, ends and the child is killed.

```ant
#!/usr/bin/env ant
//! Opens the newest log in the pager.

var logs = try File.list("/var/log/app")
if len(logs) == 0 { return Result.err(Error{ message: "no logs" }) }
var status = try Os.run("less", ["+G", `/var/log/app/${logs[len(logs) - 1]}`])
if status != 0 { Os.exit(status) }
```

Rule 4 is what lets this loop be stopped with Ctrl-C, while `git commit` opening an editor and
quitting it with Ctrl-C does not end the script:

```ant
for dir in Os.args() {
  var status = try Os.run("make", ["-C", dir])
  if status != 0 { println(`${dir}: make failed with ${status}`) }
}
```

### `Os.output`

Rules:

1. Standard output is captured and decoded as text, as `Os.exec`'s `stdout` is. One final `\n`, or
   `\r\n`, is removed, as a shell's `$(...)` does, and no other white space. A program that wants
   the output exactly as written uses `Os.exec`.
2. Standard error is captured too, and is not shown.
3. A status of 0 is `Ok` of the text. Any other status is `Err` with the message
   `<program>: exit status <n>`, followed by `: ` and standard error with its trailing white space
   removed when it is not empty. The text of standard output is discarded.
4. Everything else is as for `Os.exec`: `stdin`, the context, the `Err` for a child that could not
   start.

```ant
var head = try Os.output("git", ["rev-parse", "--short", "HEAD"])
var branch = try Result.wrapErr(Os.output("git", ["branch", "--show-current"]), "reading the branch")
// Err: git: exit status 128: fatal: not a git repository (or any of the parent directories): .git
```

`Os.output` returns `string` only. A child whose output is bytes is run with `Os.exec`, which keeps
`stdoutBytes`.

### `Os.start`, `Os.wait`, `Os.kill`

`Os.start` is the one call the others are made of. It starts the child, returns at once, and the
child runs beside the program.

1. **Streams.** Each of `stdin`, `stdout` and `stderr` is a handle or `null`. A handle the host can
   hand to a child as it is, an open file, a standard stream, a pipe end or a connection, is given
   as it is, so the child writes to the file without the program in between. `null` is the null
   device: reads are at end of file and writes are discarded. The defaults are the program's own
   streams, as Rust's `spawn`, Deno's `spawn` and Python's `Popen` have it.
2. **Pipe ends are handed over.** A handle from `Os.pipe` that is given to `Os.start` belongs to the
   child: `Os.start` closes the program's copy once the child has it, whether or not it started. So
   the reading end sees end of file when the child exits, which is what Go's `StdoutPipe` arranges
   by hand, and a program cannot leave a pipe open by forgetting to close it. Any other handle,
   files and standard streams, stays the program's and stays open.
3. **Waiting.** `Os.wait` blocks until the child has exited and is `Ok` of its status. It may be
   called any number of times, from any goroutine, and each call gets the same status. When `ctx`
   ends first it is `Err` with the context's reason and the child goes on running: the wait gave up,
   not the child. Unlike Go, waiting does not close anything, so a program may wait before or after
   it has read what the child wrote.
4. **Reaping.** The runtime reaps every child when it exits, waited for or not, so a child the
   program forgets is not left a zombie.
5. **The start's context** bounds the child's life, as `exec.CommandContext` does: when `ctx` is
   done, the child is killed and a later `Os.wait` gives its status. `Os.start` itself does not
   block, so the context never makes it `Err`, except when it is done already.
6. **Killing.** `Os.kill` sends a signal, by its name: `"SIGTERM"` by default, as the `kill` command
   and Node's `kill` have it, `"SIGKILL"`, `"SIGINT"`, `"SIGHUP"`, `"SIGQUIT"`, `"SIGUSR1"`,
   `"SIGUSR2"`. An unknown name, or one the host does not have, is `Err`. On Windows every signal
   ends the child. A child that has exited already is `Ok` and nothing is sent.
7. **The end of the run.** When the script body ends, children still running are left running, as in
   Go and Python; a program that wants them gone kills them, in a deferred block. When the host
   cancels the run, every child still running is killed, as `Os.exec`'s is today.

A server under test:

```ant
var server = try Os.start("./server", ["--port", "8080"], stdout: null)
defer { Os.kill(server) }
try waitForPort(8080)
var reply = try Http.get("http://localhost:8080/health")
println(reply.status)
```

### `Os.pipe`

A pipe is two handles, `read` and `write`. Each is an ordinary handle for `File`'s functions, and
`File.close` closes it. Together with rule 2 above, it covers streaming and pipelines, and `File`
already has the rest.

Reading output as it comes:

```ant
var out = try Os.pipe()
var build = try Os.start("make", ["all"], stdout: out.write)
while true {
  var line = try File.readLine(out.read)
  if line == null { break }
  if String.contains(line, "error") { println(`>> ${line}`) }
}
var status = try Os.wait(build)
```

`git log | grep fix`, with no shell and no copy in the program:

```ant
var p = try Os.pipe()
var log = try Os.start("git", ["log", "--oneline"], stdout: p.write)
var grep = try Os.start("grep", ["fix"], stdin: p.read)
try Os.wait(log)
var status = try Os.wait(grep)
```

A child's output into a file, and a file into a child:

```ant
var dump = try File.create("db.sql")
var status = try Os.wait(try Os.start("pg_dump", ["app"], stdout: dump))
try File.close(dump)

var input = try File.open("db.sql")
var result = try Os.exec("psql", ["app"], stdin: input)
```

Writing to a child while reading from it uses `go`, as it would in Go:

```ant
var into = try Os.pipe()
var from = try Os.pipe()
var sorter = try Os.start("sort", stdin: into.read, stdout: from.write)
go {
  for word in words { File.writeLine(into.write, word) }
  File.close(into.write)
}
var sorted = try File.readAll(from.read)
```

### `Os.which`

`Os.which("git")` is the path that `Os.run("git")` would start, or `null`. A script checks its
dependencies before it begins, rather than failing halfway with "executable file not found". It
follows Go's `LookPath`, so on Windows it tries `PATHEXT`, and it does not find a program in the
current directory through a `PATH` entry of `.` (Go 1.19's `ErrDot`).

### Exit status

A child ended by a signal has the status `128 + n`, where `n` is the signal's number, in `Os.run`,
`Os.wait` and `Os.exec`'s `Output.status` alike. A child killed by SIGKILL is `137` and one ended by
Ctrl-C is `130`, as `$?` reports them in a shell. A wrapper passes a status on with
`Os.exit(status)` and the shell that started it sees the same number it would have seen from the
child.

This replaces `-1`, and is the one change to existing behavior. The cost is that a status of `130`
from a child that exited with 130 on its own reads the same as one that SIGINT ended, as it does in
every shell.

### Buffering

`Os.stdout()`, `print` and `println` share one buffer (stdlib, Os). Every call that starts a child,
`Os.run`, `Os.exec`, `Os.output` and `Os.start`, flushes it first, whatever streams the child gets.
So `println("building...")` followed by `Os.run("make")` prints in that order. Python does not
flush, and "my print appears after the subprocess's output" is a known question there; the flush is
cheap next to starting a process.

What the program has read from its standard input into its buffer and not consumed is not seen by a
child that inherits standard input. That is the same in every language and is noted in the spec.

### What `Os.exec` becomes

Unchanged in what it does, and defined in terms of the rest: `Os.exec` is `Os.start` with standard
input from `stdin`, both outputs captured into the `Output`, and `Os.wait`. Its `stdin` also accepts
a handle, given to the child as rule 1 of `Os.start` says.

| A script wants                                          | Call                             |
| ------------------------------------------------------- | -------------------------------- |
| to run a program the user watches or types into         | `Os.run`                         |
| one value a program prints                              | `Os.output`                      |
| the status and both outputs, without failing on nonzero | `Os.exec`                        |
| the output as it comes, a pipeline, a file, or a server | `Os.start`, `Os.pipe`, `Os.wait` |

## Considered and rejected

- **A `Cmd` record to fill in and run, as in Go.** Go has no named or default parameters, so a
  struct is how it spells an optional field; Ant has both, and `Os.exec` already takes them. A
  record would also need every field in its literal (section 4, Records), so the shortest command
  would name seven fields. The calls above take exactly Go's fields: `Path` and `Args` are `program`
  and `args`, `Dir`, `Env`, `Stdin`, `Stdout`, `Stderr`, and `CommandContext`'s context; `Process`
  is `Os.Process`, `ProcessState.ExitCode` is the status, and `StdoutPipe` is `Os.pipe`.
- **A stream mode parameter, `"inherit" | "pipe" | "null"`.** Ant has no string literal types, and a
  union of empty records, `Os.Inherit{}`, is long for the common case. A handle already says where a
  stream goes, and the call says the default.
- **`Os.exec` with an `inherit: bool`.** It would return an `Output` whose two strings are always
  empty. A different result is a different call.
- **Unchecked `Os.output`, returning the status too.** That is `Os.exec`. The reason for a second
  call is that it checks.
- **A shell.** `Os.run("sh", ["-c", script])` is one line when a shell is truly wanted, and the
  quoting is then visibly the caller's.
- **Pipelines as one call, `Os.pipeline([cmd, cmd])`.** Julia and Python's `shell=True` have it.
  `Os.pipe` and two `Os.start` calls are six lines, and each child's status stays separate, which a
  shell's `pipefail` exists to recover.
- **A required output limit**, as Swift Subprocess has. See Prior art.

## Spec changes

| Where                                         | Change                                                                                                          |
| --------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| `spec/stdlib.md`, Os, the listing             | Add `Process`, `Pipe`, `run`, `output`, `start`, `wait`, `kill`, `pid`, `pipe`, `which`; widen `exec`'s `stdin` |
| `spec/stdlib.md`, Os, the `Os.exec` paragraph | Becomes a section on processes: the shared meaning of `program`, `args`, `dir`, `env`; the rules above          |
| `spec/stdlib.md`, Os, interrupts              | Rules 3–5 of `Os.run`: keyboard interrupts while a child shares the terminal                                    |
| `spec/stdlib.md`, Os, exit status             | `128 + n` for a signal, replacing `-1`                                                                          |
| `spec/stdlib.md`, File, intro                 | A handle is also a pipe end; a pipe read waits, as a connection's does                                          |
| `spec/stdlib.md`, Os, streams                 | Starting a child flushes standard output                                                                        |
| `spec/ant-lang.md` §4, Unions                 | Add `Os.Process` to the types with no discriminator                                                             |
| `spec/ant-lang.md` §9, Goroutines             | The host cancelling a run kills every child still running                                                       |
| `ant/TODO.md`                                 | Drop "Running a command on the terminal" and "Command output as text"                                           |
| `ant/tests/run/os_exec.ant`                   | `kill -9 $$` prints `137`                                                                                       |

In the Go interpreter, `Os.start` is `exec.Cmd` with `Stdin`, `Stdout` and `Stderr` set to the
`*os.File` behind a handle, or to a goroutine copying through `File`'s buffer for a connection whose
descriptor cannot be shared. `Os.run` sets `signal.Ignore(SIGINT, SIGQUIT)` for its duration,
reference-counted across goroutines, and reads `syscall.WaitStatus.Signal()` for rule 4. The reaping
of rule 4 of `Os.start` is one goroutine per child calling `Wait` and storing the status.
`cmd.WaitDelay` stays at a second.

## Non-goals

- **Replacing the process**, `syscall.Exec`, Rust's `CommandExt::exec`, Node 23's `process.execve`.
  It is Unix only and skips deferred blocks, as `Os.exit` does. `Os.run` followed by
  `Os.exit(status)` covers the wrapper case except for the process id; see Open questions.
- **A pseudo-terminal.** A program that drives an interactive child, as `expect` does, needs a pty.
  That is its own namespace.
- **Process groups, sessions, user ids, resource limits.** Go's `SysProcAttr`. Added when a program
  needs one.
- **Extra descriptors beyond three**, Go's `ExtraFiles`.
- **Running the program's own interpreter**, `os.Executable()`. A separate item with `Os.program` in
  `ant/TODO.md`.

## Open questions

- [ ] `Os.kill` and a context end the child itself, not what it started, so killing `sh -c '...'`
      leaves its children, which may hold a pipe open. Should `Os.start` put each child in its own
      process group and signal the group, as most process supervisors do? The cost is that a child
      in its own group no longer gets the terminal's Ctrl-C, so `Os.run` would keep the shared
      group.
- [ ] When a context ends a child: SIGKILL at once, as today, or SIGTERM and SIGKILL after a grace,
      as Go 1.20's `Cancel` and `WaitDelay` allow and `docker stop` does?
- [ ] Is `Os.replace` wanted after all? Its one advantage over `Os.run` is that the child keeps the
      script's process id, which matters to a supervisor that tracks it.
- [ ] `Os.output`'s `Err` message carries all of standard error. Truncate it, to its last lines or a
      size, as pytest truncates long values?
- [ ] Should `Os.output` and `Os.exec` take `stderr: File.Handle | null` too, so a child's warnings
      and progress reach the terminal while its output is captured, as `$(...)` does in a shell?
- [ ] In the REPL standard input belongs to the REPL. Should `Os.run` lend it the terminal for the
      duration, so `Os.run("vim")` works there, or give the child the null device?
- [ ] The status of a script that passes on a child's status still has to call `Os.exit`, which
      skips deferred blocks: "Exit status with deferred blocks" in `ant/TODO.md`. This proposal
      makes that item more pressing and does not solve it.
