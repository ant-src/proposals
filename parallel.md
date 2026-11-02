# Proposal: the Parallel namespace

2026-10-25 · Anton

## Summary

Add a `Parallel` namespace with five functions: `map`, `each`, `run`, `race` and `any`. Each call
starts goroutines under a scope context, returns only after all of them and their deferred blocks
have ended, and raises a callback's panic in the caller. `map` and `each` take a required `limit`.
There is no new sync object, no `Group`, no streaming result, and no new syntax. `go`, `WaitGroup`
and channels stay as they are.

## Motivation

The spec's own fan-out example, `fetchAll` in section 9, is 16 lines and still has three defects
that real programs copy:

1. **Unbounded.** It starts one goroutine per URL. With 10,000 URLs it opens 10,000 connections at
   once. Adding a limit needs a semaphore channel and roughly 10 more lines.
2. **Lost panics.** A panic in a goroutine is "reported, not returned" (section 9, Panics in
   goroutines). In `fetchAll` the slot keeps its `"pending"` placeholder, and the caller gets an
   array that looks valid.
3. **No sibling cancellation.** When one part panics, nothing tells the others to stop. This
   proposal fixes that for panics, and for the first result of `race` and `any`; stopping siblings
   on an ordinary `Err` stays the program's choice, unlike Go's errgroup.

The placeholder `Result.err(Error{ message: "pending" })` exists only because the language offers no
way to collect results in order. The worker-pool recipe the spec points to ("a channel of jobs… a
channel of results, and a `WaitGroup`") is about 25 lines and easy to get wrong.

## Prior art

Many languages now offer **structured concurrency**: children live inside a scope, the scope waits
for all of them, and a failure cancels the siblings and surfaces at the scope's end. The idea comes
from Trio's nurseries and Nathaniel J. Smith's 2018 essay "Notes on structured concurrency, or: Go
statement considered harmful". A bounded parallel map is a common thing to build on top of it.

| Approach                 | Examples                                                                                                                                                                                                                                                        | What Ant takes                                                                        |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| Structured scope         | Go `errgroup.Group`, `sourcegraph/conc`; Python `asyncio.TaskGroup`, Trio nurseries; Kotlin `coroutineScope`; Swift `TaskGroup`; Java `StructuredTaskScope`; Rust `thread::scope`, tokio `JoinSet`                                                              | Every `Parallel` call is a scope: join, shared context, panic propagation (from conc) |
| Concurrency limit        | Go `errgroup.SetLimit`, `conc` `WithMaxGoroutines`; Kotlin `Semaphore`; Trio/AnyIO `CapacityLimiter`                                                                                                                                                            | A required `limit` on `map` and `each`                                                |
| Bounded parallel map     | Elixir `Task.async_stream(max_concurrency:, ordered:)`; C# `Parallel.ForEachAsync(MaxDegreeOfParallelism)`; Rust `buffered(n)` (ordered), `buffer_unordered(n)`; Kotlin `flatMapMerge(concurrency)`; Python `ThreadPoolExecutor.map`; Haskell `mapConcurrently` | `Parallel.map`, ordered                                                               |
| Run several, race        | Haskell `concurrently`, `race`; JS `Promise.all`, `Promise.race`, `Promise.any`; OCaml Eio `Fiber.all`, `Fiber.first`                                                                                                                                           | `Parallel.run`, `Parallel.race`, `Parallel.any`                                       |
| Futures / promises       | JS `Promise`; Java `CompletableFuture`; C# `Task`; Kotlin `Deferred`                                                                                                                                                                                            | No handles. A forgotten handle leaks work and silently drops errors                   |
| `async` / `await`        | JS, Python, C#, Rust, Swift, Kotlin                                                                                                                                                                                                                             | Nothing. Function coloring; the spec rejects it in section 9                          |
| Worker pools / executors | Java `ExecutorService`, Go jobs channel                                                                                                                                                                                                                         | `Parallel.each` over a channel                                                        |
| Actors + supervisors     | Erlang/Elixir, Akka, Swift `actor`                                                                                                                                                                                                                              | Nothing now. Mailboxes are channels already                                           |
| Data parallelism         | Rust `rayon`, Java parallel streams, OpenMP                                                                                                                                                                                                                     | Nothing now. CPU-bound splitting needs its own design; this proposal targets waiting  |
| Transactional memory     | Haskell STM, Clojure refs                                                                                                                                                                                                                                       | Nothing. High cost for the runtime and the JS backend                                 |

Three lessons shape the proposal:

- **Scope over handles.** Python added `TaskGroup` in 3.11 and Java added `StructuredTaskScope`
  after years of futures; Kotlin built its coroutines around scopes from the start.
- **Panics must cross the join.** `sourcegraph/conc` exists largely because `errgroup` lets a panic
  in a goroutine crash the process. Ant's `Once` already re-raises a panic in every waiter, so this
  has precedent in the spec.
- **The limit is the feature.** Unbounded fan-out is the bug that bounded-map APIs exist to prevent.
  Elixir and C# make the limit a first-class argument.

## The `Parallel` namespace

`Parallel` is a namespace of library functions, like `Math` or `Iter`. It adds no value kind and no
syntax.

```ant
Parallel.map[T, U](xs: Array[T], f: func(T, Context): U, limit: int, ctx: Context | null = null): Array[U] | null
    // f on every element, at most limit at once, results in input order; null when some element was never admitted
Parallel.each[T, R](xs: Array[T] | Channel[T], f: func(T, Context): R, limit: int, ctx: Context | null = null): void
    // f on every element, or on every value received until the channel closes; at most limit at once; f's value is discarded
Parallel.run(fs: Array[func(Context): void], ctx: Context | null = null): void
    // every f at once
Parallel.race[T](fs: Array[func(Context): T], ctx: Context | null = null): T | null
    // the first value returned wins, Err included; null when cancelled first; panics if fs is empty
Parallel.any[T, E](fs: Array[func(Context): Result[T, E]], ctx: Context | null = null): Result[T, Array[E]] | null
    // the first Ok wins; Err of every error, in input order, when all fail; null when cancelled first
    // map and each panic if limit < 1
```

Every callback receives the scope context. A lambda has one parameter, so `map` and `each` take an
explicit two-parameter `func`. A callback that captured the caller's `ctx` instead could not see a
sibling's panic or a `race` winner, and a blocked one would hold the join forever.

`each` follows the stdlib rule that a callback whose result the library ignores is typed with a type
parameter. `run` is a deliberate exception and keeps `void`: its callbacks are a list, so one type
parameter would make every task return the same type, which the record-field idiom below does not
need. A `run` callback that ends in a value discards it with `_ = …`.

### Rules

1. **Scope.** Each call makes a scope context, a child of `ctx`. It is cancelled on every exit,
   success included, and never cancels `ctx`. The call neither returns nor panics until every
   callback it started and their deferred blocks have ended. A `go` inside a callback is not owned
   and may outlive the call. Cancellation is cooperative: the call asks callbacks to stop and cannot
   force them, so a deadline does not bound the return of a callback that ignores its context.
2. **Admission.** `map` and `each` admit array elements in index order, or channel values in receive
   order, at most `limit` at once. The order callbacks run or finish in is unspecified; `map`
   results keep input order. Every admitted callback runs exactly once and holds its slot until its
   deferred blocks have run. `run`, `race` and `any` admit the whole list together, or none of it
   when `ctx` is already done. Admission stops when the scope context ends.
3. **Bounds.** At most `limit` callbacks are live at once: the call acquires a slot before it starts
   a goroutine, or reuses `limit` workers. The limit is per call, so nested calls multiply.
4. **Channels.** `each` receives only when a slot is free, under the scope context. A value it
   receives is admitted, so cancellation cannot drop it. `each` neither closes nor drains the
   caller's channel; unreceived values stay there. It tells a `null` value from a close, so
   `Channel[T | null]` works. A producer blocked in `Channel.send` is not woken by `each` ending; it
   needs its own context or a reader, as with any channel.
5. **Outcome.** `map` returns `null` only when some element was never admitted; otherwise it returns
   the complete array, even when `ctx` ended after the last admission. `race` takes the first value
   returned, `Err` included. `any` takes the first `Ok`; an `Err` cancels nothing, and when every
   callback fails it returns `Err` of the errors in input order. Both return `null` when the scope
   is cancelled before a winner. Once chosen, the outcome is final, except that a panic overrides
   it. `each` and `run` report no status. A done `ctx` after the call says neither which elements
   ran nor whether their work finished; a program that needs to know records it per element, or uses
   `map`.
6. **Panics win.** If a callback lets a panic escape, the scope is cancelled, and after the join the
   call panics whatever rule 5 chose, including when a `race` or `any` loser panics after the
   winner. The panic is raised again with its original message and trace, as `Panic.raise` does, and
   reported once, where it finally ends a goroutine, not at every `Parallel` boundary. Later panics
   in the same call are reported and dropped.
7. **Visibility.** Every action of a callback, including its deferred blocks, happens before the
   call returns or propagates a panic, so a value a callback wrote is visible after the call without
   a lock. Siblings that touch the same location during the call still need synchronization. `xs` is
   read once, at the call, like every array a library function takes (section 11).
8. **Errors are values.** `map`, `each`, `run` and `race` never look inside a callback's value.
   `any` looks only at `Ok` versus `Err`; a callback that counts a 503 response as failure returns
   `Err` for it. Fail-fast on `Err` elsewhere is the program's choice, through `Context.cancel` on a
   context it owns.
9. **Empty input.** A `limit` below 1 panics even when `xs` is empty. Otherwise empty input never
   consults `ctx`: `map` returns `[]`, `each` and `run` return at once, `any` returns `Err([])`, and
   `race([])` panics, since it has no value to return.

`limit` is required. A caller that does want one goroutine per element of an array passes its
length, which keeps that choice visible. `run`, `race` and `any` take no limit; their lists are
fixed in the source and usually short, but that is a convention, not a check.

## The five functions

### `Parallel.map`: a known collection, results in order

`result[i]` is `f(xs[i], scope)`, whatever order the callbacks finish in.

```ant
var responses = Parallel.map(urls, func(url: string, work: Context) { Http.get(url, ctx: work) }, limit: 20, ctx: ctx)
  ?? return Result.err(Error{ message: "cancelled" })
var pages = try (responses | Result.all())
```

### `Parallel.each`: a stream of work, no results

`each` takes an array or a channel, not a `Seq`. A `Seq` pushes values and can block between them,
and nothing outside it can wake that wait. With a channel, `each` does the receiving itself under
the scope context, so a panic or a cancelled `ctx` wakes it at once. A program with a blocking `Seq`
feeds a channel from it. Over a channel, `each` is the worker pool that section 9 describes in
prose: `limit` workers drain the jobs until the channel closes.

The checker solves `T` from whichever member of `Array[T] | Channel[T]` the argument matches: an
`Array[Job]` pairs with `Array[T]` and a `Channel[Job]` with `Channel[T]`, so `T` is `Job` either
way. The checker already solves union parameters this way; the spec only has to say so (Spec
changes, §4 Inference).

```ant
Parallel.each(jobs, func(job: Job, work: Context) { process(job, work) }, limit: 8, ctx: ctx)
```

### `Parallel.run`: a fixed set of different tasks

Different result types come back through a record: each callback writes its own field, and rule 7
makes the writes visible after the call.

```ant
type Loaded = { user: User | null, orders: Array[Order] }

var loaded = Loaded{ user: null, orders: [] }
Parallel.run([
  { loaded.user = loadUser(id, x) },
  { loaded.orders = loadOrders(id, x) },
], ctx: ctx)
```

### `Parallel.race`: the first to finish wins

`race` suits tasks where any ending, success or failure, settles the matter. A session that pumps a
connection both ways ends when either direction ends:

```ant
var ended = Parallel.race([
  { readLoop(conn, x) },
  { writeLoop(conn, outbox, x) },
], ctx: ctx)
```

The winner's `Err` is the outcome, and the other loop sees its context end. When `T` includes
`null`, a `null` winner looks like cancellation, as with `Channel.receive`; wrap the value in a
record such as `type Found = { user: User | null }`.

### `Parallel.any`: the first success wins

`any` suits redundant tasks, where one failure should not end the others. A primary and a mirror:

```ant
var reply = Parallel.any([
  { Http.get(primary, ctx: x) },
  { Http.get(mirror, ctx: x) },
], ctx: ctx) ?? return Result.err(Error{ message: "cancelled" })
```

With `race`, a fast connection error from the primary would win over a good mirror response. `any`
waits for an `Ok`, and `reply` is `Err` of both errors only when both fail.

## Edge cases

| Case                                                     | Result                                                                                          |
| -------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| `map([], f, limit: 0)`                                   | Panics: the limit is checked before the input                                                   |
| `ctx` already done, non-empty input                      | No callback runs; `map`, `race` and `any` return `null`, `each` and `run` return                |
| `ctx` ends while `map` still has elements to admit       | Admitted callbacks see their context end; after the join, `null`                                |
| `ctx` ends after `map` admitted its last element         | The complete array; callbacks may hold `Err` values with the context's reason                   |
| Later elements finish first                              | `map` results and `any` errors stay in input order                                              |
| `limit: 1` and a callback with `defer`                   | The deferred block finishes before the next element starts                                      |
| `each` over `Channel[string \| null]`                    | `f` is called with each `null` value; only a close ends the stream                              |
| `ctx` ends just after `each` received a value            | That value's callback still runs, with a done context                                           |
| A `race` or `any` loser panics after the winner returned | The call panics with the loser's panic                                                          |
| Every `any` callback returns `Err`                       | `Err` of all errors, in input order; `null` only if the scope was cancelled before the last one |

## Do we need `Group`?

Not in this proposal. `Parallel` covers work whose shape is known when the call starts. The case
left over is work that adds more work to the same scope while it runs.

| Use case                               | Today                                           | With `Parallel`                         |
| -------------------------------------- | ----------------------------------------------- | --------------------------------------- |
| Fetch N URLs, keep results             | `go` + `Mutex` + `WaitGroup` + placeholder      | `Parallel.map`                          |
| Process N files, no results            | `go` + `WaitGroup`                              | `Parallel.each(files, …)`               |
| Worker pool over a job channel         | jobs channel + N workers + `WaitGroup`          | `Parallel.each(jobs, …)`                |
| A few different tasks at once          | one `go` each + `WaitGroup`                     | `Parallel.run`                          |
| Reader and writer, stop when one ends  | two `go` + `select` + `Context.cancel`          | `Parallel.race`                         |
| Primary and mirror, first success wins | channel + counter + `select` + `Context.cancel` | `Parallel.any`                          |
| Crawler that finds new URLs as it goes | `go` + `WaitGroup`                              | unchanged: needs `Group` or `WaitGroup` |

A crawler or a recursive directory walk keeps using `go` and `WaitGroup`. If such programs turn out
to be common, `Group` can be proposed later on top of the same machinery. Streaming results, a `Seq`
out of `map`, wait until source cancellation and scope lifetime for them are settled.

## Spec changes

All changes are additions. No existing signature changes, and `select` is untouched.

| Where                                  | Change                                                                                                                                                                                                                           |
| -------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `spec/ant-lang.md` §4, Inference       | State how an argument solves a union parameter: each member of the argument's type pairs with the parameter member that shares its discriminator, and a bare type parameter member takes the rest. The checker already does this |
| `spec/ant-lang.md` §9, Synchronization | Rewrite `fetchAll` with `Parallel.map` (below). Keep the `WaitGroup` version under Sharing results as the explanation of what happens underneath                                                                                 |
| §9, Waiting                            | Replace the worker-pool and future sentences: a worker pool is `Parallel.each` over a channel; first-wins waits are `Parallel.race` and `any`                                                                                    |
| §9, Visibility                         | Add an ordering: every action of a `Parallel` callback, including its deferred blocks, happens before the call returns or propagates a panic                                                                                     |
| §9, Sharing results                    | Note that a callback may write a record field or array slot that the caller reads after the call, with no lock                                                                                                                   |
| §9, Panics in goroutines               | Add one paragraph: a panic in a `Parallel` callback is raised again in the caller with its original trace and reported once                                                                                                      |
| §11, intro                             | Name `Parallel.run` as the exception to the rule that ignored callback results are typed with a type parameter                                                                                                                   |
| §11, new subsection Parallel           | The five signatures, the rules, and the edge cases                                                                                                                                                                               |
| §12, Goroutines                        | State the scope rule; a `go` inside a callback is not owned by the call                                                                                                                                                          |
| `ant/internal/prelude/prelude.d.ant`   | Doc comments for the five functions, generated from the spec text                                                                                                                                                                |

The `fetchAll` example becomes:

```ant
func fetchAll(urls: Array[string], ctx: Context): Array[Result[Http.Response, Error]] | null {
  Parallel.map(urls, func(url: string, work: Context) { Http.get(url, ctx: work) }, limit: 20, ctx: ctx)
}
```

In the Go interpreter each call is a `sync.WaitGroup`, a buffered channel as a semaphore acquired
before each goroutine starts, a child context cancelled by a deferred call, and the first panic
object under a mutex, re-raised with its trace as `runOnce` does in `ant/internal/lib/sync.go`.
`each` over a channel receives with an internal closed flag rather than through `Channel.receive`'s
nullable result. The JS backend can map it onto the scheduler that runs `go` blocks.

## Non-goals

- **Futures, promises, `async`/`await`.** They bring back handles that can leak and the function
  coloring that section 9 rejects.
- **Changing `go`.** It stays unstructured, as in Go, for fire-and-forget work, long-lived loops and
  self-spawning work.
- **`Group` and streaming results, for now.** See above.
- **Catching panics itself.** `Parallel` raises a callback's panic again in the caller; a caller
  that must survive it wraps the call in `Panic.catch` (section 8, Catching a panic), which sees the
  original message and trace.
- **Actors, supervisors, data parallelism, STM.** Out of scope for an interpreter that is meant to
  stay simple.

## Decisions

| Question                                  | Decision                                                                                     |
| ----------------------------------------- | -------------------------------------------------------------------------------------------- |
| `Parallel` or `Concurrent`?               | `Parallel`: familiar, and clear enough                                                       |
| Arrays in `each`?                         | Yes, as `Array[T] \| Channel[T]`; `Seq` is dropped because a blocking source cannot be woken |
| Inference over `Array[T] \| Channel[T]`?  | Already supported: union members pair by discriminator; §4 gains the rule in words           |
| Runtime panics table?                     | Unchanged: its library-function row covers `limit < 1` and `race([])`                        |
| Scope context for `map`/`each` callbacks? | Yes, `func(T, Context)`; a captured parent context misses sibling panics and can block       |
| `each` callback result?                   | A type parameter `R`, discarded, per the stdlib convention                                   |
| `run` callback result?                    | `void`, a documented exception: one type parameter would force one result type on the list   |
| When does `map` return `null`?            | Only when an element was never admitted                                                      |
| First-success `Parallel.any`?             | Included; `race` keeps first-returned semantics, `Err` included                              |
| Nullable `map` result?                    | Kept, for the rule that every blocking call takes a context                                  |
| Streaming results (`Seq` out)?            | Deferred until source cancellation and scope lifetime are settled                            |
| `Group`?                                  | Deferred until self-spawning work proves common                                              |
| Typed two-task `run`?                     | Deferred; record fields suffice with the visibility rule                                     |
