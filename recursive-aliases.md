# Proposal: a recursive alias takes no type parameters

2026-11-07 · Anton

## Summary

An alias that reaches itself without passing through a named record type's fields may not declare
type parameters, and neither may any alias on its cycle. `Json.Value` is unaffected. A generic one,
such as `type Nest[T] = Array[Nest[T]] | Box[T]`, becomes a compile error; generic recursion goes
through a record's field, as `type List[T] = { value: T, next: List[T] | null }` already does.

The rule removes what reflection made the checker and the interpreter carry for recursion through
generic aliases: the substituter's guard for a union body formed inside its own alias, the self-args
rule for aliases, and the per-call cycle search in `Equal`. Comparison of recursive aliases stays
coinductive, as the spec says today. The cost is one wrapper record where a generic recursive alias
was used; no `.ant` file outside the type checker's tests has one.

## Motivation

Before `type(T)` (spec "Add reflection", 2026-11-04), `ant run` never looked at a type, and the
rules for recursive aliases lived only in the checker. Reflection made the interpreter build types
at run time from declarations nobody checked, so every declaration rule now runs in both, and each
has to be fast and crash-free on invalid input in both.

Code growth since the commit before reflection (e38defa to ae9a04f, non-test Go lines):

| Package            | Before | Now   |
| ------------------ | ------ | ----- |
| `internal/types`   | 1,113  | 2,361 |
| `internal/interp`  | 5,110  | 5,559 |
| `internal/check`   | 5,515  | 5,840 |
| `internal/resolve` | 917    | 1,074 |

The Reflect library itself (`internal/lib/reflect.go`, 520 lines) is the simple part. Much of the
growth in `internal/types` exists because a generic alias can lead back to itself:

- **A substituter that stops on its own instantiations** (`outer`, `plain` and `raw` in `subst.go`),
  since substituting a recursive generic alias's body meets the alias again. Its `seen` and `done`
  serve generic records too, whose fields it validates when it instantiates them, and stay.
- **The self-args rule for aliases** (`SelfArgs`) and the per-component searches that keep it,
  `AliasCycle` and `UnionSelf` linear (`scc.go`, the `In` variants in `selfargs.go`).
- **A cycle search per `Equal` call** (`findRecursive` in `memo.go`) to tell which generic aliases
  can recur and must be remembered.
- **A second, unvalidated way to build unions** used only inside `Equal` (ed75484), because
  validating a union compared its members, which expanded a generic alias, which built the union
  again.

The review rounds of 2026-11-06 and 2026-11-07 found about twenty bugs in this area. At least five
involve recursion through a generic alias: the run-time hangs and stack overflows of
`type B[T] = D[B[D[T]]]`, `type N[T] = Array[int] | N[Array[T]]` and `type B[T] = A[B[T] | null]`,
the checker stack overflow of `type A = B[B[B[int]]]` with `type B[T] = (func(null): A | null | T)`,
and the checker hang on a union naming its alias through a later generic alias. Each fix added code.

Nobody uses what this machinery buys. Every generic alias in the repository that names itself, a few
dozen, most of them there to be rejected, is in the type checker's own tests (`tests/check/alias_*`,
`tests/run/alias_*`, `tests/unchecked/type_expr_*`). No `.ant` file outside `tests/` in ant, spec,
the site, ant-web or proposals has one. The prelude's recursive types are `Json.Value`, which takes
no type parameters, and record cycles such as `Panic`'s.

## Prior art

| Language   | Recursive aliases                                                                                                                     |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| Go         | Rejected; recursion goes through a defined type, which is nominal                                                                     |
| Rust       | Rejected ("cycle detected when expanding type alias"); recursion goes through `struct` or `enum`                                      |
| Swift      | Rejected; recursion goes through an `indirect enum` or a class                                                                        |
| Haskell    | Rejected ("cycle in type synonym declarations"); recursion goes through `data` or `newtype`                                           |
| Elm        | Rejected ("this type alias is recursive, forming an infinite type"); the compiler suggests a custom type                              |
| TypeScript | Allowed since 3.7, generic ones included, compared structurally and coinductively, with a cache for pairs in progress and a depth cap |
| Python     | Allowed through PEP 695 `type` statements, generic ones included, evaluated lazily; type checkers compare them structurally           |

Ant needs one recursive alias, because `Json.Value` must accept `[1, "a"]` without a wrapper, so it
cannot reject them outright as Go and Elm do. It keeps TypeScript's model for plain recursive
aliases, coinductive comparison included, which TypeScript also kept rather than going nominal. What
it gives up is recursion through generic aliases, where TypeScript spends its depth caps and Ant
spent most of its recent fixes.

## Proposal

### Terms

Draw a graph over a module's type declarations, with an edge from an alias to every declaration its
body names, type arguments included, and from a generic alias to every declaration a parameter's
bound names. A record's fields add no edge. An alias is **recursive** when it lies on a cycle of this
graph.

`Json.Value` is recursive. `type Tree[T] = Leaf[T] | Node[T]` with
`type Node[T] = { kids: Array[Tree[T]] }` is not: its only path back to `Tree` runs through `Node`'s
fields, which are compared by the record's tag and type arguments and never expanded. A record's
type arguments do not break a cycle, because they are compared structurally:
`type Tree[T] = Box[Tree[T]] | Leaf`, with `type Box[T] = { value: T }`, is recursive, and the
substituter meets it exactly as it meets recursion through `Array`.

### The rule

A recursive alias may not declare type parameters, and neither may any alias on its cycle:

```ant
type Value = null | bool | float64 | string | Array[Value] | Map[string, Value]   // fine
type Nest[T] = Array[Nest[T]] | Box[T]          // compile error
type Tree[T] = Box[Tree[T]] | Leaf              // compile error: Box's argument does not break the cycle

type NestNode[T] = { items: Array[Nest[T]] }    // the same shape, through a record's field
type Nest[T] = NestNode[T] | Box[T]             // fine: the cycle passes through NestNode's fields

type Opt[T] = T | null
type Doc = Opt[Array[Doc]]                      // fine: Opt is not on the cycle, since its body never names Doc

type A = Wrap[int]
type Wrap[T] = Array[A] | Box[T]                // compile error at Wrap, legal today
```

The error is reported at the generic alias and names it, with the cycle and the record form, as
Elm's does: "type Wrap is on the cycle Wrap → A → Wrap, which passes through no record's fields, so
it cannot take type parameters; hold the recursive part in a record's field".

The existing self-args rule ("passes its own type parameters unchanged and in order") becomes a rule
about records only, since only a record's fields can now lead a generic type back to itself.

### What stays

- **Coinductive comparison.** `==` on types and assignability still count a pair under test as
  holding, so `type A = Array[Array[A]]` equals `Array[A]`, `type F = func(): F | null` is a
  `type G = func(): G | int | null`, and a function value of type `func(): func(): null` is a
  `type Handler = func(): Handler | null`. The memo of pairs and its trail stay as they are.
- **The alias-cycle and union-self rules** for plain aliases. The union-self rule is about forming a
  union: flattening `P | int` in `type P = Array[P | int] | null` needs `P`'s members, which need
  that union.
- **Generic records** and their self-args rule: `type Chain[T] = { next: Chain[Box[T]] | null }` is
  still an error.
- **The instantiation memo** for long generic alias chains (`ArgsMemo`, f01f0b8 and 9da0add): those
  chains are not recursive and remain legal.
- Union discriminators, bounds, inference, and every Reflect function.

## What gets simpler

These are estimates from reading the code. A prototype (see Validation) would replace them with
numbers.

| Part                                                      | Today                                              | After                                                    |
| --------------------------------------------------------- | -------------------------------------------------- | -------------------------------------------------------- |
| Which aliases `Equal` remembers (`findRecursive`)         | A cycle search per call                            | Aliases without arguments; a generic one cannot recur    |
| Substitution (`subst.go`)                                 | `outer`, `plain`, `raw` to stop on itself          | `outer` gone; `plain` and `raw` with the union row       |
| Union building inside `Equal` (`union.go`)                | Raw unions plus validated ones, came-back checks   | One way to build a union, if the prototype confirms it   |
| `SelfArgs` and its per-component search                   | Generic aliases and records                        | Records only                                             |
| `checkDecl` in `rtype.go`, the checker's declaration pass | The rules above for generic aliases too            | The generic-alias cases gone, one new check for the rule |

## Spec changes

- **Section 4, Type declarations.** Define a recursive alias by the graph above and add the rule to
  the paragraph on types that refer to themselves, with the `Box` and `Wrap` examples. Replace the
  generic examples of the union-self rule (`type R[T] = …`, `type L[T] = Array[L[T] | int]`) with
  plain ones. Reword the self-args paragraph to apply through records' fields;
  `type Nest[T] = Array[Nest[T]] | Box[T]` moves from fine to an error, `type List[T]` stays fine.
- **Section 12.** Add the rule to the `type(T)` panic row, so an unchecked type expression reaching
  a generic recursive alias panics.

Assignability, Types as values and stdlib.md are unchanged. Assignability's "at any depth, with no
limit on how deep types nest" stays a guarantee.

## Migration

A generic recursive alias becomes a record that holds the recursive part in a field, and an alias
over it:

```ant
// before
type Tree[T] = Array[Tree[T]] | Box[T]
var t: Tree[int] = [Box{ value: 1 }, []]

// after
type Branch[T] = { kids: Array[Tree[T]] }
type Tree[T] = Branch[T] | Box[T]
var t: Tree[int] = Branch{ kids: [Box{ value: 1 }, Branch{ kids: [] }] }
```

Values gain one record literal per level, and `is Branch` replaces `is Array`. The tempting
`type Nested[T] = T | Array[Nested[T]]` is illegal already, since a type parameter may not share a
union with an array, so the record form is what such code needs today too.

The repository needs no migration outside the type checker's tests. Those that exercise generic
recursive aliases change their expected errors, and the tests of the removed machinery go away with
it.

## Considered and rejected: comparing recursive aliases by name

An earlier draft also made two different recursive aliases never the same type, hoping to drop the
coinductive memo. It does not terminate without one. Under "a recursive alias against any other type
unfolds once", `Array[A]` against `type A = Array[Array[A]]` unfolds the right side, then the left,
and returns to the starting pair without meeting two alias names. An inductive memo, where a pair
under test counts as failing, needs the same trail as the coinductive one. Unfolding only at the top
of a comparison breaks ordinary code: `func(): func(): null` is no longer a
`type Handler = func(): Handler | null`. The rule would also buy little: arrays and maps are
invariant, so a narrower `type Doc = null | string | Array[Doc] | Map[string, Doc]` is already not a
`Json.Value` today, and comparing two different recursive aliases matters only in covariant
positions, which nothing outside the checker's tests uses.

## Non-goals

- Making records structural, or aliases nominal.
- Removing recursive aliases. `Json.Value` and similar shapes keep working without wrappers.
- Changing what `ant run` checks. It still panics where the checker would reject a type it reaches.

## Open questions

1. **Error message.** The wording above needs review, in particular how much of a long cycle it
   spells.

## Validation

Before acceptance, prototype on a branch of `ant`:

1. Implement the rule in the checker and in `rtype.go`.
2. Delete what it makes unreachable, and report the lines removed per package.
3. Compare `ant check` and `ant run` output with master over every `.ant` file in the repository and
   a few thousand generated programs. The only differences should be in tests of generic recursive
   aliases. These must keep their current results, and none may hang:
   - `type A = Array[Array[A]]` with `var a: A = xs` for `xs: Array[A]`, and
     `type(A) == type(Array[A])` at run time, which prints `true`;
   - `var h: Handler = start` with `type Handler = func(): Handler | null` and
     `start = func() { return step }`, `step = func() { return null }`;
   - `type Doc = null | string | Array[Doc] | Map[string, Doc]`, rejected where a `Json.Value` is
     expected;
   - `type Opt[T] = T | null` with `type Doc = Opt[Array[Doc]]`, which stays legal.
4. Re-run the reproducers in `private/type-bugs/repro/` and the long-chain tests, and check that no
   timing gets worse.
