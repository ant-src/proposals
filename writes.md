# Proposal: when a write is delivered

2026-11-01 · Anton · Draft

## Summary

Replace the per-handle write buffer and its three special cases with one rule that Ant already
follows for `print`:

> **A write is out before the goroutine that made it waits.**

Writes to every handle (files, connections, pipe ends, standard output) are held in a buffer while
the program is busy. Before a goroutine waits on anything (a read that waits, a channel, `select`,
`Time.sleep`, a lock, a context, a child), what it wrote goes out. It also goes out when its buffer
fills, at `File.close`, at `File.flush`, and when the run ends.

```ant
var timeout = Context.new(timeout: Time.duration(seconds: 1))
try File.pipe(Os.stdin(), Os.stdout(), ctx: timeout)   // each chunk appears as it arrives
```

What changes for a program:

- `File.pipe` streams. Each read that waits delivers what the previous chunk wrote. Today the copy
  sits in a 4 KB buffer until the buffer fills or the pipe ends.
- Standard streams, connections and pipe ends lose their "a write flushes" special case. One rule
  covers all of them, and it is the rule `print` already has.
- Every wait flushes standard output, including a read of a connection or a pipe. Today a read of an
  `Os.openPipe` end does not.
- A file the program never closed is written out when the run ends. Today its last 4 KB are lost.
- Loops that write a file without waiting stay buffered, so they stay fast.

## Motivation

### `File.pipe` is not realtime

```ant
var timeout = Context.new(timeout: Time.duration(seconds: 1))
try File.pipe(Os.stdin(), Os.stdout(), ctx: timeout)
```

When `a`, then `b` half a second later, is fed in, both lines appear together when the timeout
fires. `File.pipe` runs `io.Copy(dst.w, src.r)`:

1. `src.r` is the handle's `bufio.Reader`. `io.Copy` calls its `WriteTo`.
2. The standard input pump has no `WriteTo`, so the copy becomes `dst.w.ReadFrom(pump)`.
3. `dst.w` is the handle's `bufio.Writer` over `stdoutWriter`, which has no `ReadFrom`. So the copy
   fills the 4 KB buffer and flushes it only when the buffer is full or the copy ends.

The spec says a write to a standard stream flushes. `File.pipe` is one write path that forgot to.

### Two layers, three flags, uneven waits

Writes go through two buffers today:

- **Per handle:** a `bufio.Writer` of 4 KB on every writable handle.
- **Per run:** the buffer `print`, `println` and `Os.stdout()` share. It is flushed when the program
  reads standard input, blocks, or exits (`ant-lang.md`, prelude).

The per-handle buffer helps only regular files. Every other kind of handle opts out:

- `std`: standard streams flush at every write.
- `conn`: connections flush at every write.
- `pipe`: pipe writing ends flush at every write.

Each of these is a flag that every write path has to remember.

The run-level rule, "flush before the program blocks", is the right one. But each wait carries it by
hand: channels, `select`, `Time.sleep`, `Sync` and `Context.wait` call `Flush()` themselves.
Measured on master (2026-11-01):

| The program prints, then waits on | Output appears         |
| --------------------------------- | ---------------------- |
| `Net.accept`                      | at once                |
| `File.readLine` of `Os.stdin()`   | at once                |
| `File.readLine` of `Os.openPipe`  | 2 s later, at exit     |
| a chunk read inside `File.pipe`   | at the end of the copy |

### A file never closed loses its tail

```ant
var f = try File.create("u.txt")
try File.writeLine(f, "hello")
```

This exits 0 and leaves `u.txt` empty. A script that forgets `File.close` gets no error and no data.

## Prior art

### Copying standard input to standard output

Measured on macOS 27.0.1, 2026-11-01. Input: `a`, then `b` one second later.

| Language     | Program                                            | Lines appear     |
| ------------ | -------------------------------------------------- | ---------------- |
| Go 1.27      | `io.Copy(os.Stdout, os.Stdin)`                     | as they arrive   |
| Rust 1.94    | `io::copy(&mut stdin(), &mut stdout())`            | as they arrive   |
| Node 26      | `process.stdin.pipe(process.stdout)`               | as they arrive   |
| Deno 2.8     | `Deno.stdin.readable.pipeTo(Deno.stdout.writable)` | as they arrive   |
| Ruby 2.6     | `IO.copy_stream($stdin, $stdout)`                  | as they arrive   |
| Python 3.11  | `shutil.copyfileobj(stdin.buffer, stdout.buffer)`  | together, at EOF |
| Python 3.11  | `read1` loop, `stdout.buffer.write`                | together, at EOF |
| Python 3.11  | `for l in sys.stdin: print(l, end="")`             | together, at EOF |
| Perl 5.34    | `while (<STDIN>) { print }`                        | together, at EOF |
| Lua 5.5      | `for l in io.lines() do io.write(l, "\n") end`     | together, at EOF |
| Ant (master) | `File.pipe(Os.stdin(), Os.stdout())`               | together, at end |

There are two camps:

- **Writes go to the OS when made:** Go, Node and Deno have no user-space write buffer on a handle.
  Rust buffers standard output by line, and leaves files unbuffered. Buffering is a wrapper the
  program asks for (`bufio.Writer`, `BufWriter`).
- **Writes are buffered, with flush rules:** the C stdio tradition. Python, Perl and Lua buffer
  whole blocks when output is not a terminal. Each has its own escape hatch: `PYTHONUNBUFFERED`,
  `flush=True`, `$| = 1`, `setvbuf`. "My output appears late when piped" is a perennial question in
  all of them.

C has a narrower form of Ant's rule. A line-buffered stream is flushed when input is requested, so a
prompt shows before the program reads. Python's `input()` does the same. Ant already widens this
from "before input" to "before any wait" for standard output. This proposal widens it to every
handle.

### Reads

Every runtime above keeps a read buffer where line reads need one (Rust: one global buffer behind
`Stdin`, the same shape as Ant's one handle per stream). Read buffering adds no latency as long as a
read returns whatever is available rather than waiting to fill: `read1`, `readpartial`, Rust's
`read`, Ant's `readChunk`. Ant's read side already behaves this way and is not changed here.

### What a write costs

Measured on macOS 27.0.1, Apple silicon, 200,000 lines of `line N`:

| Program                                      | Time   | Per line |
| -------------------------------------------- | ------ | -------- |
| Go, `fmt.Fprintf` to an unbuffered `os.File` | 285 ms | 1.4 µs   |
| Go, the same through `bufio.Writer`          | 8 ms   | 0.04 µs  |
| Ant, `File.writeLine` loop (buffered, today) | 46 ms  | 0.23 µs  |
| Ant, the same loop without the write         | 39 ms  | 0.20 µs  |

An Ant loop iteration costs less than a system call. Making every write a system call would make a
file-writing loop about 7× slower (46 ms → about 330 ms). That rules out the simplest design below,
unless file-writing loops are rare enough not to matter.

## Design

### The rule

Each writable handle has a write buffer. A write goes into it. The buffer is written out:

1. **before the goroutine that wrote it waits**;
2. when the goroutine ends;
3. when the buffer is full;
4. at `File.flush` and `File.close`;
5. when the run ends, `Os.exit` included.

A **wait** is anything that may block on something outside the goroutine:

- a read of standard input, a connection, a pipe or a terminal;
- `Net.accept`;
- `Channel.send` and `Channel.receive`, and `select`;
- `Time.sleep`;
- `Context.wait`, and a `Sync` lock or wait;
- `Cmd.run`, `Cmd.wait` and `Cmd.start`.

A read of a regular file never waits (as the spec already says), so it delivers nothing.

Standard output keeps its one run-wide buffer, shared by `print`, `println` and `Os.stdout()` so
they interleave in program order. Before any goroutine waits, that buffer is written out, as today.
Standard error stays unbuffered and flushes standard output first, as today.

Other handles are delivered per goroutine. A goroutine writes out the handles **it** wrote. So a
goroutine waiting on a channel never blocks behind another goroutine's connection whose peer is not
reading.

### Why "before it waits"

Output can be late only while the program is busy computing. Whenever it stops to wait for input, a
peer, a timer or another goroutine, everything it wrote is out. That is the moment a reader on the
other end could need it:

- **A prompt** is out before the read of the answer.
- **A request** is out before the read of the reply.
- **A pipe or proxy** delivers each chunk before reading the next.
- **A log line** before `Time.sleep` or a channel receive is out.
- **A loop that writes a file without waiting** fills whole buffers, at buffered speed.

The cost of a flush is paid only at a wait, which is already a slow path.

### Errors

A write-out can fail at a wait, where no write call is there to report it. The handle keeps the
first such error, and the next `write*`, `File.flush` or `File.close` of that handle returns it, as
Go's `bufio.Writer` does. `File.close` keeps its current contract: it reports the write-out and the
close. So the explicit-close advice in `defer` and `File` stays.

### `File.pipe`

`File.pipe` is a loop: write out what the handle already read ahead, then read a chunk and write it.
Each read that waits is a wait, so a pipe from standard input, a connection or a pipe streams chunk
by chunk. A pipe from a regular file never waits, so it copies in whole buffers. When the context
ends the copy, what it copied is written out, as today.

### `File.flush`

`File.flush` stays. It is for a goroutine that writes and then computes for a long time without
waiting, when a reader such as `tail -f` should see the output now. With this rule, programs rarely
need it.

## Spec changes

- `stdlib.md`, File:
  - Replace "A write to a connection is flushed as a write to a standard stream is …; `File.flush`
    is for files" with the rule above.
  - Reword the `File.flush` and `File.close` comments ("writes out what is buffered").
  - Add: an unclosed handle is written out when the run ends.
  - Add: a write-out that fails at a wait is reported by the next write, flush or close.
- `stdlib.md`, Os, Streams: standard output is flushed before any goroutine waits, and the list of
  waits is the one in File. Drop "a write to `write` is flushed" from `Os.openPipe`.
- `stdlib.md`, Net: drop "A write to a connection is flushed, as `File` says".
- `ant-lang.md`, prelude (`print`): "flushed when the program reads standard input, blocks, or
  exits" becomes a reference to the one rule.
- `stdlib.md`, Cmd: "Starting a command flushes standard output" and "each handle given as an output
  is flushed first" stay. Starting a child is a wait, and handing a handle to a child needs its
  buffer empty.

## Implementation sketch

Not part of the proposal's contract; recorded so the cost is visible.

- `Goroutine` gets a small set of handles written since its last wait. `Handle.write` adds the
  handle to it.
- `Goroutine.BeforeWait()` writes out stdout and that set. It replaces the scattered
  `Instance().Flush()` calls, and every wait calls it. `Handle.read` calls it when the handle can
  wait.
- The `std`, `conn` and `pipe` flush flags go. Standard output's handle writes straight into the run
  buffer, without a `bufio.Writer` of its own.
- `File.pipe` becomes a loop of `readChunk`-style reads and writes instead of `io.Copy`. That
  removes the `WriteTo`/`ReadFrom` shortcut that caused the bug.
- The `Instance` keeps the open handles, so that the end of the run can write them out.

## Considered and rejected

### A. Unbuffered writes (Go, Node, Deno)

Each `File.write*` is one OS write.

- **For:** the simplest model. No `File.flush`, no error deferred to close, no flush rule.
- **Against:** a file-writing loop gets about 7× slower (measured above). Getting the speed back
  needs a buffered-writer API (`File.buffered(f)`, Go's `bufio.Writer`). That brings back `flush`,
  deferred errors and a buffer for the program to remember.
- Standard output would still need its run buffer to keep `print` fast. So Ant would have two rules,
  not one.

This is the main alternative. If file-writing loops prove rare, it is simpler than the proposal.

### B. Buffer by kind: files buffered, the rest written through

Today's behaviour, with the kind fixed at open instead of by flags, and `File.pipe` fixed.

- A minimal patch.
- It keeps one special case per kind.
- It leaves the uneven waits (an `Os.openPipe` read does not flush stdout) to be fixed one by one.

### C. Line-buffered standard output (C on a terminal, Rust)

A third rule beside the two Ant has. A prompt without a newline would wait, which is why Ant flushes
before a read in the first place.

### D. Flush every handle of the run at every wait

Simpler than per goroutine. But a goroutine waiting on a channel could block writing out another
goroutine's connection whose peer is not reading.

## Non-goals

- Read buffering: unchanged.
- `O_SYNC`, `fsync` and durability: a write-out hands data to the OS, nothing more.
- A buffer size option.

## Open questions

1. **Per goroutine or per run** for handles other than stdout? Per goroutine is proposed (see D).
2. **Writing out at run end.** Should `Os.exit` write out files, given that Go's `os.Exit` does not
   flush a `bufio.Writer`? Proposed: yes. Ant already flushes stdout there, and losing a file's tail
   silently is worse.
3. **Errors at run end.** A write-out that fails when the run ends has no caller. Report it on
   standard error and exit non-zero, or drop it?
4. **A or this proposal.** Is the 7× cost on file-writing loops worth a flush rule? Measure real
   scripts before deciding.
