# Proposal: running programs

2026-10-27 · Anton · Accepted, and written into `spec/stdlib.md` with the changes in Decisions

## Decisions

Implementing it changed three things; the spec has the result.

- **`Os.exec` and `Os.output` take no handle as `stdin`.** `string | Bytes | File.Handle` is not a
  valid union: `is` tests neither `Bytes` nor `File.Handle` (`ant-lang.md` §4, Unions), so a union
  holds at most one of them. `stdin` stays `string | Bytes`; a handle as input goes through
  `Os.start(..., stdin: h)`.
- **An interrupt that ends the run kills `Os.start` children too**, not only the host's
  cancellation: both end the run before the script does.
- **A child started with a context is killed when the run ends**, whatever way, since a context ends
  with its run (`ant-lang.md` §9). A child started without one is left running when the script ends.

## Summary

Grow `Os.exec` into a small process API modelled on Go's `exec.Cmd`, spelled the Ant way: named
parameters instead of a struct to fill in, `File.Handle` for every stream, and `Result` for every
failure. Six additions and two changes:

- `Os.run`: the child runs in the foreground with the program's terminal, and the call returns its
  exit status.
- `Os.output`: the child's standard output as text, and `Err` on a nonzero status.
- `Os.start`, `Os.wait`, `Os.kill`, `Os.stop`: a child that runs in the background, in a process
  group of its own, with its streams wired to any handle.
- `Os.openPipe`: a pair of connected handles, to read a child's output as it comes or to join two
  children.
- `Os.which`: where `PATH` finds a program.
- A child ended by a signal has the status `128 + n`, as a shell reports it, not `-1`.
- When an interrupt ends a run, `ant` ends by that signal rather than exiting with 130, so the shell
  that started it can tell.

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
| POSIX `sh`           | `cmd`, in the foreground                             | `$(cmd)`: **every** trailing newline removed                      | `cmd &`: stdin is `/dev/null` without job control                | `<`, `>`, `2>&1`, `\|`                                                    |

What Ant takes:

- **Go's model of streams.** A stream is a handle, and a handle backed by a descriptor is handed to
  the child as it is, so the terminal, a file and a pipe are one case. Go needs `io.Writer` adapters
  and goroutines for everything else; Ant's runtime runs that copy itself for the few handles that
  need it (see Handing over a handle).
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
- **The shell's foreground and background.** A shell runs `cmd` in the foreground, where the
  terminal's Ctrl-C reaches it, and `cmd &` in the background, with `/dev/null` as its input when
  there is no job control. `Os.run` is the first and `Os.start` the second.
- **The shell's handling of Ctrl-C.** `system(3)` ignores SIGINT while its child runs; bash goes one
  step further and, when the child died of SIGINT, acts on it itself, by dying of SIGINT in turn
  (Cracauer, "Proper handling of SIGINT/SIGQUIT", the "wait and cooperative exit" rule). Without
  that step a loop of `Os.run` calls cannot be stopped from the keyboard; without the first, `less`
  cannot be quit with Ctrl-C without killing the script; and without dying of the signal, rather
  than exiting with 130, a shell loop that calls the script cannot be stopped either.
- **No shell, no template.** zx, Bun and Julia make a command a quoted template. Ant has no
  backtick-command syntax and keeps arguments an `Array[string]`, as `Os.exec` already does.
- **No mandatory limit.** Swift Subprocess requires a size limit on every captured output. Ant's
  other reads of whole things, `File.read` and `Http` bodies, have none; a program that fears a
  large output streams it through `Os.openPipe`.

## The API

```
type Output = { status: int, stdout: string, stderr: string, stdoutBytes: Bytes, stderrBytes: Bytes }   // unchanged
type Process                                    // opaque: a started child
type Pipe = { read: File.Handle, write: File.Handle }

Os.run(program: string, args: Array[string] = [], dir: string | null = null,
       env: Map[string, string] | null = null, ctx: Context | null = null): Result[int, Error]
                                                // in the foreground, with the program's three streams; Ok of its status
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
         stdin: File.Handle | null = null, stdout: File.Handle | null = Os.stdout(),
         stderr: File.Handle | null = Os.stderr(), ctx: Context | null = null): Result[Os.Process, Error]
                                                // in the background, in a process group of its own; returns once it is running; null is the null device
Os.wait(p: Os.Process, ctx: Context | null = null): Result[int, Error]   // its status, once it has exited; every call gets the same outcome
Os.kill(p: Os.Process, signal: string = "SIGTERM"): Result[void, Error]  // to its process group; Ok when it has exited already
Os.stop(p: Os.Process, grace: Time.Duration = Time.duration(seconds: 5)): Result[int, Error]
                                                // SIGTERM, then SIGKILL after grace; its status
Os.pid(p: Os.Process): int

Os.openPipe(): Result[Os.Pipe, Error]           // what is written to write is read from read
Os.which(program: string): string | null        // the path the calls above would start; null when none
```

`program`, `args`, `dir` and `env` mean what they mean for `Os.exec` today in every call: no shell,
and `env` is the child's whole environment. When `program` has no path separator it is looked up in
the program's own `PATH`, `Os.env("PATH")`, not in the `PATH` of `env`, as Go's `exec.Command` does;
`Os.which` takes no `env`, and must find what the calls would start.

The name is `openPipe`, not `pipe`, because `File.pipe(src, dst)` already means copying one handle
into another. Like `File.open`, it returns handles the program closes.

### `Os.run`

`Os.run` is the call for a script's foreground: the program the user is looking at.

Rules:

1. **Streams.** The child's standard input, output and error are the program's own, handed over as
   they are, and the child stays in the program's process group, so the terminal treats it as the
   foreground. Before it starts, standard output is flushed (see Handing over a handle). What the
   program has read from standard input into its buffer and not consumed stays the program's: a
   terminal is handed to the child as it is, never through a copy.
2. **Result.** The call blocks the calling goroutine until the child exits and is `Ok` of its
   status, nonzero included. It is `Err` when the child could not be started, and as Cancellation
   below says.
3. **Holding back.** While any goroutine waits in `Os.run`, every SIGINT and SIGQUIT the `ant`
   command receives is held back: the run does not end and `Os.interrupt()` is not done. A Ctrl-C
   and a `kill -INT` from elsewhere look the same to a process, so both are held. The terminal
   delivers the keyboard's signal to the child as well, which has it at its default disposition, and
   the child decides.
4. **Passing on.** When an `Os.run` child that was waiting when the signal arrived exits **because
   of** that signal, the program receives it as if it had arrived then. A SIGINT is an interrupt
   (stdlib, Os): it ends the run, or makes `Os.interrupt()` done when the program asked for it. A
   SIGQUIT does what it does when no child runs. When every such child exits otherwise, the held
   signal is dropped: it was the child's to handle, and `less`, `vim` and `python` handle it. Held
   signals do not queue, as pending signals do not: several Ctrl-C presses while one child runs are
   one interrupt, and one passed on counts once toward "a second interrupt ends the run".
5. **The rest.** SIGTERM is not held back: it ends the run, or makes `Os.interrupt()` done, as
   today, and the host's cancellation kills the child.
6. **Scope.** Holding back is process-wide, since signals are. A goroutine waiting on
   `Os.interrupt()` sees a Ctrl-C only through rule 4 while another goroutine is in `Os.run`. So a
   server does not call `Os.run`; `Os.exec`, `Os.output` and `Os.start` hold nothing back.
7. **Host.** Rules 3 to 6 are the `ant` command's. A Go program that embeds Ant owns its process's
   signals, and there `Os.run` only shares the streams. An `ant` started with SIGINT ignored, as
   `nohup` and `cmd &` in a non-interactive shell start it, keeps it ignored and holds nothing back.

```ant
#!/usr/bin/env ant
//! Opens the newest log in the pager. The logs are named by date, app-2026-10-27.log, so the
//! last in lexical order is the newest.

var dir = "/var/log/app"
var newest = Array.last(try File.list(dir)) ?? return Result.err(Error{ message: "no logs" })
var status = try Os.run("less", ["+G", `${dir}/${newest}`])
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

### How a run ends by an interrupt

Today the `ant` command exits with status 130 when an interrupt ends the run (stdlib, Os). That
breaks Cracauer's rule one level up: in `for d in *; do ./build.ant "$d"; done`, bash sees a child
that handled SIGINT and exited on its own, and goes on to the next directory.

This proposal changes it: when an interrupt ends the run, `ant` finishes ending it as today, then
resets SIGINT to its default and sends it to itself, so its parent sees a process that SIGINT ended,
which a shell reports as 130. A SIGTERM that ends the run does the same with SIGTERM. Where the host
has no signals, `ant` exits with 130 as today. A program that called `Os.interrupt()` and then ends
by its own choice exits as it chooses.

### `Os.output`

Rules:

1. Standard output is captured and decoded as text, as `Os.exec`'s `stdout` is. One final `\n`, or
   `\r\n`, is removed, and no other white space. This is Ant's rule, not the shell's: `$(...)`
   removes every trailing newline, which loses a blank last line that is data. A program that wants
   the output exactly as written uses `Os.exec`.
2. Standard error is captured too, and is not shown.
3. A status of 0 is `Ok` of the text. Any other status is `Err` with the message
   `<program>: exit status <n>`, followed by `: ` and standard error with its trailing white space
   removed when it is not empty. The text of standard output is discarded.
4. Everything else is as for `Os.exec`: `stdin`, the context, the process group, the `Err` for a
   child that could not start.

```ant
var head = try Os.output("git", ["rev-parse", "--short", "HEAD"])
var branch = try Result.wrapErr(Os.output("git", ["branch", "--show-current"]), "reading the branch")
// Err: git: exit status 128: fatal: not a git repository (or any of the parent directories): .git
```

`Os.output` returns `string` only. A child whose output is bytes is run with `Os.exec`, which keeps
`stdoutBytes`.

`Os.exec` and `Os.output` keep today's process group, the program's, as `$(...)` does, so a child
that opens `/dev/tty` itself, as `fzf` does, can still use the terminal. A Ctrl-C reaches such a
child and the program both, and ends the run, which then kills the child, as today.

### `Os.start`, `Os.wait`, `Os.kill`, `Os.stop`

`Os.start` is the background call. It starts the child, returns at once, and the child runs beside
the program.

1. **Streams.** Each of `stdin`, `stdout` and `stderr` is a handle or `null`, given as Handing over
   a handle says. `null` is the null device: reads are at end of file and writes are discarded.
   Standard output and error default to the program's, as Rust's and Deno's `spawn` have it, and
   standard input to `null`, as a shell's `cmd &` has it.
2. **No terminal input.** A terminal as `stdin` is `Err`, and the child is not started. A child
   outside the foreground process group that reads the terminal is stopped by it (SIGTTIN) and would
   hang; a child the user types into is `Os.run`'s.
3. **Process group.** The child is the leader of a process group of its own, and its children belong
   to it unless they leave. So the terminal's Ctrl-C does not reach it (rule 7 decides what happens
   to it), and `Os.kill`, `Os.stop` and the start's context signal the whole group: stopping
   `sh -c '...'` stops what it started too, and nothing is left holding a pipe open. On Windows,
   which has no process groups, they reach the child alone.
4. **Waiting.** `Os.wait` blocks until the child has exited, and until the copies the runtime runs
   for it (see Handing over a handle) have ended, a copy still running a second after the exit being
   cut off, as Go's `WaitDelay` does. It reads nothing and closes nothing. It may be called any
   number of times, from any goroutine, and each call gets the same outcome. A child that writes
   into a pipe nobody reads blocks once the pipe is full and never exits, so a program that pipes a
   child's output reads it while it waits: before the wait, or on another goroutine. The end of file
   on the reading end is a separate event and can come later than the exit, when a process the child
   started still holds the writing end.
5. **Reaping.** The runtime reaps every child when it exits, waited for or not, so a child the
   program forgets is not left a zombie.
6. **Killing and stopping.** `Os.kill` sends a signal by its name: `"SIGTERM"` by default, as the
   `kill` command and Node's `kill` have it, `"SIGKILL"`, `"SIGINT"`, `"SIGHUP"`, `"SIGQUIT"`,
   `"SIGUSR1"`, `"SIGUSR2"`. An unknown name, or one the host does not have, is `Err`. It does not
   wait. `Os.stop` is the shutdown a program can rely on: it sends SIGTERM, waits up to `grace`,
   sends SIGKILL if the child is still running, waits again, and returns the status. On a child that
   has exited already, both send nothing; `kill` is `Ok` and `stop` gives the status.
7. **The end of the run.** When the script body ends, children still running are left running, as in
   Go and Python; a program that wants them gone stops them, in a deferred block. When the host
   cancels the run, every child still running is killed, as `Os.exec`'s is today.

A server under test:

```ant
var server = try Os.start("./server", ["--port", "8080"], stdout: null)
defer { Os.stop(server) }
try waitForPort(8080)          // assumed: dials the port until it answers, or gives up
var reply = try Http.get("http://localhost:8080/health")
println(reply.status)
```

### Cancellation

A context bounds a wait, and the context of a call that starts a child also bounds the child's life,
as `exec.CommandContext` does.

| What happens                                                     | Outcome                                                                                                    |
| ---------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| The start's context is done before the child starts              | `Err` with the context's reason; nothing is started                                                        |
| The start's context ends while the child runs                    | The child, and for `Os.start` its group, is killed; the call, or every `Os.wait`, is `Err` with the reason |
| The child exits before the runtime sends the kill, then ctx ends | The child's status, as if there were no context                                                            |
| An `Os.wait` context ends first                                  | That `Os.wait` is `Err` with the reason; the child goes on running, and a later wait can still get it      |
| The child exits nonzero                                          | `Ok` of the status from `run`, `exec` and `wait`; `Err` from `output`                                      |
| The program kills it with `Os.kill` or `Os.stop`                 | `Ok` of the status, `128 + n`                                                                              |
| The host cancels the run                                         | The child is killed; the call is `Err` with the reason `cancelled`                                         |

The race between an exit and a cancellation is decided by whether the runtime sent the kill. With
these outcomes `Os.exec` is exactly `Os.start`, two captures and `Os.wait`, and its `Err` on a
cancelled context is kept. A program that passed contexts to both `Os.start` and `Os.wait` tells
them apart by `Context.isDone`.

### Handing over a handle

A handle given to a child is **borrowed**: the child gets its own copy of the descriptor, and the
program's handle stays open and stays the program's, whatever it is and whether or not the child
started. Nothing else is inherited: a child has only the three streams it was given, so a pipe's
ends reach only the children they are given to.

1. **Writes are flushed.** Every handle given as `stdout` or `stderr` is flushed before the child
   starts, and standard output always is, whatever the child gets. A header written with
   `File.writeText` and the child's output after it arrive in that order. A flush that fails makes
   the call `Err`, and the child is not started.
2. **Buffered reads are copied.** A handle given as `stdin` whose buffer holds input the program has
   read ahead and not consumed is given through a copy: the child reads a pipe, and the runtime
   writes into it the buffered input and then the rest of the handle, until end of file or until the
   child exits. The program does not read that handle while the child runs. A handle with nothing
   buffered is given as it is. Standard input is never copied (rule 1 of `Os.run`).
3. **Handles without a descriptor** to hand over, such as a connection the host serves itself, go
   through the same copy, in whichever direction they are given.
4. **Pipe writes are not buffered.** A write to a pipe end is flushed as a write to a connection is
   (stdlib, File), since the other end is waiting for it.
5. **Closing is the program's.** The reading end of a pipe sees end of file only when every writing
   end is closed, the program's included. So after `Os.start`, the program closes its copy of each
   writing end it gave away, as in C, Go's `os.Pipe` and Python. The examples below do.

The same handle may be given twice: `stdout: p.write, stderr: p.write` merges the two outputs, as
`2>&1` does. Two children may write into one pipe, and the reader sees end of file when both have
exited and the program has closed its copy.

### `Os.openPipe`

A pipe is two handles, `read` and `write`, each an ordinary handle for `File`'s functions.

Reading output as it comes:

```ant
var out = try Os.openPipe()
var build = try Os.start("make", ["all"], stdout: out.write, stderr: out.write)
try File.close(out.write)                       // the child has its own copy
while true {
  var line = try File.readLine(out.read)
  if line == null { break }
  if String.contains(line, "error") { println(`>> ${line}`) }
}
var status = try Os.wait(build)
```

`git log | grep fix`, with no shell and no copy in the program:

```ant
var p = try Os.openPipe()
var log = try Os.start("git", ["log", "--oneline"], stdout: p.write)
var grep = try Os.start("grep", ["fix"], stdin: p.read)
try File.close(p.write)
try File.close(p.read)
try Os.wait(log)
var status = try Os.wait(grep)
```

A child's output into a file, and a file into a child:

```ant
var dump = try File.create("db.sql")
try File.writeLine(dump, "-- dump of app")      // flushed before pg_dump starts
var status = try Os.wait(try Os.start("pg_dump", ["app"], stdout: dump))
try File.close(dump)

var input = try File.open("db.sql")
var result = try Os.exec("psql", ["app"], stdin: input)
```

Writing to a child while reading from it uses `go`, as it would in Go:

```ant
var into = try Os.openPipe()
var from = try Os.openPipe()
var sorter = try Os.start("sort", stdin: into.read, stdout: from.write)
try File.close(into.read)
try File.close(from.write)
go {
  for word in words { File.writeLine(into.write, word) }
  File.close(into.write)
}
var sorted = try File.readAll(from.read)
var status = try Os.wait(sorter)
```

### `Os.which`

`Os.which("git")` is the path that `Os.run("git")` would start, or `null`. A script checks its
dependencies before it begins, rather than failing halfway with "executable file not found". It
follows Go's `LookPath` over the program's own `PATH`, so on Windows it tries `PATHEXT`, and it does
not find a program in the current directory through a `PATH` entry of `.` (Go 1.19's `ErrDot`).

### Exit status

Where the host has signals, a child ended by a signal has the status `128 + n`, where `n` is the
signal's number, in `Os.run`, `Os.wait`, `Os.stop` and `Os.exec`'s `Output.status` alike. A child
killed by SIGKILL is `137` and one ended by Ctrl-C is `130`, as `$?` reports them in a shell. On
Windows the status is the exit code, and a child that `Os.kill`, `Os.stop` or a context ended has
status 1, the code `TerminateProcess` is given.

This replaces `-1`. The cost is that a status of `130` from a child that exited with 130 on its own
reads the same as one that SIGINT ended, as it does in every shell.

A wrapper passes a status on with `Os.exit(status)`, which passes the number but not that a signal
ended the child. A child that SIGINT or SIGQUIT ended under `Os.run` needs nothing more, since rule
4 passes the signal on and `ant` then ends by it. A child that SIGTERM or SIGKILL ended reaches the
wrapper's parent as an exit with `143` or `137`.

### What `Os.exec` becomes

Unchanged in what it does, and defined in terms of the rest: `Os.exec` is `Os.start` in the
program's process group, with standard input from `stdin`, both outputs captured into the `Output`,
and `Os.wait`. Its `stdin` also accepts a handle, handed over as above.

| A script wants                                          | Call                                 |
| ------------------------------------------------------- | ------------------------------------ |
| to run a program the user watches or types into         | `Os.run`                             |
| one value a program prints                              | `Os.output`                          |
| the status and both outputs, without failing on nonzero | `Os.exec`                            |
| the output as it comes, a pipeline, a file, or a server | `Os.start`, `Os.openPipe`, `Os.wait` |

## Considered and rejected

- **A command record, `Os.Command{ program, args, dir, env }`, that each call takes.** Field
  defaults (section 4) make it short to write, and spreading one gives presets:
  `Os.Command{ ...git, dir: fork }`. Not adopted in this draft. What differs between the calls is
  what they do with streams, `stdin` and the context, so a record would hold only `program`, `args`,
  `dir` and `env`, and a function is a preset already:
  `func git(args: Array[string]) { Os.output("git", args, dir: repo) }`. Go needs `Cmd` because it
  has no named or default parameters. The calls take exactly Go's fields: `Path` and `Args` are
  `program` and `args`, `Dir`, `Env`, `Stdin`, `Stdout`, `Stderr`, and `CommandContext`'s context;
  `Process` is `Os.Process`, `ProcessState.ExitCode` is the status, and `StdoutPipe` is
  `Os.openPipe`. See Open questions.
- **A stream mode parameter, `"inherit" | "pipe" | "null"`.** Ant has no string literal types, and a
  union of empty records, `Os.Inherit{}`, is long for the common case. A handle already says where a
  stream goes, and the call says the default.
- **Pipe ends handed over to the child**, closed by `Os.start` so the program cannot forget. It
  saves one close, but makes a `File.Handle` behave by where it came from: a helper that takes a
  handle could not say whether it keeps it, a failed start could not be retried with the same pipe,
  and two children could not write into one pipe. Borrowing every handle is one rule.
- **`signal.Ignore` while `Os.run` waits.** An ignored signal stays ignored across `exec`, so the
  child would start with SIGINT ignored, and many programs keep it so; Ctrl-C would do nothing. A
  caught signal is reset to its default in the child, which is what `system(3)` gets by resetting in
  the child. It would also cancel the handler behind `Os.interrupt()`.
- **Telling a keyboard SIGINT from another.** A process sees the same signal; the code that would
  tell them apart is not exposed by Go and not portable.
- **`Os.exec` with an `inherit: bool`.** It would return an `Output` whose two strings are always
  empty. A different result is a different call.
- **Unchecked `Os.output`, returning the status too.** That is `Os.exec`. The reason for a second
  call is that it checks.
- **A shell.** `Os.run("sh", ["-c", script])` is one line when a shell is truly wanted, and the
  quoting is then visibly the caller's.
- **Pipelines as one call, `Os.pipeline([cmd, cmd])`.** Julia and Python's `shell=True` have it.
  `Os.openPipe` and two `Os.start` calls are a few lines, and each child's status stays separate,
  which a shell's `pipefail` exists to recover.
- **A required output limit**, as Swift Subprocess has. See Prior art.

## Spec changes

| Where                                         | Change                                                                                                                      |
| --------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `spec/stdlib.md`, Os, the listing             | Add `Process`, `Pipe`, `run`, `output`, `start`, `wait`, `kill`, `stop`, `pid`, `openPipe`, `which`; widen `exec`'s `stdin` |
| `spec/stdlib.md`, Os, the `Os.exec` paragraph | Becomes a section on processes: the shared parameters, `PATH`, foreground and background, handing over, cancellation        |
| `spec/stdlib.md`, Os, interrupts              | Rules 3 to 7 of `Os.run`; an interrupt or SIGTERM that ends the run ends `ant` by that signal, 130 where there are none     |
| `spec/stdlib.md`, Os, exit status             | `128 + n` for a signal, replacing `-1`; the Windows status                                                                  |
| `spec/stdlib.md`, File, intro                 | A handle is also a pipe end; a pipe read waits and a pipe write is flushed, as a connection's are                           |
| `spec/stdlib.md`, Os, streams                 | Starting a child flushes standard output; a child inheriting standard input does not see what the program read ahead        |
| `spec/ant-lang.md` §4, Unions                 | Add `Os.Process` to the types with no discriminator                                                                         |
| `spec/ant-lang.md` §9, Goroutines             | The host cancelling a run kills every child still running                                                                   |
| `ant/TODO.md`                                 | Drop "Running a command on the terminal" and "Command output as text"                                                       |
| `ant/tests/run/os_exec.ant`                   | `kill -9 $$` prints `137`                                                                                                   |

In the Go interpreter, `Os.start` is `exec.Cmd` with `SysProcAttr.Setpgid`, and `Stdin`, `Stdout`
and `Stderr` set to the `*os.File` behind a handle, or to the runtime's copy (rules 2 and 3 of
Handing over a handle). Go opens every descriptor close-on-exec, which gives "nothing else is
inherited". `Os.kill` signals `-pgid`. For `Os.run`, the `ant` command keeps its `signal.Notify`
handler for SIGINT, adds SIGQUIT for the duration, and while a count of waiting `Os.run` calls is
above zero records an arriving signal instead of acting on it; the child gets both at their default,
since a caught signal is reset by `exec`. When a child exits, `syscall.WaitStatus.Signal()` decides
rule 4. Ending by a signal is `signal.Reset` and `syscall.Kill(os.Getpid(), sig)` after the run has
ended, with `os.Exit(128 + n)` as the fallback. At startup, `signal.Ignored(SIGINT)` decides rule
7's last sentence. The reaping of rule 5 of `Os.start` is one goroutine per child calling `Wait` and
storing the outcome. `cmd.WaitDelay` stays at a second.

## Non-goals

- **Replacing the process**, `syscall.Exec`, Rust's `CommandExt::exec`, Node 23's `process.execve`.
  It is Unix only and skips deferred blocks, as `Os.exit` does. `Os.run` followed by
  `Os.exit(status)` covers the wrapper case except for the process id and a child ended by SIGTERM
  or SIGKILL (see Exit status); see Open questions.
- **A pseudo-terminal.** A program that drives an interactive child, as `expect` does, needs a pty.
  That is its own namespace.
- **Job control**: moving a child between foreground and background, `tcsetpgrp`, SIGTSTP.
- **Sessions, user ids, resource limits.** The rest of Go's `SysProcAttr`. Added when a program
  needs one.
- **Extra descriptors beyond three**, Go's `ExtraFiles`.
- **Running the program's own interpreter**, `os.Executable()`. A separate item with `Os.program` in
  `ant/TODO.md`.

## Open questions

- [ ] `Os.Command` presets. If scripts that call one tool many times (git, docker, kubectl) turn out
      to write the `func git(args)` preset everywhere, a record with field defaults could be the
      first parameter, `program: string | Os.Command`, with the call's `args` after the record's.
      With it, `env` would add to the program's environment rather than replace it, as Rust, Deno,
      Ruby, Crystal, Dart and Julia's `addenv` do, `null` removing a variable and `clearEnv: true`
      starting empty. Today `env` is the whole environment, so adding one variable copies
      `Os.environ()` first.
- [ ] When a context ends a child: SIGKILL at once, as today, or what `Os.stop` does, SIGTERM and
      SIGKILL after a grace, as Go 1.20's `Cancel` and `WaitDelay` allow?
- [ ] Is `Os.replace` wanted after all? It keeps the process id and passes on every signal death,
      which matter to a supervisor that tracks the script.
- [ ] `Os.output`'s `Err` message carries all of standard error. Truncate it, to its last lines or a
      size, as pytest truncates long values?
- [ ] Should `Os.output` and `Os.exec` take `stderr: File.Handle | null` too, so a child's warnings
      and progress reach the terminal while its output is captured, as `$(...)` does in a shell?
- [ ] On Windows, should `Os.start` put the child in a job object, so `Os.kill` and `Os.stop` reach
      what it started, as the process group does elsewhere?
- [ ] In the REPL standard input belongs to the REPL. Should `Os.run` lend it the terminal for the
      duration, so `Os.run("vim")` works there, or give the child the null device?
- [ ] The status of a script that passes on a child's status still has to call `Os.exit`, which
      skips deferred blocks: "Exit status with deferred blocks" in `ant/TODO.md`. This proposal
      makes that item more pressing and does not solve it.
