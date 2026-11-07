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
   panics: `int has no field or function abs`, `Point has no field or function nrom`. A tag or a
   module record holds its functions as fields, so a miss on one keeps today's message,
   `Point has no function nrom`.

| Value                                                         | Namespace                                                                                                                                      |
| ------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| a record or named primitive value of a user type              | its type: the tag, which holds the functions declared on it (§6)                                                                               |
| a string, `Bytes`, `Html`, an array, a map, a set             | `String`, `Bytes`, `Html`, `Array`, `Map`, `Set`                                                                                               |
| `Ok`, `Err`                                                   | `Result`                                                                                                                                       |
| a value of a prelude type `N`, or of a type `N.T` it declares | `N`: `Mutex`, `Channel`, `Error`, …; `File` for `File.Handle`, `Time` for `Time.Instant`, `Regex` for `Regex.Pattern`, `Cmd` for `Cmd.Process` |
| a function                                                    | `Iter`, for sequences (Decision 7)                                                                                                             |
| a number, `bool`, `null`, `void`                              | none (Decision 4)                                                                                                                              |

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

| Type of `x`                                   | `f` is                                                                                                                                 |
| --------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| a type with a field `f`                       | the field, called as today                                                                                                             |
| a type with a namespace in the table          | the function `f` of that namespace, if it is visible here and its first parameter accepts `x`; the call is then checked as `N.f(x, …)` |
| a union whose members share a namespace       | the function `f` of that namespace, checked as `N.f(x, …)` with the union as the receiver                                              |
| any other union whose every member has an `f` | `f` for every member, field or function; each must accept the arguments, and the result is the union of results                        |
| an interface                                  | the interface's `f`, with `Self` as the interface                                                                                      |
| a type parameter                              | as for its bound                                                                                                                       |
| anything else                                 | a compile error: numbers, `bool`, anonymous record types without the field                                                             |

The checker and the interpreter agree because no value has both a field and a function of one name:
section 6 forbids it for user types, and a field never satisfies an interface function (Interfaces,
below), so a value that reaches `x.f(…)` through an interface has the function and not the field.
Width subtyping cannot add the field behind the checker's back either: an anonymous record type has
no namespace, so the checker accepts only a field call through it.

The call is checked as the direct call it stands for, `Array.sort(points)` for `points.sort()`, with
the same inference and the same errors, bounds included. The first-parameter condition decides only
whether `f` is offered on `x` at all, which makes the few namespaces that mix receivers safe: `File`
holds functions on paths and on handles, and `h.writeLine(line)` resolves while `h.write(text)` is a
compile error that says `File.write` takes a path. A function with no parameters, such as
`Point.origin()`, is never reached through a value. The condition is loose on purpose: a generic
first parameter accepts anything, so `s.from()` on a string is `String.from(s)` and `a.new()` on an
`Atomic` is `Atomic.new(a)`. Both are correct calls nobody writes, and a rule that read the
parameter's type for the namespace's name would be a second rule for a cosmetic gain (Decision 10).

The same condition makes the function row safe. Every function value's namespace is `Iter`, and
every function of `Iter` that takes a receiver takes a `Seq[T]`, so
`Iter.range(0, 9).filter(f).take(5)` checks while `handler.map(f)` on a `func(Request): Response` is
a compile error that says the function is not a sequence. On an array `.filter` is `Array.filter`,
eager; on a sequence it is `Iter.filter`, lazy. The receiver's type decides, as in JavaScript's
iterator helpers and Rust. An unchecked call of an `Iter` function on a function that is not a
sequence panics inside it, as any wrong call does.

The union rows are the field rule of section 4, Tagged unions, applied to functions. A union whose
members share a namespace is one call, checked as the direct call: a `Result` is a union of `Ok` and
`Err`, both of which the table sends to `Result`, so `r.unwrapOr(0)` is `Result.unwrapOr(r, 0)`,
which solves `E` from the union. Resolved per member it could not, since an `Ok[int]` alone fixes no
`E`, and `Result.unwrapOr(ok, 0)` on an `Ok[int]` is a compile error today for that reason.
`Cmd | Cmd.Pipeline` calls into `Cmd` the same way. Any other union resolves `f` per member, a field
on one member and a function on another included, since each value does its own lookup at run time:
a union of types that each have `area` can call `s.area()` without an interface. Its result must be
a valid union. `.` never does more than the direct call it stands for, so a bound that rejects the
union rejects the call: `[T: Time.Point]` takes a single member, so `t.add(d)` on a
`Time.Instant | Time.PlainDate` fails as `Time.add(t, d)` does, and the value is narrowed first.

A function a user declares on an alias, `func Shape.area` with `type Shape = Circle | Rect`, is not
reached through a value. A `Shape` value carries `Circle`'s tag, and the interpreter looks there. It
is still called as `Shape.area(s)`. Section 6's own example is this case, and it keeps the line that
`s.area()` does not reach it, since it is the first thing a reader tries. Only the prelude routes a
union's members to one namespace, for `Result`.

A named primitive value uses its own type's namespace, not its base's: `id.length()` on a `UserId`
looks in `UserId`, as library functions take the base kinds today. `string(id).length()` unwraps
first.

Named arguments apply when the checker knows the one function called: a field, or a namespace
function found through a single type or a union whose members share a namespace, so
`ch.receive(ctx: c)` and `r.unwrapOr(fallback: 0)` work. Through a union of several namespaces, an
interface or a type parameter the call is positional, since each implementation may name its
parameters differently and the interpreter binds names against the function it finds.

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
first parameter of each is the receiver and has the type `Self`, exactly, the satisfying type; a
signature whose first parameter has another type is a syntax error. A parameter may not carry a
default, and a function may not take type parameters of its own, though the interface may. Parameter
names document the call, since a call through an interface is positional (Calls through `.`). `Self`
is a reserved word that is legal only in an interface's signatures; anywhere else, `var x: Self` or
`type Self = …`, it is a syntax error, so it never has a second meaning. The body has at least one
function, so no interface is satisfied by everything and none is a top type. An alias may name an
interface, `type Drawable = Shape`, and stands for it as any alias does, a bound only when the
interface is.

A type `S` satisfies `I` when, for each function `f` of `I`, `.f(…)` on an `S` resolves (the checker
table above) to a namespace function, not a field, visible where the checker asks, whose type with
`Self` replaced by `S` is assignable to `f`'s signature. That is: `S` satisfies `I` when every call
the interface promises checks on an `S`. No declaration says so. A generic type satisfies an
interface for each instantiation its functions cover: `func Box.show[T](b: Box[T])` makes every
`Box[X]` a `Show`. Built-in types satisfy interfaces by the same rule, so `string`, `Array[T]` and
`Bytes` are all `interface Sized { length(s: Self): int }`, and a `File.Handle` is a `Sink` below
without a wrapper.

Visibility is the checker's, at the place it asks: where a value is widened to the interface, a
bound is solved, or one interface is assigned to another. A type whose `writeLine` is not exported
is therefore a `Sink` inside its module and not outside it, as in Go. The interpreter checks no
privacy, so the function is still called through the interface from any module the value reaches.

Satisfaction is checked as assignability is (section 4): a pair under test counts as satisfied, so
`interface Node { next(n: Self): Node | null }` holds for `func Tree.next(t: Tree): Tree | null`.
Type arguments stay invariant inside it, which is the first thing a tree runs into: `Array[Tree]` is
not an `Array[Node]`, so a `children` that returned the field as it is would not satisfy `Node`, and
the copy widens, as section 4 has it for every container.

```ant
interface Node {
  children(n: Self): Array[Node]
}

type Tree = { kids: Array[Tree] = [] }

func Tree.children(t: Tree): Array[Node] { [...t.kids] }   // a Node; returning t.kids, an Array[Tree], would not be
```

**As a type.** An interface whose functions mention `Self` only as the receiver is a type. A value
of a satisfying type is assignable to it, as is a union whose every member satisfies it, and so is a
value of an interface that has all of its functions. Containers stay invariant: `Array[Circle]` is
not an `Array[Shape]`, and `[...circles]` widens. An interface value is the value itself, with no
box: `==`, `String.from` and `Json.stringify` see the value. `is` narrows it to a satisfying type,
and the `else` branch keeps the interface, since the set of satisfying types is open. A generic
target narrows to the one instantiation that satisfies the interface, and it is a compile error when
the interface does not determine every type argument, as section 5 already rules for an anonymous
record type: `s is Box` is an error on a `Shape` and gives a `Box[int]` on a `Container[int]`.

```ant
var shapes: Array[Shape] = [Circle{ radius: 1.0 }, Rect{ width: 2.0, height: 3.0 }]
var total = shapes.map({ x.area() }).sum() ?? 0.0

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

**Type or bound.** For an interface that is a type, both forms are legal and behave the same at run
time, where types are erased. They differ only in what the checker accepts. `xs: Array[Shape]` takes
a mixed array but not an `Array[Circle]`, since containers are invariant, and its results are
`Shape`s. `[T: Shape](xs: Array[T])` takes either, `T = Shape` for a mixed array, and its results
keep `T`, so `largest` of an `Array[Circle]` is a `Circle`. The guide is the one section 4 already
gives for union bounds, `choose(a: Number, b: Number)` beside `bigger[T: Ordered]`: take the
interface, and use a bound when the result must keep the caller's type.

**What an interface is not.** It is not a value or a namespace: `Shape.area` and `is Shape` are
compile errors. `is` cannot test an interface, because at run time it could test names but not
signatures. It has no discriminator, so a union holds it only beside `null` and `void`:
`Shape | null` is fine, `Shape | Error` is not, and `Result[Shape, Error]` wraps the two instead. It
is declared only at the top level, since only top-level types have functions.

It is not data either. A type expression may not reach an interface, so `type(Shape)` and
`type(Scene)` with `type Scene = { shapes: Array[Shape] }` are compile errors, and `Reflect` gains
no shape for one. The interpreter discards the declaration, so in an unchecked run `Shape` is a name
nothing binds and the same expressions panic under the rule section 12 already has for a name in
`T`. An open set has no members a format could decode into: typed decoding targets a union, whose
members it can name, never an interface.

The interpreter keeps nothing of an interface. The declaration is parsed and discarded, like an
annotation, and every dynamic call goes through `.`.

### Choosing between the tools

The spec gets one paragraph of guidance:

- A closed set of variants is a union, narrowed with `is` and checked for exhaustion.
- An open set is an interface, called with `.`.
- A function from the receiver's own namespace may be called with `.`, `xs.map(f)`, or through the
  namespace, `xs | Array.map(f)` and `Array.map(xs, f)`. All three are legal and none is preferred;
  `ant fmt` keeps the one the author wrote. Any other function, a module's or another namespace's,
  is called plainly or through `|`: `xs | Iter.fromArray()`, `v | Json.stringify()`, `n | Port`.
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

- §2 Lexical structure: `interface` moves from reserved words to keywords; `Self` joins the reserved
  words, legal only in an interface's signatures. No new token, and the continuation rule already
  covers a leading `.`.
- §4 Types: a subsection, Interfaces: declaration, satisfaction and where it is asked, type or
  bound, the union rule, narrowing; a row in the table of Kinds; Generics drops "There are no
  interfaces" and gains interface bounds; Assignability gains the interface rules; Type declarations
  drops "interfaces" from "no classes, interfaces, or prototypes"; Types as values: a type
  expression may not reach an interface; the guidance paragraph above.
- §5 Expressions: calls through `.`, its two tables, and `?.` on them. `is`: narrowing from an
  interface, and the type arguments of a generic target. Pipes: "The pipe replaces method chaining"
  becomes "`.` chains calls of a value's own namespace; the pipe chains calls of any other
  function."
- §6 Functions: Calling: named arguments through `.` when the function called is known. Functions on
  types: "There is no method call … `p.distance(q)` reads a field of `p`, which does not exist"
  becomes the call through `.`; functions on aliases are reached only through the alias's name, and
  the `Shape.area` example says so. The rule that a function may not share its name with a field
  gains a second reason.
- §7 Nullability: Narrowing: the `else` branch of a test on an interface keeps the interface.
- §10 Modules: `export interface`, and an exported function needs an exported interface as it needs
  an exported type. Declaration files: a host's declaration file names the namespace of each type it
  declares, and a prelude or host type's fields may not shadow a function its namespace offers on
  it.
- §11 Standard library: the prelude's rows of the namespace table, with `Ok` and `Err` to `Result`
  as the one exception to a prelude type's own name; examples move from `xs | Array.map(f)` to
  `xs.map(f)`. Reflect: no shape for an interface.
- §12 Dynamic semantics: the three steps of a call through `.` and its panic; the `type(T)` row
  gains an interface that `T` reaches.
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
  satisfaction, assignability, type or bound, `is` narrowing from an interface, the union rules, and
  the rejection of a type expression that reaches an interface. Satisfaction can be cached per type
  and interface.
- **Prelude audit.** No prelude type may have a field that shadows a function its namespace offers
  on it, or `x.f()` would call the field where the user meant the function. The audit of
  `ant-lang.md` and `stdlib.md` finds no such case. `Time` values are opaque, so `Time.year(t)` has
  no field to collide with. `File.Pipe{ read, write }` shadows `File.read` and `File.write`, but
  those take a path, so they were never reachable from a pipe. `Error`, `Ok`, `Err`, `Entry`,
  `Panic`, `Cmd`, `Cmd.Output`, `Html.Element`, `Regex.Match`, `File.Info` and `Url.Parts` have no
  field whose name their namespace uses for a function on them. A script over `prelude.d.ant`, 60
  records against 492 functions, finds one shared name, `Reflect.Field.type` against `Reflect.type`,
  which takes a `Reflect.Value` and so is not offered on a `Field`. The check runs over the
  prelude's declaration file, so a new clash fails the build.
- **Tools:** `ant fmt`, `ant doc`, LSP completion and implementations, the editor grammars
  (vscode-ant, nvim-ant, intellij-ant, the Monaco package), the site and its examples. `ant fmt`
  formats both call forms and never converts one into the other.
- **Tests:** `tests/check/alias_cycles.ant` and `tests/unchecked/type_expr_bare_bound.ant` use
  `Self` as a type name and are renamed.

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
- **Dispatch by result type** (`T.zero()`, `T.parse(s)`). It needs types at run time: `ant run`
  would have to run the checker and pass type arguments to generic calls, so inference would decide
  what a program does. `reflection.md` declined the same machinery for `type(T)` inside generic
  functions and deferred the question here. If dispatch by result type is ever wanted, it is decided
  once for both, together with `Array.sum([])` returning the zero of the element type.
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

## Decisions

A review on 2026-11-04 settled seven questions, and a second on 2026-11-07 four more.

1. **No fields in interfaces.** An interface lists functions only. A structural shape over fields
   stays an anonymous record type or bound, `[T: { name: string }]`, and stays unnamed.
2. **Both call spellings stay.** `xs.map(f)`, `xs | Array.map(f)` and `Array.map(xs, f)` are all
   legal, with no warning and no preferred form. `ant fmt` keeps the spelling the author wrote. The
   cost is two ways to write one call; the gain is that every existing program, pipeline and example
   stays valid and nothing has to be migrated.
3. **Satisfaction is implicit, as in Go.** A type that has the functions satisfies the interface,
   whether or not its author had the interface in mind. Signatures are checked, which makes a wrong
   match unlikely. Prelude interfaces that replace `typed-data.md`'s hooks (see Later) follow the
   same rule when they come.
4. **Numbers have no namespace.** `x.abs()` is a compile error and a panic; `Math.abs(x)` and
   `x | Math.abs()` stay the spellings. `Math` mixes functions on every number, on `float64` only,
   and on none (`Math.random()`), and `Bits` would be left out, so no kind maps to it.

5. **Interface type and bound both stay.** `func total(xs: Array[Shape])` and
   `func total[T: Shape](xs: Array[T])` are both legal, as `Number` and `[T: Ordered]` are today.
   Take the interface; use a bound when the result must keep the caller's type (Interfaces, Type or
   bound).
6. **`Self` is a reserved word,** legal only in an interface's signatures. The interface's own name
   in the receiver position was rejected because `compare(a: Compare, b: Compare)` would mean any
   two comparable values rather than two of one type, and an explicit parameter, `[T: Compare[T]]`,
   because every use would repeat `T`.
7. **Functions use `Iter` as their namespace,** so sequences chain like arrays:
   `Iter.range(0, 1000).filter(f).take(5).toArray()`. The checker's first-parameter rule rejects a
   function that is not a `Seq`. The cost is that a future namespace for functions themselves would
   need another name.
8. **A union whose members share a namespace is one call.** `r.unwrapOr(0)` is checked as
   `Result.unwrapOr(r, 0)`, with the union as the receiver, since resolving per member could not
   solve `E` from an `Ok[int]` alone. Any other union resolves per member. `.` never does more than
   the direct call it stands for.
9. **Interfaces are not data.** A type expression may not reach an interface, and `Reflect` gains no
   shape for one: the interpreter discards the declaration, so the name is unbound at run time, and
   an open set has no members a format could decode into. Typed decoding targets a union.
10. **The first-parameter rule stays loose.** A generic first parameter accepts anything, so
    `s.from()` is `String.from(s)`. The calls are correct and rare, and a rule that read the
    parameter's type for the namespace's name would be a second rule for a cosmetic gain.
11. **`is` from an interface to a generic type follows the anonymous-record rule.** The test narrows
    to the one instantiation that satisfies the interface, and it is a compile error when the
    interface does not determine every type argument.

## Open questions

None at the moment.
