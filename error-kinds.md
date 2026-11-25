# Proposal: error kinds

2026-11-25 - draft, revised: the three OS kinds only; `Error.Closed` and the context kinds deferred

## Summary

`Error` gains a second field, `kind`, an enumeration that says what went wrong in terms a program
can branch on: `Error.NotExist`, `Error.Exist` or `Error.Permission`. It defaults to `null`, so
every `Error{ message }` literal still builds one, and `Error.wrap` keeps it, so a kind survives the
context that callers add.

```ant
type Error = { message: string, kind: Error.Kind | null = null }
type Error.Kind = Error.NotExist | Error.Exist | Error.Permission   // empty records
```

```ant
let text = File.read(path) | Result.wrapErr("read config")
if text is Err {
  if text.error.kind is Error.NotExist { return Result.ok(defaults) }
  return Result.err(text.error)
}
```

The kinds follow three of Go's sentinels, set where Go's `errors.Is` matches them, so Ant promises
no more than Go does about which OS failures are which. Kinds for a closed handle and for a context
that ended a call were in the earlier draft and are held back (Deferred).

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
  Today that is `File.exists(path)` before the read, which races with a writer. It is also wrong
  without a race: `File.exists` is false "when stat is Err for any reason" (stdlib.md), so a config
  in a directory the user may not search reads as absent, and the program runs on defaults without a
  word.
- Create only if absent: `File.write(path, text, replace: false)` is `Err` when something is there,
  and also when the directory is not writable. The two need different handling. stdlib.md says today
  that "the `Err` does not say why, as no library `Error` does", and sends a program to
  `File.exists` after the failure.
- Permissions: a hardening or setup script reports "run this as root" when an operation was denied,
  and a real error otherwise.

None of the three has a test at the call today. A field review of a rewrite of a Bash hardening
project in Ant (2026-11-19, not kept in the repository) named this the most significant remaining
gap after `File.chown`: its step scripts read optional configs and could not tell a missing file
from one they may not read.

Section 8 says that "an error that code branches on is a type of its own", and that holds for a
program's own errors. It does not help with the library's, which are all `Error`.

## Prior art

- **Go.** Errors are values compared with `errors.Is` against sentinels that a wrapped error still
  matches. Four of them classify OS failures on every platform: `fs.ErrNotExist`, `fs.ErrExist`,
  `fs.ErrPermission` and `fs.ErrClosed` (with `net.ErrClosed` for connections and listeners). A
  `syscall.Errno` matches the first three by its own `Is` method: on Unix `ENOENT` is `ErrNotExist`,
  `EEXIST` and `ENOTEMPTY` are `ErrExist`, `EACCES` and `EPERM` are `ErrPermission`; on Windows
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

type Error.Kind = Error.NotExist | Error.Exist | Error.Permission

type Error.NotExist = {}     // and one empty record for each of the others
```

`Error.Kind` is a union of empty records, which section 4 calls an enumeration, as `Bytes.Order` is.
Its members are types in the `Error` namespace, as `Panic.Frame` is in `Panic`'s and `Reflect.Null`
in `Reflect`'s. `kind` is `null` when the failure has none, as `Cmd`'s `dir` is `null` by default,
and `Error{ message: "..." }` builds such an `Error`, as before (section 4, Field defaults).

A program tests a kind with `is`, and a misspelled member is a compile error. `e.kind` is a field
path, which never narrows (section 7), so a test of `e.kind` is a test and nothing more. Most
programs test one kind and treat the rest alike, and need no more. To narrow, or to have a chain
over the kinds checked as complete, bind the kind to a local first. The field's type is
`Error.Kind | null`, so a complete chain has an arm for `null` beside one for each member:

```ant
let k = r.error.kind
if k is Error.NotExist { ... }
else if k is Error.Exist { ... }
else if k is Error.Permission { ... }
else if k is null { ... }
```

A program may set a library kind on an `Error` it builds, and the kind then claims the library's
meaning: a test double for a file system returns
`Error{ message: "open ${path}: not found", kind: Error.NotExist{} }` for a missing path. A program
cannot add kinds: an error a program branches on in its own terms, such as a user that is not found,
is a type of its own (section 8), as now.

### Adding a kind later

A kind is added when Go has a sentinel for an OS condition every platform reports and a program
needs it; Deferred has the four that were held back. Adding one is a breaking change on purpose, as
adding a node kind to `Html` is (section 11): a chain over every kind and `null`, bound to a local
and with no final `else`, stops checking until it handles the new one. A program that tests some
kinds and treats the rest alike ends its chain with `else`, and is not affected. A failure that had
no kind then has one, so a program does not take `null` to name a class of failure: it means "none
of the kinds", not "some other error".

### The library's kinds

One rule sets a kind on a standard library `Err`: an `Err` that comes from a Go error has the kind
of the first of these sentinels that `errors.Is` matches, and `null` when none does.

| `kind`             | Go sentinel        | On Unix               | Typical calls                                                      |
| ------------------ | ------------------ | --------------------- | ------------------------------------------------------------------ |
| `Error.NotExist`   | `fs.ErrNotExist`   | `ENOENT`              | reading, statting or removing a missing path                       |
| `Error.Exist`      | `fs.ErrExist`      | `EEXIST`, `ENOTEMPTY` | `replace: false` onto a file; removing a directory with entries    |
| `Error.Permission` | `fs.ErrPermission` | `EACCES`, `EPERM`     | a file the user may not read; `chown` as a user; a port below 1024 |

The sets are Go's, and the names say what Go's say, not more: `Error.Exist` also means "a directory
is not empty", since Go matches `ENOTEMPTY` to `fs.ErrExist`, so `File.remove` of a directory with
entries is `Error.Exist`. A path through a file, as `a.txt/b` (`ENOTDIR`), is not `Error.NotExist`.
A path in a directory the user may not search is `Error.Permission` and not `Error.NotExist`, which
is the difference `File.exists` cannot report. On Windows the codes are those Go's `Errno.Is` maps,
so a missing file is `Error.NotExist` there too, and a file locked by another process
(`ERROR_SHARING_VIOLATION`) has no kind. Where Ant reports one of these conditions itself instead of
passing Go's error on, it uses the same kind: `File.copyAll` when `to` exists and is not an empty
directory is `Error.Exist`.

Every other library `Err` has kind `null`:

- a parse error and a bad argument;
- a call that its context ended, whose message is the context's reason, as now, and a call on a
  closed handle or listener (Deferred);
- a network failure Go has no sentinel for, such as a refused connection or one the OS timed out;
- a check Ant makes itself, other than `File.copyAll`'s: `File.readExactly` that ends early, Go's
  `io.ErrUnexpectedEOF`, has none;
- any `Err` on a host without the facility, as the WebAssembly playground's `File` calls are, since
  Go reports those as `ENOSYS`.

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

A kind says what failed, not which call failed, and it travels as far as the error does. If
`loadConfig` also read the files a config includes, its `Err` for a missing include would be
`Error.NotExist` too, and a caller that took that kind to mean "no config" would run on defaults.
Go's `errors.Is` on a wrapped error has the same hazard. So a program tests a kind on the result of
the call it is about, as `loadConfig` does, and a function whose `Err` should not pass a kind on
drops it with `Result.mapErr`, building an `Error` of the message alone.

### Everything else is a record field

`kind` is an ordinary field, so the rest follows from the rules records already have:

- `String.from` prints it, so `Result.unwrap`'s panic and a printed `Err` show it:
  `Err{ error: Error{ message: "open a.txt: no such file or directory", kind: Error.NotExist{} } }`,
  and `kind: null` for an error of no kind.
- `==` compares it, so two errors with the same message and different kinds differ.
- `Json.stringify` writes a record without its tag: every `Error` gains `"kind":null`, and a kind is
  written as `{}`, so the JSON of an `Error` loses which kind it had (Decided).
- `Reflect` sees two fields, the second of a union type; `Reflect.makeRecord` fills `kind` with its
  default when it is left out.
- A script that ends with an `Err` of type `Error` still writes only the message (section 10,
  Scripts).
- `if r is Err && r.error.kind is Error.Permission` tests the kind; it narrows `r`, not the kind.

## Deferred

Four kinds of the earlier draft are held back until a program needs them. Each has a test at the
call today, which the three OS kinds have not, and adding one later breaks only a chain over every
kind (Adding a kind later).

- **A context ended the call:** `Error.Cancelled`, `Error.Timeout` and `Error.Interrupt`, the member
  of the reason of the context that ended a call, with `Http`'s `timeout:` as `Error.Timeout`. At
  the call the message already is the reason and nothing else (section 9, Waiting; stdlib.md, Http),
  so `r.error.message == "timeout"` tells an `Http.get(url, timeout: 2s)` that ran out of time from
  one that failed, and a caller that holds the context asks `Context.reason`. A kind would add one
  thing, surviving `Error.wrap`, for an `Err` that crossed a function which did not return the
  context. Its price is a second rule beside Go's sentinels, since the three follow `Context.reason`
  and not `errors.Is`: an interrupt is a cancellation to Go, and a Go host's own deadline is
  `"cancelled"` to Ant. It also brings two questions of its own. "The context ended" would be three
  tests, since `is` takes no alias. And a script that ends with `Error.Interrupt` would exit with
  status 1, where an interrupt elsewhere ends the run as the signal does (stdlib.md, Os,
  Interrupts). If they are added, they are three, one per reason, and `Error.Timeout` means Ant's
  timer or context: a connection the OS timed out (`ETIMEDOUT`) matches no Go sentinel and keeps
  kind `null`.
- **`Error.Closed`,** a call on a closed handle, listener or connection, Go's `fs.ErrClosed` and
  `net.ErrClosed`. A read of a handle the program closed itself is a bug, not a case to branch on.
  Its one use is the accept loop that another goroutine stops and that should end quietly, and that
  loop has a test today: `Net.accept` takes a context, so the other goroutine cancels it instead of
  closing the listener, and the loop asks `Context.isDone`.

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
- **Go hosts.** `value.ErrorValue(message)` keeps its signature and builds an `Error` of no kind, so
  a host function that returns `value.Err(value.ErrorValue("..."))` is unchanged. A host that builds
  the record itself, `value.NewRecord(value.ErrorTag, []value.Value{msg})`, leaves `kind`
  uninitialized, which is not `null`; it calls `value.ErrorValue` instead.

## Spec changes

- **ant-lang.md, section 4, Tagged unions.** "There is no `kind` field" speaks of a field that tells
  a union's members apart; reword it so it does not read against `Error.kind` (Decided).
- **ant-lang.md, section 8, `Result`.** `Error` is
  `{ message: string, kind: Error.Kind | null = null }` and `Error.Kind` its enumeration; one
  sentence on what a kind is, pointing at stdlib.md for when the library sets each.
- **ant-lang.md, section 8, Wrapping errors.** `Error.wrap` keeps `kind`. "A wrapped error keeps
  only text" becomes "keeps its text and its kind", and "an error that code branches on is a type of
  its own" gains "or, for the library's errors, its `kind`". One sentence says to test a kind on the
  result of the call it is about.
- **ant-lang.md, section 11.** The `Error` and `Error.Kind` type lines and the three members; the
  `Error.wrap` comment; the sentence that names `Panic.Frame` as a type in its namespace names
  `Error.Kind` and its members too.
- **stdlib.md, introduction.** The rule and the table, next to the paragraph on a waiting function's
  `Err`. That `Err` has no kind, and neither has any `Err` on a host without a file system.
- **stdlib.md, File, Not replacing.** "The `Err` does not say why, as no library `Error` does, so a
  program ... checks `File.exists` after it" becomes: the `Err` is `Error.Exist` when something is
  there.

## Implementation

The kinds are set in one place, and the work is in finding every place that turns a Go error into
text before it reaches it.

- **Classification.** `value.ErrorOf(err)` builds the `Error` of a Go error: its message, and the
  kind of the first of the three sentinels that `errors.Is` matches. `errResult(err)` in
  `internal/lib` is `value.Err(value.ErrorOf(err))`. `gaveUp(ctx)` is unchanged: the reason, and no
  kind.
- **Errors flattened to text first.** `File.realPath` (`fmt.Errorf("realpath %s: %s", ...)`),
  `Net.listen` and `Net.dial` (`fmt.Errorf("%s %q: %s", op, addr, err)`) and the `Http` calls build
  a new error from the old one's text, which `errors.Is` cannot see through. They wrap with `%w`, or
  classify before formatting.
- **Ant's own condition.** `File.copyAll`'s check that `to` exists sets `Error.Exist`.
- **The type.** `value.ErrorTag` gains the `kind` field and native defaults, as `CmdTag` has, so
  that `Error{ message }` and `Reflect.makeRecord` fill it; the three member tags are registered on
  `ErrorTag`, as `Cmd.Pipeline` is on `CmdTag`; the prelude declares `Error.Kind` and its members,
  each with a doc comment, and gives `Error` the field.
- **The Go API.** `value.ErrorValue(msg)` keeps its signature and fills `kind` with `value.Null`.
  `value.ErrorOf(err)` is exported, so a host function that returns the `Err` of a Go error
  classifies it by the library's rule; the embedding guide's paragraph on failures and the package
  documentation name it.
- **`Error.wrap`** copies the kind.
- **The checker** needs nothing new: a field with a default is already optional in a literal, and
  `is` already tests a field path and narrows a local.
- **Editors and the npm package** need nothing: `Error` is already highlighted, and the package's
  TypeScript types have no Ant `Error`. `ant doc`, hover and completion follow `prelude.d.ant`.
- **Tests.** Each kind gets a run test of the calls that produce it, the flattening sites among
  them. A test of `Error.Permission` is skipped when it runs as root, as Linux CI containers often
  do.

**Cost in output.** About 575 printed lines on some 180 of the 761 docs pages, about 170
expected-output lines in 38 test files and 6 lines in Go test files print an `Error` record and
change; the `Error{ message: ... }` literals in source, about 95 in tests and 45 in docs, do not.
New docs pages describe `Error.Kind` and its members under `docs/lib/Error/types/`, as `Bytes` and
`Panic` have. Not all of the docs change is mechanical: the `Error` page says errors compare by
message, which a kind makes false.

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
- **Member names as Rust, Python and Deno have them**, `Error.NotFound`, `Error.AlreadyExists`,
  `Error.PermissionDenied`. The kinds keep Go's names, since the rule is Go's sentinels.

## Non-goals

- No chain of causes, and no `errors.As`: a program's own errors stay types of their own.
- No errno numbers or platform codes.
- No kind for every Go error: a refused connection, a DNS failure, a malformed JSON text have none.
- No kind on panics: `Panic` is unchanged.
- `Http` responses with an error status are `Ok` responses, as now; their status is the status.

## Decided

- **The name is `kind`.** "Kind" is already a term of the spec: every value carries a kind (section
  4, Kinds), `is` tests kinds and tags, and section 4 says "there is no `kind` field". A sentence in
  section 4 separates the two. The name stays because it is the word for this elsewhere, as Rust's
  `ErrorKind`, and the others are taken: `code` is `ant check`'s word for a diagnostic's rule, and
  `reason` is `Context.reason`'s, a string.
- **Every field prints.** `String.from` writes `kind: null` for an error of no kind, by the record
  rule that every field prints. The alternative, a rule that `String.from` leaves out a field that
  holds its declared default, would change every record type for the sake of one field. This is the
  proposal's standing cost: most printed errors carry `kind: null`.
- **JSON is unchanged.** `Json.stringify` writes every record without its tag, so `kind` is `{}` and
  a log of errors as JSON cannot tell them apart. The same holds for every enumeration, such as
  `Bytes.Order`, so a fix belongs to `Json.stringify`: writing a member of a union of empty records
  as its name, `"Error.NotExist"`. That is a change of its own; until then a program writes
  `String.from(e.kind)`.
- **`Result.fail` takes no kind.** A program that reuses a library kind writes the literal,
  `Result.err(Error{ message: "...", kind: Error.NotExist{} })`.
- **A program that is not found.** `Cmd.new("nosuch", [])` fails with Go's `exec.ErrNotFound`, which
  is not `fs.ErrNotExist`, so it has no kind, while `Cmd.new("./nosuch", [])` fails with `ENOENT`
  and is `Error.NotExist`: the slash decides, as it does in Go. `Os.which`, null when no program of
  that name is found, is the check before the call.
