# Proposal: error wrapping

2026-10-25 · Anton

## Summary

Add two library functions, `Error.wrap` and `Result.wrapErr`, that put a line of context in front of an `Error`'s message, joined with `: ` as Go's `fmt.Errorf("...: %w")` does. `Error` stays `{ message: string }`: there is no cause field, no chain to walk, and no new syntax. A script that ends on an `Err` holding an `Error` writes the message alone to standard error, not the record.

## Motivation

The library's `Error` is a record with one field, and most messages come straight from the Go host. Some carry their own context and some carry none:

```
open /etc/app.json: no such file or directory
unexpected end of JSON input
Get "http://127.0.0.1:1/": dial tcp 127.0.0.1:1: connect: connection refused
```

Take the loader from section 8:

```ant
func loadConfig(path: string): Result[Config, Error] {
  var text = try File.read(path)
  var json = try Json.parse(text)
  Result.ok(Config.fromJson(json))
}
```

A script that loads three config files and hits a truncated one exits with status 1 and prints:

```
Error{ message: "unexpected end of JSON input" }
```

The output does not say which file failed or what the program was doing. Two things cause this:

1. **Adding context is long.** It is possible today, but each call site writes it out in full:

   ```ant
   var json = try (Json.parse(text) | Result.mapErr({ Error{ message: `parse ${path}: ${x.message}` } }))
   ```

   Few programs write this at every `try`, so most errors reach the top with no context.
2. **There is no convention.** The spec does not say how a message is phrased or joined, so each program picks its own separator and capitalization, and the joined messages do not read as one sentence.

The script exit adds a third, smaller problem. `String.from` renders the error as a record literal, so the user sees `Error{ message: "..." }` where a command-line tool would print the text.

## Prior art

| Language | Mechanism | What a wrapped error keeps |
| --- | --- | --- |
| Go | `fmt.Errorf("load config: %w", err)`, `errors.Unwrap`, `errors.Is`, `errors.As`, `errors.Join` | The message text, joined with `: `, and the cause as a value for `Is` and `As` |
| Rust | `std::error::Error::source()`; anyhow `.context("...")` and `.with_context(\|\| ...)`; thiserror `#[source]` | A cause chain; anyhow's `{:#}` joins it on one line with `: `, and `{:?}` prints it as "Caused by:" lines |
| Gleam | `Result` values, `result.map_error`; the `snag` package adds `snag.context` | A list of context lines in front of the original message |
| Zig | Error sets with no payload; error return traces in debug builds | Only the error name; the trace is built by the runtime |
| Java, Kotlin, C# | `new Exception(msg, cause)`, `InnerException` | A cause object, printed as "Caused by:" in the stack trace |
| Python | `raise X from e` (PEP 3134), `__cause__`, `__context__`, exception notes (3.11) | A cause object, plus extra lines of notes |
| JavaScript | `new Error(msg, { cause })` (ES2022) | A cause of any type; most tools print only the outer message |
| Elixir, OCaml | `{:error, reason}` and `result`; no standard wrapping | Whatever tuple or variant the program builds by hand |

Three lessons shape the proposal:

- **The message is what users read.** In Go, most `%w` wrapping is read by people, not by code. `errors.Is` and `errors.As` matter only when an error has an identity to match: a sentinel like `fs.ErrNotExist` or a typed error. Ant's library defines no sentinel errors and no error kinds, so a cause chain of library errors would be text with extra structure that nothing standard can match against. A program can keep its own sentinel `Error` and compare it with `Record.same`, but a program that matches errors is better served by a typed error (see Typed errors).
- **Context must cost one call.** anyhow's `.context()` and Gleam's `snag.context` became the norm because they cost one call. Go's `if err != nil { return fmt.Errorf(...) }` costs three lines, and Go code often skips it.
- **Typed errors carry structure in their own types.** Rust's thiserror, Swift's typed `throws`, and Ant's own tagged unions (section 8) put the fields that code branches on into a type the author declares. A library wrapping helper does not need to carry them.

## Design

### Two functions

```ant
Error.wrap(e: Error, context: string): Error                                  // Error{ message: `${context}: ${e.message}` }; e is untouched
Result.wrapErr[T](r: Result[T, Error], context: string): Result[T, Error]     // r when Ok; an Err holding Error.wrap(r.error, context) otherwise
```

`Error.wrap` returns a fresh record and leaves `e` untouched, because records are mutable and shared (section 4, Records). `Result.wrapErr` returns `r` itself on `Ok` and a fresh `Err` on `Err`. Both are functions declared on prelude types, as `Result.ok` is (section 6), so `Error` names the record type in a type and its functions in an expression. `Error{ message: "..." }` literals keep working.

The loader becomes:

```ant
func loadConfig(path: string): Result[Config, Error] {
  var text = try File.read(path)
  var json = try (Json.parse(text) | Result.wrapErr(`parse ${path}`))
  Result.ok(Config.fromJson(json))
}

var config = try (loadConfig(path) | Result.wrapErr("load config"))
```

The script exits with:

```
load config: parse /etc/app.json: unexpected end of JSON input
```

`try` binds tighter than the pipe (section 8), so a `wrapErr` stage needs parentheses. That is the same rule as `try (xs | Array.map(parse) | Result.all())`, and it needs no new syntax.

### The message convention

A new paragraph in section 8 and in the introduction of `stdlib.md` states the convention that the joined messages depend on. Go's standard library follows the same one:

- A message is lowercase and has no closing punctuation, because it may end up in the middle of a longer message.
- Context names the operation and the argument that matters: `parse /etc/app.json`, not `failed to parse config file!`. Words such as "error" and "failed" add nothing, since the text is already an error.
- The function that knows an argument adds it. `File.read` already names its path, so the caller does not repeat it. A caller adds what it was doing, which the callee cannot know.
- An error that comes from a context names no target. A blocking call whose context ends returns the context's reason, such as `cancelled` or `timeout` (section 9, Waiting), so a caller that wants the URL or path in the message adds it.

### Typed errors

Wrapping is for `Error`, the error that people read. An error that code branches on stays a type the author declares, and a cause is an ordinary field:

```ant
type BadConfig = { path: string, cause: ParseError }
type LoadError = BadConfig | Error

func loadConfig(path: string): Result[Config, LoadError] {
  var text = try File.read(path)
  var parsed = parseConfig(text)
  if parsed is Err { return Result.err(BadConfig{ path, cause: parsed.error }) }
  Result.ok(parsed.value)
}
```

`Result.wrapErr` does not take a `Result[T, ParseError]`, because a typed error has no message to join. To turn a typed error into text, the program uses `Result.mapErr` with a function that describes it, as it does today.

### The script exit

Section 10 says that a script ending on an `Err` writes `String.from` of the error to standard error. The proposal changes one case: an error with the `Error` tag writes only its `message`. Every other error type is still rendered with `String.from`, which keeps the structure of a typed error. Exit status 1 and the panic path through `Result.unwrap` do not change.

This change is separate from the two functions and can be accepted or declined alone.

## Alternatives considered

### A cause field: `{ message: string, cause: Error | null }`

The sketch in `ant/TODO.md` adds a cause and the helpers `Error.root` and `Error.chain`. It is declined:

- **It breaks every literal.** A record literal must list all of its type's fields (section 4), so each `Error{ message: "x" }` becomes `Error{ message: "x", cause: null }`. The spec has 3 such literals and the `ant` tests and examples have 121. Avoiding this needs field defaults, which would change the literal rule for every record type to serve one type.
- **It doubles the output.** `String.from` and `==` follow every field, so a wrapped error renders its full chain as nested records, and the readable form needs a custom renderer.
- **Little reads the chain.** `Error.root(e).message` holds the same text as the end of the joined message. A chain pays off when the root has an identity to match. The library defines no sentinels or kinds (see Non-goals). A program's own sentinel, compared with `Record.same`, would survive in a chain but not through `Error.wrap`, which builds a fresh record. Such a program declares a typed error instead.
- **A typed cause is a separate choice.** `cause: {} | null` would accept any record, including a program's own error, and `is` recovers its fields after a `null` test (section 5, `is`). The proposal leaves typed causes out on purpose: the readable form would render an arbitrary record with `String.from` inside a message, an error type that is not a record, such as a named primitive over `string`, does not fit `{}`, and a typed cause already has a home as a field of the typed error that wraps it.

### A `try` clause

`try Json.parse(text) else "parse config"` or a similar form would add context without parentheses. It is deferred. `else` after `try` reads as a fallback value, as in Swift's `guard` or Zig's `catch`. A new keyword costs more than the parentheses it saves. If `wrapErr` stages turn out to be in most `try` expressions, a syntax proposal can follow and point to that usage.

### A function-level handler

Go's 2018 draft had `handle err { ... }`, and Go programs use a deferred closure over a named result to wrap every error that leaves a function. Ant's `defer` cannot see the result, and giving it that access would change section 8 for one use. A function that wants one prefix on every error moves its body into a helper and wraps the helper's result once, as `loadConfig` does above.

### `Result.context`

This is anyhow's and snag's name for the function. It is declined because Ant already has a `Context` type for cancellation, and every blocking call takes a `ctx`. `wrapErr` pairs with `mapErr` and does not suggest a `Context`.

## Spec changes

| Where | Change |
| --- | --- |
| `spec/ant-lang.md` §8, `Result` | A paragraph on adding context with `Result.wrapErr`, the `loadConfig` example with one wrapped stage, and the message convention |
| §8, `try` | One sentence: a `wrapErr` stage is parenthesized with the call, like any pipeline under `try` |
| §10, Scripts | An `Err` holding an `Error` writes its `message` and a newline; any other error is written with `String.from` as now |
| §11, Prelude | `Error`'s comment points to its functions |
| §11, Result and Record | Renamed to Result, Error, and Record; adds the two signatures |
| `stdlib.md` introduction | The message convention, and that library messages name the path, address, or URL they concern, except an error that is a context's reason |
| `stdlib/prelude.ant` | Doc comments for the two functions, generated from the spec text |

In the Go interpreter both functions are a few lines in `internal/lib`, next to `Result.mapErr`, and the script exit adds one tag check where the runtime renders the final error. The JS backend needs the same code.

## Non-goals

- **Error kinds.** Programs cannot tell "not found" from "permission denied" except by matching the host's message text, which differs between operating systems and would differ again under the JS backend. That needs a stable kind on library errors and is a separate proposal. Because `Error` stays flat, `wrap` would copy such a field forward, so a kind can be matched after any number of wraps without walking a chain.
- **Joining several errors.** `errors.Join` and Python's `ExceptionGroup` collect several failures into one. `Result.all` returns the first `Err`, and `Parallel.map` returns every result, so no current API needs to join errors.
- **Stack traces on errors.** Zig's error return traces and Java's stack traces record where an error passed. Errors are values in Ant, and a trace on every `Err` costs time and memory on paths that are expected to fail. Panics keep their traces.
- **Lazy context.** anyhow's `with_context` builds the message only on failure. A template string in `wrapErr` is built on every call, including on `Ok`. The cost is small, and speed is not a priority for the interpreter.

## Open questions

- [ ] Should `Error.wrap(e, "")` return the message unchanged instead of `": message"`, or should an empty context panic?
- [ ] Should the script-exit change go further and write any error type with a `message: string` field as bare text, not only `Error`?
