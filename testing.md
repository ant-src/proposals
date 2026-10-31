# Proposal: `ant test`

2026-10-31 · Anton · Draft

## Summary

Add tests to the language. A test lives in a file whose name ends in `_test.ant` and is a
declaration, `test "name" { ... }`, whose body runs under `ant test`. The assertion is `assert`
(section 8): a failure is a panic, the panic ends the test, and the runner reports it with the
values `assert` already shows. A `test` written inside a test is a subtest, so a table of cases is a
`for` loop whose rows are reported, filtered and failed one by one. A small `Test` namespace covers
what a body cannot do alone: skip, opt in to running in parallel, a temporary directory, and a
context that ends with the test. A test file next to `foo.ant`, `foo_test.ant`, sees `foo.ant`'s
private names. Tests run one at a time in source order unless a test opts in to parallel. Goroutines
a test starts belong to it: their panics fail it, and one still running when it ends fails it too.

```ant
// stack_test.ant, beside stack.ant
test "push then pop returns the value" {
  var s = Stack.new()
  Stack.push(s, 1)
  assert(Stack.pop(s) == 1)
}

test "pop on an empty stack panics" {
  var r = Panic.catch({ Stack.pop(Stack.new()) })
  assert(r is Err)
  assert(String.contains(r.error.message, "pop on empty stack"))
}

test "parse" {
  var cases = [
    { input: "1", want: 1 },
    { input: "-7", want: -7 },
    { input: "0x10", want: 16 },
  ]
  for c in cases {
    test "${c.input}" {
      assert(parse(c.input) == c.want)
    }
  }
}
```

```
$ ant test
--- FAIL: stack_test.ant: parse/0x10 (0.00s)
    assertion failed: parse(c.input) == c.want (0 == 16)
      c.input = "0x10"
      c.want = 16
      at stack_test.ant:21:7
FAIL  1 failed, 4 passed (0.02s)
```

## Motivation

1. **There is no way to write a test.** `ant/TODO.md` lists it first among the gaps. The
   repository's own corpus compares output through `// expect` trailers, which suits the
   implementation, not a user's program.
2. **The pieces exist.** `assert` shows the values a condition was computed from and narrows
   (section 8), `Panic.catch` turns a panic into a value with a structured trace (section 8,
   Catching a panic), `defer` cleans up, and the runtime already ends a goroutine's waits on a
   cancelled context (section 9). What is missing is a place to put a test, a way to find and run
   it, and a report.
3. **Every runtime ships a runner now.** Go, Rust, Zig, Gleam, Swift, Deno, Bun and Node all include
   one. A language without one gets several incompatible libraries, each with its own assertion
   style, and editors cannot offer "run this test" until one wins.
4. **The editor and debugger are ready.** `ant lsp` can show "run | debug" above a test, and
   `ant dap` can launch one, if a test is something the tools can find without running the file.

## Prior art

### Frameworks and what they offer

| Feature                  | Go `testing`                        | Rust libtest (+ crates)                         | Zig                       | Swift Testing                            | pytest                                | Jest / Vitest                            | node:test / Deno                          | JUnit 6 / Kotest                  | ExUnit                         | Gleam                  |
| ------------------------ | ----------------------------------- | ----------------------------------------------- | ------------------------- | ---------------------------------------- | ------------------------------------- | ---------------------------------------- | ----------------------------------------- | --------------------------------- | ------------------------------ | ---------------------- |
| Declaration              | `func TestX(t *testing.T)`          | `#[test] fn x()`                                | `test "name" { }`         | `@Test("name") func x()`                 | `def test_x():`                       | `test("name", fn)`                       | `test("name", fn)`, `Deno.test`           | `@Test`; Kotest `"name" { }`      | `test "name" do`               | `pub fn x_test()`      |
| Where                    | `*_test.go`                         | `mod tests` in file, `tests/`                   | same file                 | test target                              | `test_*.py`                           | `*.test.ts`, Vitest in-source            | `*.test.js`, `*_test.ts`                  | `src/test/`                       | `test/*_test.exs`              | `test/`                |
| Private access           | same package                        | child module                                    | same file                 | `@testable import`                       | everything public                     | exports only                             | exports only                              | package-private                   | public only                    | public only            |
| Assertion                | `if` + `t.Errorf`; testify          | `assert!`, `assert_eq!`                         | `try expect`, `expectEqual` | `#expect`, `#require`                  | `assert` (rewritten to show values)   | `expect(x).toBe(y)`, ~50 matchers        | `node:assert`, `@std/assert`              | `assertEquals`; Kotest `shouldBe` | `assert` shows both sides      | `assert` keyword       |
| Keep going after failure | `t.Error`                           | no                                              | no                        | `#expect`                                | no                                    | `expect.soft`                            | no                                        | `assertAll`, soft assertions      | no                             | no                     |
| Subtests / groups        | `t.Run`                             | modules                                         | no                        | `@Suite`, nested                         | classes                               | `describe`                               | nested `test`, `t.step`                   | `@Nested`, Kotest specs           | `describe`                     | no                     |
| Parameterized            | table + `t.Run`                     | `rstest`                                        | loop                      | `@Test(arguments:)`                      | `@parametrize`                        | `test.each`                              | loop + nested                             | `@ParameterizedTest`              | loop + `test`                  | loop                   |
| Setup, teardown          | `t.Cleanup`, `TestMain`             | `Drop`                                          | `defer`                   | `init`, `deinit`                         | fixtures (DI by name, scopes)         | `beforeEach`, `test.extend`              | `before`, `after`, `t.after`              | `@BeforeEach`, extensions         | `setup`, `on_exit`             | none                   |
| Skip                     | `t.Skip`                            | `#[ignore]`                                     | `error.SkipZigTest`       | `.disabled`, `.enabled(if:)`             | `skip`, `skipif`                      | `.skip`, `.todo`                         | `skip`, `todo`                            | `@Disabled`, `assume`             | `@tag :skip`                   | no                     |
| Expected failure         | no                                  | `#[should_panic]`                               | no                        | `withKnownIssue`                         | `xfail`                               | `.failing`                               | `expectFailure`                           | `assertThrows`                    | `assert_raise`                 | no                     |
| Focus                    | no                                  | no                                              | no                        | no                                       | no                                    | `.only`                                  | `only`                                    | no                                | no                             | no                     |
| Filter                   | `-run A/b`, `-skip`                 | substring                                       | `--test-filter`           | `--filter`, tags                         | `-k expr`, `-m marker`                | `-t`, file glob                          | `--test-name-pattern`                     | tags, `--select`                  | `--only tag`                   | no                     |
| Parallel                 | opt in, `t.Parallel()`              | default, threads                                | no                        | default, `.serialized`                   | plugin                                | per file                                 | per file; `concurrency`                   | opt in                            | `async: true` per module       | per module (BEAM)      |
| Time limit               | `-timeout`, dumps all stacks        | nextest                                         | no                        | `.timeLimit`                             | plugin                                | `testTimeout`                            | `timeout`                                 | `@Timeout`                        | `@tag timeout:`                | no                     |
| Leaks                    | `goleak`, `synctest`   | no                                              | `testing.allocator`       | no                                       | no                                    | `--detectOpenHandles`                    | Deno sanitizers (ops, resources, exit)    | no                                | no                             | no                     |
| Fake time                | `testing/synctest`                  | tokio `pause`                                   | no                        | no                                       | `freezegun`                           | `vi.useFakeTimers`                       | `mock.timers`, `FakeTime`                 | no                                | no                             | no                     |
| Mocks                    | interfaces, hand-written            | `mockall`                                       | no                        | no                                       | `unittest.mock`                       | `vi.fn`, `vi.mock`                       | `mock.fn`, `mock.module`                  | Mockito, MockK                    | `Mox` (behaviours)             | no                     |
| Snapshots                | golden files, `-update` convention  | `insta`                                         | no                        | no                                       | `syrupy`                              | `toMatchSnapshot`, inline                | `t.assert.snapshot`                       | no                                | no                             | `birdie`               |
| Output check / doctests  | `Example` + `// Output:`            | doctests                                        | doctests                  | no                                       | `doctest`                             | no                                       | no                                        | no                                | `doctest`                      | no                     |
| Benchmarks               | `B.Loop`                            | `criterion`                                     | no                        | no                                       | plugin                                | `bench`                                  | `Deno.bench`                              | JMH                               | Benchee                        | no                     |
| Fuzz / property          | `F.Fuzz`, built in                  | `proptest`, `cargo-fuzz`                        | `std.testing.fuzz`        | no                                       | Hypothesis                            | `fast-check`                             | `fast-check`                              | jqwik, Kotest                     | StreamData                     | `qcheck`               |
| Coverage                 | `-cover`                            | `llvm-cov`                                      | no                        | Xcode                                    | `pytest-cov`                          | built in                                 | built in (Node: experimental)             | JaCoCo                            | `--cover`                      | no                     |
| Machine output           | `-json` events                      | JSON (unstable)                                 | no                        | JSON event stream                        | JUnit XML                             | reporters, JUnit XML                     | TAP, JUnit XML                            | XML, Open Test Reporting          | formatters                     | no                     |
| Print in a test          | shown on failure or `-v`            | captured, shown on failure                      | to stderr                 | shown                                    | captured, shown on failure            | shown                                    | shown                                     | shown                             | `capture_io`                   | shown                  |

### What the field has converged on

- **The assertion is a boolean that explains itself.** Gleam deprecated gleeunit's `should` module
  for its `assert` keyword; Swift Testing replaced XCTest's forty `XCTAssert*` functions with
  `#expect` and `#require`; pytest is chosen largely for its rewritten `assert`; Kotlin added a
  power-assert compiler plugin. Matcher libraries (Jest, AssertJ, Hamcrest, Kotest) remain where
  the language cannot show a condition's values. Ant's `assert` can.
- **Tests are statically known.** Annotations (Rust, Swift, JUnit), declarations (Zig) and naming
  conventions (Go, pytest, Gleam) let editors list tests without running code, which every
  "run | debug" code lens depends on. Registration by call (Jest, node:test, ExUnit) gives dynamic
  tests for free but needs the file executed to know what is in it.
- **Plain code over lifecycle hooks.** Go's `t.Cleanup`, Zig's `defer`, Swift's `init`/`deinit` of a
  suite. pytest fixtures are powerful and also the most magic part of pytest. `beforeEach` chains in
  nested `describe` blocks are a common source of order-dependent tests.
- **Each row of a table is a test.** Go's `t.Run`, Swift's `arguments:`, pytest's `parametrize` and
  Jest's `test.each` exist so that a table reports which row failed, keeps running the others, and
  lets a single row be rerun by name.
- **Concurrent work must not outlive its test.** Deno's sanitizers fail a test that leaves an
  operation pending or a file open; Zig's test allocator fails a leak; Go has `goleak` and, since
  1.25, `testing/synctest`; Jest has `--detectOpenHandles`. A test that leaves work running makes
  the next test fail for no visible reason.
- **Opt-in parallelism inside a file, isolation between files.** Go and JUnit run tests in a package
  sequentially unless a test opts in; Jest and node:test isolate files. Default-parallel runners
  (Rust, Swift) need per-test escape hatches for every piece of global state: Go's `t.Setenv` and
  `t.Chdir` refuse to run in a parallel test for this reason.
- **Quiet on success.** Go prints `ok` per package and only failures; pytest prints dots. Output
  printed by a passing test is noise.

## Declaring a test

Two forms fit Ant. Both put tests in `*_test.ant` files, both are statically known, and both run the
body under `Panic.catch`. They differ in what a test is.

### Option A: a `test` declaration (proposed)

```ant
test "an empty stack has length zero" {
  assert(Stack.length(Stack.new()) == 0)
}
```

`test` followed by a string literal and a block. At the top level of a test file it declares a test;
inside a test body it runs a subtest (Subtests, below).

### Option B: functions named `test...`

```ant
func testEmptyStackHasLengthZero(t: Test) {
  assert(Stack.length(Stack.new()) == 0)
}
```

Go's form: any top-level `func` in a test file whose name starts with `test` followed by an upper-case
letter or `_`, taking a `Test` or nothing. Subtests are `Test.run(t, "name", func(t: Test) { ... })`.

### Comparison

|                       | A: `test "name" { }`                                                                | B: `func testX(t: Test)`                                                                              |
| --------------------- | ----------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Name                  | A sentence: `"pop on an empty stack panics"`                                        | An identifier: `testPopOnEmptyStackPanics`                                                            |
| Grammar               | One new form; `test` becomes a keyword in test files only                           | None                                                                                                  |
| Discovery rule        | The declaration is the test                                                         | A naming rule. Ant's case carries no meaning (section 2), so `testing()` or `tests()` need a rule too |
| Callable by mistake   | No; a test is not a value                                                           | Yes; a helper can call a test, and a test can be passed around                                        |
| Subtests              | The same form nested: `for c in cases { test "${c.name}" { ... } }`                 | `Test.run(t, "name", func(t: Test) { ... })`, the `t` threaded through                                |
| Handle `t`            | Not needed: `Test.skip()` acts on the calling goroutine's test                      | Every helper that skips or makes a temp dir takes `t`                                                 |
| `try` in the body     | A test body may return an `Err`, which fails the test, as a script body may         | The function's signature says `void`; `try` needs `Result[void, Error]` spelled out                   |
| Tools                 | LSP lens on `test`; DAP launches by name                                            | Same, on a `func`                                                                                     |
| Familiar from         | Zig, Kotest, ExUnit, Jest's names, Pyret `check:`                                   | Go, pytest, Gleam                                                                                     |
| Cost                  | A keyword and a statement form in the spec, parser, formatter and checker           | A naming convention the checker must know, and the `Test` type as a parameter                         |

Option A is proposed. It is the integrated form: a test is a thing in the language, not a function
that a tool treats specially by its spelling, so it cannot be called, passed, or mistaken for a
helper. Its names read as sentences in a report. The nested form gives tables their rows without a
second API, and the body's script-like boundary lets `try` fail a test with the error. Option B's
advantage is that it needs no grammar, and for Ant that is the only one.

## The proposed design

### Test files

- A file whose name ends in `_test.ant` is a **test file**. `ant run` and `import` in a file that is
  not a test file refuse it; a test file may import another test file, for shared helpers.
- A test file is a library (section 10, Scripts): `import`, `func`, `type`, `var` and `test`
  declarations, no top-level statements and no `export`.
- **Companion scope.** `foo_test.ant` beside `foo.ant` is `foo.ant`'s companion: its top-level scope
  is nested in `foo.ant`'s, so it sees every top-level name of `foo.ant`, exported or not, and the
  names `foo.ant` imports, without importing them. A declaration in the test file that would shadow
  one of them is an error, as shadowing an import is. `foo.ant` is initialized first, once.
- A test file with no companion, such as `api_test.ant` for a whole directory, sees only what it
  imports: a black-box test, as Go's `package foo_test`.
- A script cannot have a companion: its top-level names are locals of its body (section 10,
  Scripts). Logic worth testing moves into a library the script imports.

### The `test` declaration

```
test "name" { body }
```

1. **Where.** At the top level of a test file it declares a test. Inside the body of a test, at any
   depth of blocks and loops but not inside a nested function or lambda, it runs a subtest. Anywhere
   else it is a syntax error, and `test` is a keyword only in test files, so `test` remains a plain
   name elsewhere and `Regex.test` is unaffected.
2. **Name.** A top-level test's name is a string literal without interpolation, unique in its file
   and not empty, so tools know every top-level test without running anything. A subtest's name may
   interpolate. Two subtests of one parent with the same name get ` #2`, ` #3` appended, as Go's
   `t.Run` does. The full name joins the names with `/`: `parse/0x10`.
3. **Body.** The body is a boundary, like a script body (section 10): `return` ends the test, `defer`
   registers on it, `try` returns an `Err` from it, and the checker allows `void` or an `Err` as its
   result. A test is not a value; nothing can call it.
4. **Outcome.** A test **fails** when a panic leaves its body (an `assert` or any other), when its body
   returns an `Err`, when a goroutine it owns panics, when a goroutine it owns is still running after
   it ends (Goroutines, below), or when one of its subtests fails. It is **skipped** when it calls
   `Test.skip`. Otherwise it **passes**. A failure ends the body at once, as any panic does, after
   its deferred blocks run: there is no failure that records and goes on. A failed subtest does not
   end its parent's body, which goes on to the next row and fails when it ends.
5. **Order.** Top-level tests run one at a time in source order, subtests in the order their
   statements run, except tests that call `Test.parallel()`.

### The `Test` namespace

The functions act on the **current test**: the test whose body the calling goroutine is running, or
that owns the goroutine. Called outside any test, each panics with `not in a test`. So a helper
function can skip or make a temporary directory without a handle passed to it.

```
Test.skip(reason: string): never          // ends the test as skipped; its deferred blocks run
Test.parallel(): void                     // opts the test in to running alongside other parallel tests
Test.tempDir(): string                    // a fresh directory, removed with its contents when the test ends
Test.context(): Context                   // cancelled when the test's body ends, before its deferred blocks
Test.name(): string                       // the full name, "parse/0x10"
```

- **`Test.skip`** in a subtest skips the subtest. A test that is skipped after it failed stays failed.
- **`Test.parallel`** follows Go's `t.Parallel`. The test stops at the call until every sequential
  test of the same parent (or of the file, for a top-level test) has finished, then runs alongside
  the parent's other parallel tests, at most `-parallel n` at a time. A parent ends after all of its
  parallel subtests. Tests in different files never run at once.
- **`Test.tempDir`** returns a new directory each call, under `File.tempDir`, removed after the
  test's deferred blocks.
- **`Test.context`** is the context to give blocking calls that must stop when the test does.

Nothing else is needed for the common cases: `assert` checks, `Panic.catch` checks that code panics,
`defer` cleans up, and a `for` over a table makes rows.

### Goroutines

A goroutine started while a test is current belongs to that test, and so does every goroutine it
starts.

- A panic in one of them fails the test, reported with its trace, instead of only being written to
  standard error (section 9, Panics in goroutines). The test's body is not interrupted; it fails
  when it ends.
- When the body and its deferred blocks have finished, the runner cancels the test's context and
  waits a short grace for the test's goroutines to end. One still running after it fails the test
  with `goroutine started at path:line outlived the test`, and the runner ends it the way a
  cancelled run ends every goroutine (section 9, Goroutines): at its next call or loop iteration.
  So a leaked goroutine cannot disturb the next test.

### Output and reports

- What a test prints, from any goroutine it owns, is collected and shown with the test's result when
  the test fails, or always under `-v`. A passing test prints nothing.
- A failure shows the full name, the panic's message (which for `assert` includes its values), and
  the trace from the panic to the test's body. A returned `Err` shows the error as a script's does.
- Under `-json`, the runner writes one JSON object per line for each event: a test starting, its
  output, and its outcome with its elapsed time, so editors and CI can build their own reports.

```
$ ant test -v
=== RUN   stack_test.ant: push then pop returns the value
--- PASS  stack_test.ant: push then pop returns the value (0.00s)
=== RUN   stack_test.ant: parse
=== RUN   stack_test.ant: parse/1
--- PASS  stack_test.ant: parse/1 (0.00s)
...
ok    5 passed (0.02s)
```

### The command

```
ant test [flags] [path ...]
```

- A path is a test file or a directory, searched recursively for `*_test.ant`; the default is `.`.
- Each test file runs in a fresh program: its modules are loaded and initialized for that file, so
  one file's module state never reaches another's. Files run one after another.
- `ant test`, like `ant run`, does not type-check (section 4, Checking and running). `ant check`
  accepts test files and checks them with their companions; CI runs both.

| Flag             | Meaning                                                                                                                      |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `-run regexp`    | Run only tests whose full name matches. Split on `/`, each part matches one level, as Go's `-run`: `-run 'parse/0x'`          |
| `-skip regexp`   | Skip tests that match, the same way                                                                                          |
| `-v`             | Print every test's start and result, and every test's output                                                                 |
| `-json`          | Print events as JSON lines instead                                                                                           |
| `-failfast`      | Stop starting tests after the first failure                                                                                  |
| `-parallel n`    | The most tests that run at once after `Test.parallel()`; the default is the number of CPUs                                   |
| `-timeout d`     | End the run after `d` (default `10m`), print every goroutine's stack and the tests that were running, and exit with status 1 |

Exit status, as `ant run` (section 10, Scripts): `0` when no test failed (skipped tests and no tests
at all included), `1` when a test failed or the time ran out, and `2` when a file could not be loaded
because of a syntax error, an import cycle, a missing module or a bad import.

### Editors and the debugger

- `ant lsp` lists each top-level test, and each subtest with a literal name, as a code lens with
  "run test" and "debug test", and as document symbols.
- `ant dap` gains a launch option `test` with a test file and a `-run` pattern, so a breakpoint in a
  test or in the code under test stops as it does under `ant run`.

## Examples

A table with records and a skip:

```ant
// path_test.ant, beside path.ant
test "clean" {
  var cases = [
    { path: "a//b", want: "a/b" },
    { path: "a/./b", want: "a/b" },
    { path: "a/../b", want: "b" },
    { path: "", want: "." },
  ]
  for c in cases {
    test "${c.path}" {
      assert(clean(c.path) == c.want)
    }
  }
}

test "home directory" {
  var home = Os.env("HOME") ?? Test.skip("HOME is not set")
  assert(String.startsWith(expand("~/x"), home))
}
```

Files, `try` and `defer`:

```ant
// config_test.ant, beside config.ant
test "load reads defaults for missing fields" {
  var dir = Test.tempDir()
  var path = Path.join([dir, "config.json"])
  try File.write(path, `{ "port": 9000 }`)

  var config = try load(path)
  assert(config.port == 9000)
  assert(config.host == "localhost")
}
```

A server, a context, and parallel tests:

```ant
// server_test.ant
import "./server.ant" { serve }

test "health" {
  Test.parallel()
  var addr = serve("127.0.0.1:0", Test.context())  // stops when the test does
  var response = try Http.get("http://${addr}/health")
  assert(response.status == 200)
}

test "unknown path" {
  Test.parallel()
  var addr = serve("127.0.0.1:0", Test.context())
  var response = try Http.get("http://${addr}/nope")
  assert(response.status == 404)
}
```

A panic expected from the code under test needs no extra function:

```ant
test "divide by zero panics" {
  var r = Panic.catch({ divide(1, 0) })
  assert(r is Err)
  assert(r.error.message == "division by zero")
}
```

## Considered and rejected

- **Option B, functions named `test...`.** See Declaring a test. Kept here as the main alternative
  until review.
- **A failure that records and goes on** (Go `t.Error`, Swift `#expect`, `expect.soft`). Every
  failure is a panic, so one rule covers `assert`, `panic`, `?? panic`, `Result.unwrap` and a bug
  in the code under test, and narrowing after `assert` stays sound. The case that needs several
  failures in one run is a table, and subtests give each row its own result.
- **A matcher library** (`expect(x).toBe(y)`, `Test.equal`). `assert` shows both sides of a
  comparison and the parts of any condition (section 8).
- **`Test.panics(f)` or `should_panic`.** `Panic.catch` and two `assert`s say the same and check the
  message too.
- **Lifecycle hooks** (`beforeEach`, `afterAll`) and **fixtures**. A helper function and `defer` do
  the first; a `var` at the top of the file, initialized once per file, does the second.
- **A handle parameter `t`.** The runner knows which test each goroutine belongs to, so the `Test`
  functions find it themselves. A helper that only asserts needs nothing; one that skips calls
  `Test.skip`.
- **Tests in ordinary files** (Rust's `mod tests`, Zig, Vitest in-source). A test file keeps test
  code and its imports out of a program, and `ant run` never parses it. The companion scope gives
  the same access to private names that an in-file test has.
- **Running examples and doc comments as tests** (Go's `// Output:`, doctests). Out of scope: docs
  stay docs.
- **Default-parallel tests.** See Prior art. Parallel is opt-in, per test.
- **`.only` / focus.** Committed by accident in every framework that has it; `-run` does the job.

## Non-goals

- Benchmarks, fuzzing and property testing, coverage, snapshots and golden files, mocking, fake
  time. Each can come later as its own proposal on top of `test`; none changes it.
- JUnit XML or TAP output. `-json` is the one machine format; a converter can produce the others.
- Watch mode.

## Spec changes

| Where                                      | Change                                                                                                                           |
| ------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------- |
| `spec/ant-lang.md` §2, Keywords            | `test` is a keyword in test files only                                                                                            |
| §5, new subsection or §10 after Scripts    | Test files, the companion scope, the `test` declaration and subtest statement, outcome and order                                  |
| §8, `assert`                               | A sentence that a failed `assert` in a test fails that test                                                                       |
| §9, Panics in goroutines                   | A goroutine owned by a test reports its panic to the test                                                                         |
| §11, Prelude                               | The `Test` namespace                                                                                                              |
| §12, Dynamic semantics                     | Running a test: `Panic.catch`, deferred blocks, ownership of goroutines, the grace and cancellation, collected output             |
| `spec/fmt.md`                              | Formatting of `test "name" { }`, as a `func` declaration                                                                         |
| `spec/lsp.md`, `spec/dap.md`               | Code lenses and symbols; the `test` launch option                                                                                 |
| new `spec/test.md`, or a section of a CLI doc | `ant test`: paths, flags, report, `-json` events, exit status                                                                 |
| `ant/TODO.md`                              | Drop the Test framework entry                                                                                                     |

## Open questions

1. Option A or Option B (Declaring a test).
2. The companion scope: should `foo_test.ant` see `foo.ant`'s private names, or should every test
   be black-box through `import`? Go has both; Rust and Zig only white-box; Gleam and Elixir only
   black-box.
3. Goroutine leak checking: fail by default as proposed, or warn? How long is the grace?
4. Should files run in parallel? Each file is a fresh program, so module state is safe, but the
   process's environment and working directory are shared.
5. `-count n` and `-shuffle` to find order-dependent and flaky tests.
6. `Test.setenv` and `Test.chdir`. Not needed today: Ant has no way to set the environment or change
   the working directory (stdlib `Os`), which is most of why Go refuses them in parallel tests.
7. A per-test time limit (`Test.timeout(d)`, Swift's `.timeLimit`) besides the whole-run `-timeout`.
8. Should `ant test` type-check first? It would break the rule that only `ant check` checks, but a
   test run is where a type error is least welcome as a run-time panic.
