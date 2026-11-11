# Proposal: `Result.fail`

2026-11-11 · draft

## Summary

Add one function to the `Result` namespace:

```ant
Result.fail[T](message: string): Result[T, Error]     // Result.err(Error{ message })
```

It builds the `Err` that most failing code builds, an `Error` with a message, without the nested
record literal:

```ant
return Result.err(Error{ message: "invalid quantity" })     // today
return Result.fail("invalid quantity")                      // proposed
```

There is no new syntax, no new prelude name outside `Result`, and no checker rule: the function can
be written in Ant today. `Result.err` and the `Error{ ... }` literal stay as they are.

## Motivation

The standard library reports failures as `Error`, a record `{ message: string }`, and scripts follow
it. Building one takes a constructor call around a record literal around the one thing the author
has to say:

```ant
var user = users[id] ?? return Result.err(Error{ message: "no such user" })
```

This is the common case, not a corner. In the `ant` and `spec` repositories, 90 of the 275 calls of
`Result.err` are `Result.err(Error{ ... })`; 60 of those follow a `return`, and 20 of those sit on
the right of `??`, where the line is already long. The spec's own examples write
`?? return Result.err(Error{ message: ... })` four times.

The message is all that varies between these calls, so it should be all that is written.

## Prior art

- **Go** has `errors.New("message")` for exactly this, and returns it bare, since a Go function
  returns the error beside the value and needs no wrapper.
- **Kotlin** has `Result.failure(exception)`, the constructor of the failing case, named for the
  outcome and kept in the `Result` namespace.
- **Rust** with `anyhow` writes `Err(anyhow!("message"))`, and `bail!("message")` for that plus the
  `return`. `bail!` is a macro because it leaves the function; a function cannot.
- **Effect** (TypeScript) has `Effect.fail(error)`, the same word for the same role.

## The design

```ant
func Result.fail[T](message: string): Result[T, Error] {
  Result.err(Error{ message })
}
```

`Result.fail(message)` is `Result.err(Error{ message })` under an expected `Result[T, Error]`, and
nothing more: it builds a value and does not leave the function, so a failing path still says
`return`, or is a block's last expression.

```ant
func parseOrder(row: Row): Result[Order, Error] {
  let quantity = String.toInt(row.quantity)
    ?? return Result.fail("order ${row.id}: quantity must be an integer")
  if quantity <= 0 {
    return Result.fail("order ${row.id}: quantity must be positive")
  }
  Result.ok(Order{ id: row.id, quantity })
}
```

It sits beside `Result.wrapErr`, the other function of `Result` that is about `Error` and not about
any `E`, and the two read together: one starts a message, the other adds context to it. The message
follows the convention of Wrapping errors: lowercase, no closing punctuation.

### Inference

`T` has no argument to come from, so it is solved from the expected type, as the `T` of `Result.err`
is. Every place the long form is written today has one: a `return`, the last expression of a
function body with a declared result, the right side of `??`, an annotated declaration, or an
argument whose parameter type is a `Result`.

```ant
var r: Result[int, Error] = Result.fail("typed")     // ok
var r = Result.fail("x")                             // compile error: cannot infer T
```

The second line is an error for `Result.err(Error{ message: "x" })` too, for the same reason.

### Only `Error`

The error type of the result is `Error` and no wider type. Type arguments are invariant, so a
`Result[T, Error]` is not a `Result[T, NotFound | Error]`, and `Result.fail` under that expected
type is a compile error. `Result.err` takes `E` from the expected type and still works there:

```ant
func find(id: int): Result[User, NotFound | Error] {
  if id < 0 { return Result.fail("negative id") }                       // compile error
  if id < 0 { return Result.err(Error{ message: "negative id" }) }      // ok
  ...
}
```

`Result.wrapErr` has the same limit today. In practice a function with a union of error types
returns its own typed errors and receives `Error` through `try`, which widens; building a fresh
`Error` there is the rare case, and the long form covers it.

The checker says so in those terms, as it does for any generic call whose result no type argument
fits to the expected type:

```
cannot use Result.fail where Result[User, NotFound | Error] is expected: it returns Result[T, Error]
```

The doc comment of `Result.fail` points at `Result.err` for a wider error type.

## Spec changes

- `ant-lang.md` section 11, Result, Error, Panic, and Record: the signature, after `Result.err`.
- Section 8: `Result` gains a sentence after the one on `Result.ok` and `Result.err`; the list that
  closes `try` gains `Result.fail`; Wrapping errors may say that `Result.fail` starts a message that
  `Result.wrapErr` extends.
- The examples that build an `Error` to return it move to the short form: `??` under `return` in
  section 5, `Channel.receive` in section 9, and the two `Parallel` examples in section 11.
- `prelude.d.ant`: the declaration with its doc comment; `ant doc Result.fail` shows it.

The implementation is one native function. The docs get a `lib/Result/fail` page, and
`guide/idiomatic.md` and the tour's errors page move to the short form where the error type is
`Error`.

## Non-goals

- **Returning early.** `Result.fail` is a function. A form that also leaves the function is a
  statement, and `return` already is one.
- **Wider error types.** See Only `Error`. Accepting any `E` that includes `Error` needs a lower
  bound, which Ant's generics do not have, so the function would be a builtin whose declaration in
  `prelude.d.ant` could not state its own rule.
- **Formatting.** There is no `Result.failf`; string interpolation builds the message.
- **Removing anything.** `Result.err` stays the constructor for every error type, `Error` included.

## Alternatives considered

- **A prelude function `error("message")`** ([error.md](error.md)). The shortest call, but a prelude
  name cannot be shadowed in any scope, so `error` would stop being a variable name everywhere, and
  it is the natural name for what is read out of an `Err`: the spec's examples name a variable
  `error` twice. It also reads as a conversion to `Error` beside `Error{ ... }`, and as leaving the
  function beside `panic("...")`, and does neither.
- **`Result.err` taking `E | string`.** A string would become an `Error` inside. `E | string` is not
  a legal union, since `E` may be `string`; a `Result[T, string]` could no longer be built with
  `Result.err`, because `ant run` sees only the value; and under an expected `Result[T, ParseError]`
  the call would check and build an `Error`. It is also an implicit conversion.
- **`Error.new("message")`**, as Go's `errors.New`. `Result.err(Error.new("x"))` works under wider
  error types, but saves seven characters over the literal and leaves the nesting.
- **A `fail` statement**, `fail "message"` for `return Result.err(...)`. Short, and the checker
  could widen, but it is a keyword and a second way to return, next to `try` and `return`.
- **Errors as union members**, `Order | Error` with no `Result`, as `T | null` has no `Option`. It
  removes both constructors and the invariance limit, but replaces the whole of section 8 and every
  signature in `stdlib.md` that returns a `Result`, and needs error types marked at run time for
  `try`. Too large a change to make for this.
- **Nothing.** The long form is correct and explicit. It is also a third of all `Result.err` calls.

## Open questions

1. **The name.** `fail` is short and Kotlin and Effect use the word, but it hints at control flow
   more than `err` does. `Result.error` is too close to `Result.err` to tell apart.
