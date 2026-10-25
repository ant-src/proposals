# Proposal: the Parallel namespace

2026-10-25 · Anton

## Summary

Add a `Parallel` namespace with four functions: `map`, `each`, `run` and `race`. Each call starts goroutines, returns only after all of them have ended, and raises a goroutine's panic in the caller. `map` and `each` take a required `limit`. There is no new sync object, no `Group`, and no new syntax. `go`, `WaitGroup` and channels stay as they are.

## Motivation

The spec's own fan-out example, `fetchAll` in section 9, is 16 lines and still has three defects that real programs copy:

1. **Unbounded.** It starts one goroutine per URL. With 10,000 URLs it opens 10,000 connections at once. Adding a limit needs a semaphore channel and roughly 10 more lines.
2. **Lost panics.** A panic in a goroutine is "reported, not returned" (section 9, Panics in goroutines). In `fetchAll` the slot keeps its `"pending"` placeholder, and the caller gets an array that looks valid.
3. **No sibling cancellation.** When one part panics, nothing tells the others to stop. This proposal fixes that for panics only; stopping siblings on an ordinary Err stays the program's choice, unlike Go's errgroup. Each program wires its own `Context` and `Context.cancel`.

The placeholder `Result.err(Error{ message: "pending" })` exists only because the language offers no way to collect results in order. The worker-pool recipe the spec points to ("a channel of jobs… a channel of results, and a `WaitGroup`") is about 25 lines and easy to get wrong.

## Prior art

Most mainstream languages have converged on **structured concurrency**: children live inside a scope, the scope waits for all of them, and a failure cancels the siblings and surfaces at the scope's end. The idea comes from Trio's nurseries and Nathaniel J. Smith's 2018 essay "Notes on structured concurrency, or: Go statement considered harmful". A bounded parallel map is the most common thing people build on top of it.

| Approach | Examples | What Ant takes |
| --- | --- | --- |
| Structured scope | Go `errgroup.Group`, `sourcegraph/conc`; Python `asyncio.TaskGroup`, Trio nurseries; Kotlin `coroutineScope`; Swift `TaskGroup`; Java `StructuredTaskScope`; Rust `thread::scope`, tokio `JoinSet` | Every `Parallel` call is a scope: join, shared context, panic propagation (from conc) |
| Concurrency limit | Go `errgroup.SetLimit`, `conc` `WithMaxGoroutines`; Kotlin `Semaphore`; Trio/AnyIO `CapacityLimiter` | A required `limit` on `map` and `each` |
| Bounded parallel map | Elixir `Task.async_stream(max_concurrency:, ordered:)`; C# `Parallel.ForEachAsync(MaxDegreeOfParallelism)`; Rust `buffered(n)` (ordered), `buffer_unordered(n)`; Kotlin `flatMapMerge(concurrency)`; Python `ThreadPoolExecutor.map`; Haskell `mapConcurrently` | `Parallel.map`, ordered |
| Run several, race | Haskell `concurrently`, `race`; JS `Promise.all`, `Promise.race`; OCaml Eio `Fiber.all`, `Fiber.first` | `Parallel.run`, `Parallel.race` |
| Futures / promises | JS `Promise`; Java `CompletableFuture`; C# `Task`; Kotlin `Deferred` | No handles. A forgotten handle leaks work and silently drops errors |
| `async` / `await` | JS, Python, C#, Rust, Swift, Kotlin | Nothing. Function coloring; the spec rejects it in section 9 |
| Worker pools / executors | Java `ExecutorService`, Go jobs channel | `Parallel.each` over a channel |
| Actors + supervisors | Erlang/Elixir, Akka, Swift `actor` | Nothing now. Mailboxes are channels already |
| Data parallelism | Rust `rayon`, Java parallel streams, OpenMP | Nothing. A CPU-bound concern; the tree-walking interpreter gains little from it |
| Transactional memory | Haskell STM, Clojure refs | Nothing. High cost for the runtime and the JS backend |

Three lessons shape the proposal:

- **Scope over handles.** Each language that started with futures added scopes later: Python added `TaskGroup` in 3.11, Java added `StructuredTaskScope`, and Kotlin built its coroutines around scopes from the start.
- **Panics must cross the join.** `sourcegraph/conc` exists largely because `errgroup` lets a panic in a goroutine crash the process. Ant's `Once` already re-raises a panic in every waiter, so this has precedent in the spec.
- **The limit is the feature.** Unbounded fan-out is the bug that bounded-map APIs exist to prevent. Elixir and C# make the limit a first-class argument.

## The `Parallel` namespace

`Parallel` is a namespace of library functions, like `Math` or `Iter`. It adds no value kind and no syntax.

```ant
Parallel.map[T, U](xs: Array[T], f: func(T): U, limit: int, ctx: Context | null = null): Array[U] | null
                                   // f on every element, at most limit at once; null when ctx ends before every element is admitted
Parallel.each[T](xs: Array[T] | Channel[T], f: func(T): void, limit: int, ctx: Context | null = null): void
                                   // f on every element, or on every value received until the channel closes; at most limit at once
Parallel.run(fs: Array[func(Context): void], ctx: Context | null = null): void
                                   // every f at once, each given the scope's context
Parallel.race[T](fs: Array[func(Context): T], ctx: Context | null = null): T | null
                                   // every f at once; the first value returned wins; null when ctx ends before any f returns; panics if fs is empty
                                   // map and each panic if limit < 1
```

Every call follows these rules:

1. **Scope.** The call owns the callbacks it starts and neither returns nor panics until all of them have ended. A `go` inside a callback is not owned and may outlive the call.
2. **Admission.** Elements are admitted in index order, or in receive order for a channel, with at most `limit` running. The order callbacks run or finish in is unspecified.
3. **Cancellation.** Each call has a scope context, a child of `ctx`. When `ctx` ends or a callback panics, the scope context is cancelled: the call stops admitting work, `run` and `race` callbacks see their context end, and a channel receive inside `each` wakes. Then the call joins. It requests cancellation but cannot force a callback to return, so a deadline does not bound the return time of callbacks that ignore it.
4. **Outcome.** `map` returns `null` only when some element was never admitted; if every element ran, it returns the array even if `ctx` ended meanwhile, so finished work is never discarded. `race` returns the first value returned, or `null` if the scope was cancelled before any returned. `each` and `run` return `void`, and `Context.isDone(ctx)` tells the caller whether they finished.
5. **Panics win.** If any callback panics, the call panics after the join, whatever outcome rule 4 chose. That includes a `race` loser that panics during cleanup after a winner. The original panic is raised again with its message and stack trace, as `Once` does, and the runtime reports it once, where it finally ends a goroutine, not at every `Parallel` boundary. Later panics in the same call are reported and dropped.
6. **Visibility.** Every callback's actions, including its deferred blocks, happen before the call returns or propagates a panic. A value written by one callback and read after the call needs no `Atomic` or lock.
7. **Errors are values.** No function looks inside `T` or `U` or cancels on an `Err`. Fail-fast is the program's choice, through `Context.cancel` on a context it owns.
8. **Empty input.** `map` returns `[]`, `each` and `run` return at once, and `race([])` panics, since it has no value to return.

`limit` must be positive, and there is no unbounded idiom: unbounded fan-out is the bug these functions prevent. `run` and `race` take no limit; their lists are usually short, but that is a convention, not a check.

## The four functions

### `Parallel.map`: a known collection, results in order

`result[i]` is `f(xs[i])`, and elements start in index order. An empty array returns `[]` without starting a goroutine.

```ant
var responses = Parallel.map(urls, { Http.get(x, ctx: ctx) }, limit: 20, ctx: ctx) ?? return Result.err(Error{ message: "cancelled" })
var pages = try (responses | Result.all())
```

### `Parallel.each`: a stream of work, no results

`each` takes an array or a channel, not a `Seq`. A `Seq` pushes values and can block between them, and nothing outside it can wake that wait. If a worker panicked while the source waited for its next value, the panic would be stuck until the caller's context ended. With a channel, `each` does the receiving itself under the scope context, so a panic or a cancelled `ctx` wakes it at once. Values still in the channel stay there. A program with a blocking `Seq` feeds a channel from it.

Over a channel, `each` is the worker pool that section 9 now describes in prose: `limit` workers drain the jobs until the channel closes.

```ant
Parallel.each(jobs, { process(x) }, limit: 8, ctx: ctx)
Parallel.each(files, { resize(x, ctx) }, limit: 4, ctx: ctx)
```

### `Parallel.run`: a fixed set of different tasks

`run` starts each function at once and passes it the scope context. Different result types come back through a record: each callback writes its own field, and rule 6 makes the writes visible after the call, so no `Atomic` is needed.

```ant
type Loaded = { user: User | null, orders: Array[Order] }

var loaded = Loaded{ user: null, orders: [] }
Parallel.run([
  { loaded.user = loadUser(id, x) },
  { loaded.orders = loadOrders(id, x) },
], ctx: ctx)
```

### `Parallel.race`: the first answer wins

`race` returns the first value any function returns and cancels the others' context, then waits for them to end. The first value wins even when it is an `Err`; a first-success variant is an open question.

```ant
var reply = Parallel.race([
  { Http.get(primary, ctx: x) },
  { Http.get(mirror, ctx: x) },
], ctx: ctx) ?? return Result.err(Error{ message: "cancelled" })
```

`run` and `race` callbacks get the scope context as their lambda parameter `x`. `map` and `each` callbacks do not, because the parameter holds the element. A sibling's panic therefore stops admission and wakes the channel receive, but does not interrupt a `map` or `each` callback already running; it finishes or ends with the caller's `ctx`. This is a deliberate trade for `{ Http.get(x, ctx: ctx) }` over `func(url: string, work: Context) { Http.get(url, ctx: work) }`. A panic is a bug, and at most `limit` callbacks are in flight when one happens.

## Do we need `Group`?

Not in this proposal. `Parallel` covers work whose shape is known when the call starts. The case left over is work that adds more work to the same scope while it runs.

| Use case | Today | With `Parallel` |
| --- | --- | --- |
| Fetch N URLs, keep results | `go` + `Mutex` + `WaitGroup` + placeholder | `Parallel.map` |
| Process N files, no results | `go` + `WaitGroup` | `Parallel.each(files, …)` |
| Worker pool over a job channel | jobs channel + N workers + `WaitGroup` | `Parallel.each(jobs, …)` |
| A few different tasks at once | one `go` each + `WaitGroup` | `Parallel.run` |
| Primary and mirror, first wins | channel of capacity 1 + `select` + `Context.cancel` | `Parallel.race` |
| Crawler that finds new URLs as it goes | `go` + `WaitGroup` | unchanged: needs `Group` or `WaitGroup` |

A crawler or a recursive directory walk keeps using `go` and `WaitGroup`. If such programs turn out to be common, `Group` can be proposed later on top of the same machinery. Leaving it out keeps the change to library functions: no new sync kind in section 12 and no new `select` head.

## Spec changes

All changes are additions. No existing signature changes, and `select` is untouched.

| Where | Change |
| --- | --- |
| `spec/ant-lang.md` §9, Synchronization | Rewrite `fetchAll` with `Parallel.map` (below). Keep the `WaitGroup` version under Sharing results as the explanation of what happens underneath |
| §9, Waiting | Replace the worker-pool and future sentences: a worker pool is `Parallel.each` over a channel; a first-answer-wins wait is `Parallel.race` |
| §9, Visibility | Add an ordering: every action of a `Parallel` callback, including its deferred blocks, happens before the call returns or propagates a panic |
| §9, Sharing results | Note that a callback may write a record field or array slot that the caller reads after the call, with no lock |
| §9, Panics in goroutines | Add one paragraph: a panic in a `Parallel` callback is raised again in the caller with its original trace and reported once |
| §11, new subsection Parallel | The four signatures and the eight rules |
| §12, Goroutines | State the scope rule; a `go` inside a callback is not owned by the call |
| §12, Runtime panics | Add two rows: `limit < 1` in `map` or `each`, and `race` with an empty array |
| `stdlib/prelude.ant` | Doc comments for the four functions, generated from the spec text |

The `fetchAll` example becomes:

```ant
func fetchAll(urls: Array[string], ctx: Context): Array[Result[Http.Response, Error]] | null {
  Parallel.map(urls, { Http.get(x, ctx: ctx) }, limit: 20, ctx: ctx)
}
```

In the Go interpreter each call is a `sync.WaitGroup`, a buffered channel as a semaphore, a child context, and the first panic object under a mutex, re-raised with its stack trace as `runOnce` does in `ant/internal/lib/sync.go`. The JS backend can map it onto the scheduler that runs `go` blocks.

## Non-goals

- **Futures, promises, `async`/`await`.** They bring back handles that can leak and the function coloring that section 9 rejects.
- **Changing `go`.** It stays unstructured, as in Go, for fire-and-forget work, long-lived loops and self-spawning work.
- **`Group`, for now.** See above; it can follow if self-spawning work proves common.
- **Catching panics itself.** `Parallel` raises a callback's panic again in the caller; a caller that must survive it wraps the call in `Panic.catch` (section 8, Catching a panic), which sees the original message and trace.
- **Actors, supervisors, data parallelism, STM.** Out of scope for an interpreter that is meant to stay simple.

## Open questions

Decided after review:

| Question | Decision |
| --- | --- |
| `Parallel` or `Concurrent`? | `Parallel`: familiar, and clear enough |
| Arrays in `each`? | Yes, as `Array[T] \| Channel[T]`; `Seq` is dropped because a blocking source cannot be woken |
| Scope context for `map`/`each` callbacks? | No; keep one-parameter lambdas (see The four functions) |
| When does `map` return `null`? | Only when an element was never admitted |
| First-success `Parallel.any`? | Deferred; needs all-failed and cancellation rules |
| Nullable `map` result? | Kept, for the rule that every blocking call takes a context |
| Streaming results (`Seq` out)? | Deferred until source cancellation and scope lifetime are settled |
| Typed two-task `run`? | Deferred; record fields suffice with the visibility rule |

Still open:

- [ ] Is giving up mid-flight cancellation for `map` and `each` acceptable, or should their callbacks take `func(T, Context)` after all?
