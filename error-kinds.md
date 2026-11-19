# Proposal: error kinds

2026-11-20 - draft

## Summary

`Error` gains a second field, `kind`, a string that says what went wrong in terms a program can
branch on: `"not-exist"`, `"exist"`, `"permission"`, `"closed"`, or the reason of a context that
ended the call. It defaults to `""`, so every `Error{ message }` literal still builds one, and
`Error.wrap` keeps it, so a kind survives the context that callers add.

```ant
type Error = { message: string, kind: string = "" }
```

```ant
let text = File.read(path) | Result.wrapErr("read config")
if text is Err {
  if text.error.kind == "not-exist" { return Result.ok(defaults) }
  return Result.err(text.error)
}
```

The standard library sets a kind exactly where Go's `errors.Is` matches one of a few sentinels, so
Ant promises no more than Go does about which failures are which.

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
  and also when the directory is not writable. The two need different handling.
- Permissions: a hardening or setup script reports "run this as root" when an operation was denied,
  and a real error otherwise.
- Cancellation: a loop that calls a waiting function with a context wants to stop quietly when the
  context timed out or was cancelled, and report every other `Err`.

The field review of a rewrite of a Bash hardening project in Ant (2026-11-19) named this the most
significant remaining gap after `File.chown`: "A script can't tell `ENOENT` from `EACCES` cleanly,
and the step scripts need that when reading optional configs."

Section 8 says that "an error that code branches on is a type of its own", and that holds for a
program's own errors. It does not help with the library's, which are all `Error`.

## Prior art

- **Go.** Errors are values compared with `errors.Is` against sentinels that a wrapped error still
  matches: `fs.ErrNotExist`, `fs.ErrExist`, `fs.ErrPermission`, `fs.ErrClosed`, `net.ErrClosed`,
  `context.Canceled`, `context.DeadlineExceeded`, `os.ErrDeadlineExceeded`. A `syscall.Errno`
  matches the `fs` sentinels by its own `Is` method, so `ENOENT` and Windows' `ERROR_FILE_NOT_FOUND`
  are both `fs.ErrNotExist`. The older `os.IsNotExist(err)` predicates do the same without
  unwrapping.
- **Rust.** `std::io::Error` has `kind(): ErrorKind`, an enum: `NotFound`, `PermissionDenied`,
  `AlreadyExists`, `TimedOut`, `Interrupted`, and about forty more, with `Other` for the rest.
- **Node.js.** A system error has a string `code`: `"ENOENT"`, `"EACCES"`, `"EEXIST"`. Programs
  compare it with `err.code === "ENOENT"`; the names are POSIX's.
- **Python.** `OSError` subclasses by errno: `FileNotFoundError`, `PermissionError`,
  `FileExistsError`, `TimeoutError`, `InterruptedError`, caught by class.
- **Deno.** `Deno.errors.NotFound`, `PermissionDenied`, `AlreadyExists`, `TimedOut`, checked with
  `instanceof`.

Every one of them names the same handful of cases. Go names the fewest, and a kind for each of its
sentinels covers the scripts above.

## Proposal

### The type

```ant
type Error = { message: string, kind: string = "" }
```

`kind` is a string that names the failure for a program to compare, and `""` when it has no kind.
`Error{ message: "..." }` builds an `Error` of no kind, as before (section 4, Field defaults). Any
string is a kind: a program may set one on its own errors, with the meaning the library gives it or
one of its own, as `Error{ message: "no user ${name}", kind: "not-exist" }`.

### The library's kinds

A standard library `Err` has a kind exactly when the Go error behind it matches one of these:

| `kind`         | When                                                      | Go                                     |
| -------------- | --------------------------------------------------------- | -------------------------------------- |
| `"not-exist"`  | the file, directory, or link named does not exist         | `errors.Is(err, fs.ErrNotExist)`       |
| `"exist"`      | something is already where the call would create          | `errors.Is(err, fs.ErrExist)`          |
| `"permission"` | the OS denied the operation                               | `errors.Is(err, fs.ErrPermission)`     |
| `"closed"`     | the handle or connection was closed                       | `fs.ErrClosed`, `net.ErrClosed`        |
| the reason     | the call gave up because its context was done (section 9) | `context.Canceled`, `DeadlineExceeded` |

The last row's kinds are the context reasons `Context.reason` already returns: `"cancelled"`,
`"timeout"` and `"interrupt"`. Such an `Err` has the reason as both its message and its kind, so a
caller tests `r.error.kind == "timeout"` whatever context the message gained on the way up.

Every other library `Err` has kind `""`: a parse error, a bad argument, a network failure Go has no
sentinel for, such as a refused connection. The checks Ant makes itself that mirror a Go sentinel
use its kind: a read of a closed handle is `"closed"`, as Go's `fs.ErrClosed` is, and `File.copyAll`
into a directory with entries is `"exist"`. Which platform errors count as which is Go's: on
Windows, as on Unix, a missing file is `"not-exist"` because Go's `errors.Is` says so.

### Wrapping keeps the kind

`Error.wrap(e, context)` returns `Error{ message: "${context}: ${e.message}", kind: e.kind }`, so a
kind survives `Result.wrapErr` stages and `try` as far as the error travels. `Result.fail(message)`
is `Result.err(Error{ message })`, of no kind. `Result.mapErr` is unchanged: a function that maps an
`Error` to another error decides what to keep.

```ant
func loadConfig(path: string): Result[Config, Error] {
  let text = File.read(path) | Result.wrapErr("load ${path}")
  if text is Err && text.error.kind == "not-exist" {
    return Result.ok(Config{})
  }
  let json = try (Json.parse(try text) | Result.wrapErr("parse ${path}"))
  Result.ok(Config.fromJson(json))
}
```

### Everything else is a record field

`kind` is an ordinary field, so the rest follows from the rules records already have:

- `String.from` prints it, so `Result.unwrap`'s panic and a printed `Err` show it:
  `Err{ error: Error{ message: "open a.txt: no such file or directory", kind: "not-exist" } }`.
- `==` compares it, so two errors with the same message and different kinds differ.
- `Json.stringify` writes it, and `Reflect` sees two fields.
- A script that ends with an `Err` of type `Error` still writes only the message (section 10,
  Scripts).
- Narrowing reads it as any field: `if r is Err && r.error.kind == "permission"`.

## Spec changes

- **ant-lang.md, section 8, `Result`.** `Error` is `{ message: string, kind: string = "" }`; one
  sentence on what a kind is, pointing at stdlib.md for the library's kinds.
- **ant-lang.md, section 8, Wrapping errors.** `Error.wrap` keeps `kind`. The sentence "A wrapped
  error keeps only text" becomes "keeps its text and its kind", and the remark that an error code
  branches on is a type of its own gains "or, for the library's errors, its `kind`".
- **ant-lang.md, section 11.** The `Error` type line and the `Error.wrap` comment.
- **stdlib.md, introduction.** The kinds table and the rule that a library `Err` has a kind exactly
  when Go's `errors.Is` matches, next to the paragraph on a waiting function's `Err`, which gains
  "with the reason as its kind as well".
- **stdlib.md, File and Cmd.** No per-function text: the rule covers them. The `File.exists`
  paragraph can point to `"not-exist"` for the read-then-check case.

## Implementation

Small and central. In `internal/lib`, `errResult(err)` classifies `err` with `errors.Is` against the
four sentinels and builds the `Error` with its kind; `gaveUp(ctx)` sets the context's reason as the
kind; the handle errors Ant makes itself (`errClosed`) and `copyAll`'s non-empty check set theirs.
`value.ErrorValue` takes a kind, and the prelude's `Error` type gains the field with its default.
`Error.wrap` copies the kind. The checker needs nothing new: a field with a default is already
optional in a literal.

The cost is in printed output. Every docs example and test that prints an `Error` record shows the
new field: about 616 lines on 184 docs pages and 254 lines in `tests/`. Most become
`Error{ message: "...", kind: "" }`; those that print a missing file or a cancelled context show
their kind. The change is mechanical, and the docs-update workflow carries it.

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
- **`kind: string | null`, null for none.** Every test of a kind is an `==` against a string, which
  `""` answers as well as `null`, and `""` keeps the type a plain `string`.
- **Named constants**, `Error.NOT_EXIST` for `"not-exist"`, so a typo is a compile error. They add a
  second spelling of each kind. The strings are few and appear in the docs of every function that
  can return them; see Open questions.

## Non-goals

- No chain of causes, and no `errors.As`: a program's own errors stay types of their own.
- No errno numbers or platform codes.
- No kind for every Go error: a refused connection, a DNS failure, a malformed JSON text have none.
  A kind is added when Go has a sentinel for it and a program needs it.
- No kind on panics: `Panic` is unchanged.
- `Http` responses with an error status are `Ok` responses, as now; their status is the status.

## Open questions

1. **The name.** `kind` (Rust's `ErrorKind`) or `code` (Node's `err.code`). `code` is already the
   word for `ant check`'s diagnostic codes, so `kind` avoids two meanings.
2. **Printing an error of no kind.** Printing the field in full keeps the record rule that every
   field prints, at the cost of `kind: ""` in most printed errors. The alternative is a rule that
   `String.from` leaves out a field that holds its declared default, for every record type, which is
   a larger change to make on its own, if at all.
3. **A program that is not found.** Go's `exec.ErrNotFound`, for `Cmd.new("nosuch", [])`, is not
   `fs.ErrNotExist`, so under the rule above it has no kind. A `"not-exist"` there would be
   convenient and is not what Go says. A separate `"not-found"` is another kind to learn.
4. **Typos.** `r.error.kind == "not-exists"` is never true and no tool says so. `ant check` could
   warn when `.kind` of an `Error` is compared with a string that no library function returns; that
   is a checker rule tied to a library table, and is left out of this proposal.
5. **`Result.fail` with a kind.** `Result.fail(message, kind: "not-exist")` would save the literal
   in a program that reuses the library's kinds. Left out until programs need it.
