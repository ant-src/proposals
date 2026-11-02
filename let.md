# Proposal: `let` bindings

2026-11-02 · Anton

## Summary

Add `let`, a declaration that binds a name once. `let` has the same form as `var`: it takes an
initializer, an optional type, and a name or destructuring on the left. It is allowed wherever `var`
is allowed. Any assignment to a `let` name is a compile error, and in `ant run` it panics if
reached. `let` makes the binding fixed, not the value: `let xs = [1]` still allows
`Array.push(xs, 2)`. Values are covered by `frozen.md`, and the two proposals are independent. The
checker gets one new benefit: a top-level `let`, including an imported one, can be narrowed like a
local. `var` keeps its meaning. `let` moves from the reserved words to the keywords, so no program
that is valid today stops being valid.

## Motivation

Section 3: "`var` declares a mutable binding… There is no `++`, `--`, or `const`." Every name a
program introduces can be reassigned, except functions, types and imports. That costs four things.

1. **The reader cannot see intent.** In `var total = sum(xs)`, nothing tells the reader whether
   `total` changes forty lines later. Today the only way to know is to search the scope for
   assignments, and in a script the scope is the whole file.
2. **Library constants can't be marked as constants.** `export var DefaultPort = 8080` can be
   reassigned by its own module at any time. So a top-level `var` of a library "never narrows,
   because any call may reassign it" (section 7). An imported `var Config: Settings | null` has to
   be copied into a local before every test. `prelude.d.ant` declares `var Math.PI: float64` with
   the keyword that means "mutable".
3. **Closures and goroutines.** The checker already treats "a local never assigned after its
   declaration" specially: such a local keeps its narrowing inside a lambda or `go` block (section
   7). That rule is a `let` the checker infers. Programmers can't state it, and one assignment added
   later silently drops every narrowing that relied on it. A stated `let` turns that into an error
   at the assignment.
4. **Typos in assignment.** `count = count + 1`, written where `counter` was meant, compiles when
   both are `var`. Zig made "local variable is never mutated" an error in 0.12, and fixing the
   places it flagged in Zig's own code turned up three bugs.

What `let` does not fix is changes to the value. A `let` record can still have its fields assigned.
That is the subject of `frozen.md`, and the two compose:

|               | binding reassignable   | binding fixed          |
| ------------- | ---------------------- | ---------------------- |
| value mutable | `var xs = [1]`         | `let xs = [1]`         |
| value frozen  | `var xs = freeze([1])` | `let xs = freeze([1])` |

## Prior art

| Language              | Spelling                                        | Default and enforcement                                                                                                                                                     |
| --------------------- | ----------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Swift                 | `let` / `var`                                   | Both are equal choices. The compiler warns "variable was never mutated; consider changing to `let`". `let` on a struct is deep, because structs are values                  |
| Rust                  | `let` / `let mut`                               | Immutable unless marked. Warns "variable does not need to be mutable". Deep through ownership                                                                               |
| Kotlin, Scala         | `val` / `var`                                   | Both are equal choices. IDE inspections suggest `val`. Shallow                                                                                                              |
| JavaScript            | `const` / `let` (and `var`)                     | `const` fixes the binding, shallowly. ESLint `prefer-const` is near universal. "`const` arrays can be pushed" is a common misunderstanding                                  |
| Zig                   | `const` / `var`                                 | Since 0.12, a `var` never mutated is a compile error. It found real bugs, but people complain about it while commenting out code                                            |
| Java                  | `final` locals; "effectively final"             | Lambdas may capture only effectively final locals, which the compiler infers, much as Ant's narrowing rule does                                                             |
| C#                    | `readonly` fields only                          | `readonly` locals with `let` as shorthand have been proposed since 2015 and are still not shipped. In 2023 the language team doubted the benefit was worth a second dialect |
| Dart                  | `final` / `var` (`const` is compile-time)       | Lint `prefer_final_locals`                                                                                                                                                  |
| Go                    | `var`, `:=`; `const` for compile-time constants | No immutable bindings. `const` takes only numbers, strings and booleans known at compile time                                                                               |
| Python                | `typing.Final`                                  | Checker only. Erased at run time, like TypeScript's `readonly`                                                                                                              |
| OCaml, F#, Gleam, Elm | `let`                                           | Every binding is fixed. Mutation goes through `ref` cells or `mutable` fields                                                                                               |

What follows from these for Ant:

- **Most languages with both forms treat them as equal choices.** Swift, Kotlin, Scala and
  JavaScript keep both forms and steer through tools rather than through the grammar. Only Rust
  makes the fixed binding the default, and Rust can afford that because ownership already tracks
  mutation.
- **The fixed binding is shallow everywhere except where values are copied.** Swift's `let` is deep
  only because structs are values, and Rust's only because of ownership. With reference values, as
  in JavaScript, Kotlin and Java, `let` fixes the name and nothing else. The JavaScript
  misunderstanding comes from having only one spelling. Ant would have `freeze` for the value, as a
  separate idea.
- **Enforcement by error splits opinion.** Zig shows it catches bugs. Swift and Rust warn instead,
  and Ant has no warnings.

## The proposal

### Syntax

`let` takes every form `var` takes:

```ant
let limit = 100
let name: string = "ant"
let { x, y } = origin
let [head, ...tail] = xs
let area = func(r: float64): float64 { Math.PI * r * r }
export let DefaultPort = 8080

select {
  let message = Channel.receive(inbox) { handle(message) }
}
```

`let` also goes wherever a `var` declaration goes: in a block, at the top level, after `export`, in
a test file, and in front of a `select` head. Parameters and `for` variables are discussed below.

### Rules

1. **No assignment.** An assignment whose target is a `let` name is a compile error. That covers
   `=`, compound assignments such as `+=`, and a name inside an array destructuring assignment. It
   holds whether the assignment is in the binding's own scope, a nested block, a lambda, a `go` or
   `defer` block, or a function of the module. The error message says to declare the name with
   `var`. `ant run` panics with `assignment to let name` when such an assignment is reached, as it
   does for the other checker rules (section 12). An imported `let` is already protected, because
   imported names are read-only.
2. **Initialization is unchanged.** A `let` is bound once, when its declaration runs, as a `var` is.
   Reading it earlier panics, as it does for a `var`: through a call from its own initializer, or,
   for a top-level `let`, before its initializer has run (section 10). A closure created in the
   initializer sees the binding once it is bound, so `let` declares local recursive functions as
   `var` does.
3. **Shallow.** `let` restricts the name and does nothing to the value. Fields, elements and entries
   of a `let` value can still be assigned, and the value can be passed to functions that change it.
4. **Same scoping.** `let` follows the scoping rules of `var` (sections 3 and 12): one declaration
   per name per scope, no shadowing of prelude names, top-level visibility, and the restriction on
   field defaults in local types.

### Narrowing (section 7)

A `let` narrows wherever a local that is never reassigned narrows today: after a test, in the
enclosing function, and inside every nested body. A local `let` gets nothing new, because the
checker already infers this for a local that is never reassigned.

The new part is top-level `let`s. A top-level `let`, in a script or a library, and an imported `let`
narrow in any function, lambda or block that tests them. The reason a top-level `var` never narrows
is that "any call may reassign it", and that is not true of a `let`. A read before initialization
panics rather than returning a stale value, so the narrowing is sound.

```ant
// settings.ant
export let Override: Settings | null = loadOverride()

// app.ant
import "./settings.ant" { Override }

func port(): int {
  if Override != null { Override.port } else { 8080 }    // ok: a let narrows; with var, a compile error
}
```

### Concurrency (section 9)

A `let` is written exactly once, by its initialization. A goroutine whose `go` comes after the
declaration is ordered after that write, so its reads of a local `let` can never race. That covers
every `go`, lambda and `defer` that captures it. A top-level `let` can race only with its own
initialization, by a goroutine started during module initialization that reads it before the
initializer runs. That is already a program error for a `var`, and it stays one. This guarantee is
about the binding only: whether the value can race is the value's business, and `freeze` addresses
it.

### Declaration files (section 10)

A namespace constant becomes `let Owner.NAME: T` with no initializer, so `prelude.d.ant` reads
`let Math.PI: float64`. A declaration file that still writes `var Owner.NAME` gets a compile error.
Declaration files have no assignments, so changing the keyword changes nothing else.

### Parameters and `for` variables

Today both can be reassigned. This section records the research and a recommendation, which still
needs a decision.

| Language   | Parameters                                                                | `for` variable                                           |
| ---------- | ------------------------------------------------------------------------- | -------------------------------------------------------- |
| Swift      | Fixed. `var` parameters were removed in Swift 3 (SE-0003)                 | Fixed; `for var x` allowed                               |
| Kotlin     | Fixed. `var` parameters were removed before 1.0                           | Fixed, with no opt-in                                    |
| Scala      | Fixed                                                                     | Fixed                                                    |
| Rust       | Fixed unless `mut x`                                                      | Fixed unless `for mut x`                                 |
| Zig        | Fixed; copy into a `var` to change                                        | Fixed captures                                           |
| Odin       | Fixed; `x := x` makes a mutable copy                                      |                                                          |
| C#         | Assignable                                                                | Fixed (error CS1656)                                     |
| Go         | Assignable                                                                | Assignable; a fresh variable per iteration since Go 1.22 |
| Java       | Assignable, `final` opt-in; Checkstyle `ParameterAssignment` flags writes | Assignable, `final` opt-in                               |
| JavaScript | Assignable; ESLint `no-param-reassign` (in Airbnb's style)                | `for (const x of xs)`, which `prefer-const` asks for     |
| Dart       | Assignable; lint `parameter_assignments`                                  | Assignable; lint `prefer_final_in_for_each`              |
| Python     | Assignable                                                                | Assignable                                               |

What the research shows:

1. **The change has gone one way.** Swift and Kotlin both had reassignable parameters and removed
   them. Both gave the same reason: readers took a reassignable parameter for pass by reference, and
   saving one line of copying did not pay for that. The Swift core team added that a parameter list
   is not a pattern, so neither `let` nor `var` belongs there. No language has gone the other way.
   The languages that kept assignable parameters gained lints against reassigning them.
2. **An assignable loop variable invites one particular bug.** `for x in xs { x = x * 2 }` looks
   like it updates `xs`, and it does not. C#, Swift, Kotlin, Rust and Zig reject the assignment. Go
   needed a release, 1.22, to fix the opposite bug, a variable shared across iterations, which Ant
   avoided from the start.
3. **Ant code doesn't reassign either.** A scan of the 590 `.ant` files across the repositories
   found about 1,320 parameters and 153 `for` variables. Apart from `_` and tests of assignment
   itself, none is ever assigned.
4. **Ant's reference values make the misreading more likely.** `p.x = 1` changes the caller's
   record, while `p = q` changes nothing the caller sees. With fixed parameters the rule is simple:
   you can change a parameter's value through it, but you cannot rebind it. The checker gains
   nothing, because it already infers this for narrowing.

**Recommendation.** Make parameters and `for` variables fixed, as `let` bindings, with no opt-in
form: no `var` parameter and no `for var`. Every name in a program would then be fixed except one
declared with `var`, including functions, types, imports, parameters and loop variables. The cost is
the normalizing idiom: since a body cannot declare a parameter's name again, it binds a new name, as
in `var c = ctx ?? Context.new()`. `_` stays assignable, as the write-only discard. The corpus shows
nothing would break.

**The alternative is to leave both assignable**, as Go, Java and JavaScript do. That is consistent
with leaving the choice of keyword to the programmer, and it costs nothing today. What it gives up
is the loop-variable error and the clean rule above.

### No preferred form

Ant does not prefer one keyword over the other. A `var` that is never reassigned is as valid as a
`let`, and no tool reports or rewrites it: `ant check` has no warnings, the LSP shows no hint, and
`ant fmt` changes layout, not declarations. A program uses `let` where it wants the guarantee. The
LSP marks a `let` name with the `readonly` semantic-token modifier, so an editor can color it.

### The REPL

A `let` entered in the REPL holds for the rest of the session. A later input may declare other names
again, as it may today, but not a name bound by `let`, and it cannot assign one.

## Spec changes

- **Section 2, Keywords.** Move `let` from the reserved words to the keywords. `const` stays
  reserved.
- **Section 3, Variables.** Introduce `let` beside `var`, give the four rules and the style
  sentence, and replace "There is no `++`, `--`, or `const`" with "There is no `++` or `--`".
- **Section 5.** Destructuring and `select` heads accept `let` wherever they accept `var`.
- **Section 6.** "Inside a block, bind a function with `var`" becomes "with `let` or `var`".
- **Section 7, Narrowing.** Top-level and imported `let`s narrow. The rule for locals never
  reassigned stays, and now names `let` as the way to state it.
- **Section 9, Data races.** Add the paragraph above.
- **Section 10.** Module items, `export let`, `let` in declaration files, and imported names.
- **Section 12.** In Names and scopes, a binding records whether it is a `let`. In Runtime panics,
  add a row for assignment to a `let`, in the column of errors the checker rules out.
- **`lsp.md`.** The `readonly` modifier. Inlay hints cover `let` as they cover `var`.
- **`dap.md`.** `setVariable` refuses a `let`, which is listed with the `readOnly` attribute.
- **`doc.md`.** A doc comment attaches to a `let` as it does to a `var`.

## Implementation

- **Lexer and parser.** `let` becomes a keyword token, and `let` declarations reuse the `var`
  declaration node with a flag.
- **Resolver and checker.** Report an assignment to a `let`, extend narrowing to top-level `let`s,
  and require `let` for namespace constants in declaration files.
- **Interpreter.** A binding slot records that it is a `let`, and the assignment path panics on it.
  A `let` costs nothing at run time beyond that check on assignment.
- **Tools.** Keyword lists in `highlight`, the REPL, and the Monaco, VS Code, Neovim and IntelliJ
  grammars. The LSP gets the modifier, the DAP the `readOnly` attribute, and `ant doc` shows `let`.
- **Tests.** Each assignment form, assignments from closures and goroutines, narrowing of top-level
  and imported `let`s, the runtime panic, declaration files, and a recursive local function bound
  with `let`.
- **Migration.** `prelude.d.ant` namespace constants become `let`. Nothing else has to change.

## Non-goals

- Immutable values: that is `frozen.md`.
- Compile-time constants, as in Go's `const`. A `let` initializer is any expression, run in source
  order.
- Readonly record fields.
- Making `let` the only binding form: `var` stays.

## Alternatives considered

- **No keyword; the checker infers it.** This is what Ant does now for locals, and Java does the
  same with "effectively final". It cannot cover top-level and imported names, because an exporting
  module's own code may reassign them later. It also does not show intent, and it cannot turn an
  accidental assignment into an error.
- **`const`.** In JavaScript, `const` is exactly this binding, so JavaScript programmers would read
  it correctly. But in Go, C, C++, Rust and Zig, `const` means a compile-time constant or a deep
  constant, and even JavaScript users misread `const` as an immutable value. Ant reserved `const` so
  that it would never have that meaning. `let` beside `var` is Swift's pair, where `let` means a
  fixed binding.
- **`val`.** This is Kotlin's and Scala's spelling, but `val` and `var` differ by one letter and
  look alike when scanning code.
- **`let` for mutable and `const` for fixed, as in JavaScript.** That would rename `var` in every
  program, and the mutable form would end up with the keyword Swift, Rust and the ML family use for
  the fixed one.
- **Rust's `let` and `let mut`.** Fixed by default suits Rust because its ownership system relies on
  it. In Ant it would change every existing declaration, and the choice between `let` and `var` is
  left to the programmer.

## Decisions

1. **A `var` that is never reassigned stays valid.** No error, no hint, no rewrite: programmers use
   either keyword as they please (2026-11-02).
2. **The REPL gets no exception.** A `let` holds for the rest of the session and cannot be declared
   again (2026-11-02).
3. **Parameters and `for` variables:** pending. The research and recommendation are above.
