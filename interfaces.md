# Proposal: interfaces and `->`

2026-11-03 · Anton

## Summary

Two additions, one for the run time and one for the checker.

- **`x->f(a)`** calls the function `f` declared on the type of `x`'s tag: `p->norm()` is
  `Point.norm(p)` when `p` is a `Point`. The interpreter finds `f` on the value's tag at the call,
  so the same expression calls `Circle.area` or `Rect.area` according to what `s` holds.
- **`interface Shape { area(s: Self): float64 }`** names a set of functions. A record or named
  primitive type satisfies it by having those functions; nothing declares that it does. An interface
  is a type, `Array[Shape]`, and a bound, `[T: Shape]`. It exists only in the checker: the
  interpreter discards it as it discards an annotation.

The interpreter gains one rule, the `->` call, which is a field read of a tag that it already does
for `Point.norm`. The checker gains an interface type and a satisfaction check. `interface` is
already a reserved word, and `->` is unused. Types stay advisory: no evaluation rule consults a
written type, and nothing is passed in behind the program's back.

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

A smaller irritation applies to every user type: a function on a type is always called through the
type's name, `Point.norm(p)` or `p | Point.norm()`, so an editor cannot offer a value's functions,
and code that holds a union or a type parameter has no call that picks the member's function.

## Prior art

| Approach                                       | Examples                                                                                        | What Ant takes                                                                                                                             |
| ---------------------------------------------- | ----------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Implicit, structural method sets               | Go interfaces; TypeScript interfaces; Python `Protocol` (PEP 544)                               | Satisfaction by having the functions, with no `implements`. Consumers define the interface they need, which Go names as its main lesson    |
| Explicit conformance                           | Java, Kotlin; Swift protocols; Rust traits; Carbon `impl`                                       | Not the declaration. Ant's namespaces are closed, so the type's module is the one place an implementation can be: coherence without `impl` |
| Type classes, dictionary passing               | Haskell; Rust and Swift generics; Scala `given`; Roc abilities                                  | Not the mechanism. Dictionaries come from static types, which Ant erases. Dispatch by result type, `T.zero()`, is excluded as a result     |
| Constraint-only interfaces                     | Go type sets; Swift protocols with associated types; Rust traits that are not dyn-compatible    | The split: an interface that mentions `Self` beyond the receiver is a bound only                                                           |
| Call syntax that searches the type's namespace | Lean generalized field notation (`xs.map f` is `List.map xs f`); D and Nim UFCS; Koka dot calls | The `->` call. Lean is closest: the namespace of the value's type is searched, and the value becomes the first argument                    |
| A separate operator for the call               | Odin `x->f()`; PHP `->`; Elixir and Gleam pipes                                                 | A token distinct from `.`, so that a field read and a call stay distinguishable to the reader and the interpreter                          |
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
- **Only tagged values have functions of their own.** Records and named primitive values carry a
  tag, and section 6 stores a type's functions on it. `int`, `string`, arrays, maps, functions and
  the prelude's resources have no tag. `type Celsius = float64` is how a number gets functions, as
  `type Celsius float64` is in Go.
- **`.` is a field read.** A record of closures calls `w.write(data)`, a field read and a call; that
  stays. A call of a function on a type needs its own token.
- **Namespaces are closed.** Only a type's module declares its functions (section 6), so a type has
  one function per name, and no orphan or overlap rules are needed. A library type cannot be made to
  satisfy a user interface by anyone but its author, which is Go's rule too.

## The proposal

### The `->` call

`x->f(a, b)` is a postfix call, at the precedence of `.` and calls, so it chains:
`p->scale(2.0)->norm()`. At run time it:

1. evaluates `x`;
2. reads the tag of the value: a record's tag, or a named primitive value's constructor. A value
   with no tag panics: `int has no functions`;
3. reads the function `f` declared on that type, which section 6 already stores on the tag. A tag
   without one panics: `Circle has no function area`;
4. evaluates the arguments and calls the function with `x` first.

That is the whole run-time rule. `p->norm()` and `Point.norm(p)` call the same function when `p` is
a `Point`, and the type name stays the way to call a function that is not on the receiver's type,
such as `Point.origin()`. The interpreter checks neither signatures nor privacy, as for any field
read of a tag.

The checker resolves `x->f` by the static type of `x`:

| Type of `x`                            | `f` is                                                                                      |
| -------------------------------------- | ------------------------------------------------------------------------------------------- |
| a named record or named primitive type | the function `f` on that type, if it is visible here                                        |
| a union of those                       | `f` on every member; each must accept the arguments, and the result is the union of results |
| an interface                           | the interface's `f`, with `Self` as the interface                                           |
| a type parameter                       | as for its bound                                                                            |
| anything else                          | a compile error: built-in kinds, anonymous record types, aliases' own functions             |

The union row is the field rule of section 4, Tagged unions, applied to functions: a union of types
that each declare `area` can call `s->area()` without an interface. Its result must be a valid
union.

A function declared on an alias, `func Shape.area` with `type Shape = Circle | Rect`, is not reached
by `->`. A `Shape` value carries `Circle`'s tag, and the interpreter looks there. It is still called
as `Shape.area(s)`.

Named arguments apply when the checker knows the one function called: the receiver's static type is
a named type. Through a union, an interface or a type parameter, the call is positional, since each
implementation may name its parameters differently and the interpreter binds names against the
function it finds.

`x?->f()` is not part of the proposal. A nullable receiver is narrowed first (open question 4).

`->` joins `.`, `?.`, `|` and `else` in the line continuation rule (section 2), so a chain may break
before `->`.

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

A type `S` satisfies `I` when `S` is a named record type or a named primitive type, and for each
function `f` of `I`, `S` declares `f` (section 6), visible where the checker asks, whose type with
`Self` replaced by `S` is assignable to `f`'s signature. No declaration says so. A generic record
satisfies an interface for each instantiation its functions cover: `func Box.show[T](b: Box[T])`
makes every `Box[X]` a `Show`.

**As a type.** An interface whose functions mention `Self` only as the receiver is a type. A value
of a satisfying type is assignable to it, as is a union whose every member satisfies it, and so is a
value of an interface that has all of its functions. Containers stay invariant: `Array[Circle]` is
not an `Array[Shape]`, and `[...circles]` widens. An interface value is the value itself, with no
box: `==`, `String.from` and `Json.stringify` see the record. `is` narrows it to a satisfying type,
and the `else` branch keeps the interface, since the set of satisfying types is open:

```ant
var shapes: Array[Shape] = [Circle{ radius: 1.0 }, Rect{ width: 2.0, height: 3.0 }]
var total = shapes | Array.map({ x->area() }) | Array.sum()

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

func sorted[T: Compare](xs: Array[T]): Array[T] { Array.sortWith(xs, func(a: T, b: T): int { a->compare(b) }) }

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
annotation, and every dynamic call goes through `->`.

### Choosing between the tools

The spec gets one paragraph of guidance:

- A closed set of variants is a union, narrowed with `is` and checked for exhaustion.
- An open set is an interface, called with `->`.
- A function on the receiver's own type is called with `->`; any other function, a module's, a
  namespace's, or a library's on a built-in kind, through `|` or a plain call.
- State captured at construction, such as a handler and its configuration, stays a record of
  closures.

## Examples

A sink with a test double and no adapter:

```ant
// log.ant
export interface Sink {
  write(s: Self, line: string): Result[void, Error]
}

export func info(sink: Sink, msg: string): Result[void, Error] { sink->write("info ${msg}") }

// main.ant
import "./log.ant" { info }

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

A user ordering in a generic container:

```ant
type Heap[T] = { items: Array[T] = [] }

func Heap.push[T: Compare](h: Heap[T], v: T): void {
  Array.push(h.items, v)
  var i = len(h.items) - 1
  while i > 0 && h.items[i]->compare(h.items[(i - 1) / 2]) < 0 {
    var p = (i - 1) / 2
    [h.items[i], h.items[p]] = [h.items[p], h.items[i]]
    i = p
  }
}

var jobs: Heap[Job] = Heap{}
jobs->push(Job{ priority: 2, name: "build" })
```

## Spec changes

- §2 Lexical structure: `interface` moves from reserved words to keywords; `->` is a token; `Self`
  is a type name inside an interface body; `->` joins the continuation rule.
- §4 Types: a subsection, Interfaces: declaration, satisfaction, type or bound, the union rule; a
  row in the table of Kinds; Generics drops "There are no interfaces" and gains interface bounds;
  Assignability gains the interface rules; Type declarations drops "interfaces" from "no classes,
  interfaces, or prototypes"; the guidance paragraph above.
- §5 Expressions: the `->` call, its checker table, and its precedence. Pipes: "The pipe replaces
  method chaining" becomes "The pipe chains calls of any function; `->` chains calls of a value's
  own functions."
- §6 Functions on types: "There is no method call" becomes the `->` call; functions on aliases are
  reached only through the alias's name.
- §12 Dynamic semantics: the four steps of `->` and its two panics.
- `fmt.md`: the declaration, and breaking a chain before `->`. `doc.md`: an interface's page, and
  the interfaces a type satisfies among those its module sees. `lsp.md`: completion after `->`, go
  to implementations of an interface function.

## Implementation

- **Parser:** the `->` postfix, the `interface` declaration, `Self`.
- **Interpreter:** the `->` call, a few lines beside the field read. Interface declarations are
  dropped like annotations. Nothing else.
- **Checker:** most of the work: the resolution table for `->`, the interface type, satisfaction,
  assignability, type or bound, `is` narrowing from an interface, the union rule. Satisfaction can
  be cached per type and interface.
- **Tools:** `ant fmt`, `ant doc`, LSP completion and implementations, the editor grammars
  (vscode-ant, nvim-ant, intellij-ant, the Monaco package), the site.

## Non-goals

- **`->` on built-in kinds.** `"abc"->trim()` would need a map from each kind to a prelude
  namespace, and the namespaces are not organized by receiver: `File.write` takes a path, not a
  handle, and numbers have no namespace. It would also make `|` nearly redundant. Library calls stay
  `s | String.trim()`.
- **Changing `.`.** `x.f` stays a field read, and `x.f()` calls a function stored in a field.
- **Dispatch by result type** (`T.zero()`, `T.parse(s)`). It needs types at run time.
- **Default implementations, explicit `implements`, embedding with `...`, generic interface
  functions.** Each adds rules for a convenience. Embedding and generic functions are in Later.
- **`is` with an interface**, for the reason in Interfaces.

## Later

- **Prelude interfaces as hooks.** A `toString` for `String.from`, a `compare` that lets
  `Array.sort` take user types, and `reflect.md`'s `fromData`/`toData`/`validate`, as interfaces
  with checked signatures instead of names. Each changes what a built-in function does to a record,
  so each is its own decision.
- **I/O interfaces** over `File.Handle`, network connections and user types, as in Go. Prelude
  resources have no tag, so this needs `->` on them, which is the built-in-kinds question again.
- **Embedding**, `interface ReadWriter { ...Reader, ...Writer }`, if listing the functions again
  proves common.
- **Generic interface functions.** The interpreter would need nothing, since `->` ignores type
  arguments; only the checker's satisfaction rule grows.

## Open questions

1. **Fields in interfaces.** `interface Named { name: string }` would give a structural shape a
   name, which today is impossible. It mixes a structural field rule with a function rule in one
   declaration.
2. **Bound or type for `Self`-receiver interfaces.** `func total(xs: Array[Shape])` and
   `func total[T: Shape](xs: Array[T])` overlap. The rule here follows Go: take the interface type,
   and use a bound when the result must keep `T`.
3. **Two spellings for one call.** `p->norm()` and `p | Point.norm()` call the same function. The
   guidance picks `->`; `ant fmt` cannot enforce it without types, but the checker or the LSP could
   suggest it.
4. **Nullable receivers.** `x?->f()`, yielding `null` when `x` is `null`, as `?.` does for fields.
5. **Accidental satisfaction.** A type with a `close` of the right shape satisfies every `Closer`.
   Go accepts this; `reflect.md` rejected hooks that a function could become by accident. The two
   proposals should agree.
6. **`Self`.** A reserved word, or a type name only inside interface bodies.
