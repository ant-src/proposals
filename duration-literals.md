# Duration Literals

**Status:** Proposal

## Motivation

Ant already has `Time.Duration`, constructed with `Time.duration(...)` and parsed from ISO 8601 with
`Time.parseDuration`. This is precise but verbose for common timeouts, sleeps, and intervals:

```ant
Time.sleep(Time.duration(milliseconds: 500))
let response = try Http.get(url, timeout: Time.duration(seconds: 5))
```

Give short, exact time spans a convenient spelling without adding a new type or changing calendar
arithmetic.

## Proposal

Add duration literals using Go-style units:

```ant
Time.sleep(500ms)
let timeout = 2h30m
let interval = 1.5s
let response = try Http.get(url, timeout: 5s)
```

A duration literal has type `Time.Duration`, independent of context, and needs no import. It has the
same runtime value as the `Time.duration(...)` call with the fields as written:

```ant
500ms  // Time.duration(milliseconds: 500)
2h30m  // Time.duration(hours: 2, minutes: 30)
90m    // Time.duration(minutes: 90)
1.5h   // Time.duration(hours: 1, minutes: 30)
0s     // Time.duration()
```

### One way for each job

The literal and the constructor do not overlap:

- A **literal** is a constant span of hours and smaller units: `5s`, `2h30m`.
- **`Time.duration(...)`** is every other duration: a computed one, `Time.duration(seconds: n)`, a
  calendar one, `Time.duration(days: 1)`, and a negative one.

The spec and the docs write a constant exact span as a literal. There is no duration arithmetic, so
`n * 1s` is a compile error as before, and `Time.duration(seconds: n)` stays the way to build a span
from a number.

### Syntax

- Units: `h`, `m`, `s`, `ms`, `us`, `ns` (hours through nanoseconds). `m` means minutes.
- One or more _number + unit_ components, with no whitespace: `2h30m15s`.
- Units appear once each, in descending order.
- Numbers use Ant's decimal spelling (including `_` between digits), but not exponents or base
  prefixes.
- A fraction is allowed only in the final component: `1h30.5m`, not `1.5h30m`.
- A fraction is spread into the smaller fields exactly as `Time.parseDuration` spreads one: `1.5h`
  is 1 hour and 30 minutes, `1h30.5m` is 1 hour, 30 minutes and 30 seconds, `1.5s` is 1 second and
  500 milliseconds, `0.5ms` is 500 microseconds. It is evaluated exactly, never through `float64`,
  and must come out as a whole number of nanoseconds: `0.1ns` is invalid.
- Values obey the existing `Time.Duration` bounds. Invalid syntax, precision, or range is a source
  error in both `ant check` and `ant run`.
- The lexer recognizes the entire adjacent sequence as one literal. An unknown suffix or trailing
  characters are an error, not separate identifiers.
- A `.` after the unit is member access, since the number has ended:
  `90m.roundDuration(largestUnit: Time.Hour{})`. A `.` after a digit still begins a fraction, so
  `1.s` is an error as `1.x` is.

These are **not** general-purpose numeric suffixes. There are no `d`, `w`, `mo`, or `y` literals:
calendar units remain explicit with `Time.duration(days: 1)` and similar calls. In particular, `24h`
means exactly 24 hours, not the next calendar day across a DST transition.

### Equality

A literal keeps its fields as written, like every other `Time.Duration`, and `==` on durations
compares the ten fields. So two literals of the same length are not always equal:

```ant
90m == 1h30m      // false: 90 minutes; 1 hour and 30 minutes
60s == 1m         // false
1000ms == 1s      // false
1.5h == 1h30m     // true: the fraction spreads to the same fields

Time.compareDuration(90m, 1h30m) == 0   // true: the same length
```

This is the existing rule (`stdlib.md`, Time), but a literal makes it easier to trip over than
`Time.duration(minutes: 90)` does, since the same spelling in Go compares by length. The spec states
it next to the literal, with these examples, and `ant check` reports `==` or `!=` between two
duration literals of the same length and different fields, naming `Time.compareDuration`.

Balancing a literal at compile time, so that `90m` were 1 hour and 30 minutes, was considered and
rejected: the literal would differ from the constructor call it stands for and from
`parseDuration("PT90M")`, and `==` would still be field-wise for every computed duration.

### Negative durations

There is no negative literal and unary `-` still needs a number. A negative span is
`Time.negate(500ms)`, and stepping back in time is `Time.subtract(t, 5m)`.

Extending unary `-` to `Time.Duration` was considered and rejected. It would be the only operator on
a library type, and it suggests `a + b`, `d * 2` and `a < b`, which stay `Time.addDuration` and
`Time.compareDuration`. Negative constants are also rare: `Time.sleep`, `Time.ticker`, `Context.new`
and the `Http` timeouts all refuse or ignore one.

## Parsing and formatting

`Time.parseDuration` also takes the strings Go's `time.ParseDuration` takes, since that is the
notation configuration files, flags and environment variables already use, and the one
`ant test -timeout` documents:

```ant
try Time.parseDuration("PT2H30M")  // ISO 8601, as before
try Time.parseDuration("2h30m")    // Go notation
try Time.parseDuration("-500ms")   // negative
try Time.parseDuration("1.5µs")    // as Go prints it
```

A string whose first character after an optional sign is `P` is ISO 8601, parsed as today. Any other
string is Go notation, with Go's grammar rather than the literal's:

- An optional `+` or `-`, then one or more _number + unit_ components. `"0"`, with an optional sign,
  is the zero duration.
- Units are `h`, `m`, `s`, `ms`, `us`, `µs` (U+00B5), `μs` (U+03BC) and `ns`.
- Components come in any order, a unit may repeat, and any component may carry a fraction: `"1m2h"`,
  `"1h1h"` and `"1.5h30m"` are accepted, as in Go.
- Each component adds to its own field, and a fraction spreads into the smaller fields as above, so
  `"1h1h"` is 2 hours and `"1.5h30m"` is 1 hour and 60 minutes.
- Digits of a fraction below a nanosecond are dropped, as Go drops them: `"0.1ns"` is zero.
- A value outside the `Time.Duration` bounds is `Err`.

Every string Go's `Duration.String` prints therefore parses, and so does every string that is a
valid literal. Bad input returns `Err` rather than panicking; a bad literal is a source error.
`ant test -timeout` reads its argument with the same parser.

The literal is stricter than the string on purpose: source is written once and read often, so it has
one order and no repeats, while a string comes from outside and is taken as Go takes it.

Keep **ISO 8601 as the only canonical output**. `String.from`, string interpolation, and JSON
serialization are unchanged:

```ant
String.from(500ms)   // "PT0.5S"
String.from(2h30m)   // "PT2H30M"
```

## Examples to test

| Input                                                  | Result                                          |
| ------------------------------------------------------ | ----------------------------------------------- |
| `500ms`, `2h30m`, `1.5h`, `1h30.5m`, `0s`, `1_000ms`   | Valid duration literals                         |
| `1m2h`, `1h1h`, `1.5h30m`                              | Invalid order, repetition, or fraction position |
| `1d`, `1mo`, `1M`, `1min`, `1e3s`, `0x1s`, `1µs`       | Unsupported spelling                            |
| `0.1ns`, `1h30`, `1h 30m`, `1.s`                       | Nonrepresentable precision or invalid syntax    |
| `-5s`, `5s + 1s`, `2 * 1s`                             | Compile error: the operator needs a number      |
| `90m == 1h30m`                                         | `false`, and reported by `ant check`            |
| `Time.compareDuration(90m, 1h30m)`                     | `0`                                             |
| `1.5s == Time.duration(seconds: 1, milliseconds: 500)` | `true`                                          |
| `90m.roundDuration(largestUnit: Time.Hour{})`          | `1h30m`                                         |
| `try Time.parseDuration("2h30m")`                      | `Ok(2h30m)`                                     |
| `try Time.parseDuration("1m2h")`, `("1h1h")`           | `Ok`: 2 hours and 1 minute; 2 hours             |
| `try Time.parseDuration("1.5µs")`, `("0")`, `("-5s")`  | `Ok`                                            |
| `try Time.parseDuration("bad")`, `("5")`, `("1d")`     | `Err`                                           |

## Alternatives rejected

- **`Time.span("500ms")`:** another API for the same type, while `Time.parseDuration` already
  exists.
- **`Time.duration("500ms")`:** overloads an existing numeric-fields constructor and complicates its
  signature.
- **General user-defined suffixes (`64MiB`, `10km`, `20%`):** introduce name resolution, ambiguity,
  and a larger language feature for a narrow use case.
- **Separate `Span` type:** duplicates `Time.Duration` and forces conversions at existing APIs.
- **Canonical Go-style formatting:** breaks the current ISO 8601 text and JSON contract.
- **Unary `-` on durations:** see Negative durations.
- **Balanced literals:** see Equality.
- **A strict short syntax in `parseDuration`:** a third dialect next to ISO 8601 and Go's, which
  would reject strings Go itself prints, such as `1.5µs`.

## Compatibility and implementation

No currently valid Ant source changes meaning: numeric suffixes are currently forbidden, and
`Time.parseDuration` returns `Err` today for every string it would newly accept.

- **Spec:** the literal in `ant-lang.md` (Literals, the lexical grammar, the "no literal suffixes"
  sentence); `parseDuration`, the equality note and `Cmd.stop`'s default in `stdlib.md`; examples
  that spell a constant span, such as `Time.sleep(Time.duration(seconds: 1))`, rewritten as
  literals; `-timeout` in `test.md`; the literal in `fmt.md`, which prints it as written.
- **ant:** a duration-literal token, a literal AST and value case, typing in `ant check` with the
  equality report, evaluation in `ant run`, and the Go-notation branch of `Time.parseDuration`. The
  literal and the string share the number and fraction code, not the grammar.
- **Editors:** the token in the `vscode-ant`, `intellij-ant` and `nvim-ant` grammars and in the
  Monaco package.
- **Docs:** `ant/docs` pages that build a constant span.

References: [Ant language spec](https://github.com/ant-src/spec/blob/master/ant-lang.md) ·
[Ant stdlib](https://github.com/ant-src/spec/blob/master/stdlib.md) ·
[Go duration format](https://pkg.go.dev/time#ParseDuration)
