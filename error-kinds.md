# Proposal: error kinds

2026-11-20 - draft, revised: `kind` is an enumeration; review findings addressed

## Summary

`Error` gains a second field, `kind`, an enumeration that says what went wrong in terms a program
can branch on: `Error.NotExist`, `Error.Exist`, `Error.Permission`, `Error.Closed`, or the reason a
context ended the call. It defaults to `null`, so every `Error{ message }` literal still builds one,
and `Error.wrap` keeps it, so a kind survives the context that callers add.

```ant
type Error = { message: string, kind: Error.Kind | null = null }
type Error.Kind = Error.NotExist | Error.Exist | Error.Permission | Error.Closed
  | Error.Cancelled | Error.Timeout | Error.Interrupt   // empty records
```

```ant
let text = File.read(path) | Result.wrapErr("read config")
if text is Err {
  if text.error.kind is Error.NotExist { return Result.ok(defaults) }
  return Result.err(text.error)
}
```

Four kinds follow four of Go's sentinels, set where Go's `errors.Is` matches them, so Ant promises
no more than Go does about which OS failures are which. The other three follow the reason of the
context that ended a call, as `Context.reason` already reports it.

## Motivation

The standard library returns `Result[T, Error]`, and `Error` is `{ message: string }`. A program
that must tell one failure from another has only the message text, which is written for people,
names the path, and varies by platform:

```
open /etc/app.conf: no such file or directory      // Linux, macOS
open C:\app.conf: The system cannot find the file specified.   // Windows
```

The cases that come up in every script are few and well known:

- An optional file: read it if it is there, use defaults if it is not, but fail on anything else.
  Today that is `File.exists(path)` before the read, which races with a writer and still cannot tell
  "missing" from "unreadable" when the read fails.
- Create only if absent: `File.write(path, text, replace: false)` is `Err` when something is there,
  and also when the directory is not writable. The two need different handling. stdlib.md says today
  that "the `Err` does not say why, as no library `Error` does", and sends a program to
  `File.exists` after the failure.
- Permissions: a hardening or setup script reports "run this as root" when an operation was denied,
  and a real error otherwise.
- Cancellation that crossed a function: a helper that takes a context and returns `Err` loses the
  context on the way up, so its caller, which only sees the `Err`, cannot tell "timed out" from
  "failed" without the message. `Http.get(url, timeout:)` has no context the caller could ask.
- A server's accept loop: another goroutine closes the listener to stop it, and the loop ends
  quietly on that `Err` and reports any other.

The field review of a rewrite of a Bash hardening project in Ant (2026-11-19) named this the most
significant remaining gap after `File.chown`: "A script can't tell `ENOENT` from `EACCES` cleanly,
and the step scripts need that when reading optional configs."

Section 8 says that "an error that code branches on is a type of its own", and that holds for a
program's own errors. It does not help with the library's, which are all `Error`.

## Prior art

- **Go.** Errors are values compared with `errors.Is` against sentinels that a wrapped error still
  matches. Four of them classify OS failures on every platform: `fs.ErrNotExist`, `fs.ErrExist`,
  `fs.ErrPermission` and `fs.ErrClosed` (with `net.ErrClosed` for connections and listeners). A
  `syscall.Errno` matches them by its own `Is` method: on Unix `ENOENT` is `ErrNotExist`, `EEXIST`
  and `ENOTEMPTY` are `ErrExist`, `EACCES` and `EPERM` are `ErrPermission`; on Windows
  `ERROR_FILE_NOT_FOUND` and `ERROR_PATH_NOT_FOUND` are `ErrNotExist`, and so on. Go has other
  sentinels, such as `fs.ErrInvalid`, `io.ErrUnexpectedEOF` and `exec.ErrNotFound`, that classify
  something else. `context.Canceled` and `context.DeadlineExceeded` say why a context ended.
- **Rust.** `std::io::Error` has `kind(): ErrorKind`, an enum: `NotFound`, `PermissionDenied`,
  `AlreadyExists`, `TimedOut`, `Interrupted`, and about forty more, with `Other` for the rest. Its
  `Interrupted` is `EINTR` and its `TimedOut` is `ETIMEDOUT`, OS conditions, not a person or a timer
  ending the work.
- **Node.js.** A system error has a string `code`: `"ENOENT"`, `"EACCES"`, `"EEXIST"`. Programs
  compare it with `err.code === "ENOENT"`; the names are POSIX's.
- **Python.** `OSError` subclasses by errno: `FileNotFoundError`, `PermissionError`,
  `FileExistsError`, `TimeoutError`, `InterruptedError` (`EINTR`), caught by class.
- **Deno.** `Deno.errors.NotFound`, `PermissionDenied`, `AlreadyExists`, `TimedOut`, checked with
  `instanceof`.

Every one of them names the same handful of cases. Rust, Python and Deno make the set a type, so a
misspelled case does not compile; Node's strings do not catch one.

## Proposal

### The type

```ant
type Error = { message: string, kind: Error.Kind | null = null }

type Error.Kind = Error.NotExist | Error.Exist | Error.Permission | Error.Closed
  | Error.Cancelled | Error.Timeout | Error.Interrupt

type Error.NotExist = {}     // and one empty record for each of the others
```

`Error.Kind` is a union of empty records, which section 4 calls an enumeration, as `Bytes.Order` is;
it is written over several lines with a leading `|`, as `Reflect.Shape` is. Its members are types in
the `Error` namespace, as `Panic.Frame` is in `Panic`'s and `Reflect.Null` in `Reflect`'s. `kind` is
`null` when the failure has none, as `Cmd`'s `dir` is `null` by default, and
`Error{ message: "..." }` builds such an `Error`, as before (section 4, Field defaults).

A program tests a kind with `is`, and a misspelled member is a compile error. `e.kind` is a field
path, which never narrows (section 7), so a test of `e.kind` is a test and nothing more. To narrow,
or to have a chain over the kinds checked as complete, bind the kind to a local first:

```ant
let k = r.error.kind
if k is Error.NotExist { ... }
else if k is Error.Permission { ... }
else { ... }
```

A program may set a library kind on an `Error` it builds, and the kind then claims the library's
meaning: a test double for a file system returns
`Error{ message: "open ${path}: not found", kind: Error.NotExist{} }` for a missing path. A program
cannot add kinds: an error a program branches on in its own terms, such as a user that is not found,
is a type of its own (section 8), as now.

### Adding a kind later

A kind is added when Go has a sentinel for an OS condition every platform reports and a program
needs it. Adding one is a breaking change on purpose, as adding a node kind to `Html` is (section
11): a chain over every kind, bound to a local and with no final `else`, stops checking until it
handles the new one. A program that tests some kinds and treats the rest alike ends its chain with
`else`, and is not affected.

### The library's kinds

Two rules set a kind on a standard library `Err`.

**OS failures.** An `Err` that comes from a Go error has the kind of the first of these sentinels
that `errors.Is` matches, and `null` when none does:

| `kind`             | Go sentinel                     | On Unix               | Typical calls                                                      |
| ------------------ | ------------------------------- | --------------------- | ------------------------------------------------------------------ |
| `Error.NotExist`   | `fs.ErrNotExist`                | `ENOENT`              | reading, statting or removing a missing path                       |
| `Error.Exist`      | `fs.ErrExist`                   | `EEXIST`, `ENOTEMPTY` | `replace: false` onto a file; removing a directory with entries    |
| `Error.Permission` | `fs.ErrPermission`              | `EACCES`, `EPERM`     | a file the user may not read; `chown` as a user; a port below 1024 |
| `Error.Closed`     | `fs.ErrClosed`, `net.ErrClosed` |                       | a call on a closed handle, listener or connection                  |

The sets are Go's, and the names say what Go's say, not more: `Error.Exist` also means "a directory
is not empty", since Go matches `ENOTEMPTY` to `fs.ErrExist`, so `File.remove` of a directory with
entries is `Error.Exist`. A path through a file, as `a.txt/b` (`ENOTDIR`), is not `Error.NotExist`.
On Windows the codes are those Go's `Errno.Is` maps, so a missing file is `Error.NotExist` there
too, and a file locked by another process (`ERROR_SHARING_VIOLATION`) has no kind. Where Ant reports
these conditions itself instead of passing Go's error on, it uses the same kind: a call on a handle
Ant has closed is `Error.Closed`, as is `Net.accept` on a listener closed by `Net.close`, and
`File.copyAll` when `to` exists and is not an empty directory is `Error.Exist`. No other check Ant
makes itself has a kind: `File.readExactly` that ends early, Go's `io.ErrUnexpectedEOF`, has none.

**A context ended the call.** A call that gives up because its context was done (section 9, Waiting)
has the context's reason as its message, as now, and the member of that reason as its kind:
`"cancelled"` is `Error.Cancelled`, `"timeout"` is `Error.Timeout` and `"interrupt"` is
`Error.Interrupt`. The three follow `Context.reason`, Ant's own word for why a context ended, and
not Go's sentinels: an interrupt is a cancellation to Go, and a Go host's own deadline is
`"cancelled"` to Ant. `Http`'s `timeout:` ends the call as a context would, so its `Err` is
`Error.Timeout` as well. `Context.reason` stays a string.

Every other library `Err` has kind `null`: a parse error, a bad argument, a network failure Go has
no sentinel for, such as a refused connection, and any `Err` on a host without the facility, as the
WebAssembly playground's `File` calls are, since Go reports those as `ENOSYS`.

### Wrapping keeps the kind

`Error.wrap(e, context)` returns `Error{ message: "${context}: ${e.message}", kind: e.kind }`, so a
kind survives `Result.wrapErr` stages and `try` as far as the error travels. `Result.fail(message)`
is `Result.err(Error{ message })`, of no kind. `Result.mapErr` is unchanged: a function that maps an
`Error` to another error decides what to keep.

```ant
func loadConfig(path: string): Result[Config, Error] {
  let text = File.read(path) | Result.wrapErr("load ${path}")
  if text is Err && text.error.kind is Error.NotExist {
    return Result.ok(Config{})
  }
  let json = try (Json.parse(try text) | Result.wrapErr("parse ${path}"))
  Result.ok(Config.fromJson(json))
}
```

### Everything else is a record field

`kind` is an ordinary field, so the rest follows from the rules records already have:

- `String.from` prints it, so `Result.unwrap`'s panic and a printed `Err` show it:
  `Err{ error: Error{ message: "open a.txt: no such file or directory", kind: Error.NotExist{} } }`,
  and `kind: null` for an error of no kind.
- `==` compares it, so two errors with the same message and different kinds differ.
- `Json.stringify` writes a record without its tag: every `Error` gains `"kind":null`, and a kind is
  written as `{}`, so the JSON of an `Error` loses which kind it had (Open questions).
- `Reflect` sees two fields, the second of a union type; `Reflect.makeRecord` fills `kind` with its
  default when it is left out.
- A script that ends with an `Err` of type `Error` still writes only the message (section 10,
  Scripts).
- `if r is Err && r.error.kind is Error.Permission` tests the kind; it narrows `r`, not the kind.

## Compatibility

Programs that build and test errors keep working, with these exceptions:

- **Printed errors change.** Every `Error` that a program prints, or that `Result.unwrap` panics
  with, shows `kind`.
- **JSON of errors changes.** Every `Error` written with `Json.stringify` gains a `"kind"` member.
- **Equality can change.** `==` between a library `Err`'s error and a hand-written
  `Error{ message: "..." }` with the same message is false when the library's has a kind.
- **Spreading an `Error` into a program's own type** of the shape `{ message: string }` stops
  checking, since the spread would copy `kind`, which the target lacks; under `ant run` it panics.
  Such a program names the field instead: `Mine{ message: e.message }`.

## Spec changes

- **ant-lang.md, section 4, Tagged unions.** "There is no `kind` field" refers to unions in general;
  reword it so it does not read against `Error.kind`, or settle Open question 1 another way.
- **ant-lang.md, section 8, `Result`.** `Error` is
  `{ message: string, kind: Error.Kind | null = null }` and `Error.Kind` its enumeration; one
  sentence on what a kind is, pointing at stdlib.md for when the library sets each.
- **ant-lang.md, section 8, Wrapping errors.** `Error.wrap` keeps `kind`. "A wrapped error keeps
  only text" becomes "keeps its text and its kind", and "an error that code branches on is a type of
  its own" gains "or, for the library's errors, its `kind`".
- **ant-lang.md, section 9, Waiting.** "A `Result` is `Err` with the context's reason as its
  message" gains "and the reason's member as its `kind`".
- **ant-lang.md, section 11.** The `Error` and `Error.Kind` type lines and the seven members; the
  `Error.wrap` comment; the sentence that names `Panic.Frame` as a type in its namespace names
  `Error.Kind` and its members too.
- **stdlib.md, introduction.** The two rules and the table, next to the paragraph on a waiting
  function's `Err`, and the host without a file system, whose `Err` values have no kind.
- **stdlib.md, File, Not replacing.** "The `Err` does not say why, as no library `Error` does, so a
  program ... checks `File.exists` after it" becomes: the `Err` is `Error.Exist` when something is
  there.
- **stdlib.md, Http.** The `timeout:` paragraphs: the `Err` is `Error.Timeout`.
- **stdlib.md, Net.** `Net.accept` on a closed listener is `Error.Closed`.

## Implementation

The kinds are set in a few places, and the work is in finding every place that turns a Go error into
text before it reaches them.

- **Classification.** `errResult(err)` in `internal/lib` matches `err` against the four sentinels
  with `errors.Is` and builds the `Error` with its kind. `gaveUp(ctx)` sets the member of the
  context's reason, and `Http`'s timeout goes through it.
- **Errors flattened to text first.** `File.realPath` (`fmt.Errorf("realpath %s: %s", ...)`),
  `Net.listen` and `Net.dial` (`fmt.Errorf("%s %q: %s", op, addr, err)`) and the `Http` calls build
  a new error from the old one's text, which `errors.Is` cannot see through. They wrap with `%w`, or
  classify before formatting.
- **Ant's own conditions.** The closed-handle error (`errClosed`) and the closed-listener error
  (`errListenerClosed`) become values that match `fs.ErrClosed` and `net.ErrClosed`, or set
  `Error.Closed` where they are built; `File.copyAll`'s check sets `Error.Exist`.
- **The type.** `value.ErrorTag` gains the `kind` field and native defaults, as `CmdTag` has, so
  that `Error{ message }` and `Reflect.makeRecord` fill it; the seven member tags are registered on
  `ErrorTag`, as `Cmd.Pipeline` is on `CmdTag`; `value.ErrorValue` takes a kind; the prelude
  declares `Error.Kind` and its members, each with a doc comment, and gives `Error` the field.
- **`Error.wrap`** copies the kind.
- **The checker** needs nothing new: a field with a default is already optional in a literal, and
  `is` already tests a field path and narrows a local.
- **Editors and the npm package** need nothing: `Error` is already highlighted, and the package's
  TypeScript types have no Ant `Error`. `ant doc`, hover and completion follow `prelude.d.ant`.
- **Tests.** Each kind gets a run test of the calls that produce it. A test of `Error.Permission` is
  skipped when it runs as root, as Linux CI containers often do.

**Cost in output.** About 575 printed lines on the docs pages and about 214 in 57 test files print
an `Error` record and change, and 7 lines in Go test files; the `Error{ message: ... }` literals in
source, about 80, do not. New docs pages describe `Error.Kind` and its members under
`docs/lib/Error/types/`, as `Bytes` and `Panic` have. Not all of the docs change is mechanical: the
`Error` page says errors compare by message, which a kind makes false.

## Alternatives considered

- **Distinct error types in the library's signatures**, such as
  `File.read(path): Result[string, File.NotExist | File.Denied | Error]`. Checked, and `is` narrows
  it, but every signature grows, and since type arguments are invariant, `try File.read(p)` in a
  function returning `Result[T, Error]` stops compiling. It breaks most programs to serve a few.
- **Predicates over a hidden kind**, `File.isNotExist(e)` as Go's `os.IsNotExist`. A record would
  carry state that `==`, `String.from`, `Json.stringify` and `Reflect` do not see, which no other
  Ant value does. A field is the same information, visible.
- **A cause chain**, a `cause: Error | null` field with an `Error.is` that walks it, as Go's
  `errors.Is` walks `Unwrap`. Declined in the error wrapping proposal (2026-10-25): wrapping keeps
  text, and an error that code branches on is a type of its own. A kind is the part of a cause a
  program actually tests, without the chain.
- **Errno names**, `"ENOENT"` and `"EACCES"` as Node has them. Unix-specific, more of them than a
  program tests, and not what Go exposes; a few kinds that Go defines on every platform are the
  portable subset.
- **A string**, `kind: string = ""` with `"not-exist"` and the others, as Node's `err.code`.
  Programs could add kinds of their own, and JSON would keep the kind. But
  `r.error.kind == "not-exists"` is never true and nothing says so, editors cannot complete a kind,
  and a chain over the kinds cannot be checked for completeness. The first draft of this proposal
  had it; the enumeration replaced it because the library's kinds are a closed set, which is what
  section 4's enumerations are for.
- **Named constants over a string**, `Error.NOT_EXIST` for `"not-exist"`. A typo in the constant's
  name is caught, but not one in a string written directly, and each kind has two spellings.
- **`Context.isDone(ctx)` instead of context kinds.** A caller that holds the context can ask it why
  the call ended, today. It cannot once the `Err` has crossed a function that did not return the
  context, or for `Http`'s `timeout:`, which has no context to ask; see Open question 2.
- **Member names as Rust, Python and Deno have them**, `Error.NotFound`, `Error.AlreadyExists`,
  `Error.PermissionDenied`. The four OS kinds keep Go's names, since the rule is Go's sentinels; the
  three context kinds take `Context.reason`'s words, Ant's spelling `Cancelled` included. Rust's and
  Python's `Interrupted` is `EINTR`, an OS condition, where Ant's `Interrupt` is a person pressing
  Ctrl-C.

## Non-goals

- No chain of causes, and no `errors.As`: a program's own errors stay types of their own.
- No errno numbers or platform codes.
- No kind for every Go error: a refused connection, a DNS failure, a malformed JSON text have none.
- No kind on panics: `Panic` is unchanged.
- `Http` responses with an error status are `Ok` responses, as now; their status is the status.

## Open questions

1. **The name.** "Kind" is already a term of the spec: every value carries a kind (section 4,
   Kinds), `is` tests kinds and tags, and section 4 says "there is no `kind` field". `Error.kind`
   and `Error.Kind` beside "a kind is what the runtime sees" will confuse a reader of the spec and
   of hover docs. Keeping it means a sentence in section 4 that separates the two. The other names:
   `code` is `ant check`'s word for a diagnostic's rule; `reason` is `Context.reason`'s word, which
   the three context members already follow, and reads well as `e.reason is Error.NotExist`.
2. **One context kind or three.** Three members make "the context ended" three tests,
   `k is Error.Cancelled || k is Error.Timeout || k is Error.Interrupt`, since `is` takes no alias.
   One member, `Error.Cancelled` for any context end with the reason in the message and in
   `Context.reason`, is leaner and still serves the case of an `Err` that crossed a function. Three
   tell a timeout from a person's cancel without parsing the message.
3. **`Error.Closed`.** Its use is the accept loop that ends when another goroutine closes the
   listener; a read of a handle the program closed itself is a bug, not a case to branch on. It
   could be left out until a program needs it, per the lean rule.
4. **JSON loses the kind.** `Json.stringify` writes every record without its tag, so `kind` is `{}`
   and a log of errors as JSON cannot tell them apart. The same holds for every enumeration, such as
   `Bytes.Order`, so the fix, if any, is in `Json.stringify`: writing a member of a union of empty
   records as its name, `"Error.NotExist"`. That is a change of its own; until then a program writes
   `String.from(e.kind)`.
5. **A script that ends with `Error.Interrupt`.** A script that returns the `Err` of an interrupted
   call prints `interrupt` and exits with status 1, where an interrupt elsewhere ends the run as the
   signal does (stdlib.md, Os, Interrupts). It could exit as an interrupt instead.
6. **Printing an error of no kind.** Printing the field in full keeps the record rule that every
   field prints, at the cost of `kind: null` in most printed errors. The alternative is a rule that
   `String.from` leaves out a field that holds its declared default, for every record type, which is
   a larger change to make on its own, if at all.
7. **`Result.fail` with a kind.** `Result.fail(message, kind: Error.NotExist{})` would save the
   literal in a program that reuses the library's kinds. Left out until programs need it.

## Decided

- **A program that is not found.** `Cmd.new("nosuch", [])` fails with Go's `exec.ErrNotFound`, which
  is not `fs.ErrNotExist`, so it has no kind, while `Cmd.new("./nosuch", [])` fails with `ENOENT`
  and is `Error.NotExist`: the slash decides, as it does in Go. `Os.which`, null when no program of
  that name is found, is the check before the call.
