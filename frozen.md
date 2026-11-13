# Proposal: frozen values

2026-11-02 · Anton

## Summary

Add one prelude function, `freeze`, and one type operator, `Frozen[T]`. `freeze(v)` returns a deep,
immutable copy of `v`. Arrays, maps, sets and records reached from `v` are copied once each, and
values that are already immutable are shared. A frozen value has the same kind, `==`, hash,
rendering and `is` result as its mutable original. The only difference is that a write to it panics.
The checker types the result as `Frozen[T]`, rejects writes through that type, and treats frozen
containers as covariant. Writes are also guarded at run time, because the checker cannot see every
write: `ant run` executes code that was never checked, and `Frozen[T]` is assignable to `T`, so a
frozen value can reach code that writes through a plain type. The proposal adds no keyword and no
new collection types, and reference semantics stay as they are.

## Motivation

Section 4 says: "Fields are mutable, and there is no `readonly` or frozen form. Strings, time
values, and patterns are immutable; records, arrays, maps, sets, and `Bytes` are not." That leaves
five problems that every non-trivial program runs into.

1. **Sharing across goroutines needs a lock even when nobody writes.** A configuration loaded once
   and read by every handler is a data race (section 9) if anything could write it, and nothing in
   the language says that nothing will. The usual pattern is to keep the current config in an
   `Atomic` and replace it on reload. In Ant that pattern is only safe by convention, because a
   reader holding an `Atomic.load` result can still write `config.routes[k] = v`.
2. **Composite keys can break silently.** Section 4: a map key or set member "is hashed when it is
   inserted, and mutating it afterwards is a program error like a data race: memory stays safe, but
   the entry may become unreachable." The language invites records and arrays as keys and then
   offers no way to make them safe.
3. **Library state leaks.** A module that exports `var Weekdays = ["mon", …]` or returns an internal
   array hands every caller write access to it. Imports are read-only bindings (section 10), but the
   values they hold are not. The only protection is a defensive copy on every call.
4. **Shared defaults are a trap.** `tags: Array[string] = shared` shares one array among all records
   (section 4, Field defaults). The spec warns about it because nothing can make the sharing safe.
5. **Snapshots and history.** Undo stacks, a UI's previous state (`ui.md`), a cache entry, and a
   message on a channel all want to know that a value will not change underneath them. Today the
   receiver has to copy on receipt and trust the sender to copy on send.

Strings, time values and patterns show what immutability buys: such a value can be shared without
copying and serves as a map key. This proposal extends that property to the five mutable containers
without adding a second set of them. `Bytes` was immutable until the mutable-bytes change (spec
923b171) made it the working storage for binary data; `Frozen[Bytes]` gives the immutable bytes back
where a program wants them.

## Prior art

| Approach                              | Examples                                                                                                                                                                                                            | What Ant takes                                                                                                                                                                            |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Read-only view in the type system     | TypeScript `readonly`, `ReadonlyArray`, `as const`; C++ `const`; Go proposals #22876, #29192, #31464 (none accepted)                                                                                                | The static half only. Unchecked code and mutable aliases still write, and `const` spreads through every signature that touches the value ("contagion")                                    |
| Freeze in place                       | JavaScript `Object.freeze`; Ruby `freeze`                                                                                                                                                                           | Nothing. It is shallow, it freezes the value under every other name that holds it, and it races with a writer in another goroutine                                                        |
| Deep freeze to share between threads  | Ruby `Ractor.make_shareable` (and the open `deep_freeze`, Feature #21962); Python PEP 795 (subinterpreters, not accepted); Dart VM deeply immutable objects for isolates                                            | The purpose and the depth: a value that crosses to another thread must be immutable all the way down. Ant copies where these freeze in place                                              |
| Frozen copy, same interface           | Java `List.copyOf`, `Map.copyOf` (throw on mutation); Python `tuple`, `frozenset`, `frozendict` (PEP 814, Python 3.15); .NET `ToFrozenDictionary`, `ToFrozenSet`                                                    | The core model: a copy that is immutable, built once and read many times. Java shows the cost of runtime-only errors, so the checker catches direct writes                                |
| Read-only interface over mutable      | Kotlin `List` vs `MutableList`; .NET `IReadOnlyList`                                                                                                                                                                | Covariance. Kotlin's `List<out E>` is covariant because it has no writes, and `Frozen[Array[T]]` gets the same rule for the same reason                                                   |
| Build mutable, publish frozen         | Clojure transients and `persistent!`; Immer.js `produce` (Redux Toolkit), which also freezes its result                                                                                                             | The workflow: build or edit with ordinary writes, then `freeze` once. Ant's mutable default plays the role of Clojure's transient                                                         |
| Persistent collections                | Clojure; Scala; Immutable.js; Rust `im`, `rpds`; Kotlin `kotlinx.collections.immutable`; .NET `ImmutableDictionary`                                                                                                 | Not now. A second collection API breaks "one obvious way". The representation is left open so that a later proposal can add O(log n) updates (see Later)                                  |
| Frozen value with structural equality | TC39 Records & Tuples (withdrawn April 2025), its successor Composites (Stage 2)                                                                                                                                    | Confirmation. JavaScript dropped new deep-immutable primitives and kept frozen objects with structural equality. Ant already has structural `==` and keys, and lacks only the frozen part |
| Value semantics, copy on write        | Swift `Array`, `Dictionary`; Hylo and Mojo mutable value semantics                                                                                                                                                  | Nothing. Changing what assignment means would break section 9's sharing idioms and every program that relies on aliasing                                                                  |
| Immutable by default                  | Elm, Gleam, Elixir, Haskell; Roc, Koka and Lean, which reuse memory in place through reference counting (Perceus, FBIP)                                                                                             | Nothing. Ant is an imperative language on a garbage collector; in-place reuse needs precise reference counts that the Go and JavaScript runtimes do not have                              |
| Declaration-level immutability        | Python `@dataclass(frozen=True)`; Kotlin `val` properties; C# `readonly record struct`, `init`; Java `record`, `final` fields, value classes (JEP 401, preview); Julia `struct` (immutable unless `mutable struct`) | Left open (see Open questions). Shallow fields do not give the sharing guarantee, but a frozen-by-declaration type would                                                                  |
| Capabilities for sharing              | Pony `val`, `iso`; Swift `Sendable`; Rust `Send`, `Sync` with ownership; OxCaml `portable` and `contended` modes                                                                                                    | The runtime fact that a frozen value is safe to share. Proving race freedom statically needs ownership, which Ant does not have                                                           |

Three lessons shape the proposal:

- **In Ant, the guarantee must live in the value.** Two paths let a write reach a frozen value
  without the checker seeing it: `ant run` executes unchecked code (section 1, "Types are
  advisory"), and `Frozen[T]` is assignable to `T` to avoid `const` contagion. Type erasure alone is
  not the reason: a fully checked program without frozen-to-mutable assignability would need no
  run-time check. A `readonly` annotation closes neither path. Python (unchecked) and Java and .NET
  (frozen collections behind the mutable interface) put the property on the object for the same
  reasons.
- **Copy, don't freeze in place.** `Object.freeze` changes a value that other names already hold,
  and in Ant that might be another goroutine in the middle of writing it. A copy leaves the original
  alone and costs only what is still mutable, since frozen parts are shared.
- **Don't fork the collection API.** Clojure and Immutable.js need a parallel world of types and
  conversions at every boundary. A frozen array is still an array, so `Array.map`, `for`, `len` and
  destructuring work on it unchanged.

### Where languages are going

The last two years show three trends. Ant's design follows the first two and leaves the third open.

1. **Mutable languages add frozen collections beside their mutable ones, not instead of them.**
   Python accepted `frozendict` for 3.15 in February 2026 and cited free threading as a reason. .NET
   8 added `FrozenDictionary` and `FrozenSet` next to its persistent `Immutable*` types, because
   most immutable tables are built once and read many times. JavaScript withdrew Records & Tuples
   and moved on to Composites, frozen objects usable as map keys. None of these languages changed
   what assignment means or made mutation the exception.
2. **Sharing between threads is what pushes freezing to go deep.** `frozendict` and Composites are
   shallow, and both say so. Every design aimed at sharing between threads goes deep: Ruby's
   `Ractor.make_shareable`, Python's PEP 795 for subinterpreters, and the Dart VM's deeply immutable
   objects for isolates. Ant has shared-memory goroutines, so it needs the deep form. The main
   criticism of PEP 795 was what a deep freeze does to classes, functions and modules it reaches.
   Ant has no classes, and rule 2 keeps functions and sync objects by reference.
3. **Record types are becoming immutable by declaration.** Examples are Java records and value
   classes (JEP 401), Kotlin's planned deeply immutable value classes, C# `readonly record struct`,
   and Julia, where `struct` is immutable and `mutable struct` is the marked case. Julia's reason,
   that immutability makes identity irrelevant, applies to Ant too, whose `==` is already
   structural. It is open question 2 here, not part of the proposal, because Ant's records are
   mutable today and changing the default would break every program.

Static approaches have not spread. TypeScript's `readonly` is used mostly at API edges, and Go has
declined every read-only proposal. Swift, Rust, Pony and OxCaml get race freedom from a type system
that tracks ownership or modes, which Ant deliberately does not have (section 1, "Types are
advisory").

### Fit with Ant

| Principle or decision                   | Fit     | Why                                                                                                                                                             |
| --------------------------------------- | ------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Types are advisory                      | Strong  | Unchecked runs and `Frozen[T]`-to-`T` assignment both bypass the checker, and the flag on the value catches both. This is the main reason not to use `readonly` |
| One obvious way                         | Good    | One function, and no second collection API. The cost is that every container has two run-time states                                                            |
| Minimal syntax                          | Strong  | No keyword and no literal. `freeze([…])` covers constants                                                                                                       |
| Structural `==`, composite map keys     | Strong  | Ant already has what Composites add to JavaScript. Frozen keys close the "mutating a key" hole                                                                  |
| Goroutines and shared memory            | Strong  | A frozen value sent on a channel can't be changed by the sender afterwards, which is a common Go bug. `Atomic` plus `freeze` gives snapshots                    |
| `string`, `Html` are immutable          | Good    | The same property, extended to containers, `Bytes` among them since it became mutable                                                                           |
| Library close to Go                     | Weak    | Go has nothing like it and relies on convention and the race detector. Ant already goes beyond Go with structural `==`                                          |
| Interpreter stays simple                | Good    | One flag, about a dozen checks, and one graph walk. Persistent structures are deferred for this reason                                                          |
| JavaScript backend, exact results       | Good    | Uses the same flag, without `Object.freeze`, so behavior is identical                                                                                           |
| A checked program panics only on values | Partial | `Frozen[T]` assignable to `T` lets a frozen value reach a plain-typed write. That is the Java trade-off, made smaller by the checker                            |

Overall, the core fits Ant better than it fits most of the languages above. Ant runs unchecked code
and lets frozen values flow into plain types, so only a run-time property can give a guarantee. Its
equality is already structural, so frozen values are good keys with no further work. Its concurrency
is shared memory, so deep immutability is the cheapest way to share safely. The weak points are
deliberate: the checker gap avoids `const` contagion, and O(n) updates keep the interpreter simple.

## The proposal

```
freeze[T](v: T): Frozen[T]       // a deep immutable copy of v; v itself when v is already immutable
```

`freeze` is a prelude function. `Frozen[T]` is a type operator built into the checker, as `len` is.

### Run time

1. **What freezes.** An array, map, set, record or `Bytes` that is not frozen is copied. Each
   element, key, value and field of the copy is the frozen form of the original's. The copy is
   marked frozen before `freeze` returns it, so no other goroutine can see it mutable.
2. **What is shared.** A value that is already frozen is returned as it is, with no copy, so
   `freeze(freeze(x))` is `freeze(x)` and costs nothing. Numbers, strings, `Html`, tagged
   primitives, time values, patterns, tags, `null` and `void` are immutable already and are returned
   unchanged. Functions, sync objects, `Random` generators and resources are kept by reference too.
   They have identity rather than contents, and a frozen config that holds a channel or a handler
   holds that same channel or handler.
3. **Graphs.** Each mutable value reached more than once is copied once, so the frozen copy keeps
   the original's sharing and cycles. The cost is linear in the mutable part of the graph.
4. **Reads.** `freeze` reads its argument the way every library function does (section 11). A writer
   in another goroutine is a data race, and `freeze` sees that writer's stores or not, as section 9
   allows. To freeze shared state, call `freeze` under the lock that guards it.
5. **Writes panic.** These all panic on a frozen value: `a.b = v`, `a[i] = v` (including compound
   assignment and destructuring assignment), and the nine mutating functions `Array.push`, `extend`,
   `pop`, `insert`, `removeAt`, `Map.set`, `Map.remove`, `Set.add` and `Set.remove`. The message
   names the kind: `assignment to a frozen record`, `Array.push on a frozen array`. The guard sits
   on the container's store paths only: field, element, entry and member stores and the nine
   functions. Reads, iteration, `len`, hashing and `==` never check the flag, and neither does
   assigning to a variable: `p = q` rebinds `p` and writes nothing frozen, whatever `p` or `q`
   holds. Storing a frozen value into a mutable container is an ordinary write.
6. **Everything else is unchanged.** A frozen value has the same kind, so `is Array`, `is Point`,
   `==`, hashing, `String.from`, `Json.stringify`, `for`, `len`, indexing and destructuring treat it
   like any other value of that kind, and `freeze([1]) == [1]` is `true`. `Array.same` and the other
   identity functions see a new value: `Record.same(p, freeze(p))` is `false` unless `p` was already
   frozen.
7. **New values are mutable.** A function that returns a new value returns a mutable one, even when
   its input is frozen: `Array.map`, `filter`, `slice`, `sort`, `Map.merge`, `Set.union`, array and
   record spreads, and so on. Their elements are whatever the input held, so frozen elements stay
   frozen. This keeps one rule: only `freeze` makes a frozen value.

```ant
var p = freeze(Point{ x: 1.0, y: 2.0 })
var q = Point{ ...p, x: 3.0 }             // a new, mutable Point; p is untouched
q.x = 4.0                                 // fine
p.x = 4.0                                 // compile error; with ant run, panics: assignment to a frozen record
```

### Concurrency

No goroutine can write a frozen value, so no access to it can be a data race. After a goroutine
obtains a reference through any ordering of section 9, whether a channel, a lock, an `Atomic`, a
`go` expression or a `WaitGroup`, it can read the value forever without locks. The reference itself
still lives in an ordinary variable, field or element, and section 9's rules apply to that slot
unchanged. This is Go's guarantee and no stronger: a frozen value published through a racy write is
still a racy program.

That gives Ant the copy-on-write snapshot pattern, Clojure's atom, without any new synchronization
object:

```ant
type Config = { port: int, routes: Map[string, string], admins: Set[string] }

var current: Atomic[Frozen[Config]] = Atomic.new(freeze(load()))

func route(path: string): string | null {
  var config = Atomic.load(current)       // one load; no lock, no copy
  config.routes[path]
}

func reload(): void {
  Atomic.store(current, freeze(load()))   // readers keep the snapshot they loaded
}
```

### The checker

`Frozen[T]` is the type of a frozen value whose mutable form has type `T`. The checker computes it
structurally:

- `Frozen[T]` is `T` when `T` is immutable already: a primitive, `string`, `Html`, a named primitive
  type, a time type, `Regex.Pattern`, a function type, a sync, random or resource type, `null`,
  `void` or `never`.
- `Frozen[A | B]` is `Frozen[A] | Frozen[B]`, and `Frozen[Frozen[T]]` is `Frozen[T]`.
- Reading an element of `Frozen[Array[T]]` gives `Frozen[T]`, and likewise a key or value of
  `Frozen[Map[K, V]]`, a member of `Frozen[Set[T]]`, and a field of `Frozen[R]` for a record type
  `R`. `Frozen[Bytes]` holds bytes, which have nothing inside to freeze; its writes panic. `is`
  narrows inside: `x is Circle` on `Frozen[Shape]` gives `Frozen[Circle]`.
- For a type parameter, `Frozen[T]` stays as written until `T` is known.

Assignability (section 4) gains two rules:

- **A frozen value is a value of its type.** `Frozen[T]` is assignable to `T`, so every function
  that takes an `Array[T]` takes a frozen one, and inference binds `T` to the frozen element type:
  `Array.map(frozenPoints, f)` calls `f` with `Frozen[Point]`. No library signature changes.
- **Frozen containers are covariant.** Section 4 makes type arguments invariant "because contents
  are mutable". Frozen contents are not, so `Frozen[Array[Circle]]` is assignable to
  `Frozen[Array[Shape]]`, and likewise for `Map` values, `Set` members and `Frozen` record fields.

The checker rejects a write whose target has a `Frozen` static type: a field or element assignment,
or a call of one of the nine mutating functions with a `Frozen` first argument. The checker knows
those nine by name, as it knows `len`. A value that has a plain type but is frozen at run time,
because it was passed to a `func f(xs: Array[int])` that pushes, panics at the write. That is the
trade-off this proposal makes: checked code can pass a frozen value to a mutating function and panic
at run time. Rejecting it would need `Frozen[T]` not assignable to `T` (see Alternatives). It is
listed in section 12's last column, beside an index out of range, and it is never memory corruption.
The run-time guard is also what keeps covariance sound: a frozen array of circles seen as
`Frozen[Array[Shape]]` and then as a plain `Array[Shape]` panics when a square is pushed, instead of
storing it.

```ant
var circles = freeze([Circle{ radius: 1.0 }])
var shapes: Frozen[Array[Shape]] = circles       // ok: frozen arrays are covariant
Array.push(circles, Circle{ radius: 2.0 })       // compile error: Array.push mutates its argument

func grow(xs: Array[Circle]): void { Array.push(xs, Circle{ radius: 2.0 }) }
grow(circles)                                    // checks; panics: Array.push on a frozen array

var mutable: Array[Shape] = shapes               // checks: Frozen[T] is assignable to T
Array.push(mutable, Square{ side: 1.0 })         // checks; panics instead of storing a Square
```

The checker could close that gap by inferring which parameters a function writes. That inference
would have to follow every store into a field, closure or channel, and the result would show up in
every signature, which is the `const` contagion this proposal avoids (see Alternatives).

### Idioms

```ant
// A constant a library exports: importers can neither rebind it (section 10) nor change it.
export var Weekdays = freeze(["mon", "tue", "wed", "thu", "fri"])

// A composite key that cannot go stale.
var visited: Set[Frozen[Cell]] = Set.of([])
Set.add(visited, freeze(cell))

// A shared default that is safe to share.
var NoTags: Frozen[Array[string]] = freeze([])
type Post = { title: string, tags: Frozen[Array[string]] = NoTags }

// An update: copy what changes, share what doesn't.
var state = freeze(State{ count: 0, todos: [] })
state = freeze(State{ ...state, count: state.count + 1 })          // copies one record; todos is shared
state = freeze(State{ ...state, todos: [...state.todos, todo] })   // copies the array's references
```

The last line is O(n) in the number of todos. For state that changes one entry at a time in a large
collection, that is the cost the Later section addresses.

## Spec changes

- **Section 4, Records.** Replace "there is no `readonly` or frozen form" with a pointer to the new
  subsection. Field defaults: point to `Frozen` for shared defaults. Arrays, maps, and sets:
  recommend frozen keys and members, and leave mutating a non-frozen key a program error.
- **Section 4, new subsection "Frozen values".** `freeze`, `Frozen[T]`, rules 1–7, the checker rules
  and the gap.
- **Section 4, Assignability.** The two rules above, and the invariance paragraph now says "mutable
  contents".
- **Section 9, Data races and Visibility.** Frozen values are never written, so access to them
  cannot race. Publication of the reference is unchanged.
- **Section 11, Prelude.** `freeze`. The `mutates` note names the frozen panic.
- **Section 12, Values.** Array, map, set and record values carry a frozen flag, set only by
  `freeze`. Runtime panics: a write to a frozen value, in the last column.
- **stdlib.md.** No changes. `Atomic`, channels and `Once.value` take frozen values like any other.

## Implementation

- **Interpreter.** Add a `frozen` flag to `Array`, `Map`, `Set`, `Record` and `Bytes`. Set it before
  the value escapes `freeze`, and never clear it. Check it in the store paths of those five kinds
  (rule 5); reads and variable assignment are untouched. `freeze` walks the graph with a memo table
  that keeps sharing and cycles. A frozen map or set could skip its read lock, and a frozen key
  could cache its hash. Both are optional and need the benchmarks below to justify them. This adds
  no new value kind, so `==`, hashing and rendering need no changes.
- **Checker.** Add the `Frozen` type node with its normalization, the two assignability rules, the
  write check, and inference through `Frozen[Array[T]]` against `Array[T]`. Hover and `ant doc`
  print `Frozen[Point]`.
- **JavaScript backend.** Use the same flag on the runtime's array, map and record objects. Nothing
  depends on `Object.freeze`.
- **Tests.** Every rule above, the nine mutating functions, compound and destructuring assignment, a
  cyclic graph, freezing shared subgraphs once, and a race test with a frozen map read by many
  goroutines while another reloads the `Atomic`. Covariance through mutable aliases: for arrays, map
  values, set members and record fields, a frozen container widened covariantly, assigned to its
  plain type and written must panic and leave the value unchanged, both checked and under `ant run`.
  A checked call that passes a frozen value to a mutating function must check and then panic.
- **Benchmarks.** The guard adds a branch to every container write, and its cost is unknown until
  measured. Before acceptance, compare the interpreter with and without the flag on mutation-heavy
  workloads: an array filled by `push` and rewritten by index, compound assignment in a loop, a map
  built and drained with `Map.set` and `Map.remove`, a visited `Set` in a graph search, and a record
  field updated in a hot loop. Add a read-heavy loop as a control and `freeze` on large, shared and
  cyclic graphs. Run the same set on the JavaScript backend and record the results here. If the
  write cost shows, revisit where the flag lives before the spec change.

## Non-goals

- No `const`, no `readonly` fields, and no read-only views. Section 3 keeps "There is no `++`, `--`,
  or `const`."
- No change to reference semantics or to what assignment copies.
- No compile-time proof of race freedom and no ownership.
- No persistent collections in this proposal.

## Alternatives considered

- **`readonly T` in the checker only** (TypeScript, C++). It costs nothing at run time, but that is
  also what it guarantees. Every library signature would need to choose between `readonly` and
  mutable, which is the contagion users of TypeScript and C++ complain about. It fixes none of
  problems 1, 2 and 5, because a reader cannot know whether another alias writes.
- **`Frozen[T]` not assignable to `T`.** This closes the checker gap, but every function that only
  reads would then have to say so, through a third "readable" type or by writing
  `Array[T] | Frozen[Array[T]]` everywhere. That is the same contagion, moved around. Java and
  Python live with frozen collections behind the mutable interface, and Ant's checker still catches
  the direct writes that Java misses.
- **Freeze in place.** It is cheaper by one copy, but `var a = [1]; var b = a; freeze(a)` would
  change what `b` can do, and freezing a value that another goroutine is writing would be a new kind
  of race. A copy is what Python's `frozenset(s)` and .NET's `ToFrozenSet()` do.
- **Shallow freeze.** A frozen config whose `routes` map is still mutable is no snapshot at all, and
  shallow freezing gives no sharing guarantee or stable keys.
- **Separate immutable types** (`Vector`, `PMap` and so on). Splitting `Bytes` from `Array[byte]`
  was worth a second type because of a 24× memory difference, and its mutability is that of arrays.
  Here the data is the same, and a second API would double section 11.

## Later: persistent updates

If programs come to need O(log n) updates on large frozen collections, a later proposal can add
`Array.with(xs, i, v)`, `Map.with(m, k, v)`, `Map.without(m, k)`, `Set.with` and `Set.without`. Each
takes a frozen collection and returns a frozen one, and an implementation could back them with a
persistent vector and a HAMT without changing anything this proposal specifies. That is a
substantial addition to a tree walker that is meant to stay simple, so it waits for a program that
needs it.

## Open questions

1. **Freeze keys on insert?** `Map.set` and `Set.add` could store `freeze(k)` for an array, map, set
   or record key. That would remove the "mutating a key is a program error" rule from section 4
   altogether. The costs are a copy per insert of a mutable composite key and keys that are no
   longer `same` as the inserted value. Python takes a similar line by rejecting mutable keys.
2. **Frozen-by-declaration types.** Something like
   `type Money = frozen { amount: int, currency: string }`, whose literals are always frozen, as
   Python's `@dataclass(frozen=True)` is. This would be useful for value objects and keys, but it is
   a new form in the type grammar.
3. **Functions and sync objects inside frozen values.** Rule 2 keeps them by reference. The stricter
   option is to panic, as Swift's `Sendable` rejects non-sendable captures. The current rule follows
   Go and promises nothing about what a closure captures.
4. **A query.** Should there be `isFrozen(x)`? This draft has none, because `freeze` is idempotent
   and free on a frozen value, so code that needs a frozen value calls it.
5. **The name.** `freeze` matches Ruby, JavaScript and .NET's `ToFrozen*`, but those freeze in place
   (except .NET). `snapshot` describes a copy better but reads oddly on a literal.
