# Proposal: unbuffered writes and `File.buffered`

2026-11-01 · Anton · Draft

## Summary

Make writes behave as they do in Go. Buffering happens only where the program asks for it.

- **Every write is one OS write.** This covers `File.write*` on any handle, `print`, `println`,
  `eprint`, `eprintln`, connections and pipe ends. Nothing is held back, so nothing needs a flush,
  and output appears in program order.
- **`File.buffered(f)` is the opt-in.** It returns a handle whose writes collect in a buffer until
  the buffer is full, `File.flush` is called, or `File.close` is called. It is Go's `bufio.Writer`,
  as a `File.Handle`.
- **Reads are unchanged.** A handle keeps one read buffer, so `readLine` and `readAll` can be mixed,
  and a read returns what is available without waiting to fill it.

```ant
var timeout = Context.new(timeout: Time.duration(seconds: 1))
try File.pipe(Os.stdin(), Os.stdout(), ctx: timeout)   // each chunk appears as it arrives

var out = File.buffered(try File.create("big.txt"))   // a loop of many small writes
defer { File.close(out) }
for row in rows { try File.writeLine(out, row) }
try File.close(out)                                     // writes out the buffer, then closes
```

This removes the following from the spec and the runtime:

- the run-wide standard output buffer, and every rule for flushing it: before a read of standard
  input, before a wait, before a write to standard error, before starting a command, and at exit;
- the per-handle write buffer and its three opt-outs (`std`, `conn`, `pipe`);
- the case of a write that succeeds and then fails at `File.close`, for every handle that is not
  buffered.

It also fixes three bugs found on master that come from that machinery (see Motivation).

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

### Two buffers, three flags, uneven waits

Writes go through two buffers today:

- **Per handle:** a 4 KB `bufio.Writer` on every writable handle.
- **Per run:** the buffer that `print`, `println` and `Os.stdout()` share. It is flushed when the
  program reads standard input, blocks, or exits (`ant-lang.md`, prelude).

The per-handle buffer helps regular files only. Every other kind of handle opts out with a flag, and
every write path has to remember the flag:

- `std`: standard streams flush at every write.
- `conn`: connections flush at every write.
- `pipe`: pipe writing ends flush at every write.

The run-wide buffer is flushed before a wait only where each kind of wait remembers to do it.
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

This exits 0 and leaves `u.txt` empty.

### The rules are the bug

Each of these comes from the same place: writes are held, and some rule must let them go. The spec
spends paragraphs on those rules:

- when standard output is flushed;
- why standard error flushes standard output;
- why a connection is flushed;
- why `File.pipe` flushes;
- why a write can fail at close;
- why the close of a file that was written must be explicit.

The runtime spends flags and scattered `Flush()` calls on them. Go has none of this. A write is a
write, and a program that wants a buffer says so.

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
  program asks for: Go's `bufio.Writer`, Rust's `BufWriter`.
- **Writes are buffered, with flush rules:** the C stdio tradition. Python, Perl and Lua buffer
  whole blocks when output is not a terminal. Each has its own escape hatch: `PYTHONUNBUFFERED`,
  `flush=True`, `$| = 1`, `setvbuf`. "My output appears late when piped" is a perennial question in
  all of them.

Ant follows the standard library of Go (see `stdlib.md`), and the newer runtimes are all in the
first camp.

### Reads

Every runtime above keeps a read buffer where line reads need one. Rust keeps one global buffer
behind `Stdin`, the same shape as Ant's one handle per stream. Read buffering adds no latency as
long as a read returns whatever is available rather than waiting to fill: Python's `read1`, Ruby's
`readpartial`, Rust's `read`, Ant's `readChunk`. Ant's read side already behaves this way.

Go's `os.File` has no line read; a Go program wraps it in a `bufio.Reader` or `bufio.Scanner`. Ant's
`File.readLine` is that wrapper, built in. So Ant stays close to Go by keeping reads buffered and
making writes not.

### What a write costs

Measured on macOS 27.0.1, Apple silicon, 200,000 lines of `line N`:

| Program                                      | Time   | Per line |
| -------------------------------------------- | ------ | -------- |
| Go, `fmt.Fprintf` to an unbuffered file      | 285 ms | 1.4 µs   |
| Go, `fmt.Println` unbuffered, to a pipe      | 73 ms  | 0.37 µs  |
| Go, `fmt.Println` unbuffered, to `/dev/null` | 330 ms | 1.6 µs   |
| Go, `bufio.Writer`, to a pipe                | 15 ms  | 0.08 µs  |
| Ant, `File.writeLine` loop to a file, today  | 46 ms  | 0.23 µs  |
| Ant, `println` loop, today                   | 49 ms  | 0.25 µs  |
| Ant, the same loop without any write         | 39 ms  | 0.20 µs  |

These are the costs of the proposal:

- An unbuffered `File.writeLine` loop to a file gets about 7× slower: 46 ms becomes about 330 ms.
- A `println` loop gets about 2× slower to a pipe, and up to about 7× slower to a file or
  `/dev/null`.
- A program that writes enough for this to matter wraps its output in `File.buffered`. That is the
  same trade Go makes, and Go programs live with it.

## Design

### Writes

A write to a handle that is not buffered is one write to the OS. It returns once the OS has taken
the bytes, and reports the OS's error. This applies to:

- `File.writeLine`, `writeText`, `writeChunk` and `pipe`;
- `print`, `println`, `eprint` and `eprintln`, which write to `Os.stdout()` and `Os.stderr()`;
- every kind of handle: files, standard streams, connections and `Os.openPipe` ends.

Since nothing is held, output from `print`, `Os.stdout()`, standard error and children comes out in
the order the program wrote it. A prompt shows before the read that follows it.

`File.pipe` copies chunk by chunk:

- It first writes what the source handle had already read ahead.
- Then it reads a chunk and writes it, until the end of the source.
- A source that waits (standard input, a connection, a pipe) streams.
- A regular file is copied in chunks of the read buffer's size.
- When the context ends the copy, what was read has been written.

### Reads

Unchanged. Each handle that reads has one read buffer, shared by all its reads.

- A read returns what is there and waits only while there is nothing.
- A read that the context ended keeps what it had consumed for the next read.
- A file with something read ahead is handed to a child through a copy, as `Cmd.start` already says.

### `File.buffered`

```ant
File.buffered(f: File.Handle, size: int = 65536): File.Handle   // writes held until full, flush or close
File.flush(f: File.Handle): Result[void, Error]                 // writes out a buffered handle; Ok on any other
File.close(f: File.Handle): Result[void, Error]                 // a buffered handle: writes out, then closes f
```

`File.buffered` returns a new handle over `f`:

- **Writes.** A write to the new handle goes into a buffer of `size` bytes. The buffer is written to
  `f` when a write does not fit, at `File.flush`, and at `File.close`. Nothing else writes it out:
  no wait, no read and no exit. A `size` below 1 is `Err`.
- **Reads.** A read of the new handle reads `f`, through `f`'s one read buffer. A buffered
  connection can still `readLine` its replies, after `File.flush`, as with Go's `bufio.ReadWriter`.
- **Errors.** The first error from writing the buffer out is kept. Every later write, flush and
  close of the buffered handle returns it, as Go's `bufio.Writer` does.
- **Close.** `File.close` of the buffered handle writes the buffer out and closes `f`, and reports
  the first error of the two. So one close serves both, and a written file is checked as it is
  today: `try File.close(out)`. On a standard stream the close writes out and leaves the stream
  open, as a close of a standard stream does nothing today. A close after the first does nothing and
  is `Ok`.
- **Goroutines.** One goroutine may write the buffered handle while another reads it, as with any
  handle.
- **Writing past the buffer.** A write to `f` directly, or a `print` while
  `File.buffered(Os.stdout())` holds output, bypasses the buffer. That output can come out before
  what the buffer still holds. The spec says so, as Go's documentation does.
- **Children.** Handing the buffered handle to `Cmd.start` as an output writes the buffer out first,
  then hands over `f`.

`File.flush` of a handle that is not buffered does nothing and is `Ok`. So a function that takes a
handle may flush it without knowing what it is.

### Examples

```ant
// A report of many lines: one buffer, one checked close.
func writeReport(path: string, rows: Array[Row]): Result[void, Error] {
  var out = File.buffered(try File.create(path))
  defer { File.close(out) }
  for row in rows { try File.writeLine(out, format(row)) }
  try File.close(out)
}

// Bulk output to standard output.
var out = File.buffered(Os.stdout())
for i in Iter.range(0, 1_000_000) { try File.writeLine(out, "line ${i}") }
try File.flush(out)

// A line protocol: the request goes out in one write, then the reply is read.
var conn = File.buffered(try Net.dial("localhost:6379"))
try File.writeText(conn, "PING\r\n")
try File.flush(conn)
var reply = try File.readLine(conn)
```

For a whole file, building the text and writing it once is simpler still and as fast:
`File.write(path, Array.join(lines, "\n"))`.

## Spec changes

`stdlib.md`, File:

- Add `File.buffered` to the listing. Reword the `File.flush` and `File.close` comments as above.
- Replace "A write to a connection is flushed as a write to a standard stream is …; `File.flush` is
  for files" with: a write is one write to the OS, and `File.pipe` writes each chunk it reads.
- Narrow the close paragraph ("A write to a file can succeed into the buffer …") to buffered
  handles. A close of a written file is still checked, since a close can fail, but the reason given
  is the close itself.
- Add the `File.buffered` paragraph from the Design.

`stdlib.md`, Os, Streams:

- Drop "`Os.stdout()`, `print` and `println` write to one buffer … flushes it first".
- Drop "Standard error is not buffered, and a write to it … flushes standard output first … The
  write therefore waits while standard output cannot be written".
- In their place: the streams are not buffered, so writes come out in program order, and
  `File.buffered(Os.stdout())` is the way to buffer them.
- `Os.openPipe`: drop "and a write to `write` is flushed".

`stdlib.md`, Cmd:

- Drop "Starting a command flushes standard output first".
- "Each handle given as an output is flushed first" stays, for buffered handles.

`stdlib.md`, Net:

- Drop "A write to a connection is flushed, as `File` says".

`ant-lang.md`, prelude:

- `print`: drop "output is buffered and flushed when the program reads standard input, blocks, or
  exits".
- `eprint`/`eprintln`: drop "which is not buffered, and flush standard output first".

`ant-lang.md`, `defer`:

- The example keeps its explicit `File.close(output)`. The sentence after it says a close can fail
  and a deferred block's value is discarded, without mentioning a buffer.

## Implementation sketch

Not part of the proposal's contract; recorded so the size is visible.

**Removed:**

- `Instance.out` and its `Flush()`.
- The `Instance().Flush()` calls in `sync.go`, `context.go`, `chan.go`, `select.go`, `process.go`,
  `test.go`, `debug.go` and `Handle.read`.
- `Config.WriteThrough` and the `evaluating` check.
- `errWriter`'s flush of standard output.
- The `std`, `conn` and `pipe` flags.
- The flush in `Handle.close`, for handles that are not buffered.

**Changed:**

- `Handle.w` becomes the plain `io.Writer`.
- A buffered handle carries a `*bufio.Writer` over its parent's writer and a pointer to the parent
  for reads and close.
- `File.pipe` becomes a loop over the source's read buffer and `Read`, writing each chunk. It
  replaces `io.Copy` and the `WriteTo`/`ReadFrom` shortcut that caused the bug.

**Hosts:**

- A host that embeds Ant receives each write as it is made.
- A host for which a write is costly, such as the playground posting messages to a page, may buffer
  its own writer. That is the host's choice, not the language's.
- `ant test`, the REPL and `ant dap` capture or forward output as it is written.

## Considered and rejected

### Flush before every wait

The previous draft of this proposal:

- every handle buffered;
- a goroutine's writes written out before it waits, when its goroutine ends, at close, and at the
  end of the run.

It keeps loops fast without a new function. But it is a rule every wait must implement and every
reader of the spec must learn, and it still defers errors to close for every handle. It is the
current design made consistent, not made simple.

### A `Buffer` namespace

`Buffer.new(f)`, `Buffer.writeLine(b, …)` and so on. Ant has no interfaces. So either every write
function gets a copy in `Buffer`, or every function that takes a handle (`File.pipe`, `Cmd.start`, a
`Csv` or `Json` writer) takes `File.Handle | Buffer`. Returning a `File.Handle` avoids both. The
name `Buffer` is better kept for an in-memory byte builder, Go's `bytes.Buffer`, should Ant need
one.

### An option at open

`File.create(path, buffered: true)` reaches files only. Standard output, connections and pipe ends
would each need the option on their own constructor. `File.buffered(f)` works on any handle.

### Line buffering for standard output

Rust buffers standard output by line, and C does on a terminal. That is a third rule: a prompt
without a newline would wait for a flush, which is the problem the flush-before-read rule exists to
fix today.

### Buffer by kind

Files buffered, everything else written through: today's behaviour with the kind fixed at open. It
keeps the deferred close error and a flush rule for files, and it leaves `print` with its own buffer
and rules.

## Non-goals

- Read buffering: unchanged.
- `O_SYNC`, `fsync` and durability: a write hands data to the OS, nothing more.
- A buffered reader: reads are already buffered.
- An in-memory byte builder.

## Open questions

1. **Exit with a buffered handle not flushed.** Go loses the buffered tail when a program returns
   from `main` or calls `os.Exit`. Ant could write out every buffered handle at the end of the run.
   That costs a list of buffered handles per run, and it removes a silent loss that a script is
   likely to hit. It promises more than Go; the alternative is to say plainly that the tail is lost.
2. **Default size.** 64 KB is proposed, the size of a pipe's buffer on Linux and macOS. Go's default
   is 4 KB.
3. **Name.** `File.buffered` is proposed. Alternatives: `File.bufferWrites`, `File.withBuffer`.
4. **Nested buffering.** `File.buffered(File.buffered(f))` is allowed and harmless. Should it return
   the inner handle instead?
