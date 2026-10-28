# Proposal: `ant dap`, a debugger for editors

2026-10-28 · Anton

## Summary

A new command, `ant dap`, serves the Debug Adapter Protocol (DAP) over standard input and standard
output, as `ant lsp` serves the Language Server Protocol. It runs the program in its own process,
under the interpreter, with breakpoints, stepping, pause, goroutines as threads, stack frames,
variables and break on panic. VS Code talks to it with a few lines of configuration. IntelliJ talks
to it through the platform's debugger API and a small DAP client. The language does not change; a
new contract, `spec/dap.md`, defines the command.

```
ant dap                     serve the debug protocol on standard input and standard output
```

## Motivation

- `ant lsp` gives both editors errors, completion and navigation. Running a program is one click in
  both. Seeing why it misbehaves is still `Os.println`.
- A panic trace (section 8) shows where a program died, not the values that got it there.
- Goroutines and channels (section 9) are where print debugging is weakest: output interleaves, and
  a deadlocked program prints nothing. A debugger that lists every goroutine and where it is blocked
  answers that directly.
- DAP is the one protocol both editors speak, so one adapter serves both, and later any other DAP
  client (Neovim, Helix, Zed, Emacs).

## Prior art

- **Delve** (`dlv dap`) is Go's debugger and serves DAP directly, with no separate adapter.
  All-stop: when one goroutine stops, all do; goroutines are DAP threads. Ant follows it on
  both points.
- **debugpy** and **js-debug** run in-process or next to the runtime and add a "just my code"
  filter that skips library frames. Ant's prelude is library code in this sense.
- **Lua and Ruby debuggers** (`local-lua-debugger`, `debug.gem`) hook a tree or bytecode
  interpreter at each line or statement, the approach proposed here.
- **JetBrains.** CLion 2025.3 added DAP debug profiles, but only in CLion. The platform has an
  experimental `dap` extension point with no documentation. LSP4IJ has a DAP client, but
  `intellij-ant` moved off LSP4IJ to the native LSP API. Language plugins that debug in every
  JetBrains IDE implement the XDebugger API (`XDebugProcess`, `XStackFrame`, `XValue`) themselves.

## The command

`ant dap` takes no arguments. It writes nothing to standard output but protocol messages, framed
with `Content-Length` as in `ant lsp`; what it logs goes to standard error. It serves one client
and one debug session, and ends when the client disconnects.

The adapter and the program share one process: `launch` loads the program as `ant run` does and
runs it on the interpreter with the debugger attached. The program's standard output and standard
error become `output` events, because the real standard output carries the protocol.

### Launch

| Attribute     | Meaning                                                          | Default            |
| ------------- | ---------------------------------------------------------------- | ------------------ |
| `program`     | the file to run, as given to `ant run`                           | required           |
| `args`        | the program's arguments (`Os.args`)                              | `[]`               |
| `cwd`         | the working directory                                            | the program's dir  |
| `env`         | environment variables added to the adapter's own                 | `{}`               |
| `stopOnEntry` | stop before the first statement of the first module that runs    | `false`            |

A program that fails to load (a syntax error, a missing import) ends the session with the error in
an `output` event and an exit status of `ant run`'s. The `exited` event carries `ant run`'s exit
status.

`terminate` delivers an interrupt, as Ctrl-C does under `ant run`: a program that asked for
interrupts (`Os.interrupt`) gets the chance to stop cleanly. `disconnect` ends the run at once.

Attach to a running `ant run` is not in this proposal (see Non-goals).

## The runtime model in DAP terms

### Stop points

The debugger stops only between statements: before each statement of every block, including the
body of a function, a lambda, a `go` block, a `defer` block, a `for` body and an `if` or `match`
branch. A breakpoint on a line binds to the first statement that starts on that line. If no
statement starts there, it moves to the next line that has one, in the same body. A line past the
end of every body stays unverified.

A statement with a lambda in it, such as `xs.map { it * 2 }`, has stop points inside the lambda's
body. Step over goes past them; step in stops at the first one.

### Threads

Each live goroutine is a thread. The main goroutine is thread 1, named `main`. Others are numbered
in the order they start and named after their `go` block: `go at server.ant:42`. A goroutine that
ends is reported with a `thread` event.

All-stop: when any goroutine stops (breakpoint, step, panic or pause), every goroutine stops at its
next stop point. A goroutine blocked in a channel operation, `select`, `WaitGroup.wait`, a sleep
or I/O stays where it is. Its stack is shown with the blocking call as its top frame and a
`blocked on receive`-style label. If the operation completes while the program is stopped, the
goroutine stops before it returns to Ant code. `continue` resumes every goroutine, and so does a
step: the stepping goroutine stops at the step's end and the rest stop with it.

Time does not stop. A sleep, a timer or a `Context` deadline that passes while the program is
stopped fires on resume.

### Stack frames

A thread's frames are the frames of its panic trace (section 8, `panic`): one per function or lambda
call and one per module body running, with the same names. Each frame carries the position of the
statement it is at, or of the call it is in. A builtin written in Go appears as a frame with no
source. A function from the prelude has a source the client fetches by `sourceReference`, and is
marked `deemphasize`.

### Scopes and variables

Each frame has three scopes:

- **Locals**: the bindings of the frame's body that are in scope at the current statement, innermost
  block first. A name shadowed by an inner binding is hidden. A binding not yet declared is not
  shown.
- **Closure**: for a lambda, a `go` block or a `defer` block, the bindings of the enclosing bodies
  it can see.
- **Module**: the fields of the frame's module.

Values show as `String.from` would show them, with strings quoted. Records, arrays, maps, sets, enum
variants with a payload, `Result` and `Bytes` expand to their parts; arrays and `Bytes` page
through `indexedVariables`. A function shows its name and parameters.

The type column shows a value's runtime type. `ant run` discards annotations (section 1, Types are
advisory), so the declared type is not available; `ant lsp`'s hover still shows it in the editor.

### Stepping

Stepping is defined on frames, not on stack depth:

- **Step in**: stop at the next stop point on this goroutine, in any frame. It does not follow a
  `go` block into the new goroutine; a breakpoint there does.
- **Step over**: stop at the next stop point on this goroutine whose frame is the current frame or
  one of its callers.
- **Step out**: stop at the next stop point on this goroutine in a caller of the current frame.

A `for` body over a sequence runs inside the sequence's `yield` call, deeper on the stack than the
loop, but its statements belong to the loop's frame. So step over on a `for` stops in the body, and
step over at the end of the body goes to its next iteration. A `defer` block belongs to the frame
that registered it, so step over at the end of a block stops in its deferred blocks.

With "just my code" on (the default), step in does not stop in the prelude, and step out of a
prelude frame stops in the first caller outside it.

### Break on panic

Two exception filters:

| Filter     | Stops at                                                    | Default |
| ---------- | ----------------------------------------------------------- | ------- |
| `uncaught` | a panic no `Panic.catch` on its goroutine will stop         | on      |
| `all`      | every panic, including one `Panic.catch` turns into a value | off     |

The stop happens at the panic's position, before any frame unwinds, so every frame's variables can
still be read. `exceptionInfo` gives the message and the trace as section 8 prints it.

## Capabilities by phase

| Feature                                     | Phase |
| ------------------------------------------- | ----- |
| launch, `stopOnEntry`, output, exit status  | 1     |
| line breakpoints                            | 1     |
| threads, stack, scopes, variables           | 1     |
| continue, pause, step in, over, out         | 1     |
| break on panic (`uncaught`, `all`)          | 1     |
| `terminate`, `disconnect`                   | 1     |
| `evaluate` (watch, hover, debug console)    | 2     |
| conditional and hit-count breakpoints       | 2     |
| logpoints                                   | 2     |
| `setVariable`                               | 3     |
| function breakpoints                        | 3     |

Phase 2 shares one piece: evaluating an expression in a stopped frame. It parses the expression,
resolves its names against the frame's scopes, and runs it on the stopped goroutine. An expression
may call functions, so it can block or panic; a panic in it is the evaluation's error, not the
program's.

## Interpreter changes

Not part of the contract, listed to size the work. The interpreter stays a tree walker; with no
debugger attached, each change costs one predictable branch.

- **Statement hook.** `evalStmts` calls the debugger before each statement when one is attached. All
  stop logic (breakpoints, steps, pause) is behind it.
- **Slot names.** The resolver records, for each slot of a `BlockScope`, its name and declaration
  position. Today a scope keeps only slot counts, so locals cannot be named.
- **Frames with state.** A `Frame` gains its `Env` and its current position. Today it keeps the call
  site only, and the innermost position lives in the trace builder.
- **Goroutine registry.** The instance keeps live goroutines with IDs, start position and state
  (running, blocked on what, stopped). A pause flag joins the `done` check at calls and loop
  back-edges, and blocking builtins check it before they return.
- **Panic hook.** A panic calls the debugger before unwinding, with whether a `Panic.catch` on the
  goroutine will stop it.
- **Program I/O.** Standard output and error go through `Config.Stdout` and `Config.Stderr`
  already, so redirecting them to `output` events needs nothing new.

DAP framing is the same as LSP's, so the adapter reuses the hand-written JSON-RPC layer of
`internal/lsp` and adds no dependency.

## Editors

**VS Code (`vscode-ant`).** In `package.json`, `breakpoints: [{ "language": "ant" }]` and a
`debuggers` entry of type `ant` with the launch attributes above. A `DebugAdapterDescriptorFactory`
starts `ant.path` with `dap`, and only in a trusted workspace, as for the language server. A
`DebugConfigurationProvider` lets F5 debug the active file with no `launch.json`. "Ant: Run File"
gets a "Debug File" twin.

**IntelliJ (`intellij-ant`).** The existing Ant run configuration gets the Debug executor. An
`XDebugProcess` starts `ant dap` and speaks DAP through a small client, either `lsp4j.debug` or
hand-written. An `XLineBreakpointType` covers `.ant` files, and `XStackFrame`, `XValue` and
`XExecutionStack` map frames, variables and goroutines. This runs in every JetBrains IDE that runs
the plugin, unlike CLion's DAP profiles. If the platform's `dap` extension point becomes public and
documented, the client can move to it without changing `ant dap`.

## Spec changes

- New `spec/dap.md`: the command, launch attributes, stop points, threads, frames, scopes, stepping,
  exception filters and capabilities, in the style of `lsp.md`.
- `ant-lang.md`: none. Stopping a program does not change what it computes, except that time passes
  (Threads).
- `spec/README.md` lists `dap.md` with the other tool documents.

## Non-goals

- **Attach** to a program already running under `ant run`. It would mean a debug listener in every
  run. Launch covers the editor case.
- **Remote debugging**, over TCP or in a container.
- **Reverse or time-travel debugging.**
- **Hot reload** of edited code into a stopped program.
- **Debugging the JS backend's output.** A compiled program would need source maps and the browser's
  or Node's debugger. Separate design.
- **Debugging inside the REPL.**
- **Showing declared types** in the variables view (see Scopes and variables).

## Open questions

1. **Standard input.** In-process launch gives the program an empty `Os.stdin()`. A program that
   reads input needs a terminal: `runInTerminal` would start a separate `ant run` that connects back
   to the adapter, which is half of attach. Ship phase 1 without it?
2. **Step granularity.** Statements only, or also stop points at each call in a long pipeline
   (`xs | filter(...) | map(...)`) on one line?
3. **Evaluate side effects.** Hover runs an expression without the user asking. Should hover
   evaluate only names and field paths (`user.address.city`), with calls left to watch and the
   console?
4. **Library frames.** Should "just my code" also skip imported modules outside the workspace, or
   only the prelude?
5. **IntelliJ client.** `lsp4j.debug`, a library already shaped for DAP, or a hand-written client
   with no dependency?
