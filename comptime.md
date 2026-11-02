# Proposal: `comptime` blocks

2026-11-02 · Anton

## Summary

Add `comptime { … }`, a block that runs before the program starts, with no I/O, and whose value is
frozen into the program. Code in the block is ordinary Ant and can call ordinary functions. Every
command evaluates it the same way:

- `ant check` evaluates it and reports a panic as a compile error.
- `ant run` and `ant test` evaluate it while loading, before any module initializes.
- A compiler, meaning the JavaScript backend or generated Go, evaluates it at build time and emits
  the result as a literal.

Ant is deterministic once I/O, time and goroutines are excluded, so every command gets the same
value. One new library function, `File.embed`, reads a file next to the module at comptime. The
result is a deep frozen value (`frozen.md`), and `let X = comptime { … }` makes `X` comptime-known
(`let.md`), so later comptime blocks can use it.

This is Zig's `comptime` for values. Zig's other half, types as values, comptime parameters and
`@typeInfo`, does not fit a language whose types are erased and whose checker checks every function.
That part is left out on purpose (see Not from Zig).

## Motivation

1. **Bad constant data fails at run time.** `Result.unwrap(Url.parse("https//api.example.com"))`, a
   malformed JSON default, a regex built from a constant string, or a time zone name all type-check
   and then panic, possibly in production on a rarely used path. In a comptime block, `ant check`
   reports them.
2. **Tables are built on every start.** A CRC table, a keyword set, or a word list parsed from a
   file is rebuilt each time a CLI starts. That is small for the interpreter and pure overhead for a
   compiled program, which could have stored the table.
3. **Assets need I/O at run time.** A SQL schema, an HTML template, help text or test fixtures need
   `File.read` relative to the working directory, which breaks when the program runs from elsewhere.
   A JavaScript bundle has no file system at all. Go solved this with `//go:embed`, Rust with
   `include_str!`, Zig with `@embedFile`, and C23 and C++26 with `#embed`.
4. **Invariants go unchecked.** "This table has 256 entries" or "these two enums agree" can only be
   checked by a test or an `assert` that runs at startup.
5. **Constants inside functions are rebuilt per call.**
   `Set.has(Set.of(["a", "e", "i", "o", "u"]), c)` builds the set on every call. Hoisting it to a
   top-level `var` moves it away from its only use.

A top-level `let` already runs its initializer once, but it can do I/O, it runs after imports
initialize, `ant check` never evaluates it, and a compiler must ship the code that computes it. A
comptime block adds the guarantees: no effects, evaluation before anything else, checked by
`ant check`, and a value a compiler can embed.

## Prior art

| Language    | Mechanism                                                                                                                                  | What Ant takes                                                                                                                                                           |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Zig         | `comptime` expressions, blocks, top-level `comptime {}`, parameters; types as values; `@embedFile`; `@compileError`; `@setEvalBranchQuota` | The keyword, blocks, top-level blocks for assertions, the same language at both times, no I/O except embedding, and a budget. Not types or parameters                    |
| D           | CTFE: any pure function runs at compile time; string imports (`import("file")`, only from `-J` directories); `static if`; mixins           | One set of functions for both times, and reading files only from allowed places                                                                                          |
| C++         | `constexpr`, `consteval` (must run at compile time), `constinit`; C++26 static reflection (P2996, `std::meta`); `#embed`                   | `consteval`'s guarantee that the block runs at compile time, never "maybe". Not reflection                                                                               |
| Rust        | `const` items, `const fn`, const generics, `include_str!`/`include_bytes!`, `build.rs`, proc macros                                        | A warning: `const fn` is a subset of the language that has grown slowly (heap allocation and const traits are still unstable). Ant runs the whole language minus effects |
| Mojo        | `comptime` (renamed from `alias`), comptime parameters `[n: Int]`, `comptime if`, `comptime for`, `comptime assert`                        | Confirms the keyword. Not parameters                                                                                                                                     |
| Nim         | `const`, `static:` blocks, `staticRead`, macros                                                                                            | Static blocks and file reading. Not macros                                                                                                                               |
| Jai         | `#run`: any code at compile time, including I/O                                                                                            | Nothing. A build that depends on the network or the clock can't be reproduced                                                                                            |
| Elixir      | The module body runs at compile time; `File.read!` plus `@external_resource`                                                               | Recording embedded files as dependencies, so a change re-evaluates                                                                                                       |
| Go          | `const` for basic types only; `//go:embed` (package-level, no `..`, no symlinks); `go generate` as a separate author step                  | Embedding rules close to Go's. Go's policy that a build never runs user code is why Ant's comptime is pure                                                               |
| Swift       | SE-0359 build-time constants, returned for revision in 2022; macros since 5.9                                                              | A warning: constants without a clear evaluation model stalled                                                                                                            |
| JavaScript  | Bun macros: `import … with { type: "macro" }` run at bundle time and inline the return value; `bun run` does not run them                  | Inlining results into the output. Not the split: Ant evaluates comptime in every command, so `run` and a build agree                                                     |
| Lisp, Julia | Macros, generated functions                                                                                                                | Nothing. Code generation is a separate proposal if ever                                                                                                                  |

What follows from these:

- **Use the same language at both times.** Zig, D and Jai run ordinary functions at compile time.
  Rust and C++ started with a restricted subset and have spent a decade widening it. Ant's
  interpreter already runs every function, so a comptime block can call anything pure.
- **Keep it hermetic, with one door for files.** Zig, D, Rust's constants and Go's builds run no
  I/O, and each offers exactly one way to read a file. Jai, Elixir and Bun allow anything, so their
  builds depend on the machine they run on.
- **Make every mode agree.** Bun's macros run under `bun build` but not under `bun run`, so the two
  can differ. Ant evaluates comptime in check, run, test and build. Because the result is
  deterministic, all four give the same value.
- **Types are where Zig's power is, and where its cost is.** Zig analyzes lazily: a generic function
  is checked only per instantiation, and an uncalled function is not checked at all, a well-known
  complaint. Ant's checker checks "code nothing calls" (section 4), and Ant's types are erased at
  run time, so types cannot be comptime values.

## The proposal

### Syntax and placement

`comptime` is a keyword followed by a block, like `go` and `defer`. It is an expression, and its
value is the block's last expression, frozen.

```ant
let CrcTable = comptime { Array.map(Array.range(0, 256), crcEntry) }

func isVowel(c: string): bool {
  Set.has(comptime { Set.of(["a", "e", "i", "o", "u"]) }, c)
}

comptime {
  assert(len(CrcTable) == 256)
  assert(CrcTable[0] == uint32(0), "the table starts at 0")
}
```

It takes only a block, not a bare operand. As a prefix operator, `comptime a | f()` would bind like
`try`, and `f` would quietly run at run time. With braces, the extent is visible.

A comptime block may appear anywhere an expression may, including inside function bodies, lambdas
and generic functions. It may also stand alone as a top-level item. A top-level comptime block is
not a statement for the purpose of section 10: it does not make a library a script, because it runs
no program logic. Its value is discarded, so it is the place for assertions.

A comptime block is outside every boundary (section 5), as a field default is. `return` and `try` in
it, outside a lambda, are syntax errors. Code that needs a value from a `Result` writes
`Result.unwrap`, and a failure becomes a compile error.

### What a comptime block may name

A comptime block may name only comptime-known things:

- literals, and its own locals: a `var` inside the block is an ordinary mutable local, so a block
  can build a table imperatively;
- functions: top-level `func`s of any module, functions declared on types, and prelude functions
  allowed at comptime (below);
- types, tags and constructors;
- a `let` whose initializer is a comptime block, from this module above the block or from an import.
  `export let Table = comptime { … }` exports a comptime-known value.

It may not name a parameter, a local of the enclosing function, a top-level `var`, or a `let` with
an ordinary initializer. Those exist only at run time. The checker rejects each name directly in the
block. A function called at comptime that reads such a name panics at that read,
`x is not comptime-known`, and that panic becomes the comptime error. `var X = comptime { … }` is
allowed: the value is computed at comptime, but `X` can be reassigned, so `X` is not comptime-known.

### What a comptime block may do

Anything deterministic. Specifically:

- **Allowed:** all of `Array`, `Map`, `Set`, `String`, `Bytes`, `Html`, `Regex`, `Iter`, `Math`,
  `Bits`, `Result`, `Error`, `Record`, `Json`, `Csv`, `Encoding`, `Url`, `freeze`, `assert`,
  `panic`, `len` and the conversions; `Panic.catch`; `Path` functions that only manipulate strings;
  `Time` functions that compute rather than read the clock; `Crypto` hashes; `Random.new(seed)` with
  a seed and every draw from it, which section 11 fixes to the bit; HTML expressions and regex
  literals.
- **Not allowed:** everything that reads or changes the outside world, and everything that waits:
  `File` (except `File.embed`), `Http`, `Net`, `Os`, `Cmd`, `Time.now`, `Time.sleep` and the
  tickers, `Crypto.randomBytes`, `Random.new()` without a seed, `print` and its siblings, `go`,
  channels, contexts and the other sync objects. Calling one panics,
  `File.read is not available at comptime`. The checker reports direct calls. `prelude.d.ant` marks
  each such declaration so `ant doc` shows it.

These rules make a block's result a function of the source and the embedded files alone. Map and set
order is insertion order, floats follow IEEE 754, and rendering and seeded random numbers are fixed
by the spec. So the checker, the interpreter and every backend compute the same value.

### The value

The block's result passes through `freeze` (`frozen.md`), so `comptime { e }` has type `Frozen[T]`
when `e` has type `T`. It must be data: numbers, `bool`, `string`, `null`, `void`, `Bytes`, `Html`,
tagged primitives, time values, patterns, tags, and arrays, maps, sets and records of these. A
function, sync object, generator, resource or `Random` in the result is a comptime error, because a
compiler could not write it out as a literal. Results that are equal compare equal. Each evaluation
of the program yields one value per block, shared by every run of the code around it, so the
`isVowel` set above is built once, not once per call.

### `File.embed`

```
File.embed(path: string): string        // the file's text, at comptime only; panics if missing or not UTF-8
File.embedBytes(path: string): Bytes    // its bytes, at comptime only
```

These follow `File.read` and `File.readBytes`, with Go's embedding rules:

- The path is relative to the module's file, as an import path is.
- It must name a file in the module's directory or below it: no `..`, no absolute path, and no
  symbolic link that leads outside.
- The argument must be a string literal, so that tools can see dependencies without evaluating
  anything.
- The embedded file is a dependency of the module. The LSP and any cache re-evaluate when it
  changes.

Called outside a comptime block, they panic, and the checker rejects the call. Reading a directory
or a glob is left for later.

```ant
let Schema = comptime { File.embed("./schema.sql") }
let Words = comptime { File.embed("./words.txt") | String.lines() | Set.of() }
let Logo = comptime { File.embedBytes("./assets/logo.png") }
```

### When it runs

Loading a program has a new phase between parsing and initialization. Every comptime block of every
module is evaluated in the order modules initialize: imports depth first, then each module's blocks
in source order, including blocks inside function bodies. A block that reads a comptime `let` that
has not been evaluated yet panics, as reading a `var` before its initializer does, and the checker
reports the same thing statically. The phase runs on one goroutine with nothing else running, so it
needs no synchronization.

- **`ant check`** runs the phase. Each comptime panic is a compile error at the block, with the
  panic's message and a trace of the calls made at comptime. Code after a failing block is still
  checked. A block whose names the checker already rejected is not evaluated.
- **`ant run` and `ant test`** run the phase while loading. A comptime panic stops the program
  before any module initializes, with exit status 2, like a syntax error or an import cycle: the
  program could not be loaded.
- **A compiler** runs the phase at build time and writes each value as a frozen literal. Functions
  used only at comptime need not be emitted.

### Budget

The phase must end, because `ant check` and the LSP run it as a user types. Each block gets a budget
of evaluation steps, ten million in this draft, counted across the functions it calls. The
call-depth limit of section 12 applies as usual. A block that exceeds its budget is a comptime
error, `comptime evaluation exceeded its budget`. There is no way to raise it in this draft; see
Open questions. The LSP evaluates in the background and drops a run when the document changes.

### Tools

- **LSP.** Hovering over a comptime block or a comptime `let` shows its value, rendered by
  `String.from`, truncated. Comptime errors appear as diagnostics.
- **DAP.** The comptime phase runs before the program starts, and breakpoints are not hit during it.
- **`ant doc`** shows `comptime` in a `let`'s signature and marks functions that are not available
  at comptime.
- **`ant fmt`** formats a comptime block as it formats `go` and `defer`.

## Examples

```ant
//! A CRC-32 whose table is computed once, before the program starts.

func crcEntry(n: int): uint32 {
  var c = uint32(n)
  for _ in Array.range(0, 8) {
    c = if Bits.and(c, uint32(1)) == uint32(1) {
      Bits.xor(uint32(0xEDB88320), Bits.shiftRight(c, 1))
    } else {
      Bits.shiftRight(c, 1)
    }
  }
  c
}

export let CrcTable = comptime { Array.map(Array.range(0, 256), crcEntry) }

comptime { assert(CrcTable[1] == uint32(0x77073096)) }
```

```ant
// Constant inputs checked by ant check, not on first use.
let Api = comptime { Result.unwrap(Url.parse("https://api.example.com/v2")) }
let Defaults = comptime { Result.unwrap(Json.parse(File.embed("./defaults.json"))) }
let Berlin = comptime { Result.unwrap(Time.zone("Europe/Berlin")) }
```

```ant
// A page template built once, as Html.
let Footer = comptime { <footer><p>Built with Ant</p></footer> }
```

## Not from Zig

| Zig feature                                       | In Ant                                                                                                                                                              |
| ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Types as values, `fn List(comptime T: type) type` | No. Types exist only in the checker and are erased at run time (section 1), so no evaluation can produce one. Generic types already cover `List[T]`                 |
| Comptime parameters, `comptime n: usize`          | No. They need a specialized copy of a function per call site, which a tree walker doesn't make, and generics cover the type case. Open question 3                   |
| `@typeInfo`, `@Type`, generating types            | No. The checker would have to run code to learn a type, and then it could no longer check every function before any runs. Tags already keep field names at run time |
| `inline for`, `inline while`                      | No. Unrolling is an optimization, not semantics                                                                                                                     |
| Conditional compilation on the target             | No. A comptime block cannot see the platform, so the checker, the interpreter and a cross-compiling backend agree. `Os.platform()` stays a run-time call            |
| `@compileError`                                   | `panic` and `assert` in a comptime block already produce compile errors                                                                                             |
| `@embedFile`                                      | `File.embed`, `File.embedBytes`                                                                                                                                     |
| Top-level `comptime {}`                           | Yes, for assertions                                                                                                                                                 |

## Fit with Ant

| Principle or decision                                     | Fit     | Why                                                                                                                                                    |
| --------------------------------------------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Types are advisory                                        | Good    | Values only; nothing about types depends on evaluation                                                                                                 |
| `ant run` does no static work beyond parsing              | Partial | The comptime phase is evaluation, not analysis, and it runs only code the program wrote in comptime blocks. The checker still never runs for `ant run` |
| One obvious way                                           | Partial | A top-level `let` also runs once. The rule: use comptime when the value should be checked, pure, or embedded; otherwise a `let`                        |
| Minimal syntax                                            | Good    | One keyword and a block, the shape of `go` and `defer`                                                                                                 |
| Errors are values                                         | Good    | Inside a block, `Result.unwrap` turns an `Err` into a compile error; at run time nothing changes                                                       |
| Deterministic library                                     | Strong  | Ant already fixes map order, rendering, float behavior and seeded random numbers, so every mode computes the same value                                |
| Interpreter stays simple                                  | Good    | A load phase, one flag the effectful built-ins check, a step counter, and a check that the result is data. No specialization or caching required       |
| JavaScript backend, exact results                         | Strong  | The bundle gets literal data and embedded files, with no file system needed                                                                            |
| Library close to Go                                       | Partial | Go never runs user code in a build. Ant runs only pure code, and `File.embed` follows `go:embed`'s rules                                               |
| `ant check` and the LSP are safe to run on any repository | Good    | No I/O except reading files under the module's directory, plus a step budget                                                                           |

## Dependencies

- **`frozen.md`.** The result is frozen. Without it, a comptime value would be a mutable value that
  every run of the surrounding code shares, so one call could change what the next one sees.
- **`let.md`.** A comptime-known name has to be a binding that cannot be reassigned.

## Spec changes

- **Section 2.** `comptime` becomes a keyword.
- **Section 5.** New subsection "`comptime`": the syntax, comptime-known names, allowed effects, the
  value, and the boundary rule.
- **Section 10.** Top-level comptime blocks in libraries; the comptime phase in the load sequence;
  exit status 2 for a comptime failure; `export let` of a comptime value.
- **Section 11.** A note on comptime availability, and the marking in `prelude.d.ant`.
- **Section 12.** The comptime phase in Dynamic semantics. Runtime panics gains a row for an
  effectful call at comptime, a non-comptime-known name, a result that is not data, and an exceeded
  budget.
- **`stdlib.md`.** `File.embed` and `File.embedBytes`, and comptime availability per namespace.
- **`lsp.md`, `dap.md`, `doc.md`, `fmt.md`, `test.md`.** As in Tools.

## Implementation

- **Parser.** A `comptime` block node. A top-level comptime block does not mark a module as a
  script.
- **Loader.** The comptime phase after parsing the graph. A frame flag marks comptime evaluation,
  effectful built-ins check it, and a step counter enforces the budget. The result is frozen and
  stored on the node, and evaluating the node later returns it.
- **Checker.** The comptime-known name rule, direct effectful calls, `Frozen[T]` typing, and the
  string-literal argument of `File.embed`. It then runs the phase and turns panics into diagnostics.
- **LSP.** Background evaluation with cancellation, hover values, and file dependencies for
  re-evaluation.
- **Backends.** Write frozen values as literals.
- **Tests.** Each allowed and refused namespace, name rules through calls, ordering across modules,
  the budget, `File.embed` path rules, results that are not data, identical values from check and
  run, and the exit status.

## Non-goals

- Types, comptime parameters, reflection, unrolling, and conditional compilation (see Not from Zig).
- Macros or code generation.
- I/O at comptime beyond reading files under the module's directory.
- Caching comptime results between runs. An implementation may cache by module and embedded-file
  hashes, but nothing requires it.

## Alternatives considered

- **`const NAME = expr`** (Rust, C++ `constexpr`, Go). `const` is reserved, and Go and Rust readers
  would understand it. But a declaration form doesn't cover a constant inside an expression, such as
  the vowel set, or top-level assertions, so an expression form would be needed anyway. Two forms
  for one idea break "one obvious way". `let X = comptime { … }` composes the two concepts.
- **A prefix operator, `comptime expr`.** It is shorter, but its precedence would be `try`'s, and
  `comptime a | f()` would run `f` at run time without anyone noticing.
- **Evaluate only in `ant check` and builds; `ant run` evaluates lazily, like an ordinary `let`.**
  `ant run` would then differ from builds in when failures surface, which is the Bun split. Running
  the phase at load keeps every mode the same.
- **Allow I/O at comptime** (Jai, Elixir, Bun). This would make results depend on the machine,
  network and time, and `ant check` on an untrusted repository could reach the network.
- **Embedding as a declaration** (`//go:embed`). A directive is a second syntax with its own rules.
  A function called in a comptime block is ordinary code, and the block makes its timing explicit.

## Open questions

1. **The budget.** Is ten million steps the right number? Should a block be able to raise it, as
   Zig's `@setEvalBranchQuota` does, for example `comptime(steps: 100_000_000) { … }`?
2. **Directories.** `File.embedDir(path): Map[string, Bytes]` for a folder of assets, with Go's
   rules on hidden files.
3. **Comptime parameters.** `func f(comptime n: int)` could require a comptime argument and let the
   body use `n` in comptime blocks. Its main use would be validating constant arguments at call
   sites, such as a format string. Worth a separate proposal if programs ask for it.
4. **Printing.** Zig's `@compileLog` helps debugging. Should `println` at comptime write to the
   checker's output instead of panicking?
