# Proposal sketch: a UI framework on `Html`

2026-10-27 · Anton

Status: sketch, for later discussion. It records the research done while designing `html.md` and the
decisions a full proposal must make. Nothing here is designed yet.

## Goal

Once Ant compiles to JavaScript, the same `Html` values that a server renders to a string should
also:

1. render to DOM nodes in the browser, with event handlers and state;
2. take over server-rendered markup without rebuilding it (hydration);
3. work as React Server Components do: components that run only on the server, read data directly,
   and send their output to the browser, with client components inside it left as references that
   the browser runs.

`html.md` keeps the way open for each of these (its section Designed for later). This sketch is what
comes after it.

## Prior art

| System                  | Model                                                                                          | Notes for Ant                                                                                       |
| ----------------------- | ---------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| React                   | Components rerun on every state change; hooks keyed by call order; virtual DOM diff            | Needs component elements, not calls, so hooks know their component; the hook rules are hidden rules |
| React Server Components | `'use client'` marks a module boundary; server output streams as a tree with client references | Props across the boundary must serialize; server functions (`'use server'`) are RPC endpoints       |
| Solid, Leptos           | Components run once; signals update DOM nodes directly; no virtual DOM                         | Components stay plain functions; dynamic parts are closures                                         |
| Svelte 5                | Compiler; runes (`$state`) are signals                                                         | Same model as Solid, with compiler syntax Ant would avoid                                           |
| Qwik                    | Resumes the server's state instead of hydrating                                                | Avoids re-running components on load; heavy serialization of closures                               |
| Astro, Fresh            | Islands: static HTML with interactive components hydrated alone                                | Close to server components with client references, without a stream                                 |
| Phoenix LiveView, htmx  | State on the server; the browser receives HTML over a socket or per request                    | Needs no JavaScript backend at all; `html.md` already serves htmx                                   |

## The decision: how the client updates

**Fine-grained signals**, as Solid and Leptos. A component is a function that runs once and builds
nodes. State is a signal, `Signal.new(0)`, read with `Signal.get` and written with `Signal.set`, in
the style of `Atomic`. A function value in content or an attribute is a region the renderer calls
again when a signal it read changes, so the laziness is a lambda the reader can see:

```ant
func counter(start: int): Html {
  var count = Signal.new(start)
  <button onclick={{ Signal.set(count, Signal.get(count) + 1) }}>
    Clicked {{ Signal.get(count) }} times
  </button>
}
```

The inner braces are an ordinary Ant lambda inside an interpolation. On the server the region is
called once and rendered as text; in the browser it is called again on each change.

**Rerendering**, as React. A component reruns on every change and the renderer diffs the result. It
is the model most people know, but it needs a component element that defers the call
(`<counter start={1} />` or a library function), state that knows which component it belongs to by
call order, and a diff. That conflicts with components as ordinary calls and with Ant's dislike of
hidden rules.

The sketch leans to signals. Server components work with either.

## Server and client

- **Marking a client module.** React marks the module, and everything it imports ships with it. Ant
  would need a declaration at the top of a module, since it has no directives; or the mark goes on
  the import. Open.
- **Client components in server code.** The server build binds an imported client component to a
  function that returns a reference node, the fourth node kind `html.md` leaves room for, holding
  the module, the name, and the arguments. Rendering it to a string on the server renders the
  component there, as React's server rendering does. To the caller it is a call that returns `Html`
  either way.
- **What crosses.** Every Ant value has a kind and a tag and there are no classes, so the rule is
  simpler than React's: primitives, `Bytes`, arrays, maps, sets, records, named primitives, time
  values, `Html`, and server-function references cross; functions, sync objects, resources, and
  `Random` do not. A record's tag needs an identity both sides agree on, such as the module's path
  and the type's name, with a position for a type declared in a block.
- **Server functions.** An exported function of a server module, called from client code, becomes a
  request. Every argument and result must cross, and the call can fail on the network, so it should
  return a `Result`. It is a public endpoint and must check its caller.
- **Async and streaming.** Ant has no `async`: a server component that reads a database blocks, as
  any function does. Streaming is a boundary that renders its subtree on a goroutine and sends a
  fallback first. Whether a client component needs a value that is not ready yet, React's promise
  prop, is open.

## Open questions

- [ ] Signals or rerendering.
- [ ] How a module is marked client or server.
- [ ] The wire format: React's Flight rows, or a format of Ant's own.
- [ ] Context, the value a subtree reads without passing it down, on the server and the client.
- [ ] Structure the browser repairs. Hydration walks the DOM the browser built from the server's
      markup, and that DOM differs from the `Html` tree where the parser repairs it: a `<tbody>`
      inserted into a table, a `<p>` closed before a block element, nested `<a>` or `<form>` split.
      Either the checker rejects those few cases (an open question in `html.md`) or the hydrator
      tolerates them.
- [ ] Hydration markers in the string renderer's output, or a format that needs none.
- [ ] Whether the DOM renderer is written in Ant, which needs `Html` to be readable by a library, or
      is part of the JavaScript backend.
