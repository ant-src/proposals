# Proposal: recursive aliases are plain and named

2026-11-07 · Anton

## Summary

Two restrictions on aliases that refer to themselves, so that the checker and the interpreter can
drop most of the machinery reflection pulled in:

1. **A recursive alias takes no type parameters.** An alias that reaches itself without passing
   through a named record type, such as `Json.Value`, is fine. A generic one, such as
   `type Nest[T] = Array[Nest[T]] | Box[T]`, is a compile error; generic recursion goes through a
   record, as `type List[T] = { value: T, next: List[T] | null }` already does.
2. **A recursive alias is compared by name.** It is the same type as its own expansion and as
   itself, and as no other recursive alias. With `type F = func(): F | null` and
   `type G = func(): G | null`, an `F` is no longer a `G`.

Records, unions, generics, `Json.Value` and every program outside the type checker's own tests are
unchanged. The cost is one wrapper record where a generic recursive alias was used; the corpus has
none outside `tests/`.

## Motivation

Before `type(T)` (spec "Add reflection", 2026-11-04), `ant run` never looked at a type, and the
equi-recursive rules for aliases lived only in the checker. Reflection made the interpreter build
types at run time from declarations nobody checked, so every declaration rule now runs in both, and
each has to be fast and crash-free on invalid input in both.

Code growth since the commit before reflection (e38defa, non-test Go lines):

| Package            | Before | Now   |
| ------------------ | ------ | ----- |
| `internal/types`   | 1,113  | 2,285 |
| `internal/interp`  | 5,110  | 5,559 |
| `internal/check`   | 5,515  | 5,772 |
| `internal/resolve` | 917    | 1,067 |

The Reflect library itself (`internal/lib/reflect.go`, 520 lines) is the simple part. Most of the
growth in `internal/types` serves recursive aliases:

- **Coinductive equality and assignability.** `Equal` and `Assignable` assume a pair holds while
  testing it, and a memo with a trail that drops answers when an assumption fails (`memo.go`).
- **A cycle search per call** to find which aliases are recursive (`findRecursive`, `scc.go`).
- **The self-args rule for aliases** (`SelfArgs`) and per-component variants of it, of `AliasCycle`
  and of `UnionSelf`, to keep them linear.
- **A substituter that stops on its own instantiations** (`seen`, `done`, `plain`, `outer` in
  `subst.go`), since substituting a recursive generic alias body meets the alias again.
- **A second, unvalidated way to build unions** used only inside `Equal` (ed75484), because
  validating a union compared its members, which expanded the alias, which built the union again.

The review rounds of 2026-11-06 and 2026-11-07 found about twenty bugs in this area. At least six
involve a recursive generic alias or the structural comparison of two recursive aliases: the
run-time hangs and stack overflows of `type B[T] = D[B[D[T]]]`,
`type N[T] = Array[int] | N[Array[T]]` and `type B[T] = A[B[T] | null]`, the exponential check of a
ring of function aliases, the checker stack overflow of `type A = B[B[B[int]]]` with
`type B[T] = (func(null): A | null | T)`, and the checker hang on a union naming its alias through a
later generic alias. Each fix added code.

Nobody uses what this machinery buys. The 43 generic aliases in the repository that name themselves
in their own body, most of them there to be rejected, are all in the type checker's own tests
(`tests/check/alias_*`, `tests/run/alias_*`, `tests/unchecked/type_expr_*`); the examples, the site
and the standard library have none. The one recursive alias the library needs, `Json.Value`, takes
no type parameters and is never compared with another recursive alias.

## Prior art

| Language   | Recursive aliases                                                                                        |
| ---------- | -------------------------------------------------------------------------------------------------------- |
| Go         | Rejected; recursion goes through a defined type, which is nominal                                        |
| Rust       | Rejected (cycle when expanding a type alias); recursion goes through `struct` or `enum`                  |
| Swift      | Rejected; recursion goes through an `indirect enum` or a class                                           |
| Haskell    | Rejected ("cycle in type synonym declarations"); recursion through `data` or `newtype`                   |
| Elm        | Rejected ("this type alias is recursive, forming an infinite type"); the compiler suggests a custom type |
| TypeScript | Allowed since 3.7, compared structurally, with depth limits ("type instantiation is excessively deep")   |
| Python     | Allowed through PEP 695 `type` statements, evaluated lazily; type checkers compare them structurally     |

Ant sits between the two groups. It needs one recursive alias, because `Json.Value` must accept
`[1, "a"]` without a wrapper, so it cannot reject them outright as Go and Elm do. It does not need
TypeScript's structural comparison of two different recursive aliases, or recursion through generic
aliases, which are where TypeScript spends its depth limits. In the terms of Pierce's _Types and
Programming Languages_ (chapters 20 and 21), Ant moves from equi-recursive to iso-recursive
comparison for aliases: a recursive alias unfolds once, at the top, and is otherwise a name.

## Proposal

### Terms

An alias is **recursive** when it reaches itself through alias bodies and array, map, set and
function types, without passing through a named record type's fields. `Json.Value` is recursive.
`type Tree[T] = Leaf[T] | Node[T]` with `type Node[T] = { kids: Array[Tree[T]] }` is not: its cycle
passes through the record `Node`, which is compared by tag and type arguments and never expanded.

### Rule 1: a recursive alias takes no type parameters

A recursive alias may not declare type parameters, and neither may any alias on its cycle:

```ant
type Value = null | bool | float64 | string | Array[Value] | Map[string, Value]   // fine
type Nest[T] = Array[Nest[T]] | Box[T]          // compile error: a recursive alias takes no type parameters

type NestNode[T] = { items: Array[Nest[T]] }    // the same shape, through a record
type Nest[T] = NestNode[T] | Box[T]             // fine: Nest's cycle passes through NestNode
```

The message names the cycle and suggests the record, as Elm's does. The rule makes the existing
self-args rule ("passes its own type parameters unchanged and in order") a rule about records only,
since only a record can now lead a generic type back to itself.

### Rule 2: a recursive alias is compared by name

Two types, compared by `==` on type values or by assignability, are related as they are today,
except where both sides are recursive aliases:

- the same recursive alias on both sides is the same type;
- two different recursive aliases are never the same type and never assignable, whatever their
  bodies;
- a recursive alias against any other type unfolds once and is compared by its body.

So `Value` equals `null | bool | float64 | string | Array[Value] | Map[string, Value]` written out,
and `Array[Value]` is assignable to `Value`, as today. With `type F = func(): F | null` and
`type G = func(): G | null`, `F` and `G` are now different types. A comparison either reaches a pair
of recursive aliases, which it answers at once, or unfolds one of them against a finite type, which
it walks into. So it ends with no assumption about pairs under test.

Hashing a type value needs no change: the relation only gets finer, so equal types still hash equal.

### What stays

- The alias-cycle rule (`type A = A | null`) and the union-self rule
  (`type P = Array[P | int] | null`), both for plain aliases.
- Generic records and their self-args rule: `type Chain[T] = { next: Chain[Box[T]] | null }` is
  still an error.
- Union discriminators, bounds, inference, and every Reflect function.
- A memo of decided alias pairs, for speed on aliases that share structure (bug 6 of the first
  round). With no assumptions in play it is a plain cache, without the trail.

## What gets simpler

These are estimates from reading the code. A prototype (see Validation) would replace them with
numbers.

| Part                                                          | Today                                                    | After                                                               |
| ------------------------------------------------------------- | -------------------------------------------------------- | ------------------------------------------------------------------- |
| `Equal`, `Assignable` (`types.go`, `assign.go`, `memo.go`)    | Coinductive, trail of pairs under test, reset on failure | Structural with a name check; plain cache                           |
| Finding recursive aliases (`findRecursive`, `scc.go`)         | A cycle search per `Equal` call                          | One flag per declaration, set when it is resolved                   |
| Substitution (`subst.go`)                                     | `seen`, `done`, `plain`, `outer` to stop on itself       | Terminates structurally; records are never expanded                 |
| Union building inside `Equal` (`union.go`)                    | Raw unions plus validated unions, came-back checks       | One way to build a union                                            |
| `SelfArgs`, `UnionSelf`, `AliasCycle` and their `In` variants | Generic aliases and records, per component               | Self-args for records only; union-self and cycles for plain aliases |
| `checkDecl` in `rtype.go`                                     | Five declaration rules, per-expression components        | The same rules with the generic cases gone                          |

The instantiation memo for long generic alias chains (`ArgsMemo`, f01f0b8 and 9da0add) stays: those
chains are not recursive and remain legal.

## Spec changes

- **Section 4, Type declarations.** Define a recursive alias and add rule 1 to the paragraph on
  types that refer to themselves. Replace the generic examples of the union-self rule
  (`type R[T] = …`, `type L[T] = Array[L[T] | int]`) with plain ones. Reword the self-args paragraph
  to apply through records; `type Nest[T] = Array[Nest[T]] | Box[T]` moves from fine to an error,
  `type List[T]` stays fine.
- **Section 4, Assignability.** Replace the paragraph on recursive aliases ("a pair already under
  test counts as assignable", with `F` and `G`) by rule 2, and invert the example.
- **Section 4, Types as values.** In the `==` bullet, replace "compared as a cyclic value is: a pair
  already under comparison counts as equal" by rule 2.
- **Section 12.** Add the rule 1 case to the `type(T)` panic row, so an unchecked type expression
  reaching a generic recursive alias panics.
- **stdlib.md.** No change: `Json.Value` is plain.

## Migration

A generic recursive alias becomes a record that holds the recursive part, and an alias over it:

```ant
// before
type Tree[T] = Array[Tree[T]] | Box[T]
var t: Tree[int] = [Box{ value: 1 }, []]

// after
type Branch[T] = { kids: Array[Tree[T]] }
type Tree[T] = Branch[T] | Box[T]
var t: Tree[int] = Branch{ kids: [Box{ value: 1 }, Branch{ kids: [] }] }
```

Values gain one record literal per level, and `is Branch` replaces `is Array`. Two different
recursive aliases used interchangeably need one of them to be declared as an alias of the other
(`type G = F`), which then is not recursive itself and expands to `F`.

The repository needs no migration outside the type checker's tests. Those that exercise generic
recursive aliases change their expected errors, and the tests of the removed machinery go away with
it.

## Non-goals

- Making records structural, or aliases nominal in general. Only recursive aliases change.
- Removing recursive aliases. `Json.Value` and similar shapes keep working without wrappers.
- Changing what `ant run` checks. It still panics where the checker would reject a type it reaches.

## Open questions

1. **Does rule 2 make the union-self rule unnecessary?** That rule exists because the outer union of
   `type P = Array[P | int] | null` is formed from the inner one, which is formed from `P`. Under
   rule 2 the inner `P` might be kept as a name until the outer union is formed. If so, the rule and
   its checks can go too.
2. **Generic recursion that passes its parameters unchanged.**
   `type Nest[T] = Array[Nest[T]] | Box[T]` is legal today because of the self-args rule. Allowing
   exactly that case would keep most of the substituter machinery, so this proposal says no. Is
   there a use that justifies it?
3. **Error message.** The message should name the cycle and show the record form. A wording such as
   "type Nest refers to itself without a record, so it cannot take type parameters; hold the
   recursive part in a record, as `type NestNode[T] = { items: Array[Nest[T]] }`" needs review.

## Validation

Before acceptance, prototype on a branch of `ant`:

1. Implement both rules in the checker and in `rtype.go`.
2. Delete what they make unreachable, and report the lines removed per package.
3. Compare `ant check` and `ant run` output with master over every `.ant` file in the repository and
   a few thousand generated programs. The only differences should be in tests of generic recursive
   aliases and of comparing two different recursive aliases.
4. Re-run the reproducers in `private/type-bugs/repro/` and the long-chain tests, and check that no
   timing gets worse.
