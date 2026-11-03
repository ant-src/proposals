# Proposal: interfaces

2026-11-03 · Anton

## Summary

Add `interface` declarations: a named set of functions that a type provides through its functions on
types (section 6). A record or named primitive type satisfies an interface when it has a function of
each name with the right signature; nothing declares that it does. The interface's name is a
namespace whose functions dispatch on the run-time tag of their first argument, so `Shape.area(s)`
calls `Circle.area` or `Rect.area` according to what `s` holds. An interface is usable as a bound,
`[T: Shape]`, and, when it mentions `Self` only as the receiver, as a type, `Array[Shape]`.

The run-time cost is one dispatcher per interface function, which reads a field of the argument's
tag. The checker gets a new kind of type and a satisfaction check. `interface` is already a reserved
word. Types stay advisory: no evaluation rule consults a written type, and dictionaries are never
passed.

This is a research draft. It records the design space, rejects three alternatives, and recommends
one. Nothing here is part of the language.

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
   map or over a remote store; a storage layer with a SQL and an in-memory implementation. A union
   names every implementation in the library that consumes them, so a user cannot add one.
2. **Test doubles.** A function that sends mail takes the mailer as a parameter so a test can pass a
   fake. With a record of closures the fake is easy, but the real mailer has to be wrapped at every
   call site that passes it.
3. **Generic algorithms over user types.** Sorting records by their natural order, a priority queue
   of jobs, a `max` over user values. `Array.sortWith(xs, Job.compare)` works, but a generic
   `PriorityQueue[T]` has to take the comparator at construction and carry it in every value.
4. **Library protocols.** Go's standard library is built on small interfaces: `io.Reader`,
   `io.Writer`, `fmt.Stringer`, `sort.Interface`, `error`. Ant's library mirrors Go's API but has to
   spell each of these as a concrete type, as `File.Handle` does for every writer.

The record-of-closures idiom covers all four, and it is what Gleam and Elm programs do. It is
explicit and costs no language feature. It is also verbose at the point that matters most, where a
value of a concrete type is handed to code that wants the abstraction:

```ant
type Writer = { write: func(Bytes): Result[void, Error] }

type Buffer = { chunks: Array[Bytes] = [] }

func Buffer.writer(b: Buffer): Writer {
  Writer{ write: func(data: Bytes): Result[void, Error] { Array.push(b.chunks, data); Result.ok(void) } }
}

func report(w: Writer, rows: Array[string]): Result[void, Error] {
  for row in rows { try w.write(Bytes.fromString("${row}\n")) }
  Result.ok(void)
}

report(Buffer.writer(buf), rows)      // every caller adapts
```

The adapter is a second value with identity semantics. Two `Writer`s over one buffer are not `==`,
since functions compare by identity. A `Writer` cannot be narrowed back to a `Buffer`. A function
that wants a writer and a flusher needs a second record type.

## Prior art

| Approach                             | Examples                                                                                            | What Ant takes                                                                                                                                               |
| ------------------------------------ | --------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Implicit, structural method sets     | Go interfaces; TypeScript interfaces (static only); Python `Protocol` (PEP 544, static only)        | The core: satisfaction by having the functions, with no `implements`. Consumers define the interface they need, which Go names as its main lesson            |
| Explicit conformance                 | Java and Kotlin interfaces; Swift protocols; Rust traits; Carbon `impl`; Mojo traits                | Not the declaration. Ant's namespaces are closed, so the type's module is the only place an implementation can be, which gives coherence without `impl`      |
| Type classes with dictionary passing | Haskell; Rust and Swift generics (static dispatch); Scala `given`; Lean; Roc abilities              | Not the mechanism. Dictionaries come from static types, and Ant erases them (section 1). Return-type polymorphism such as `T.zero()` is excluded as a result |
| Constraint-only interfaces           | Go type sets (1.18); Swift protocols with associated types; Rust traits that are not dyn-compatible | The split: an interface that mentions `Self` beyond the receiver is a bound only, like Go's constraint interfaces. Ant's union bounds are already type sets  |
| Records of functions                 | Gleam; Elm; OCaml first-class modules; Zig `std.mem.Allocator`, and `std.Io.Writer` since 0.15      | The run-time model: a call goes through a table of functions. Ant's table is the tag, which already holds the type's functions                               |
| Built-in constraints only            | Elm `comparable`, `number`, `appendable`                                                            | What Ant has today with `Ordered` and `Number`. Elm's long-standing complaint is that user types cannot be `comparable`                                      |
| Run-time structural checks           | Python `@runtime_checkable`, which tests names and not signatures                                   | A warning. A run-time `is Shape` could only test names, and would let a wrongly typed function through. So `is` does not take an interface                   |

### Where languages are going

1. **Generics end up next to dynamic dispatch, not instead of it.** Go added generics in 1.18 and
   kept interfaces as the dynamic half; it approved generic methods on concrete types in 2026 but
   not on interface methods, because its dispatch tables are built per instantiation. Ant has no
   instantiation at run time, so it does not have that limit.
2. **Duck-typed generics give way to explicit tables at library boundaries.** Zig 0.15 replaced the
   generic `anytype` writers, which made every function that touched one generic, with a concrete
   `std.Io.Writer` that calls through a table of function pointers.
3. **Existentials become visible.** Swift now spells an interface used as a type `any P`, and Rust
   renamed object safety to dyn compatibility. Both mark the cost and the restrictions of using an
   interface as a type. Ant's rule below draws the same line without the extra syntax: an interface
   that needs `Self` in other positions is not a type at all.

### Fit with Ant

| Principle or decision                    | Fit    | Why                                                                                                                           |
| ---------------------------------------- | ------ | ----------------------------------------------------------------------------------------------------------------------------- |
| Types are advisory                       | Strong | Dispatch reads the value's tag, which the interpreter already keeps. No rule consults a written type and nothing is passed in |
| Functions on types are fields of the tag | Strong | Section 6 already stores `Circle.area` on the `Circle` tag. A dispatcher is one field read of the argument's tag              |
| Closed namespaces                        | Strong | Only a type's module declares its functions, so there is one implementation per type and name: no orphan rules, no overlap    |
| No method call                           | Good   | The call stays `Shape.area(s)` or `s \| Shape.area()`, a namespace call like `Point.distance`. No new call syntax             |
| One obvious way                          | Good   | Unions are closed and checked for exhaustion; interfaces are open. The guide is one sentence. The weak spot is bound vs type  |
| Unions need discriminators               | Fair   | An interface value has no discriminator, so it joins a union only with `null` and `void`, as a type parameter nearly does     |
| No top type                              | Good   | An interface with no functions is rejected, so `interface Any {}` cannot reintroduce one                                      |
| Library close to Go                      | Strong | Go's library is built on small interfaces, and this is Go's model                                                             |
| Interpreter stays simple                 | Strong | One new declaration that binds a namespace of dispatchers, about fifty lines                                                  |
| JavaScript backend                       | Strong | The same dispatcher, a property read on the tag object                                                                        |

## Design constraints from Ant

Four facts rule out most designs before taste enters.

- **Dispatch must come from a value.** `ant run` discards annotations. Inside
  `func f[T: Show](x: T)` the interpreter does not know `T`; it knows `x`. An interface function
  therefore needs an argument whose tag selects the implementation, and functions such as `T.zero()`
  or `T.parse(s)`, which select by the result type, are out. Haskell's `mempty` and Rust's `Default`
  have no Ant equivalent under any design that keeps types advisory.
- **Only tagged values can dispatch.** Records and named primitive values carry a tag; `int`,
  `string`, arrays, maps, sets, functions and the prelude's resources do not, or their namespaces
  are closed to user code. `type Celsius = float64` is the way to give a number functions, as
  `type Celsius float64` is in Go.
- **There are no methods.** `p.area` reads a field. Any call syntax for interfaces must be a
  namespace call.
- **Namespaces are closed.** A module declares functions only on its own types (section 6). That is
  a restriction on retrofitting, since a library type cannot be made to satisfy a user interface by
  anyone but its author. It is also what Go has, and Go's ecosystem shows that consumer-defined
  interfaces over producer-defined types work: the producer writes the functions, the consumer names
  the subset it needs.

## Alternatives

### A. Keep records of closures, document the idiom

No language change. Section 4 gains a paragraph on `type Writer = { write: func… }`. It is what
Gleam and Elm do, and Zig's new writer is the same table made by hand.

Rejected as the answer, kept as an idiom. It works for state captured at construction, such as a
handler with its configuration. It does not give user types an ordering, it loses `==` and `is` on
the original value, and every caller adapts.

### B. Type classes with dictionary passing

`[T: Show]` makes the checker pass `Show`'s implementation for `T` as a hidden argument, as Haskell
and Rust do. It supports `T.zero()` and selects implementations by type, not by value.

Rejected. It makes a checked program behave differently from an unchecked one, which section 1
forbids. The reflection proposal rejected reified generics for the same reason.

### C. Hooks by name

Fixed names that the library looks up on a tag: `toString` for `String.from`, `compare` for
`Array.sort`, `fromData` and `toData` as in `reflect.md`. No user-defined interfaces.

Rejected as the general mechanism. It serves the library and not user code, and it is what this
proposal provides anyway: a prelude interface is a hook with a declared signature (see Later).

### D. Interface as a named structural record type

`interface Named { name: string }` names a shape that `{ name: string }` spells today. It closes the
gap in the Motivation table, since an anonymous record type cannot be named. It does not add
dispatch. It is left to open question 1, as a possible second half of this proposal.

## The proposal

### Declaration

```ant
interface Shape {
  area(s: Self): float64
  name(s: Self): string
}
```

`interface` declares a name at the top level of a module, and `export interface` exports it. The
body lists function signatures, one per line, in the syntax of a function type with parameter names.
Inside the body, `Self` stands for the satisfying type. The first parameter of every function is the
receiver and has type `Self`; that is what dispatch reads. The body has at least one function, so no
interface is satisfied by everything.

An interface may take type parameters, and it may include another interface's functions with a
spread, as a record literal does:

```ant
interface Reader { read(r: Self, max: int): Result[Bytes | null, Error] }
interface Writer { write(w: Self, data: Bytes): Result[void, Error] }
interface ReadWriter { ...Reader, ...Writer }

interface Container[T] {
  get(c: Self, i: int): T | null
  size(c: Self): int
}
```

An interface function may have type parameters of its own, `map[U](c: Self, f: func(T): U): …`. Go
forbids this because its dispatch tables are built per instantiation. Ant's dispatch ignores type
arguments, so the only cost is in the checker.

### Satisfaction

A type `S` satisfies interface `I` when:

- `S` is a named record type or a named primitive type, or a union whose every member satisfies `I`
  when `I` mentions `Self` only as the receiver; and
- for each function `f` of `I`, `S` has a function `f` declared on it (section 6), visible where the
  checker asks, whose type, with `Self` replaced by `S` and the function's own type parameters
  solved, is assignable to `f`'s signature under the rules of Assignability.

No declaration says that `S` satisfies `I`. A generic record satisfies an interface for every
instantiation its functions cover: `func Box.show[T](b: Box[T]): string` makes every `Box[X]` a
`Show`, and `func Box.total(b: Box[int]): int` makes only `Box[int]` a `Summable`.

Visibility is checked where a value meets the interface: at an assignment to an interface type or at
a bound being solved. A type whose `area` is not exported satisfies `Shape` inside its own module
and in its test files, and nowhere else.

### As a type

An interface whose functions mention `Self` only as the first parameter is a type. A value of any
satisfying type is assignable to it, and an interface `J` is assignable to `I` when `J` has every
function of `I`. Generic containers stay invariant, so `Array[Circle]` is not an `Array[Shape]` and
`[...circles]` widens, as for unions.

```ant
var shapes: Array[Shape] = [Circle{ radius: 1.0 }, Rect{ width: 2.0, height: 3.0 }]
var total = shapes | Array.map(Shape.area) | Array.sum()
```

An interface value is the value itself: no box, no second identity. `==`, `String.from`, and
`Json.stringify` see the underlying record. `is` narrows it to a satisfying type, and the `else`
branch keeps the interface, since the set of satisfying types is open:

```ant
if s is Circle { println(s.radius) }
```

`is` does not take an interface on its right. At run time it could test only that the tag has
functions of the right names, not their signatures, which is Python's `runtime_checkable` gap.

An interface has no discriminator, so a union may hold one only beside `null` and `void`.
`Shape | null` is fine, and `Shape | Error` is not, since a type that satisfies `Shape` today may
gain or lose a function tomorrow and move the line. `Result[Shape, Error]` wraps them instead.

### As a bound

Every interface is a bound: `[T: Shape]` admits any type that satisfies `Shape`. An interface that
mentions `Self` beyond the receiver is only a bound, since a value of it would let `compare` receive
a `Circle` and a `Rect`:

```ant
interface Compare {
  compare(a: Self, b: Self): int
}

type Job = { priority: int, name: string }

func Job.compare(a: Job, b: Job): int { b.priority - a.priority }

func sorted[T: Compare](xs: Array[T]): Array[T] { Array.sortWith(xs, Compare.compare) }

var queue: Array[Compare] = []       // compile error: Compare mentions Self in compare's second parameter
```

Within a bound, a union solves `T` only when every member satisfies the interface and the interface
is a type, matching the rule that a union bound needs a single member: `Compare` with
`Circle | Rect` would compare across members.

A bound and an interface type overlap for receiver-only interfaces. `func total(xs: Array[Shape])`
takes a mixed array; `func largest[T: Shape](xs: Array[T]): T` keeps the element type in the result.
The guide in section 4 is to take the interface type, and to reach for a bound when the result must
keep `T` or when the interface is a bound only.

### Calls

An interface's name is a namespace (section 10) holding one function per interface function. Its
type is the signature with `Self` a type parameter bounded by the interface: `Shape.area` is
`func[S: Shape](s: S): float64`, so it is a value like any other, and
`Array.map(shapes, Shape.area)` needs no lambda. `s | Shape.area()` works through the pipe.

There is no `s.area()`. `p.area` is still a field read. A satisfying type's own function stays
callable directly: `Circle.area(c)` calls the implementation with no dispatch.

### Run time

The interpreter binds an interface's name to a namespace record. Each field is a dispatcher: it
reads the tag of its first argument, reads the field of that name from the tag, and calls it with
all the arguments. A named primitive value's functions live on its constructor, as section 12 says,
and the dispatcher reads them there.

A dispatcher panics when its first argument has no tag, or when the tag has no function of that
name: `Shape.area: int has no function area`, or `Shape.area: Circle has no function area`. Only an
unchecked program reaches either panic. The interpreter does not check signatures, so a function of
the right name and the wrong shape is called and fails as any wrong call does. Privacy is not
checked either, as with any field read of a tag.

A dispatcher is not a frame of its own: a panic trace, the debugger, and coverage show the call
going straight from the caller to `Circle.area`.

An interface name used where a tag is expected, after `is` or as a literal head, is a compile error
and a panic.

### Local declarations

An interface is declared only at the top level, since functions on types are, and a local type has
none (section 4, Type declarations).

## Examples

A writer with a test double and no adapter:

```ant
// log.ant
export interface Sink {
  write(s: Self, line: string): Result[void, Error]
}

export func info(sink: Sink, msg: string): Result[void, Error] { Sink.write(sink, "info ${msg}") }

// main.ant
import "./log.ant" { Sink, info }

type FileSink = { handle: File.Handle }

func FileSink.write(s: FileSink, line: string): Result[void, Error] { File.writeLine(s.handle, line) }

var out = FileSink{ handle: try File.create("app.log") }
try info(out, "started")

// log_test.ant
import "./log.ant" { info }

type Memory = { lines: Array[string] = [] }

func Memory.write(m: Memory, line: string): Result[void, Error] {
  Array.push(m.lines, line)
  Result.ok(void)
}

test "info prefixes" {
  var m = Memory{}
  Result.unwrap(info(m, "hi"))
  assert(m.lines == ["info hi"])
}
```

A user type with an ordering, used by a generic container:

```ant
type Heap[T] = { items: Array[T] = [] }

func Heap.push[T: Compare](h: Heap[T], v: T): void {
  Array.push(h.items, v)
  var i = len(h.items) - 1
  while i > 0 && Compare.compare(h.items[i], h.items[(i - 1) / 2]) < 0 {
    var p = (i - 1) / 2
    [h.items[i], h.items[p]] = [h.items[p], h.items[i]]
    i = p
  }
}
```

## Spec changes

- §1 Principles: none. The design is chosen to need none.
- §2 Lexical structure: `interface` moves from reserved words to keywords. `Self` becomes a type
  name inside an interface body (open question 3).
- §4 Types: a new subsection, Interfaces, with declaration, satisfaction, type vs bound, and the
  union rule; a row in the table of Kinds ("its argument's", `is`: none, union: with `null` and
  `void`); Generics drops "There are no interfaces" and gains interface bounds; Assignability gains
  the two interface rules; Type declarations drops "interfaces" from "no classes, interfaces, or
  prototypes"; a sentence on when to use a union and when an interface.
- §6 Functions on types: a function on a type may satisfy interfaces; nothing else changes.
- §10 Namespaces: a fourth kind of namespace, an interface, whose functions dispatch.
- §12 Dynamic semantics: the dispatcher, its two panics, and its absence from traces.
- `fmt.md`: formatting of the declaration. `doc.md`: an interface's page, and the interfaces a type
  satisfies among those its module can see. `lsp.md`: go to implementations from an interface
  function, and the interface functions a type's function satisfies.

## Implementation

- **Parser:** the declaration, the spread, `Self`.
- **Interpreter:** bind the namespace of dispatchers; skip dispatcher frames in traces, the
  debugger's stack, and coverage. Small.
- **Checker:** most of the work. An interface type, satisfaction with `Self` substitution and
  generic functions, the two assignability rules, the type-vs-bound rule, bounds solved by
  satisfaction, `is` narrowing from an interface, and the union rule. Satisfaction results should be
  cached per (type, interface).
- **Tools:** `ant fmt`, `ant doc`, LSP (implementations, hover, completion after `Shape.`), the
  editor grammars (vscode-ant, nvim-ant, intellij-ant, the Monaco package), the site.

## Non-goals

- **Method call syntax.** `s.area()` would make a field read and a call depend on what the field is,
  and the spec chose namespace calls on purpose.
- **Return-type dispatch** (`T.zero()`, `T.parse(s)`). It needs types at run time.
- **Default implementations** in an interface. An interface is a list of signatures; a shared helper
  is an ordinary function that takes the interface.
- **Built-in types satisfying user interfaces.** `int`, `string`, arrays, maps and the prelude's
  resources have no tag or a closed namespace. Wrap them in a named type (see Later).
- **`is` with an interface**, for the reason in As a type.
- **Explicit `implements`.** Closed namespaces already give one implementation per type; a
  declaration would only add a place to forget.

## Later

- **Prelude interfaces as hooks.** Once interfaces exist, the library's customization points can be
  interfaces with checked signatures instead of names: a `toString` for `String.from`, a `compare`
  that lets `Array.sort` take user types, and `reflect.md`'s `fromData`/`toData`/`validate`. Each is
  a separate decision, since each changes what a built-in function does to a record.
- **I/O interfaces.** `Io.Reader` and `Io.Writer` over `File.Handle`, network connections and user
  types, as in Go. This needs dispatch on a resource's kind as well as on a tag, since prelude
  resources have no tag; the prelude could declare which of its functions satisfy which prelude
  interface.
- **Built-in kinds in dispatch.** The same mechanism would let `string` satisfy an interface through
  `String.*` functions, if the prelude declares it.

## Open questions

1. **Fields in interfaces.** Should an interface also list fields,
   `interface Named { name: string }`, checked as an anonymous record type is? It would let a
   structural shape have a name, which today is impossible, and one keyword would cover both. It
   also mixes a structural field rule and a nominal function rule in one declaration.
2. **Bound or type for receiver-only interfaces.** Two spellings for one job is a "one obvious way"
   cost. The alternative is Go's rule that only a bound keeps `T`, which this proposal follows, or
   requiring a marker such as Swift's `any`.
3. **`Self`.** A reserved word, a contextual name inside interface bodies, or the interface's own
   name in the receiver position (`area(s: Shape)`), which is shorter but reads as "takes any Shape"
   and cannot express `compare(a: Self, b: Self)`.
4. **Unions.** The rule allows an interface only beside `null` and `void`. A looser rule would allow
   a named type that does not satisfy the interface, at the price of a distant union breaking when a
   type gains a function.
5. **Accidental satisfaction.** A type with a `close` function of the right shape satisfies every
   `Closer`. Go accepts this; `reflect.md` rejected hooks that a function could become "by
   accident". Signatures being checked makes it unlikely to matter, but the two proposals should
   agree.
6. **Privacy at run time.** A dispatcher calls a private function of a type in an unchecked program.
   That matches field reads of tags today; it is noted here so that the decision is visible.
