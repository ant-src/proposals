# Proposal: code coverage

2026-11-02 · Anton

## Summary

Add statement and branch coverage to `ant test` and `ant run`. `-cover` counts how many times each
statement of the covered modules runs and how many times each way through a branch is taken, and
prints the share of each that ran, per file and in total. `-coverprofile file` also writes the
counts as an LCOV tracefile, which CI services, `genhtml`, and editors already read. A statement is
a stop point of the debugger (`dap.md` section 4), so the language has one definition of "a place
the program reaches". A branch is an expression that picks between two or more ways on: `if`, `&&`,
`||`, `??`, `?.`, `try`, a loop, and `select`. Branches matter more in Ant than in Go, because Ant
handles errors and nulls in expressions (`try`, `??`, `?.`) rather than in blocks, and statement
coverage cannot see inside an expression. There is one mode, counting, with no new syntax, no
exclusion comments, and no threshold flag.

## Motivation

`ant test` says whether the tests pass. It cannot say what they left out:

1. **Untested branches go unnoticed.** An `else` that handles a malformed input, the `Err` arm after
   a `try`, or a clause of a `select` that only fires on timeout can be wrong for months, because no
   test reaches it and nothing points at it.
2. **Untested files are invisible.** A module that no test file imports looks the same as one with
   full tests. Section 10 asks scripts to move their logic into libraries so that it can be tested;
   without coverage there is no way to see whether it was.
3. **Reviews and CI have no number.** Every hosted CI service and code review tool shows coverage on
   a pull request if it is given a report. Ant cannot produce one, so Ant projects get nothing
   there.
4. **Scripts are tested by running them.** A command-line script is often checked end to end, by
   running it on sample input. Which of its lines those runs reach is the same question, outside
   `ant test`.

Coverage is not a measure of test quality, and the proposal does not treat it as one: it adds no
threshold and no gate. It answers "which code did this run reach", which is a fact about the run.

## Prior art

| Tool                          | Unit                                | Modes                              | Report                                                                        | What Ant takes                                                                                         |
| ----------------------------- | ----------------------------------- | ---------------------------------- | ----------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Go `go test -cover`           | basic block, with statement counts  | `set` (default), `count`, `atomic` | `coverage: 78.3% of statements`; text profile; `go tool cover -func`, `-html` | The flags, `-cover` and `-coverprofile`, the one-line total, and statements as the unit. Not the modes |
| Go `go build -cover` (1.20)   | the same, for a whole program       | the same                           | `GOCOVERDIR`, merged with `go tool covdata`                                   | Coverage of a program run, not only of tests: `ant run -coverprofile`                                  |
| Go 1.22                       | —                                   | —                                  | packages with no test files reported at 0%                                    | Files no test loads count, at zero                                                                     |
| Python `coverage.py`          | line; branch (arcs) with `--branch` | —                                  | terminal table, HTML, XML, LCOV, JSON                                         | The per-file terminal table. Not `# pragma: no cover`                                                  |
| JavaScript Istanbul / c8 / V8 | statement, branch, function, line   | —                                  | LCOV, HTML, JSON                                                              | LCOV as the one file format; function records in it                                                    |
| Rust `cargo llvm-cov`, LLVM   | regions, with branch and MC/DC      | —                                  | LCOV, HTML, JSON                                                              | Nothing beyond LCOV: regions and MC/DC need a compiler's view of every subexpression                   |
| Deno `deno test --coverage`   | line, branch, function (from V8)    | —                                  | `deno coverage --lcov`, HTML                                                  | Confirmation that an interpreter-hosted test runner reports LCOV                                       |
| Dart `dart test --coverage`   | line                                | —                                  | JSON, LCOV via `format_coverage`                                              | Nothing new                                                                                            |

Four lessons:

- **Go's three modes are a compiler's trade-off.** `set` exists because a counter increment in
  compiled code is a measurable cost and a store of `1` is less. In a tree-walking interpreter a
  statement already costs far more than an atomic add, so Ant counts always and has no mode flag.
- **LCOV is the format everything reads.** Codecov, Coveralls, SonarQube, GitLab, `genhtml`, VS
  Code's Coverage Gutters, and JetBrains' JavaScript tooling all take LCOV. Go's own profile format
  is read mainly by Go's tools. Writing LCOV alone means one format to specify and no converter to
  ship.
- **Go can do without branches; Ant cannot.** Go reports statements only, and gets away with it
  because its error path is a block: `if err != nil { return err }` holds a statement that stays
  uncovered until a test reaches it. Ant's error path is `try f()` or `f() ?? return x`, inside one
  statement, so statement coverage marks it covered whether or not the `Err` or the `null` was ever
  seen. Python's `--branch`, Istanbul, and LLVM count branches for the same reason: conditions and
  short circuits inside one line.
- **Exclusion comments rot.** `# pragma: no cover` and `/* istanbul ignore next */` hide code from
  the report without proving it is unreachable, and they stay after the code changes. Go has none
  and its reports are still read. Ant has no pragmas and this proposal adds none.

## Proposal

### What is counted

A **statement** for coverage is a stop point of `dap.md` section 4 other than a pipeline stage:

- each statement of a block: a function body, a script body, a lambda, a `go` block, a `defer`
  block, a loop body, a branch of `if`, a clause of `select`, and a plain block;
- each top-level `var` and `let` of a library module, as its initializer runs.

Its **count** is the number of times the program reached it, counted before it runs, so a statement
that panics part way counts as reached. A statement is **covered** when its count is above zero.

This makes the unit the same one the debugger stops at and a breakpoint binds to, so "this line was
never reached" in a report and "a breakpoint here never hit" agree. Some consequences, all intended:

- `if c { a } else { b }` is one statement, plus `a` and `b` in their branches, so an untaken branch
  shows, also when the `if` is an expression inside another statement:
  `var x = if c { 1 } else { 2 }` has three statements.
- A lambda's body is a block, so `Array.map(xs, { x -> x * 2 })` with an empty `xs` leaves `x * 2`
  uncovered. So does markup's `{if user != null { <a>…</a> }}`, whose branch is a block.
- `a && b`, `a ?? b`, `a?.b`, `try f()`, and `f() ?? return x` are inside one statement: statement
  coverage does not say which way they went. Branches do (below).
- A pipeline stage is not a statement of its own. A stage is reached whenever its statement is,
  unless an earlier stage panics, and counting it would only split one line into several numbers.
- `func`, `type`, `import`, and `test` declarations are not statements, nor is a declaration file
  (section 10) or the prelude, which is not written in Ant. A function with an empty body has none.
- The top-level `var` and `let` of a library module run once, when the module is initialized, so a
  module loaded by any test has them covered. They still count, so a file that nothing loads reads
  0% and not "no statements".

### Branches

A **branch point** is an expression that goes on one of a fixed set of ways. Each way is a
**branch**, with a count of the times it was taken:

| Branch point                         | Branches, in this order                                             |
| ------------------------------------ | ------------------------------------------------------------------- |
| `if c { … }`, with or without `else` | `c` true; `c` false, also when there is no `else`                   |
| `a && b`                             | `b` evaluated; `b` skipped, because `a` was false                   |
| `a \|\| b`                           | `b` evaluated; `b` skipped, because `a` was true                    |
| `a ?? b`                             | `a` null, so `b` evaluated; `a` not null, so `b` skipped            |
| `a?.b`, `a?.[i]`, `f?.(x)`           | the left side not null; the left side null, so the result is `null` |
| `try a`                              | `a` is `Ok`; `a` is `Err`, so `try` returned                        |
| `for x in e`, `while c`              | the body ran at least once; the body did not run                    |
| `select`                             | one branch per clause, in source order, an `else` clause included   |

For the operators and loops, branch 0 is the way that evaluates more; `if` lists true first.

- Each `if` of an `else if` chain is a branch point of its own, and so is each `?.` of a chain
  `a?.b?.c`. An `if` inside markup, `{if open { … }}`, is an `if`.
- A loop's branches are counted once each time the loop starts, not per iteration, so "the body did
  not run" means the loop was reached with nothing to do. A `while true` loop has no branch point,
  since its body always runs.
- A branch point that is never reached has its branches neither taken nor not taken: they count
  toward the total as uncovered, and the profile writes them as `-` (below).
- A branch is **covered** when its count is above zero. `assert` is not a branch point: its failing
  way is a panic, which a test that passes never takes.
- Parameter and field defaults (section 4, Field defaults; section 6) are not branch points. Whether
  a default was used is a fact about a call, not a way through the function, and a default is
  usually a literal. A default expression that is never used goes unreported.

Each branch of `if` and `select` whose block holds a statement is also visible as that statement.
The branch adds the ways that have no statement of their own: an `if` with no `else`, an empty
block, and every operator in the table.

### Counting

Counting is per program run and atomic: statements and branches reached by goroutines, by parallel
tests, and by a goroutine that a leaked-goroutine check ends (section 10, Tests) all count, up to
the moment each stopped. A branch is counted when the way is decided, before the code on it runs:
`try` counts its `Err` before it returns.

### Which files are covered

**`ant test`.** The covered files are the modules found by the same search that finds test files
(`test.md` section 1): every `.ant` file under the given paths, or `.`, that is not a test file and
not a declaration file, in directories not starting with `.`. Every one of them is covered whether
or not a test loads it; one that none loads is parsed for its statements and reported at 0%. A
module that a test loads from outside the paths is not covered. Each test file runs as a program of
its own, and the counts of all of them are summed per statement and per branch. A covered file that
no test loads and that does not parse is reported as a file that could not be loaded (`test.md`
section 4), on standard error and with exit status `2`, and is left out of the totals.

So `ant test` in a project's root covers the whole project, and `ant test ./parser` covers only that
directory, the way `go test ./parser` covers that package. Scripts under the paths are covered and,
since a test cannot import a script, read 0% unless their logic has moved into a library: the report
says what section 10 recommends.

**`-coverfiles glob`.** Replaces that set with the `.ant` files under `.` that match `glob`, as
`Path.match` matches (`stdlib.md`), with `**` across directories; it may be given more than once.
`ant test -coverfiles 'src/**' ./tests` runs the tests in `./tests` and covers `./src`. Test files
and declaration files never count, even when they match.

**`ant run`.** The covered files are the modules the program loads, the script and its imports.
There is no search: a file the program does not load is not part of its run. So here `-coverfiles`
does not replace the set but narrows it: it keeps the loaded modules that match, and a matching file
the program does not load is not reported.

### Flags

Both commands take, before the file or paths:

| Flag                 | Meaning                                                                            |
| -------------------- | ---------------------------------------------------------------------------------- |
| `-cover`             | Count statements and branches, and print the coverage report (below) after the run |
| `-coverprofile file` | Count, and write an LCOV tracefile to `file`; it implies `-cover`                  |
| `-coverfiles glob`   | Cover the files that match instead (above); it implies `-cover`; repeatable        |

Without them nothing is counted, so a run without coverage costs nothing more than today.

### The report

`ant test` prints the report after its summary line, and `ant run` to standard error after the
program ends, so that the program's standard output stays its own:

```
ok    14 passed (0.08s)
coverage: 81.4% of statements, 63.0% of branches
  geometry.ant        100.0%   24/24    100.0%    6/6
  parse.ant            76.9%   40/52     55.0%   22/40
  scripts/build.ant     0.0%    0/11      0.0%    0/8
```

The first line is the total over all covered files, with one decimal, as Go prints it: covered
statements over statements and taken branches over branches, `100.0%` only when every one ran. Then
one line per covered file, as a path relative to the working directory, sorted by path, with its
shares and its counts; a file with no branch point shows `-` for branches. With no statement at all
the line is `coverage: [no statements]`, again as in Go, and with statements but no branch point the
line ends after the statements. Under `-json`, the report is one more event before `done`:

| `action`     | Other fields                                                                                                                                     |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `"coverage"` | `statements`, `statementsCovered`, `branches`, `branchesCovered`, and `files`: an array of objects with `file` and the same four counts, by path |

The report and the profile are written however the run ends once a program has run: when tests pass
or fail, since a failing run's coverage is still a fact about it; when a panic ends the program;
when `-timeout` runs out or an interrupt ends the run; and when `Os.exit` ends it. The command
writes them before it exits with the program's status; no deferred block runs, as `Os.exit`
promises. Under `ant test`, a file that could not be loaded is reported and the others still run and
count, so a run that exits with status `2` still has a report. Only a usage error, or an `ant run`
whose program could not be loaded, writes neither.

### The profile

`-coverprofile` writes an LCOV tracefile, one record per covered file, in the order of the report:

```
TN:
SF:/home/me/proj/parse.ant
FN:12,parse
FN:40,Token.next
FNDA:31,parse
FNDA:0,Token.next
FNF:2
FNH:1
BRDA:13,0,0,31
BRDA:13,0,1,0
BRDA:41,0,0,-
BRDA:41,0,1,-
BRF:4
BRH:1
DA:3,1
DA:13,31
DA:14,31
DA:15,0
LF:4
LH:3
end_of_record
```

- `SF` is the file's absolute path, as `dap.md` names sources, so tools do not need the working
  directory to find it.
- `DA:line,count` has one record for each line on which at least one statement starts, with the
  smallest count among them. A line where any statement never ran so reads as not covered, which is
  what an editor's gutter should show for `if c { a }` written on one line with `a` never reached.
  `LF` and `LH` count those lines.
- `FN:line,name` names each function declared at the top level of the module, as the debugger names
  them (`parse`, `Point.dist`), at the line of its `func`. `FNDA` is how many times its body was
  entered. Lambdas, local functions, and `go` blocks have no `FN` record; their statements have `DA`
  records like any other.
- `BRDA:line,block,branch,count` has one record per branch. `line` is the line of the branch point's
  keyword or operator (`if`, `&&`, `??`, `?.`, `try`, `for`, `while`, `select`); `block` numbers the
  branch points on that line from 0 in source order; `branch` numbers its branches from 0 in the
  order of the table above. `count` is `-` when the branch point was never reached. `BRF` and `BRH`
  count branches and taken branches.
- Lines, functions, and branches are LCOV's units. The statement share in the report is finer than
  lines: a file can show every line covered in the profile and still be below 100% of statements in
  the report, when two statements share a line. The terminal gives the exact numbers and the profile
  gives the picture.

The file is written whole when the run ends, replacing any file of that name. Merging profiles from
several runs is left to the tools that read LCOV, which all do it.

### Interaction with other tools

- **`ant dap`.** The debugger and coverage share their points but not their runs: `ant dap` takes no
  coverage flags in this proposal. A debug session is not a measurement.
- **`ant lsp` and editors.** Editors read the LCOV file. A later change can let the editor plugins
  run tests with coverage and show it with each editor's coverage view. VS Code's test coverage API
  takes statement ranges and branches, which LCOV's lines lose (Later, below).
- **Other backends.** Statements are stop points, which any backend that supports breakpoints has,
  and branch points are defined by the language's own operators. A JavaScript backend reports the
  same counts for the same run.

## Spec changes

- `spec/test.md` section 1: `-cover`, `-coverprofile`, `-coverfiles` in the flag table. A new
  section, Coverage, with statements, branch points and their order, which files, the report, the
  `coverage` JSON event, and the profile. Section 4: the report and profile on each exit path.
- `spec/ant-lang.md` section 10, Scripts, or wherever `ant run`'s flags are listed: the same three
  flags, pointing to `test.md`'s Coverage section for their meaning.
- `spec/dap.md` section 4: one sentence noting that coverage counts the statement stop points, so a
  change to the stop points is a change to coverage. Branch points are defined in `test.md` alone,
  since the debugger does not stop at them.

## Implementation sketch

The interpreter already calls `stopPoint` before each statement (`internal/interp/debug.go`). With
coverage on, the loader numbers the statement stop points of each covered module when it parses it,
gives each module a `[]atomic.Uint64`, and the statement node carries its index, so the hook is one
atomic add, with no map lookup. Branch points get the same treatment: each `if`, `&&`, `||`, `??`,
`?.`, `try`, loop, and `select` node carries the index of its first branch counter, and the
evaluator, which already knows which way it went, adds one to that index plus the way. Files under
the search that a test file does not load are parsed without being run, only to number their
statements and branches. `ant test` sums the counters of each test file's program into one table
keyed by path and counter index, and writes the report and profile from it. The parser's spans give
each statement's start line; the function records come from the top-level `func` declarations.

## Non-goals

- **Condition and MC/DC coverage.** Whether each operand of `a && b || c` independently changed the
  outcome. It is required in avionics (DO-178C) and rarely read anywhere else, it needs a record of
  which operands decided each evaluation rather than a counter, and few tools besides LLVM's report
  it.
- **Exclusion comments.** No `// coverage: ignore`. Code that should not count belongs outside the
  covered files, which `-coverfiles` already controls.
- **A threshold flag.** No `-coverfail 80`. A CI step can read the report or the profile, and a
  number to beat invites tests written for the number.
- **An HTML report.** `genhtml` and every CI service render LCOV. A built-in viewer is a second
  product to keep in step.
- **Per-test coverage**, which test reached which line. It is a different profile, with a dimension
  per test, and no common format carries it.
- **Coverage of the prelude.** It is not written in Ant.

## Later

- **Editor integration.** Run with coverage from the test code lenses and show the result inline.
- **A profile with exact ranges.** A JSON profile with each statement's and branch's start and end,
  for editors whose coverage views take ranges, as VS Code's does.
- **Coverage of `ant dap` launches**, if users want a debug launch and a measurement in one.

## Open questions

1. **Per-file lines by default.** The report lists every covered file. A large project might want
   only the total, with the table under `-v`. Go prints one line per package and nothing finer.
2. **Scripts in the default set.** Covering scripts under the paths at 0% is honest but noisy for a
   project with an `examples/` directory. The alternative is to leave scripts out unless
   `-coverfiles` names them.
3. **`ant run` report destination.** Standard error keeps the program's output clean but mixes the
   report with the program's own errors. A `-coverprofile` without `-cover` could print nothing.
