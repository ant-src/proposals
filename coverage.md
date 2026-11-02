# Proposal: code coverage

2026-11-02 · Anton

## Summary

Add statement coverage to `ant test` and `ant run`. `-cover` counts how many times each statement of
the covered modules runs and prints the share that ran, per file and in total. `-coverprofile file`
also writes the counts as an LCOV tracefile, which CI services, `genhtml`, and editors already read.
A statement is a stop point of the debugger (`dap.md` section 4), so the language has one definition
of "a place the program reaches" and coverage adds a counter to it. There is one mode, counting,
with no new syntax, no exclusion comments, and no threshold flag.

## Motivation

`ant test` says whether the tests pass. It cannot say what they left out:

1. **Untested branches go unnoticed.** An `else` that handles a malformed input, the `Err` arm after
   a `try`, or a `case` of a `select` that only fires on timeout can be wrong for months, because no
   test reaches it and nothing points at it.
2. **Untested files are invisible.** A module that no test file imports looks the same as one with
   full tests. Section 10 asks scripts to move their logic into libraries so that it can be tested;
   without coverage there is no way to see whether it was.
3. **Reviews and CI have no number.** Every hosted CI service and code review tool shows coverage on
   a pull request if it is given a report. Ant cannot produce one, so Ant projects get nothing there.
4. **Scripts are tested by running them.** A command-line script is often checked end to end, by
   running it on sample input. Which of its lines those runs reach is the same question, outside
   `ant test`.

Coverage is not a measure of test quality, and the proposal does not treat it as one: it adds no
threshold and no gate. It answers "which code did this run reach", which is a fact about the run.

## Prior art

| Tool                                  | Unit                                          | Modes                       | Report                                                       | What Ant takes                                                                                      |
| ------------------------------------- | --------------------------------------------- | --------------------------- | ------------------------------------------------------------ | --------------------------------------------------------------------------------------------------- |
| Go `go test -cover`                   | basic block, with statement counts            | `set` (default), `count`, `atomic` | `coverage: 78.3% of statements`; text profile; `go tool cover -func`, `-html` | The flags, `-cover` and `-coverprofile`, the one-line total, and statements as the unit. Not the modes |
| Go `go build -cover` (1.20)           | the same, for a whole program                 | the same                    | `GOCOVERDIR`, merged with `go tool covdata`                  | Coverage of a program run, not only of tests: `ant run -coverprofile`                               |
| Go 1.22                               | —                                             | —                           | packages with no test files reported at 0%                   | Files no test loads count, at zero                                                                  |
| Python `coverage.py`                  | line; branch (arcs) with `--branch`           | —                           | terminal table, HTML, XML, LCOV, JSON                        | The per-file terminal table. Not `# pragma: no cover`                                               |
| JavaScript Istanbul / c8 / V8         | statement, branch, function, line             | —                           | LCOV, HTML, JSON                                             | LCOV as the one file format; function records in it                                                 |
| Rust `cargo llvm-cov`, LLVM           | regions, with branch and MC/DC                | —                           | LCOV, HTML, JSON                                             | Nothing beyond LCOV: regions and MC/DC need a compiler's view of every subexpression                |
| Deno `deno test --coverage`           | line, branch, function (from V8)              | —                           | `deno coverage --lcov`, HTML                                 | Confirmation that an interpreter-hosted test runner reports LCOV                                    |
| Dart `dart test --coverage`           | line                                          | —                           | JSON, LCOV via `format_coverage`                             | Nothing new                                                                                         |

Three lessons:

- **Go's three modes are a compiler's trade-off.** `set` exists because a counter increment in
  compiled code is a measurable cost and a store of `1` is less. In a tree-walking interpreter a
  statement already costs far more than an atomic add, so Ant counts always and has no mode flag.
- **LCOV is the format everything reads.** Codecov, Coveralls, SonarQube, GitLab, `genhtml`, VS
  Code's Coverage Gutters, and JetBrains IDEs all take LCOV. Go's own profile format is read mainly
  by Go's tools. Writing LCOV alone means one format to specify and no converter to ship.
- **Exclusion comments rot.** `# pragma: no cover` and `/* istanbul ignore next */` hide code from
  the report without proving it is unreachable, and they stay after the code changes. Go has none
  and its reports are still read. Ant has no pragmas and this proposal adds none.

## Proposal

### What is counted

A **statement** for coverage is a stop point of `dap.md` section 4 other than a pipeline stage:

- each statement of a block: a function body, a script body, a lambda, a `go` block, a `defer`
  block, a loop body, a branch of `if`, a clause of `select`, a test body, and a plain block;
- each top-level `var` and `let` of a library module, as its initializer runs.

Its **count** is the number of times the program reached it, counted before it runs, so a statement
that panics part way counts as reached. A statement is **covered** when its count is above zero.

This makes the unit the same one the debugger stops at and a breakpoint binds to, so "this line was
never reached" in a report and "a breakpoint here never hit" agree. Some consequences, all intended:

- `if c { a } else { b }` is one statement, plus `a` and `b` in their branches, so an untaken branch
  shows, also when the `if` is an expression inside another statement: `var x = if c { 1 } else { 2
  }` has three statements.
- A lambda's body is a block, so `Array.map(xs, { x -> x * 2 })` with an empty `xs` leaves `x * 2`
  uncovered. So does markup's `{if user != null { <a>…</a> }}`, whose branch is a block.
- `a && b`, `a ?? b`, `a?.b`, and a `?? return` are inside one statement: which side ran is not
  recorded, as in Go. Branch coverage is left for later (Later, below).
- A pipeline stage is not a statement of its own. A stage is reached whenever its statement is,
  unless an earlier stage panics, and counting it would only split one line into several numbers.
- `func`, `type`, `import`, and `test` declarations are not statements, nor is a declaration file
  (section 10) or the prelude, which is not written in Ant. A function with an empty body has none.
- The top-level `var` and `let` of a library module run once, when the module is initialized, so a
  module loaded by any test has them covered. They still count, so a file that nothing loads reads
  0% and not "no statements".

Counting is per program run and atomic: statements reached by goroutines, by parallel tests, and by
a goroutine that a leaked-goroutine check ends (section 10, Tests) all count, up to the moment each
stopped.

### Which files are covered

**`ant test`.** The covered files are the modules found by the same search that finds test files
(`test.md` section 1): every `.ant` file under the given paths, or `.`, that is not a test file and
not a declaration file, in directories not starting with `.`. Every one of them is covered whether
or not a test loads it; one that none loads is parsed for its statements and reported at 0%. A
module that a test loads from outside the paths is not covered. Each test file runs as a program of
its own, and the counts of all of them are summed per statement.

So `ant test` in a project's root covers the whole project, and `ant test ./parser` covers only that
directory, the way `go test ./parser` covers that package. Scripts under the paths are covered and,
since a test cannot import a script, read 0% unless their logic has moved into a library: the report
says what section 10 recommends.

**`-coverfiles glob`.** Replaces that set with the `.ant` files under `.` that match `glob`, as
`Path.match` matches (`stdlib.md`), with `**` across directories; it may be given more than once.
`ant test -coverfiles 'src/**' ./tests` runs the tests in `./tests` and covers `./src`. Test files
and declaration files never count, even when they match.

**`ant run`.** The covered files are the modules the program loads, the script and its imports.
There is no search: a file the program does not load is not part of its run. `-coverfiles` narrows
the set the same way.

### Flags

Both commands take, before the file or paths:

| Flag                 | Meaning                                                                         |
| -------------------- | ------------------------------------------------------------------------------- |
| `-cover`             | Count, and print the coverage report (below) after the run                      |
| `-coverprofile file` | Count, and write an LCOV tracefile to `file`; it implies `-cover`               |
| `-coverfiles glob`   | Cover the files that match instead (above); it implies `-cover`; repeatable     |

Without them nothing is counted, so a run without coverage costs nothing more than today.

### The report

`ant test` prints the report after its summary line, and `ant run` to standard error after the
program ends, so that the program's standard output stays its own:

```
ok    14 passed (0.08s)
coverage: 81.4% of statements
  geometry.ant        100.0%   24/24
  parse.ant            76.9%   40/52
  scripts/build.ant     0.0%    0/11
```

The first line is the total over all covered files, with one decimal, as Go prints it: covered
statements over statements, `100.0%` only when every statement ran. Then one line per covered file,
as a path relative to the working directory, sorted by path, with its share and its counts. With no
covered statement at all the line is `coverage: [no statements]`, again as in Go. Under `-json`,
the report is one more event before `done`:

| `action`     | Other fields                                                                             |
| ------------ | ---------------------------------------------------------------------------------------- |
| `"coverage"` | `covered`, `statements`, and `files`: an array of `{ file, covered, statements }` by path |

The report is printed when the run ends normally, after test failures too: a failing run's coverage
is still a fact about it. It is not printed when the command exits with status `2`, or when
`Os.exit` ends it, which ends the command at once (`test.md` section 4); the profile is still
written in the second case, since a script that exits with a status has run to its end.

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
- Lines and functions are LCOV's units. The total in the report is over statements, which is finer:
  a file can show every line covered in the profile and still below 100% in the report, when two
  statements share a line. The terminal gives the exact number and the profile gives the picture.

The file is written whole when the run ends, replacing any file of that name. Merging profiles from
several runs is left to the tools that read LCOV, which all do it.

### Interaction with other tools

- **`ant dap`.** The debugger and coverage share their points but not their runs: `ant dap` takes no
  coverage flags in this proposal. A debug session is not a measurement.
- **`ant lsp` and editors.** Editors read the LCOV file. A later change can let the editor plugins
  run tests with coverage and show it with each editor's coverage view (VS Code's test coverage API
  takes per-line counts).
- **Other backends.** The definition is in terms of stop points, which any backend that supports
  breakpoints has. A JavaScript backend reports the same counts for the same run.

## Spec changes

- `spec/test.md` section 1: `-cover`, `-coverprofile`, `-coverfiles` in the flag table. A new
  section, Coverage, with what is counted, which files, the report, the `coverage` JSON event, and
  the profile. Section 4: the report and profile on each exit path.
- `spec/ant-lang.md` section 10, Scripts, or wherever `ant run`'s flags are listed: the same three
  flags, pointing to `test.md`'s Coverage section for their meaning.
- `spec/dap.md` section 4: one sentence noting that coverage counts the statement stop points, so a
  change to the stop points is a change to coverage.

## Implementation sketch

The interpreter already calls `stopPoint` before each statement (`internal/interp/debug.go`). With
coverage on, the loader numbers the statement stop points of each covered module when it parses it,
gives each module a `[]atomic.Uint64`, and the statement node carries its index, so the hook is one
atomic add, with no map lookup. Files under the search that a test file does not load are parsed
without being run, only to number their statements. `ant test` sums the counters of each test file's
program into one table keyed by path and statement index, and writes the report and profile from it.
The parser's spans give each statement's start line; the function records come from the top-level
`func` declarations.

## Non-goals

- **Branch, condition, and MC/DC coverage.** They need a point for each side of `&&`, `??`, `?.`,
  and each arm of a match, which the language does not otherwise have. See Later.
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

- **Branch coverage.** LCOV has `BRDA` records. If added, the points would be each operand of `&&`
  and `||` that may be skipped, the right side of `??`, the access after `?.`, and each branch of
  `if` and clause of `select`, including the implicit `else`. It needs those points defined once,
  next to the stop points.
- **Editor integration.** Run with coverage from the test code lenses and show the result inline.
- **Coverage of `ant dap` launches**, if users want a debug launch and a measurement in one.

## Open questions

1. **Per-file lines by default.** The report lists every covered file. A large project might want
   only the total, with the table under `-v`. Go prints one line per package and nothing finer.
2. **Scripts in the default set.** Covering scripts under the paths at 0% is honest but noisy for a
   project with an `examples/` directory. The alternative is to leave scripts out unless
   `-coverfiles` names them.
3. **`ant run` report destination.** Standard error keeps the program's output clean but mixes the
   report with the program's own errors. A `-coverprofile` without `-cover` could print nothing.
