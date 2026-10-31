# Proposal: `assert`

2026-10-26 · Anton · Accepted, in spec 50ec384 + 9e8b2ee and ant 9e65a9e

## Summary

Add `assert(cond, message?)` to the prelude. It panics when `cond` is false, with a message that
quotes the condition as written and shows the values it was computed from: the two sides of a
comparison, and each variable, field and call inside it. It narrows like `if !cond { panic(...) }`,
so `assert(user != null)` leaves `user` a `User` below it. It is always on: no flag, build mode or
release setting removes it. The message is evaluated only on failure. `assert` is built into the
checker and the evaluator and, unlike every other prelude name, is not a value: its report needs the
condition's source text, which only a call site has. No keyword, no new syntax, no debug-only
variant.

## Motivation

Ant already says what a failed check is: "`panic` is for bugs" (section 1). What it lacks is a short
way to write the check, and a good report when it fails.

1. **The spelling is long.** An invariant today is `if !(i < len(xs)) { panic("i out of range") }`.
   The condition is written negated, the message repeats the condition in prose, and the line is
   long enough that people skip it.
2. **The report says too little.** `panic("i out of range")` gives a message and a trace. It does
   not say what `i` was, or which of two similar checks on the line failed. Every author writes the
   values into the message by hand, `panic("i out of range: ${i} >= ${len(xs)}")`, or does not.
3. **Tests need it.** The test framework sketched in `ant/TODO.md` proposes `Test.equal` and
   `Test.true` so that a failure shows both values. If the language's own check shows both values,
   most of that API is unnecessary: a test is a function that calls `assert`, and the runner already
   turns a panic into a failure through `Panic.catch` (section 8, Catching a panic).
4. **`??` covers only null.** `value ?? panic("unreachable")` is a good one-liner for absence. There
   is no equivalent for `is`, for a range, or for a relation between two values.

## Prior art

Every mainstream language has an assertion. They differ on four questions: is it a statement, a
macro or a function; can it be turned off; what does the failure say; and does the type checker
learn from it.

| Language    | Form                                                                                            | Turned off by                                                                              | On failure                                                                                      | Narrows                            |
| ----------- | ----------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------- | ---------------------------------- |
| C, C++      | `assert(e)` macro                                                                               | `NDEBUG`                                                                                   | file, line, text of `e`; abort                                                                  | no                                 |
| C++26       | `contract_assert(e)`, `pre`, `post`                                                             | evaluation semantic: ignore, observe, enforce, quick-enforce                               | handler gets source location and text                                                           | no                                 |
| Java        | `assert e : msg;` statement                                                                     | **off by default**, `-ea` turns on                                                         | `AssertionError(msg)`                                                                           | no                                 |
| C#          | `Debug.Assert(e, msg)`                                                                          | removed outside debug builds; `Trace.Assert` stays                                         | dialog or listener                                                                              | via `[DoesNotReturnIf(false)]`     |
| Python      | `assert e, msg` statement                                                                       | `-O` strips it, message included                                                           | `AssertionError(msg)`; pytest rewrites asserts to show values                                   | no (mypy: yes)                     |
| JavaScript  | `console.assert(e, msg)`; Node `assert(e)`                                                      | —                                                                                          | `console.assert` **only logs and continues**; Node throws, and shows the source text            | TypeScript: `asserts e` signatures |
| Go          | none                                                                                            | —                                                                                          | —                                                                                               | —                                  |
| Rust        | `assert!`, `assert_eq!`, `assert_ne!`, `debug_assert!` macros                                   | `debug_assert!` only, in release                                                           | `assertion failed: e`; `_eq` shows left and right                                               | no                                 |
| Swift       | `assert`, `precondition`, `assertionFailure`, `fatalError`; Swift Testing `#expect`, `#require` | `-O` removes `assert`; `-Ounchecked` removes `precondition` and assumes it true            | message, file, line; `#expect` shows every subexpression's value                                | `#require` unwraps                 |
| Kotlin      | `assert`, `require`, `check`, `requireNotNull`, `checkNotNull`, `error`                         | `assert` needs JVM `-ea`; the others always run                                            | `IllegalArgumentException` / `IllegalStateException`; power-assert compiler plugin shows values | yes, through contracts             |
| Zig         | `std.debug.assert(ok)`                                                                          | in ReleaseFast it becomes `unreachable`, **undefined behavior** that the optimizer assumes | panic with trace                                                                                | no                                 |
| D           | `assert(e, msg)`, `in`/`out` contracts, `invariant`                                             | `-release`, except `assert(0)`                                                             | `AssertError`                                                                                   | no                                 |
| Eiffel, Ada | `require`/`ensure`/`invariant`; `Pre`/`Post`, `pragma Assert`                                   | per-class or `Assertion_Policy`                                                            | exception                                                                                       | no                                 |
| OCaml       | `assert e` keyword                                                                              | `-noassert`, except `assert false`                                                         | `Assert_failure (file, line, col)`                                                              | no                                 |
| Haskell     | `Control.Exception.assert`                                                                      | ignored under `-O`                                                                         | file and line                                                                                   | no                                 |
| Nim         | `assert`, `doAssert`                                                                            | `assert` off under `-d:danger`; `doAssert` always                                          | text of the condition                                                                           | no                                 |
| Gleam       | `assert e` keyword (1.11)                                                                       | never                                                                                      | text of `e`, and both sides of a binary operator evaluated                                      | no                                 |
| Elixir      | `assert` in ExUnit only; production code matches and crashes                                    | —                                                                                          | ExUnit shows left and right of `==`                                                             | pattern match binds                |

What Ant takes:

- **Always on, never assumed.** Turning assertions off is the most regretted part of the design
  space. C's `assert(x = f())` and Python's `assert do_work()` vanish under a flag. Java's default
  of off means assertions almost never run in production, which is where invariants break. Zig and
  Swift's `-Ounchecked` go further and let the optimizer assume the check, so a wrong assertion is
  undefined behavior. Rust, Kotlin's `require`/`check`, Nim's `doAssert`, Gleam and Swift's
  `precondition` keep the check, and the newer languages default that way. Ant is interpreted and
  has no release mode to strip it in.
- **Show the values.** Rust's `assert_eq!`, pytest's rewriting, Swift Testing's `#expect`, Groovy
  and Kotlin power-assert, ExUnit and Gleam all exist because "assertion failed" alone sends the
  reader to a debugger. Gleam and `#expect` do it without a second name: the one assertion looks at
  the comparison it was given. Ant can too, since the checker and evaluator see the condition, and
  like power-assert it also shows the variables and calls inside it.
- **Narrow.** TypeScript added `asserts x is T` signatures and Kotlin added contracts because an
  assertion that the checker ignores forces a second, redundant check. Ant's narrowing already
  treats `if !cond { panic(...) }` this way (section 7); `assert` gets the same rule.
- **Lazy message.** Python, Java, Rust, Swift (`@autoclosure`) and Kotlin (a lambda) evaluate the
  message only on failure, so an expensive message costs nothing on the passing path.
- **One name.** Kotlin's `require`/`check` and Swift's `assert`/`precondition` split one idea by who
  is to blame or by build mode. Ant already has the split that matters: a bad input is a `Result`, a
  bug is a panic. One name, `assert`, for the second.
- **Not Go's answer.** Go's FAQ leaves assertions out because programmers "use them as a crutch to
  avoid thinking about proper error handling and reporting." Ant answers that with `Result`, not by
  making the check for a bug longer to write. The guidance below keeps `assert` off input
  validation.

## The `assert` function

```ant
assert(cond: bool, message: string = ""): void   // panics if cond is false; message is evaluated only then
```

### Semantics

1. **Evaluation.** `cond` is evaluated once. If it is `true`, `assert` returns and `message` is
   never evaluated. If it is `false`, `message` is evaluated and the goroutine panics, exactly as
   `panic` does: deferred blocks run, `Panic.catch` can stop it, and an uncaught panic is reported
   with its trace (section 12, Deferred blocks and panics). The innermost frame is the `assert`
   call. The message is lazy exactly as the message of `if !cond { panic(message) }` is.
2. **The message.** The panic message is `assertion failed: ` followed by the condition as `ant fmt`
   prints it on one line, then the values of rule 3 when there are any, then `: ` and `message` when
   it is not empty. Comments and line breaks in the source do not appear.
3. **Values.** On failure the report shows the values `cond` was computed from, each rendered in
   full by `String.from` as inside an array, so a string is quoted. They are the values the
   evaluation already produced; nothing is evaluated a second time.
   - **The comparison.** When the outermost operator of `cond` is `==`, `!=`, `<`, `<=`, `>`, `>=`
     or `is`, the two values follow the condition in parentheses, so they stand next to the text
     they belong to: `total == expected (9 == 10)`. For `is`, the left value is shown in full and
     the right side is the left value's actual tag. A failed `!=` against the literal `null` names
     the other side instead, since both values are `null`: `user != null (user is null)`.
   - **The parts.** Below the first line comes one line `text = value` for each name, field access,
     index expression and call inside `cond` that ran, in the order it ran: `assert(isSorted(xs))`
     shows `xs = [3, 1, 2]`. Left out are literals, the callee of a call, the object of a field
     access or index (the access itself is shown), the comparison's two operands (already in
     parentheses), the whole condition, the bodies of lambdas, and a part whose value is a function.
     A text that repeats is shown once, with its first value. A part that short-circuiting skipped
     did not run and is not shown, so `assert(a && b)` shows `a = true` and `b = false`, or only
     `a = false`.
4. **Narrowing.** After `assert(cond)`, the checker applies what `if !cond { panic(...) }` would:
   the complement of the false branch holds for the rest of the block (section 7, Narrowing). So
   `assert(x != null)`, `assert(s is Circle)` and `assert(a != null && b != null)` narrow.
5. **Not a value.** `assert` may only be called by name. `Array.map(flags, assert)` or
   `var check = assert` is a compile error and, unchecked, a syntax error from the resolver, which
   already knows the name cannot be shadowed (section 3). A direct call is therefore always the
   prelude's `assert`. This makes `assert` the one prelude name that is not a value: `len`,
   `println` and the conversions are built into the checker too, but each can still be passed
   (`Array.map(words, len)`). `assert` cannot, because rules 2 and 3 need the condition as written
   and its operands, which exist only at a call site.
6. **Unchecked runs.** Rules 1–3 are run-time rules and hold under `ant run` without `ant check`. A
   `cond` that is not a `bool` panics as `if` does (section 12, Runtime panics), and a `message`
   that is not a string panics as `panic` does.
7. **Never removed.** There is no flag, environment variable or build mode that skips `assert`.

### Reports

```ant
func pop[T](stack: Array[T]): T {
  assert(len(stack) > 0, "pop on empty stack")
  ...
}
```

```
panic: assertion failed: len(stack) > 0 (0 > 0): pop on empty stack
  stack = []
  at pop (stack.ant:2:3)
  at main (main.ant:14:9)
```

```ant
var total = Array.sum(prices)
assert(total == expected)
// panic: assertion failed: total == expected (9 == 10)

assert(isSorted(merged))
// panic: assertion failed: isSorted(merged)
//   merged = [1, 3, 2]

assert(lo <= mid && mid < hi)
// panic: assertion failed: lo <= mid && mid < hi
//   lo = 0
//   mid = 5
//   hi = 5

var shape = load(id)
assert(shape is Circle)
println(shape.radius)                       // shape: Circle here
// panic: assertion failed: shape is Circle (Square{ side: 2 } is Square)

var user = find(users, id)
assert(user != null, "user ${id} seeded above")
println(user.name)                          // user: User here
// panic: assertion failed: user != null (user is null): user 7 seeded above
```

The message follows the error convention of section 8: lowercase, no closing punctuation, context
first. A message names the reason the condition should hold, not the condition again.

### Tests

With the values in the report, a test needs no assertion library:

```ant
// math_test.ant
func testAdd() {
  assert(add(1, 2) == 3)
  assert(add(-1, 1) == 0)
}
// FAIL testAdd: assertion failed: add(1, 2) == 3 (4 == 3)
//   at testAdd (math_test.ant:3:3)
```

The runner calls each test through `Panic.catch`, as `ant/TODO.md` suggests, and prints the caught
panic. `Test.equal` and `Test.true` become unnecessary; `Test.fail`, `Test.stop` and `Test.skip`
stay with the test framework, which is a separate proposal. A test that should keep going after a
failure collects its own failures; `assert` always stops.

## When to use it

`assert` states something the program believes is true, so that a bug shows where it is made instead
of where it is noticed.

| Situation                                                                       | Use                                                                                                   |
| ------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| A caller passed a bad value to a library function that documents a precondition | `assert`, and say so in the doc comment: "panics if n < 0" (section 11)                               |
| An invariant of a data structure, a loop or a state machine                     | `assert`                                                                                              |
| A value that cannot be null here                                                | `value ?? panic("...")` when the value is used at once, `assert(value != null)` when it is used later |
| A branch that cannot run                                                        | `panic("unreachable")`                                                                                |
| Input from a user, a file, the network, another service                         | `Result`. Never `assert`                                                                              |
| A test                                                                          | `assert`                                                                                              |

The rule: if the program can recover by doing something else, it is a `Result`. If the only fix is
to change the code, it is an `assert`.

## Considered and rejected

- **A keyword, `assert cond, msg`.** Python, Java, OCaml and Gleam spell it as syntax. Ant has one
  prefix keyword (`try`) and keeps everything else a prelude name, including `panic`. A call with a
  default message parameter reads like every other call, needs no grammar change and no new reserved
  word. The cost is that `assert` is a prelude name with special evaluation, and the only one that
  is not a value (rule 5).
- **`assertEqual`, `assert_eq`.** A second name for the most common case. Rule 3 gives `==` the same
  report, and covers `<` and `is` too.
- **`debugAssert` or a `-O` flag.** See Prior art. A check too expensive for production is guarded
  by the program's own flag: `if Config.paranoid { assert(isSorted(xs)) }`.
- **Assumed assertions.** Letting a compiler assume `cond` is the Zig and `-Ounchecked` model. Ant
  has no optimizer that would benefit, and a wrong assertion must stay a panic.
- **`require` and `check` as in Kotlin.** They split the same panic by who is at fault. In Ant the
  caller's fault on input is a `Result`; on a documented precondition it is a bug like any other.
- **A power-assert diagram.** Groovy and Swift's `#expect` draw each value under its subexpression.
  Rule 3 lists the same values one per line instead, which survives a proportional font, a log
  prefix and line wrapping, and needs no layout rules.
- **Contracts (`pre`, `post`, `invariant`).** Eiffel, D, Ada and C++26 attach checks to
  declarations. They need syntax on `func` and `type`, rules for inheritance that Ant does not have,
  and give little over an `assert` at the top and bottom of a body.
- **Static assertions.** Ant has no compile-time evaluation to assert about.

## Spec changes

All additions. `panic` and narrowing are unchanged in meaning.

| Where                                     | Change                                                                                                                                            |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| `spec/ant-lang.md` §1, Principles         | None. "`panic` is for bugs" already covers it                                                                                                     |
| §7, Narrowing                             | Add to the second bullet: `assert(cond)` narrows the rest of the block as `if !cond { panic(...) }` does                                          |
| §8, new subsection `assert` after `panic` | Semantics rules 1–7, the report format, and the When to use it table                                                                              |
| §11, Prelude                              | Add the signature, and a paragraph that `assert` is built into the checker and the evaluator and, unlike every other prelude name, is not a value |
| §11, Html                                 | None. Its example `assert(badge(3) == <b>3</b>)` already calls `assert`, which this proposal adds                                                 |
| §12, Expressions                          | A call of `assert`: evaluation order, the lazy message, the values recorded for the report                                                        |
| §12, Runtime panics                       | Add a row: `assert` with a non-`bool` condition (ruled out) or a false condition (possible)                                                       |
| `spec/fmt.md`                             | Note that the report prints the condition as `ant fmt` does                                                                                       |
| `ant/TODO.md`, Test framework             | Drop `Test.equal` and `Test.true` in favor of `assert`                                                                                            |
| `ant/internal/prelude/prelude.d.ant`      | Declaration of `assert` with its doc comment                                                                                                      |

In the Go interpreter, the resolver rejects `assert` outside a call's callee, and the evaluator
handles the call as a node of its own (condition, lazy message, and the list of parts rule 3 shows).
It records each part's value as the condition evaluates and drops them when it is true. The texts of
the condition and its parts come from `internal/format` once, when the module is loaded, not on each
call. The JS backend emits the texts as constants at the call site.

## Non-goals

- **Turning assertions off.** By design, see Prior art.
- **A testing framework.** `ant test`, discovery, filtering and `Test.skip` stay in their own
  proposal; this one only makes `assert` good enough to be its assertion.
- **Custom failure handlers.** C++26 and Java let a program replace what a failed check does. A
  program that must survive a bug in one unit of work uses `Panic.catch` at that unit's edge.
- **Shortening values.** Values print in full by `String.from`. A pretty printer that folds a long
  value, or a diff that lists where two structures differ, can come later without changing `assert`.
- **Panic values other than a message.** An assertion's panic is an ordinary `Panic` record. The
  operand values are in its message, not in fields.

## Open questions

Decided after review:

| Question                                         | Decision                                                  |
| ------------------------------------------------ | --------------------------------------------------------- |
| Truncate long values?                            | No; `String.from` in full for now, a pretty printer later |
| `assert(false)` and `assert(true)` errors?       | No                                                        |
| Show variables and calls, not only the operands? | Yes, rule 3's parts, listed one per line                  |
| Lazy `message`?                                  | Yes, as the message of `if !cond { panic(message) }`      |
| A diff for a failed `==` between structures?     | Not now; both values print in full by `String.from`       |
