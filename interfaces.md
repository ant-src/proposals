# Proposal: interfaces and calls through `.`

2026-11-03 · Anton

## Summary

Two additions, one for the run time and one for the checker.

- **`x.f(a)`**, when `x` has no field `f`, calls `f` from the namespace of `x`'s type, with `x`
  first. `p.norm()` is `Point.norm(p)` when `p` is a `Point`, `s.trim()` is `String.trim(s)`, and
  `xs.map(f)` is `Array.map(xs, f)`. The interpreter finds the namespace from the value at the call,
  so the same expression calls `Circle.area` or `Rect.area` according to what `s` holds.
- **`interface Shape { area(s: Self): float64 }`** names a set of functions. Any type for which
  those calls resolve satisfies it; nothing declares that it does. An interface is a type,
  `Array[Shape]`, and a bound, `[T: Shape]`. It exists only in the checker: the interpreter discards
  it as it discards an annotation.

The interpreter gains one rule, for a call `x.f(…)` that finds no field `f`: find the receiver's
namespace, read a field of it, and call. For a user type the namespace is the tag, which already
holds the type's functions; for a built-in value it is a fixed table of kinds. The checker gains an
interface type and a satisfaction check. `interface` is already a reserved word, and no token is
added. Types stay advisory: no evaluation rule consults a written type, and nothing is passed in
behind the program's back.

This is a draft. Nothing here is part of the language.

## Motivation

The spec says (section 4, Generics): "There are no interfaces, no `where` clauses, and no bounds on
type constructors. A user type cannot customize `String.from` or ordering." What Ant has instead:

| Need                                  | Today                                                   | Cost                                                                             |
| ------------------------------------- | ------------------------------------------------------- | -------------------------------------------------------------------------------- |
| A closed set of variants              | Tagged union and an `is` chain, checked for exhaustion  | None. This is the right tool and stays                                           |
| An operator over a fixed set of kinds | A union bound: `[T: Ordered]`, `[T: Number]`            | Only the prelude's kinds; a user type cannot join `Ordered`                      |
| Anything with a given field           | Anonymous record type or bound: `[T: { name: string }]` | Fields only, and the shape cannot be named: every `type X = { … }` is nominal    |
| An open set of implementations        | A record of closures: `type Writer = { write: func… }`  | Each implementation adapts by hand, allocates a record per value, and loses `==` |
| Iteration over a user type            | `Seq[T]`, a function type                               | None; the spec notes that sequences "need no other protocol"                     |

Four kinds of program run into the open-set row:

1. **Pluggable back ends.** A logger that writes to a file, a buffer, or a test fake; a cache over a
   map or over a remote store. A union names every implementation in the library that consumes them,
   so a user cannot add one.
2. **Test doubles.** A function that sends mail takes the mailer as a parameter so a test can pass a
   fake. With a record of closures the fake is easy, but the real mailer is wrapped at every call
   site that passes it.
3. **Generic algorithms over user types.** Sorting records by their natural order, a priority queue
   of jobs. `Array.sortWith(xs, Job.compare)` works, but a generic `Heap[T]` has to take the
   comparator at construction and carry it in every value.
4. **Library protocols.** Go's standard library is built on small interfaces: `io.Reader`,
   `io.Writer`, `fmt.Stringer`, `sort.Interface`. Ant's library mirrors Go's API but has to spell
   each as a concrete type, as `File.Handle` does for every writer.

The record-of-closures idiom covers all four, and it is what Gleam and Elm programs do. It is
verbose at the point that matters most, where a value of a concrete type is handed to code that
wants the abstraction:

```ant
type Writer = { write: func(Bytes): Result[void, Error] }

type Buffer = { chunks: Array[Bytes] = [] }

func Buffer.writer(b: Buffer): Writer {
  Writer{ write: func(data: Bytes): Result[void, Error] { Array.push(b.chunks, data); Result.ok(void) } }
}

report(Buffer.writer(buf), rows)      // every caller adapts
```

The adapter is a second value. Two `Writer`s over one buffer are not `==`, since functions compare
by identity, and a `Writer` cannot be narrowed back to a `Buffer`.

A smaller irritation applies to every value: a function is always called through a namespace's name,
`Point.norm(p)`, `p | Point.norm()`, `s | String.trim()`, so an editor cannot offer a value's
functions after the value, and code that holds a union or a type parameter has no call that picks
the member's function.

## Prior art

| Approach                                       | Examples                                                                                        | What Ant takes                                                                                                                             |
| ---------------------------------------------- | ----------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Implicit, structural method sets               | Go interfaces; TypeScript interfaces; Python `Protocol` (PEP 544)                               | Satisfaction by having the functions, with no `implements`. Consumers define the interface they need, which Go names as its main lesson    |
| Explicit conformance                           | Java, Kotlin; Swift protocols; Rust traits; Carbon `impl`                                       | Not the declaration. Ant's namespaces are closed, so the type's module is the one place an implementation can be: coherence without `impl` |
| Type classes, dictionary passing               | Haskell; Rust and Swift generics; Scala `given`; Roc abilities                                  | Not the mechanism. Dictionaries come from static types, which Ant erases. Dispatch by result type, `T.zero()`, is excluded as a result     |
| Constraint-only interfaces                     | Go type sets; Swift protocols with associated types; Rust traits that are not dyn-compatible    | The split: an interface that mentions `Self` beyond the receiver is a bound only                                                           |
| Call syntax that searches the type's namespace | Lean generalized field notation (`xs.map f` is `List.map xs f`); D and Nim UFCS; Koka dot calls | The call through `.`. Lean is closest: the namespace of the value's type is searched, and the value becomes the first argument             |
| Fields and methods share `.`                   | Go, where a struct's field and method names may not collide; Odin's separate `x.f()`            | Go's rule, which section 6 already has. Odin's separate token is the rejected alternative (see Alternatives)                               |
| Records of functions                           | Gleam; Elm; OCaml first-class modules; Zig `std.Io.Writer` since 0.15                           | Stays the idiom for state captured at construction, such as a handler with its configuration                                               |
| Run-time structural checks                     | Python `@runtime_checkable`, which tests names, not signatures                                  | A warning: `is` does not take an interface                                                                                                 |

Two trends inform the design. Go added generics in 1.18 and kept interfaces as the dynamic half; it
approved generic methods on concrete types in 2026, but not on interface methods, because its
dispatch tables are built per instantiation. Zig 0.15 replaced its duck-typed `anytype` writers with
a concrete writer that calls through a table of functions. Both point at dynamic dispatch through a
table as the boundary tool, and in Ant the tag already is that table.

## Design constraints from Ant

- **Dispatch must come from a value.** `ant run` discards annotations. Inside
  `func f[T: Shape](x: T)` the interpreter does not know `T`; it knows `x`. So a call picks its
  function from its receiver's tag, and functions that would be picked by a result type, `T.zero()`
  or `T.parse(s)`, are out under any design that keeps types advisory.
- **A value knows its kind and its tag, nothing more.** Records and named primitive values carry a
  tag, and section 6 stores a type's functions on it. Strings, arrays, maps and the other built-in
  kinds carry no tag, but the prelude already groups their functions in one namespace per kind,
  almost all of them receiver first: `Array.map(xs, f)`, `Map.get(m, k)`, `Regex.find(p, s)`,
  `Channel.send(c, v)`. A fixed table from kind to namespace is all a call needs for them.
- **A type's fields and functions never share a name.** Section 6 already makes that a compile
  error, so on any one value `.f` can mean a field or a function, never both, and `.` can carry
  both. A record of closures keeps calling `w.write(data)` as a field read and a call.
- **Namespaces are closed.** Only a type's module declares its functions (section 6), so a type has
  one function per name, and no orphan or overlap rules are needed. A library type cannot be made to
  satisfy a user interface by anyone but its author, which is Go's rule too.

## The proposal

### Calls through `.`

`x.f(a, b)` keeps its meaning when `x` has a field `f`: read the field, call its value. When it has
none, the call goes to the namespace of `x`'s type and passes `x` first. So
`users.filter({ x.active }).map({ x.name }).sort()` chains, and `p.norm()` is `Point.norm(p)`.

At run time a call whose callee is `x.f`:

1. evaluates `x`;
2. reads the field `f` of `x` when there is one, which covers records, module namespaces
   (`Geometry.distance(p, q)`), tags (`Point.origin()`) and prelude namespaces (`Array.map(xs, f)`),
   evaluates the arguments and calls the field's value with them, as today;
3. otherwise finds the value's namespace in the table below, reads `f` from it, evaluates the
   arguments, and calls `f` with `x` first. A value with no namespace, or a namespace without `f`,
   panics: `int has no field or function abs`, `Point has no field or function nrom`.

| Value                                                         | Namespace                                                                                                                                      |
| ------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| a record or named primitive value of a user type              | its type: the tag, which holds the functions declared on it (§6)                                                                               |
| a string, `Bytes`, `Html`, an array, a map, a set             | `String`, `Bytes`, `Html`, `Array`, `Map`, `Set`                                                                                               |
| `Ok`, `Err`                                                   | `Result`                                                                                                                                       |
| a value of a prelude type `N`, or of a type `N.T` it declares | `N`: `Mutex`, `Channel`, `Error`, …; `File` for `File.Handle`, `Time` for `Time.Instant`, `Regex` for `Regex.Pattern`, `Cmd` for `Cmd.Process` |
| a number, `bool`, `null`, `void`, a function                  | none (open questions 6 and 7)                                                                                                                  |

That is the whole run-time rule. Only the call form changes: `x.f` outside a call is a field read,
as today, so a namespace function is reached only by calling it. `var g = p.norm` is a compile error
and a panic, and `{ p.norm() }` is the function value. Go would bind `p` into a closure here; Ant
does not, so `(p.norm)()` and `p.norm()` differ only in that the first is an error.

The table's rows for the prelude are a list the prelude fixes, as it fixes its names, and a host's
declaration file adds rows for the types it declares. `p.norm()` and `Point.norm(p)` call the same
function when `p` is a `Point`, and a namespace's name stays the way to call any other function:
`Point.origin()`, `Net.peer(h)`, `Json.stringify(v)`. The interpreter checks neither signatures nor
privacy, as for any field read of a tag; an unchecked `h.write("x")` on a handle calls `File.write`,
which takes a path, and panics there as any wrong call does.

The checker resolves `x.f(…)` by the static type of `x`, fields first:

| Type of `x`                          | `f` is                                                                                        |
| ------------------------------------ | --------------------------------------------------------------------------------------------- |
| a type with a field `f`              | the field, called as today                                                                    |
| a type with a namespace in the table | the function `f` of that namespace, if it is visible here and its first parameter accepts `x` |
| a union whose every member has one   | `f` for every member; each must accept the arguments, and the result is the union of results  |
| an interface                         | the interface's `f`, with `Self` as the interface                                             |
| a type parameter                     | as for its bound                                                                              |
| anything else                        | a compile error: numbers, functions, anonymous record types without the field                 |

The checker and the interpreter agree because no value has both a field and a function of one name:
section 6 forbids it for user types, and a field never satisfies an interface function (Interfaces,
below), so a value that reaches `x.f(…)` through an interface has the function and not the field.
Width subtyping cannot add the field behind the checker's back either: an anonymous record type has
no namespace, so the checker accepts only a field call through it.

The first-parameter condition makes the few namespaces that mix receivers safe: `File` holds
functions on paths and on handles, and `h.writeLine(line)` resolves while `h.write(text)` is a
compile error. A function with no parameters, such as `Point.origin()`, is never reached through a
value.

The union row is the field rule of section 4, Tagged unions, applied to functions: a union of types
that each have `area` can call `s.area()` without an interface, and `Cmd | Cmd.Pipeline` calls into
`Cmd` for both. Its result must be a valid union. A `Result` is a union of `Ok` and `Err`, both of
which the table sends to `Result`, so `r.unwrapOr(0)` works.

A function a user declares on an alias, `func Shape.area` with `type Shape = Circle | Rect`, is not
reached through a value. A `Shape` value carries `Circle`'s tag, and the interpreter looks there. It
is still called as `Shape.area(s)`. Only the prelude routes a union's members to one namespace, for
`Result`.

A named primitive value uses its own type's namespace, not its base's: `id.length()` on a `UserId`
looks in `UserId`, as library functions take the base kinds today. `string(id).length()` unwraps
first.

Named arguments apply when the checker knows the one function called: the receiver's type is not a
union, an interface or a type parameter, so `ch.receive(ctx: c)` works. Through those three the call
is positional, since each implementation may name its parameters differently and the interpreter
binds names against the function it finds.

`x?.f()` yields `null` when `x` is `null` and calls `f` otherwise, as `?.` already does for fields.
A chain may break before `.` or `?.` under the existing continuation rule (section 2).

### Interfaces

```ant
interface Shape {
  area(s: Self): float64
  name(s: Self): string
}

interface Container[T] {
  get(c: Self, i: int): T | null
  size(c: Self): int
}
```

`interface` declares a name at the top level of a module, and `export interface` exports it. The
body lists signatures, one per line, in the syntax of a function type with parameter names. The
first parameter of each is the receiver, of type `Self`, the satisfying type. The body has at least
one function, so no interface is satisfied by everything and none is a top type. An interface may
take type parameters; its functions may not take their own.

A type `S` satisfies `I` when, for each function `f` of `I`, `.f(…)` on an `S` resolves (the checker
table above) to a namespace function, not a field, visible where the checker asks, whose type with
`Self` replaced by `S` is assignable to `f`'s signature. That is: `S` satisfies `I` when every call
the interface promises checks on an `S`. No declaration says so. A generic type satisfies an
interface for each instantiation its functions cover: `func Box.show[T](b: Box[T])` makes every
`Box[X]` a `Show`. Built-in types satisfy interfaces by the same rule, so `string`, `Array[T]` and
`Bytes` are all `interface Sized { length(s: Self): int }`, and a `File.Handle` is a `Sink` below
without a wrapper.

**As a type.** An interface whose functions mention `Self` only as the receiver is a type. A value
of a satisfying type is assignable to it, as is a union whose every member satisfies it, and so is a
value of an interface that has all of its functions. Containers stay invariant: `Array[Circle]` is
not an `Array[Shape]`, and `[...circles]` widens. An interface value is the value itself, with no
box: `==`, `String.from` and `Json.stringify` see the value. `is` narrows it to a satisfying type,
and the `else` branch keeps the interface, since the set of satisfying types is open:

```ant
var shapes: Array[Shape] = [Circle{ radius: 1.0 }, Rect{ width: 2.0, height: 3.0 }]
var total = shapes.map({ x.area() }).sum()

for s in shapes {
  if s is Circle { println(s.radius) }
}
```

**As a bound.** Every interface is a bound. One that mentions `Self` beyond the receiver is only a
bound, since a value of it could pass a `Circle` and a `Rect` to one `compare`:

```ant
interface Compare {
  compare(a: Self, b: Self): int
}

type Job = { priority: int, name: string }

func Job.compare(a: Job, b: Job): int { b.priority - a.priority }

func sorted[T: Compare](xs: Array[T]): Array[T] { xs.sortWith(func(a: T, b: T): int { a.compare(b) }) }

var queue: Array[Compare] = []       // compile error: Compare is a bound only
```

A union solves an interface bound only when the interface is a type, matching the rule that a union
bound takes a single member.

**What an interface is not.** It is not a value or a namespace: `Shape.area` and `is Shape` are
compile errors. `is` cannot test an interface, because at run time it could test names but not
signatures. It has no discriminator, so a union holds it only beside `null` and `void`:
`Shape | null` is fine, `Shape | Error` is not, and `Result[Shape, Error]` wraps the two instead. It
is declared only at the top level, since only top-level types have functions.

The interpreter keeps nothing of an interface. The declaration is parsed and discarded, like an
annotation, and every dynamic call goes through `.`.

### Choosing between the tools

The spec gets one paragraph of guidance:

- A closed set of variants is a union, narrowed with `is` and checked for exhaustion.
- An open set is an interface, called with `.`.
- A function from the receiver's own namespace is called with `.`: `xs.map(f)`, `p.norm()`. Any
  other function, a module's or another namespace's, is called plainly or through `|`:
  `seq | Iter.take(5)`, `v | Json.stringify()`, `n | Port`.
- State captured at construction, such as a handler and its configuration, stays a record of
  closures.

## Examples

A sink that a file handle satisfies as it is, with a test double:

```ant
// log.ant
export interface Sink {
  writeLine(s: Self, line: string): Result[void, Error]
}

export func info(sink: Sink, msg: string): Result[void, Error] { sink.writeLine("info ${msg}") }

// main.ant
import "./log.ant" { info }

var out = try File.create("app.log")
try info(out, "started")                     // File.writeLine makes File.Handle a Sink

// log_test.ant
import "./log.ant" { info }

type Memory = { lines: Array[string] = [] }

func Memory.writeLine(m: Memory, line: string): Result[void, Error] {
  m.lines.push(line)
  Result.ok(void)
}

test "info prefixes" {
  var m = Memory{}
  Result.unwrap(info(m, "hi"))
  assert(m.lines == ["info hi"])
}
```

A user ordering in a generic container:

```ant
type Heap[T] = { items: Array[T] = [] }

func Heap.push[T: Compare](h: Heap[T], v: T): void {
  h.items.push(v)
  var i = len(h.items) - 1
  while i > 0 && h.items[i].compare(h.items[(i - 1) / 2]) < 0 {
    var p = (i - 1) / 2
    [h.items[i], h.items[p]] = [h.items[p], h.items[i]]
    i = p
  }
}

var jobs: Heap[Job] = Heap{}
jobs.push(Job{ priority: 2, name: "build" })
```

Library calls on built-in values, before and after:

```ant
var names = users
  | Array.filter({ x.age >= 18 })
  | Array.map({ x.name })
  | Array.sort()

var names = users
  .filter({ x.age >= 18 })
  .map({ x.name })
  .sort()

var port = (m | Map.get("port")) ?? "80" | String.trim()
var port = (m.get("port") ?? "80").trim()
```

## Spec changes

- §2 Lexical structure: `interface` moves from reserved words to keywords; `Self` is a type name
  inside an interface body. No new token, and the continuation rule already covers a leading `.`.
- §4 Types: a subsection, Interfaces: declaration, satisfaction, type or bound, the union rule; a
  row in the table of Kinds; Generics drops "There are no interfaces" and gains interface bounds;
  Assignability gains the interface rules; Type declarations drops "interfaces" from "no classes,
  interfaces, or prototypes"; the guidance paragraph above.
- §5 Expressions: calls through `.`, its two tables, and `?.` on them. Pipes: "The pipe replaces
  method chaining" becomes "`.` chains calls of a value's own namespace; the pipe chains calls of
  any other function."
- §6 Functions on types: "There is no method call … `p.distance(q)` reads a field of `p`, which does
  not exist" becomes the call through `.`; functions on aliases are reached only through the alias's
  name. The rule that a function may not share its name with a field gains a second reason.
- §10 Declaration files: a host's declaration file names the namespace of each type it declares, and
  a prelude or host type's fields may not shadow a function its namespace offers on it.
- §11 Standard library: the prelude's rows of the namespace table; examples move from
  `xs | Array.map(f)` to `xs.map(f)`.
- §12 Dynamic semantics: the three steps of a call through `.` and its panic.
- `stdlib.md`: its types' rows, and its examples, as in §11.
- `fmt.md`: the declaration. `doc.md`: an interface's page, and the interfaces a type satisfies
  among those its module sees. `lsp.md`: namespace functions in completion after `.`, go to
  implementations of an interface function.

## Implementation

- **Parser:** the `interface` declaration and `Self`. Calls are parsed as today.
- **Interpreter:** a call whose callee is `x.f` reads the field as today and, on a miss, switches on
  the value's kind, or follows the tag to its namespace, which is the tag itself for a user type.
  Interface declarations are dropped like annotations. Nothing else.
- **Checker:** most of the work: fields first, then the namespace table, the interface type,
  satisfaction, assignability, type or bound, `is` narrowing from an interface, the union rule.
  Satisfaction can be cached per type and interface.
- **Prelude audit.** No prelude type may have a field that shadows a function its namespace offers
  on it, or `x.f()` would call the field where the user meant the function. The audit of
  `ant-lang.md` and `stdlib.md` finds no such case. `Time` values are opaque, so `Time.year(t)` has
  no field to collide with. `File.Pipe{ read, write }` shadows `File.read` and `File.write`, but
  those take a path, so they were never reachable from a pipe. `Error`, `Ok`, `Err`, `Entry`,
  `Panic`, `Cmd`, `Cmd.Output`, `Html.Element`, `Regex.Match`, `File.Info` and `Url.Parts` have no
  field whose name their namespace uses for a function on them. The check runs over the prelude's
  declaration file, so a new clash fails the build.
- **Tools:** `ant fmt`, `ant doc`, LSP completion and implementations, the editor grammars
  (vscode-ant, nvim-ant, intellij-ant, the Monaco package), the site and its examples. A rewrite
  from `x | N.f(…)` to `x.f(…)` needs types, so it is an LSP code action, not `ant fmt`.

## Alternatives

### A separate token, `x->f()`

An earlier draft of this proposal used `->` for the call and left `.` a pure field read, as Odin
does. It is the more explicit design: a reader sees whether `x` is passed, a field call and a
namespace call never share syntax, and `(x.f)(a)` and `x.f(a)` stay the same expression. It also
costs a token, a `?->` for nullable receivers, an entry in the continuation rule, and a second
operator for one idea where `Geometry.distance`, `Array.map` and `Point.origin` already use `.`.
Section 6's rule that a type's fields and functions do not share names removes the ambiguity that
the separate token guarded against, so `.` was chosen. The cost that remains is the reader's: in
`cfg.handler(req)`, whether `cfg` is passed depends on whether `Cfg` has a field `handler`. Go has
the same property and the same collision rule.

### Bound method values

`p.norm` without a call could be a closure over `p`, as Go's method values are. It makes `x.f` mean
three things: a field, a namespace function bound to `x`, or an error. Ant keeps two, and the lambda
`{ p.norm() }` costs four characters.

## Non-goals

- **Method values**, for the reason above.
- **Dispatch by result type** (`T.zero()`, `T.parse(s)`). It needs types at run time.
- **Default implementations, explicit `implements`, embedding with `...`, generic interface
  functions.** Each adds rules for a convenience. Embedding and generic functions are in Later.
- **`is` with an interface**, for the reason in Interfaces.

## Later

- **Prelude interfaces as hooks.** A `toString` for `String.from`, a `compare` that lets
  `Array.sort` take user types, and `typed-data.md`'s `fromData`/`toData`/`validate`, as interfaces
  with checked signatures instead of names. Each changes what a built-in function does to a record,
  so each is its own decision.
- **I/O interfaces in the prelude**, `Io.Reader` and `Io.Writer` as in Go. A user can already
  declare one that `File.Handle` satisfies; the prelude version needs its `File` and `Net` functions
  named consistently across handles and connections.
- **Embedding**, `interface ReadWriter { ...Reader, ...Writer }`, if listing the functions again
  proves common.
- **Generic interface functions.** The interpreter would need nothing, since a call ignores type
  arguments; only the checker's satisfaction rule grows.

## Open questions

1. **Fields in interfaces.** `interface Named { name: string }` would give a structural shape a
   name, which today is impossible. It mixes a structural field rule with a function rule in one
   declaration.
2. **Bound or type for `Self`-receiver interfaces.** `func total(xs: Array[Shape])` and
   `func total[T: Shape](xs: Array[T])` overlap. The rule here follows Go: take the interface type,
   and use a bound when the result must keep `T`.
3. **Two spellings for one call.** `xs.map(f)` and `xs | Array.map(f)` call the same function. The
   guidance picks `.`; `ant fmt` cannot enforce it without types, but the checker could warn and the
   LSP offer the rewrite. The stricter choice is a compile error for a pipe or a plain call into the
   receiver's own namespace, which makes `.` the only spelling.
4. **Accidental satisfaction.** A type with a `close` of the right shape satisfies every `Closer`.
   Go accepts this; `typed-data.md` rejected hooks that a function could become by accident. The two
   proposals should agree.
5. **`Self`.** A reserved word, or a type name only inside interface bodies.
6. **Numbers.** No namespace today. Mapping every numeric kind to `Math` would give `x.abs()` and
   `x.sqrt()`, but `Math` also holds `Math.random()` and functions on `float64` only, and `Bits`
   would be left out. The alternative is nothing, and `Math.abs(x)` as now.
7. **Sequences.** A `Seq[T]` is a function, so `seq.filter(f)` would mean every function value
   searches `Iter`. Useful for pipelines, odd for every other function. The alternative is
   `seq | Iter.filter(f)` as now.
