# Proposal: diagnostics with codes, fixes, and warnings

2026-11-19 · accepted

## Summary

Make `ant check` tell a reader, human or program, what went wrong, how to fix it, and everything
that is wrong in one run. Four changes, which share one data structure:

1. **Codes.** Every diagnostic carries a stable code naming the rule it enforces, such as
   `type-mismatch` or `unused-result`. Messages may change; codes do not. `ant check -explain code`
   prints what a code means, with a wrong and a right example.
2. **Help and fixes.** A diagnostic may carry a `help` line and fixes, each a set of text edits. A
   fix the checker is certain of is marked `auto`, and `ant code fix` applies those. The language
   server offers every fix as a quick fix, in place of its message-matching regular expressions.
3. **Warnings.** A second severity for code that is legal but almost always a bug: a discarded
   `Result`, an unused import or variable, and a named type interpolated as `Month("2026-09")`.
   Warnings print but leave the exit status 0; `-strict` makes them fail. There are no suppression
   comments: each warning is silenced by writing the intent, such as `_ = f()`.
4. **Every error in one run.** `ant check` parses with the recovering parser the language server
   already uses, reports every syntax error of a file, and type-checks what parsed. An error that
   only follows from an earlier one is not reported.

Nothing changes in `ant run`, and no program that checks today fails `ant check` without `-strict`.
Two things a consumer of `ant check` sees do change, and the proposal treats them as breaking
(Compatibility, below): a program with no error may now print warnings, and `-json` writes objects
that are not errors.

## Motivation

An agent wrote five programs (about 1,150 lines) in Ant from the docs alone, and recorded every
failed `ant check`, every bug that only showed up at run time, and every API it had to guess. Most
mistakes took one round trip, because the messages are good. The cost was in four places, and this
proposal addresses each:

1. **One error at a time.** 18 runs of `ant check` failed. In 9 of them, a syntax-stage error (a
   reserved word, a prelude name, a nested `func`, a bad escape, a test name that is not a string)
   hid every other error in the file, because a file that cannot be loaded is "reported and not
   checked" (`ant-lang.md` section 4). Each one cost an extra round trip.
2. **A fix the checker knows but does not say.** `groups.join(",")` fails with "Array[string] has no
   field or function join". The checker knows the prelude has
   `String.join(parts: Array[string], sep: string)`, whose first parameter takes the receiver. Six
   of the 18 failures were guessed APIs like this, and a `help` line would have turned each into a
   one-step fix.
3. **Bugs that check clean.** This program checks with no error and goes wrong in two places:

   ```ant
   type UserId = string
   func save(path: string): Result[void, Error] { File.write(path, "x") }
   save("/nonexistent/dir/file")   // the Err is dropped
   let id = UserId("u1")
   println("user ${id}")           // prints: user UserId("u1")
   ```

   Errors are values in Ant, and dropping one is legal: only a discarded HTML expression is an error
   (section 5). Nothing fails and nothing prints. For a person this is a review comment; for an
   agent, which trusts `ant check` exiting 0, it is a shipped bug.

4. **Tools match messages.** The language server's quick fixes (state an inferred type, add missing
   fields, import a name) find their diagnostics by regular expressions over the message text
   (`internal/lsp/codeaction.go`). Rewording a message silently breaks a quick fix. Any outside tool
   that wants to act on a diagnostic has the same problem, and a `fix` subcommand of `ant code` has
   been waiting for codes for that reason.

## Prior art

| Tool                      | Codes                                       | Fixes                                                                                    | Warnings                                                                 | What Ant takes                                                                                         |
| ------------------------- | ------------------------------------------- | ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------ |
| rustc                     | `E0308`, numbered; `rustc --explain E0308`  | `help:` lines; suggestions with an applicability: machine-applicable, maybe-incorrect, … | lints on by default; `#[must_use]` on `Result` warns when one is dropped | `help:`, `--explain`, the applicability split as `auto`, and a must-use `Result` warning on by default |
| mypy                      | named, after the message: `[arg-type]`      | none                                                                                     | some checks optional                                                     | Named codes, printed in brackets at the end of the line                                                |
| Go (`go build`, `go vet`) | none in the compiler; vet analyzers by name | gopls quick fixes; `go fix` for API moves                                                | unused imports and variables are errors; vet's checks fail `go vet`      | vet's bar for a check: correct, frequent, precise. No suppression comments                             |
| TypeScript                | `TS2322`, numbered                          | code fixes in the language service, "fix all" per code                                   | none in the compiler                                                     | Fixes as data in the checker's output, not only in the editor                                          |
| clang                     | flag names, `-Wunused-variable`             | fix-it hints, `-fixit` applies them                                                      | `-Werror` turns warnings into errors                                     | A fix shown in text and applicable from the command line                                               |
| ESLint, Ruff              | rule names, `no-unused-vars`, `F401`        | `--fix`; Ruff separates safe and unsafe fixes, applying only safe ones by default        | configurable per rule, with inline disables                              | Safe fixes only by default. Not the configuration or the inline disables                               |

Five lessons:

- **Names beat numbers.** `E0308` and `TS2322` mean nothing until looked up. `[arg-type]` and
  `no-unused-vars` say what they are about, and an agent can match on them and guess their meaning.
- **A fix is data.** clang, rustc, TypeScript, and Ruff all attach edits to the diagnostic, so the
  editor, the command line, and other tools apply the same fix. Ant's quick fixes live only in the
  language server, recomputed from the message.
- **Applicability matters.** Rust and Ruff both learned that an automatic fix must not change what a
  correct program does. Rust marks a suggestion `MachineApplicable` only when it is certain; Ruff
  applies "unsafe" fixes only when asked. Ant's `auto` is the same idea.
- **Dropping a `Result` is the one warning everyone agrees on.** Rust's `unused_must_use` is on by
  default and fires on `Result`. Go has `errcheck` outside the toolchain because the compiler does
  not check it, and its absence is a long-standing complaint.
- **Configuration and inline disables rot.** ESLint's per-rule configuration means two projects run
  different checkers. Go vet has no switches and no disable comments, and keeps that workable by
  admitting only precise checks. Ant takes Go's position: few warnings, each silenced by writing
  what you mean.

## Proposal

### The shape of a diagnostic

A diagnostic has:

| Field                                    | Meaning                                                                                                      |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| `path`, `line`, `column`, `start`, `end` | as today                                                                                                     |
| `severity`                               | `"error"` or `"warning"`                                                                                     |
| `code`                                   | the rule, from the catalog below                                                                             |
| `message`                                | as today                                                                                                     |
| `help`                                   | optional: one sentence on how to fix it, when the message does not already say                               |
| `fixes`                                  | optional: each a `title`, an `auto` flag, and `edits`, each a `path`, `start`, `end`, and replacement `text` |
| `related`                                | optional: other places that explain the diagnostic, each a `path`, `line`, `column`, and `message`           |

### Text output

The first line keeps its form, `path:line:column: message`, with two additions: a warning's message
starts with `warning: `, and every line ends with the code in brackets, as mypy prints it. `help`
and the title of each fix follow the marked source line, and `related` places follow those. The
count at the end names both severities. Everything goes to standard error, as errors do today, so
standard output stays empty:

```text
ledger.ant:33:19: Array[string] has no field or function join [no-member]
    │
 33 │   "${sign}${groups.join(",")}"
    │             ^^^^^^^^^^^
    = help: join is String.join, which takes the array first
    = fix: String.join(groups, ",")

probe.ant:3:1: warning: the Result of save is discarded, so its Err is lost [unused-result]
    │
  3 │ save("/nonexistent/dir/file")
    │ ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    = help: propagate it with try, or discard it on purpose with _ = save(...)

1 error, 1 warning
```

A `related` place prints as `= note: path:line:column: message`.

### JSON output

`-json` keeps one object per diagnostic and adds the fields above. Warnings are included. The span
is `groups.join`, which the carets mark; the fix replaces the whole call, `groups.join(",")`, so its
edit ends one column later than the span:

```json
{
  "path": "ledger.ant",
  "line": 33,
  "column": 19,
  "start": { "line": 33, "column": 13 },
  "end": { "line": 33, "column": 24 },
  "severity": "error",
  "code": "no-member",
  "message": "Array[string] has no field or function join",
  "help": "join is String.join, which takes the array first",
  "fixes": [
    {
      "title": "String.join(groups, \",\")",
      "auto": true,
      "edits": [
        {
          "path": "ledger.ant",
          "start": { "line": 33, "column": 13 },
          "end": { "line": 33, "column": 29 },
          "text": "String.join(groups, \",\")"
        }
      ]
    }
  ]
}
```

### Exit status

| Status | When                                                                                 |
| ------ | ------------------------------------------------------------------------------------ |
| `0`    | no error; warnings may have been reported                                            |
| `1`    | an error was reported, or under `-strict` a warning                                  |
| `2`    | a file cannot be loaded (now with all of its errors reported), or the usage is wrong |

A syntax error, a missing module, and a forbidden import still make a file one that cannot be
loaded, since `ant run` cannot run it, so they keep status `2`. A redeclared prelude name, which no
longer stops the file (Every error in one run, below), moves from `2` to `1`.

### Codes

A code names a rule. Several messages may share a code, a message may be reworded without changing
its code, and a code is never reused for another rule. A new rule gets a new code. A code may be
split into finer codes, such as `syntax` into `reserved-word` and others: the old code is then
retired, listed in the catalog as split, and never reused, so a tool that matched it learns from the
catalog which codes replaced it. The initial catalog groups the checker's present messages (about
180 formats in `internal/check` and `internal/resolve`, and the parser's) by the rule they enforce:

| Code                | Rule                                                                                        | Examples of today's messages                                               |
| ------------------- | ------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| `syntax`            | the source does not parse                                                                   | 'as' is a reserved word; bad escape                                        |
| `missing-module`    | an import names no readable file                                                            | cannot find module "./b.ant"                                               |
| `import`            | an import is not allowed: a cycle, a script, a declaration file, a test file from a library | import cycle through x.ant                                                 |
| `undefined`         | a name or type is not declared                                                              | x is not defined; unknown type Foo                                         |
| `not-exported`      | a name is used outside its module without `export`                                          | f is not exported by ./a.ant                                               |
| `redeclared`        | a name is declared twice in a scope, or shadows a prelude name                              | Entry is a prelude name and cannot be redeclared                           |
| `declaration-order` | a name is read before it is initialized                                                     | x is declared later in the module                                          |
| `wrong-kind`        | a name is used as a kind of thing it is not                                                 | Math is a namespace, not a value; T is a type, not a function              |
| `type-mismatch`     | a value's type does not fit where it is used                                                | cannot use string \| null where string is expected                         |
| `no-member`         | a field or function does not exist on a type or namespace                                   | Math has no member mod; Point has no field z                               |
| `needs-narrowing`   | a member exists only on some members of a union                                             | Shape has no field radius on every member: narrow the value first          |
| `not-nullable`      | `??` or `?.` on a value that cannot be `null`                                               | ?. needs a nullable left side, got Op                                      |
| `operands`          | an operator's operands do not fit it                                                        | + needs both operands of one type, got int \| null and int                 |
| `arguments`         | a call's arguments do not fit the function                                                  | Result.ok needs an argument for value; f takes 2, got 3                    |
| `needs-annotation`  | the checker cannot infer a type and asks for one                                            | cannot infer the element type of an empty array; annotate it               |
| `no-common-type`    | branches, elements, or returns have no type in common                                       | the branches of this if do not form a valid type                           |
| `bound`             | a type argument is outside its bound, or a type does not satisfy an interface               | T would be bool, which is not a member of its bound Ordered                |
| `record-literal`    | a record literal or spread is malformed                                                     | the field x is listed twice                                                |
| `missing-field`     | a record literal leaves out a field with no default                                         | Server literal is missing the field addr                                   |
| `missing-else`      | an `if` without `else` is used as a value: a union not fully handled, or a missing branch   | an if without else has type void when the condition is false, …            |
| `try`               | `try` outside a function that returns a compatible `Result`, or on a value that is not one  | try would return Err[Error], but the function returns int                  |
| `assign`            | an assignment to something that cannot be assigned                                          | cannot assign to x, a let; declare it with var to reassign it              |
| `lambda`            | a lambda's shape does not fit its expected type                                             | a lambda has at most one parameter, x; …                                   |
| `useless-test`      | an `is` or comparison whose result is known, or an `else` that cannot run                   | the test is always true; this else cannot run                              |
| `duration-equality` | `==` between duration literals of one length and different fields                           | 60s == 1m compares the fields as written, …                                |
| `markup`            | a value or attribute that markup does not take                                              | markup takes a string, a number, void, Html, or an array of them, got bool |
| `conversion`        | a conversion between types it does not apply to                                             | int converts a number or a named primitive type over a number, got string  |
| `void-value`        | `_` read, or `void` bound                                                                   | _ is write-only and cannot be read                                         |
| `reflect`           | `type(...)` used where it cannot be                                                         | type(...) takes a type, not a value                                        |
| `test`              | a test declaration that is not allowed                                                      | expected the test's name, a string, got identifier "c"                     |

The warnings below add four more. The catalog is part of the spec, so a tool can rely on it.

### Help and fixes

The checker attaches `help` where the message alone does not say what to write, and a fix where the
edit is clear. The first set, each from a mistake in the agent report:

| Code               | Written                        | Help                                                                                  | Fix                                               | Auto |
| ------------------ | ------------------------------ | ------------------------------------------------------------------------------------- | ------------------------------------------------- | ---- |
| `no-member`        | `groups.join(",")`             | join is String.join, which takes the array first                                      | `String.join(groups, ",")`                        | yes¹ |
| `no-member`        | `Math.mod(l, r)`               | Math has no mod; its functions nearest by name are …                                  |                                                   |      |
| `no-member`        | `Result.flatMap(f)`            | Result has no flatMap; chain results with try                                         |                                                   |      |
| `arguments`        | `Result.ok()`                  | the expected type is Result[void, Error]: pass void                                   | `Result.ok(void)`                                 | yes  |
| `type-mismatch`    | `Map{ [k]: v }`                | a computed key is written in parentheses                                              | `Map{ (k): v }`                                   | no³  |
| `syntax`           | `x as Op`                      | Ant has no casts: test the type with is, which narrows a local                        |                                                   |      |
| `test`             | `test c[0] { }`                | a test's name is a string; interpolate the value                                      | `test "${c[0]}" { }`                              | yes  |
| `syntax`           | `Result.ok({ a, b })`          | a record literal starts with its type's name                                          |                                                   |      |
| `type-mismatch`    | `json.value` after `is Map`    | a field path does not narrow; bind it to a local first (related: the test on line 28) |                                                   |      |
| `redeclared`       | `type Entry = …`               | Entry is the prelude type Entry[K, V], an entry of a map                              |                                                   |      |
| `missing-else`     | a union `if` chain             | the chain does not handle Blue                                                        | an `else if light is Blue` branch                 | no²  |
| `needs-annotation` | a binding the checker can type | (as today)                                                                            | the inferred type, as the language server adds it | yes  |
| `missing-field`    | a record literal               | (as today)                                                                            | the missing fields, with placeholders             | no²  |
| `undefined`        | a name another module exports  | (as today)                                                                            | the import, one fix per module that exports it    | no⁴  |

¹ When exactly one prelude function in the receiver type's namespaces, or `String`, `Array`, `Map`,
`Set`, and `Iter`, has the name and takes the receiver first. With more than one, each is a fix and
none is `auto`. ² A fix with a placeholder is never `auto`. ³ The author may have meant an array key
in a map of the wrong type. ⁴ A misspelled name can match an export by accident, and an import adds
a module whose `var` initializers then run (section 10).

A fix is `auto` only when it is the only fix offered and applying it removes the diagnostic, and in
addition:

- **For an error,** no other edit could be meant. The program does not check, so there is no
  behavior to keep; what makes the fix safe to apply unread is that the error admits one repair.
  `Result.ok()` where `Result[void, Error]` is expected can only mean `Result.ok(void)`, and a
  `needs-annotation` error can only be answered with the type the checker inferred. A misspelled
  name that happens to match an export can mean many things, so an import is never `auto`.
- **For a warning,** the program's behavior is unchanged. Removing an unused import qualifies only
  while the module stays imported, by another name of the same import or by another module of the
  program, since dropping a module drops its `var` initializers.

A fix is never `auto` in a file with a syntax error (Every error in one run, below).

### Warnings

A warning reports code the checker accepts that is almost always a mistake. To be added, a warning
must meet Go vet's bar: it catches a real bug, the bug happens in practice, and it rarely fires on
correct code. Each warning is silenced by writing the intent in the code, never by a comment or a
setting. The initial set:

| Code                  | Fires on                                                                                                                                                                                                                                                                                                            | Silenced by                                                                                                                                                    |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `unused-result`       | a value the program discards whose type is a `Result` or a union with a `Result` member, such as `Result[T, E] \| null`: a statement other than a block's value, the value of a block whose value is discarded, or the last value of a callback whose expected type returns `void`. Not anywhere in a `defer` block | `_ = f()`, or `try f()`                                                                                                                                        |
| `unused-import`       | an imported name the module never uses                                                                                                                                                                                                                                                                              | removing it                                                                                                                                                    |
| `unused-variable`     | a local `let` or `var`, or a binding of a destructuring, that is never read. Not `_`, a parameter, or a top-level declaration                                                                                                                                                                                       | removing it, or binding `_`                                                                                                                                    |
| `named-interpolation` | a value of a named primitive type in `"${...}"`, which renders as `UserId("u1")`. Markup already rejects one, since "rendering as a constructor call would put `UserId("u-1")` on a page" (section 5)                                                                                                               | the base type's conversion for the value, as `string(id)` for `type UserId = string` and `int(c)` for `type Count = int`; `String.from(x)` for the tagged form |

`defer` and `go` are treated differently on purpose, though section 8 gives neither block a result.
A deferred close whose error is dropped is the language's idiom: the error of a close that matters
is taken by closing explicitly before the block ends, and the deferred close then does nothing. A
goroutine that drops an `Err` has no such idiom and loses the error for good, so `go` blocks warn.
In a `go` block `try` is a compile error, so the silencer there is `_ = f()`.

`unused-import` has an `auto` fix under the condition above. `unused-result` offers `_ = f()`, and
`try f()` inside a function that returns a compatible `Result`, and marks neither, since only the
author knows whether the error should propagate. The other two offer fixes that are not `auto`.

### Every error in one run

`ant check` loads with the recovering parser (`loader.Config.Recover`, which the language server
uses today). It reports every syntax error of a file, then type-checks what parsed, as the editor
already does. The file still counts as one that cannot be loaded, so the status is still `2`, but
its other errors are no longer hidden.

What recovery drops is gone from the tree, and the uses in it with it. So in a file with a syntax
error, the checker reports errors only, computes no warnings, and attaches no `auto` fix: an
`unused-import` there may name an import the unparsed part needs, and removing it would delete live
code.

A `redeclared` prelude name no longer stops the file. It is reported, and in that module the
module's own declaration is used for the rest of the check, so its uses are checked and
`ant code rename` can find them.

**Cascades.** An error is not reported where it follows only from another error: when an operand,
callee, receiver, or initializer has a type the checker could not determine, or a name's binding was
itself an error, such as the parameters of a lambda rejected for its arity. No message names a type
the checker could not determine, so `?` never appears in a message.

### `ant check -explain`

`ant check -explain code` prints the code's rule, a short wrong example, and the right one, then
exits 0. An unknown code, or a retired one, which names the codes that replaced it, is a usage
error. The catalog in the spec is normative: it fixes each code's rule. The explanations, with their
examples, are entries of `docs/tools/check.md`, tested like every docs example, and must agree with
it.

### `ant code fix`

```sh
ant code fix [-diff] [-code list] [-root dir] [path...]
```

`fix` applies every `auto` fix in the modules under the paths, the current directory by default,
checks again, and repeats until no `auto` fix is left. The rounds run on the files held in memory,
and the files are written once, at the end, or under `-diff` printed as one unified diff of the
first version against the last, and not written. A file with a syntax error is left as it is.
`-code` limits both the fixes applied and the diagnostics reported afterwards to the listed codes,
comma-separated. `fix` prints how many fixes it applied in each file, as `rename` does, and the
diagnostics left go to standard error with their count.

If `auto` fixes remain after 10 rounds, `fix` stops, writes nothing, and reports the fixes still
pending: fixes that keep producing fixes are a bug in the checker, not something to apply.

The exit status follows `ant code`: 0 when no error is left, whether or not a fix was applied; 1
when errors are left, or warnings under `-strict`; 2 for a usage, read, or write error, or when the
10 rounds run out. Warnings left without `-strict` do not change the status, as in `ant check`.

### Interaction with other tools

- **`ant test`** reports errors as it does now, with the new fields in its `"error"` events, whose
  `severity` is always `"error"`. It does not report warnings, so that its output stays the test
  report; `ant check` and the editor report them. The field is kept so that an `"error"` event and
  an `ant check -json` object stay one shape.
- **`ant run`** is unchanged: it does not check.
- **The language server** publishes each diagnostic with its `code`, a severity of 2 for a warning,
  its `related` places as `relatedInformation`, and the `Unnecessary` tag on `unused-import` and
  `unused-variable`, which editors render faded. Its quick fixes are the diagnostic's fixes, marked
  preferred when `auto` or when it is the only fix offered, as an import from the one module that
  exports a name is today. The regular expressions in `codeaction.go` go away.
- **`ant code rename`** refuses programs with errors unless `-force` is given. An error that the
  rename itself removes, a `redeclared` name being renamed, no longer counts.

## Compatibility

Section 4 promises that `ant check` "prints nothing and exits with `0`" on a program with no error,
and that `-json` writes "one JSON object per error". Both change:

- In text mode, a program with no error may print warnings to standard error. The exit status is
  still `0`, and standard output stays empty. A script that treats any output on standard error as
  failure, rather than reading the status, starts failing.
- Under `-json`, a warning is an object with `"severity": "warning"`. A consumer that counts lines
  as errors must now read `severity`.
- A redeclared prelude name exits with `1`, not `2`.

The language server's clients see new fields only. No program that checks today fails without
`-strict`.

## Spec changes

- `spec/ant-lang.md` section 4, Checking and running: the diagnostic fields, the text form with the
  code and `warning: `, the count of both severities, the exit status, recovery and cascades, and
  `-strict` and `-explain`. "Prints nothing" and "one JSON object per error" become "per
  diagnostic". A pointer to the catalog.
- A new `spec/check.md`, beside `fmt.md`, `test.md`, and `doc.md`: the catalog of codes with their
  rules, the warnings with their exact conditions and how each is silenced, the help and fixes the
  checker must offer, and the definition of `auto`.
- `spec/ant-lang.md` section 2: the duration-literal sentence names its code, `duration-equality`.
- `spec/test.md` section 2: `"error"` events carry `severity`, `code`, `help`, `fixes`, and
  `related`; warnings are not reported.
- The language server's design document, section 4: codes, severities, related information, tags,
  and code actions from fixes.
- The `ant code` design document: the `fix` subcommand, and the `rename` exception.

## Implementation sketch

`loader.Diagnostic` gains `Code`, `Severity`, `Help`, `Fixes`, and `Related`, and the public
`ant.Diagnostic` follows. `Checker.errorf` and `errorIn` take a code as their first argument, which
touches each of their roughly 250 call sites once. The parser's `fail` sets `syntax`, and the loader
sets its own codes. Help and fixes are attached at the sites the table lists.

`cmd/ant/check.go` sets `Recover: true`. The checker already tolerates recovered trees, since the
language server runs it on them. Cascade suppression extends the existing `types.IsInvalid` checks
to the sites that still report on an invalid type.

Warnings run after checking, and only for files that parsed, from what `check.Info` already records
for the editor: `unused-import` and `unused-variable` from the uses of each declaration, which
`ant code refs` reads; and `unused-result` and `named-interpolation` from the types of statements,
callbacks, and interpolations.

To measure precision before the warnings ship, run them over every example in `ant/docs` (all
tested, many hundreds of programs) and `ant/examples`. A warning there is either a bug in a doc or a
false positive, and each must be resolved one way or the other.

## Non-goals

- **Configuration.** No file and no flag enable or disable individual warnings. A warning that needs
  one does not meet the bar.
- **Suppression comments.** No `// ant:ignore`. Every warning has a way to write the intent.
- **Style.** No naming, length, or complexity rules. `ant fmt` owns layout and nothing owns taste.
- **Warnings in `ant run`.** It does not check, by design (section 1, Types are advisory).
- **Numeric codes.** Names only.
- **Fixes that change behavior applied automatically.** `auto` is reserved for edits that cannot.

## Later

- `regex-group-index`: `m.groups[k]` with a constant `k` past the group count of the regex literal
  `m` was matched with. It needs the pattern to reach the match, which is local flow analysis.
- `unused-declaration`: a private top-level function or type of a library module that nothing uses.
- `float-to-int`: `int(x)` of a float that may have a fraction, which panics. Left out because the
  spec makes `int(x)` the deliberate exact conversion, so an author who wants the panic has no way
  to say so: `Math.floor` changes the result and `Math.toInt` adds a `null` branch. It also fires on
  calls like `int(Math.sqrt(n))` and `int(x * 100.0)` whose fractions the author has ruled out. It
  needs an asserting form, and a false-positive rate from the docs examples, before it meets the
  bar.
- Search by type for `help`: a matcher like a future `ant doc -find` would rank candidates by their
  whole signature, not only the first parameter.

## Decided in review

1. **Codes stay in the text output.** The code is what a reader passes to `-explain`, and Rust and
   mypy both print it.
2. **`ant test` does not report warnings.** Go's `go test` runs a subset of `go vet` and fails on
   it; Ant keeps the test report to tests.
3. **`useless-test` and `duration-equality` stay errors.** They report legal code that is almost
   always a mistake, but turning them into warnings would loosen a checker that rejects them today.
4. **`unused-variable` leaves out a script's top-level bindings.** Scripts are also explored by
   hand, where an unused binding is normal.
5. **`float-to-int` moved to Later**, which settles its precision question.
