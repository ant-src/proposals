# Duration Literals

**Status:** Proposal

## Motivation

Ant already has `Time.Duration`, constructed with `Time.duration(...)` and parsed from ISO 8601 with `Time.parseDuration`. This is precise but verbose for common timeouts, sleeps, and intervals:

```ant
Time.sleep(Time.duration(milliseconds: 500))
let response = try Http.get(url, timeout: Time.duration(seconds: 5))
```

Give short, exact time spans a convenient spelling without adding a new type or changing calendar arithmetic.

## Proposal

Add duration literals using Go-style units:

```ant
Time.sleep(500ms)
let timeout = 2h30m
let interval = 1.5s
let response = try Http.get(url, timeout: 5s)
```

A duration literal has type `Time.Duration`, independent of context. It needs no import, does not use a variable named `Time`, and has the same runtime value as `Time.duration(...)`:

```ant
500ms  // Time.duration(milliseconds: 500)
2h30m  // Time.duration(hours: 2, minutes: 30)
1.5h   // Time.duration(hours: 1, minutes: 30)
0s     // Time.duration()
```

### Syntax

- Units: `h`, `m`, `s`, `ms`, `us`, `ns` (hours through nanoseconds). `m` means minutes.
- One or more *number + unit* components, with no whitespace: `2h30m15s`.
- Units appear once each, in descending order. No duplicates or mixed signs.
- Numbers use Ant's decimal spelling (including `_` between digits), but not exponents or base prefixes.
- A fraction is allowed only in the final component: `1h30.5m`, not `1.5h30m`.
- Fractions are evaluated exactly, never through `float64`; the result must be representable to a nanosecond. For example, `0.1ns` is invalid.
- Values obey the existing `Time.Duration` bounds. Invalid syntax, precision, or range is a source error in both `ant check` and `ant run`.
- The lexer recognizes the entire adjacent sequence as one literal. An unknown suffix or trailing characters are an error, not separate identifiers.

These are **not** general-purpose numeric suffixes. There are no `d`, `w`, `mo`, or `y` literals: calendar units remain explicit with `Time.duration(days: 1)` and similar calls. In particular, `24h` means exactly 24 hours, not the next calendar day across a DST transition.

### Negative durations

Extend unary `-` to accept `Time.Duration` as well as numbers, delegating to the semantics of `Time.negate`:

```ant
let retryOffset = -500ms
let d = 2h
let earlier = -d
```

This does not introduce binary duration arithmetic: combine computed durations with the existing `Time.addDuration`.

## Parsing and formatting

Extend the existing parser, without adding `Time.span` or another constructor:

```ant
try Time.parseDuration("PT2H30M")  // ISO 8601
try Time.parseDuration("2h30m")    // short notation
try Time.parseDuration("-500ms")   // negative
```

The short string syntax follows the literal's unit, order, precision, and range rules, with an optional leading `+` or `-`. The string `"0"` also denotes a zero duration. Bad input returns `Err` rather than panicking; a bad literal is a source error. The short syntax is deliberately stricter than Go's `time.ParseDuration` and is **not** promised to accept every Go string.

Keep **ISO 8601 as the only canonical output**. `String.from`, string interpolation, and JSON serialization are unchanged:

```ant
String.from(500ms)   // "PT0.5S"
String.from(2h30m)   // "PT2H30M"
```

Integral components retain their existing fields: `90m` stays 90 minutes rather than being automatically balanced to 1 hour 30 minutes. A fractional component is decomposed into the smallest exact fields because `Time.Duration` fields are integers.

## Examples to test

| Input | Result |
| --- | --- |
| `500ms`, `2h30m`, `1.5h`, `1h30.5m`, `0s`, `-5s` | Valid duration literals |
| `1m2h`, `1h1h`, `1.5h30m` | Invalid order, repetition, or fraction position |
| `1d`, `1mo`, `1M`, `1min`, `1e3s` | Unsupported spelling |
| `0.1ns`, `1h30`, `1h 30m` | Nonrepresentable precision or invalid syntax |
| `try Time.parseDuration("2h30m")` | `Ok(Time.Duration)` |
| `try Time.parseDuration("bad")` | `Err` |

## Alternatives rejected

- **`Time.span("500ms")`:** another API for the same type, while `Time.parseDuration` already exists.
- **`Time.duration("500ms")`:** overloads an existing numeric-fields constructor and complicates its signature.
- **General user-defined suffixes (`64MiB`, `10km`, `20%`):** introduce name resolution, ambiguity, and a larger language feature for a narrow use case.
- **Separate `Span` type:** duplicates `Time.Duration` and forces conversions at existing APIs.
- **Canonical Go-style formatting:** breaks the current ISO 8601 text and JSON contract.

## Compatibility and implementation

No currently valid Ant source changes meaning: numeric suffixes are currently forbidden. This proposal requires a duration-literal token, a literal AST/value case, `ant check` typing, `ant run` evaluation, unary `-` support for durations, and parser/formatter tests. Reuse one exact short-duration parser for source literals and `Time.parseDuration`; keep ISO 8601 parsing and output unchanged.

References: [Ant language spec](https://github.com/ant-src/spec/blob/master/ant-lang.md) · [Ant stdlib](https://github.com/ant-src/spec/blob/master/stdlib.md) · [Go duration format](https://pkg.go.dev/time#ParseDuration)
