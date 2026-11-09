# Proposal: private fields

2026-11-09 - revised draft

This revision is based on the attached 2026-11-07 draft. It retains its syntax and checker-only
model. The main additions clarify structural views, spreading, reflective compatibility,
serialization, and the limits of invariant protection. Research notes and a proposed regression
matrix follow the specification changes.

## Summary

A field in the record body of a `type` declaration may be marked `private`. Its name is available
only in the declaring module and in a test file that imports that module, following Ant's existing
test visibility rules. Fields remain public by default.

Outside that visibility boundary, ordinary checked code cannot read or write the field, name it in a
literal or destructuring, copy it with a spread, or obtain it through structural widening. A private
field without a default prevents an external record literal from constructing the type.

Privacy restricts the source API, not access to runtime data. `ant run` does not enforce it.
Reflection remains able to inspect and construct private fields. Equality, hashing, `String.from`,
and `Json.stringify` retain their existing treatment of record data.

```ant
// span.ant
export type Span = {
  private start: int,
  private end: int,
}

export func Span.new(start: int, end: int): Result[Span, Error] {
  if start > end {
    return Result.err(Error{ message: "span: start ${start} after end ${end}" })
  }
  Result.ok(Span{ start, end })
}

export func Span.length(s: Span): int { s.end - s.start }
```

An importing module can name `Span`, hold its values, and call its exported functions. It cannot
construct `Span` with an ordinary literal or directly modify either endpoint. Reflection and any
access deliberately exposed by `span.ant` are explicit exceptions to that protection.

## Motivation

An exported record type should not require every part of its representation to be an ordinary public
field API. Libraries need to hide invariant-sensitive fields, mutable bookkeeping, and
implementation details while keeping the type name available in signatures.

For `Span`, a validating factory is not useful as an ordinary API boundary if another module can
write `Span{ start: 9, end: 1 }` or later assign an invalid endpoint. For a pool or cache, exposing
internal arrays, maps, and synchronization objects invites callers to bypass the functions that
coordinate them.

The intended guarantee is limited: correctly checked client code must use the access paths the
module exposes, unless it explicitly uses reflection. The guarantee is not that every instance must
have passed a particular factory, nor that a private field's contents are confidential.

Private fields are not immutable. A module can expose its representation accidentally by returning a
mutable array, map, or record stored in a private field. It can also deliberately expose a writable
structural view. Field visibility does not prevent mutation through such an alias.

Public fields remain public and mutable. A `Pool` with public `max` still permits `pool.max = -1`.
If a field participates in an invariant that clients must not break, it must be private and the
module must control how it changes.

Finally, hiding a field's name does not make changes to that field unobservable. Equality, hashing,
raw JSON, debugging output, and reflection may still depend on private state. This proposal provides
source-level encapsulation, not full representation independence.

## Prior art

**Go** provides package-scoped access to unexported fields and excludes them from ordinary
`encoding/json` struct encoding. Its reflection API is more restrictive than this proposal: an
unexported field is not settable, and converting a reflect value obtained from it to an interface
panics, although kind-specific readers can inspect primitive values. Ant should not describe its
unrestricted reflection as equivalent to Go's. [G1, G2, G3]

**Rust** requires field visibility for struct construction and functional update syntax, including
fields supplied implicitly by the update base. This supports a conservative rule for spreads. Rust's
module hierarchy and visibility modifiers need not be imported into Ant. [R1]

**TypeScript** distinguishes checker-only `private` from runtime-enforced JavaScript `#` fields. It
also uses private-member declaration origin in class compatibility. Ant's named-record identity
already serves the relevant nominal role; this proposal does not add a second private-field brand.
[T1, T2]

**Serde** demonstrates that serialization policy need not be identical to source visibility: its
examples serialize non-`pub` fields and use explicit attributes to skip selected data. Private
access and wire-format design are separate decisions. [S1, S2]

## The design

### Syntax and placement

`private` precedes a field name:

```ant
type Cache[K, V] = {
  limit: int,
  private entries: Map[K, V] = Map{},
  private order: Array[K] = [],
}
```

It is a contextual keyword only in that position. A field named `private` remains legal:

```ant
type Flags = { private: bool }
```

The next token distinguishes the modifier from that field name. No global keyword or new visibility
level is introduced.

The modifier is allowed only in the record body directly declared by a `type` declaration, including
a local type declaration. It is not allowed in an anonymous record type, a nested anonymous record
body, or an inline record member of a union. A named type with private fields may still occur in a
union.

### Visibility and ownership

A private field belongs to its original record declaration and that declaration's module. Import
renaming, namespace access, passing a value through another module, and generic instantiation do not
change that owner.

The field is accessible throughout its declaring module, not only in functions declared on its type.
A test file that imports the module has the same access granted by Ant's test visibility rules. This
does not grant production code access through test-only helpers.

Where the field is inaccessible, the checker rejects:

| Operation                                                                            | Example                                   |
| ------------------------------------------------------------------------------------ | ----------------------------------------- |
| Reading, optional reading, or calling a function-valued field                        | `s.end`, `s?.end`, `s.end()`              |
| Assignment or compound assignment                                                    | `s.end = 0`, `s.end += 1`                 |
| A literal entry, including shorthand                                                 | `Span{ start: 1, end: 2 }`, `Span{ end }` |
| A literal omitting a required private field                                          | `Span{}`                                  |
| Destructuring that names the field                                                   | `var { end } = s`                         |
| A spread that reads or supplies the field                                            | `Other{ ...s }`, `Span{ ...other }`       |
| Structural widening that requires the field                                          | `var v: { end: int } = s`                 |
| A common field access through a union when a relevant member's field is inaccessible | `value.end`                               |

Diagnostics identify the declaring type and module, for example:

```text
Span.end is private to span.ant
```

An inaccessible field is not treated as absent so that lookup can silently select a different
operation with the same name.

### Construction and defaults

An external literal may omit a private field only when that field has a default. It may never
explicitly supply that field, even with the same value as its default.

```ant
// pool.ant; Conn is the pool's existing connection type.
export type Pool = {
  max: int = 10,
  private idle: Array[Conn] = [],
  private lock: Mutex = Mutex.new(),
}
```

```ant
// An importing module.
var p = Pool{ max: 4 } // allowed: both private fields use their defaults
```

If any private field lacks a default, an external literal cannot construct the record. This does not
introduce a constructor mechanism: ordinary module functions provide the supported construction API,
and reflection retains its separate construction facilities.

Defaults retain their existing evaluation rules. They run when an omitted field needs a value, in
the declaring scope, and are evaluated afresh for each construction. An expression such as `[]`
creates fresh storage; an expression that returns existing shared storage can still share it.
Privacy changes neither rule.

Providing defaults for every private field intentionally permits external literal construction. It
does not validate arbitrary public field combinations. A type that needs cross-field validation
should require construction through its functions instead.

### Structural views, nominal identity, and generics

This proposal does not change named-record identity or generic invariance. A structurally similar
record does not become a `Span` merely because its fields have the same names and types.

For widening to an anonymous record type, every required source field must have the existing
required type and must be accessible where the widening is checked. Inaccessible extra fields do not
prevent a view that requires only public fields.

```ant
// account.ant
export type Account = {
  name: string,
  private token: string,
}
```

```ant
// Another module; account is an Account.
var label: { name: string } = account // allowed
var secret: { token: string } = account // compile error
```

The same accessibility requirement applies to implicit conversions, arguments, returns, generic
bound satisfaction, and inferred structural views. A generic helper requiring `{ token: string }`
must not accept `Account` from outside its owning module merely because generic inference discovers
that field.

The owner may deliberately expose a structural view containing a private field:

```ant
// Inside span.ant.
export func Span.endView(s: Span): { end: int } { s }
```

The returned view permits external code to read and write `end`. It aliases the original record; it
is not a copy or a read-only view. This is an intentional delegation by the owner, not a privacy
violation. A scalar getter avoids exposing that writable slot.

A structural view does not remove runtime fields and is not a serialization filter. An import rename
or an `is` narrowing back to the named record does not grant access to its private fields.

Accessibility is contextual, while type identity is not. The checker must not globally cache an
owner-authorized conversion as valid in all modules. Any cached visibility-dependent result must
distinguish the access context, including test privileges. This applies recursively where
assignability and interface checks consult fields.

### Spreads

Keep the existing runtime meaning of spread: it copies actual source fields, and later entries
replace earlier ones. Do not introduce a public-fields-only spread or make its runtime behavior
depend on a static structural view.

The existing requirement for a named record spread operand remains in force. An anonymous structural
view is not an exact list of runtime fields and cannot be used to evade the privacy rule by
spreading a narrowed view.

For each otherwise legal spread, the checker requires access to every source field it reads and
every destination field it supplies. A field that is overwritten by a later entry is still part of
that spread; later replacement does not authorize an earlier inaccessible access.

Outside `span.ant`, both of these are rejected:

```ant
Span{ ...s }
Span{ ...s, start: 0 }
```

The first is rejected even though it specifies no replacement fields. Copying an already valid
wrapper is not generally guaranteed to preserve its invariant: a shallow copy may share a mutable
backing array while duplicating an independent counter, or duplicate logical ownership of a
resource. A module can provide an explicit copy function with the appropriate semantics.

An ordinary assignment that shares a reference is not a spread and remains allowed:

```ant
var another = s // the same Span, not a new record shell
```

Spreading public configuration into a record whose remaining private fields have defaults is
allowed:

```ant
type PoolOptions = { max: int }
var p = Pool{ ...PoolOptions{ max: 4 } }
```

If the source also contains `idle` or `lock`, that spread would supply inaccessible destination
fields and is rejected, even though those fields have defaults.

### Runtime behavior, equality, and rendering

Ordinary interpreter operations do not check caller-module visibility. Record slots acquire no
access-control state. Unchecked code may access private fields as it accesses public ones.

The modifier is retained in declaration metadata so that reflection can describe it. That is not
runtime access enforcement and does not require keeping a variable's inferred type or a privacy
check on each field operation.

Private fields continue to participate in equality and hashing. `Record.same` is unchanged.
`String.from` and `Json.stringify` continue to inspect runtime data under their existing rendering
rules; unsupported values and cycles retain their existing behavior.

Consequently, adding a private cache or changing private bookkeeping can affect equality, hashing,
and serialized output. Privacy does not promise that these operations depend only on the public API.
Custom equality, redaction, and representation-independent serialization are separate design
questions.

### Reflection and reflective compatibility

Add one property to `Reflect.Field`:

```ant
type Reflect.Field = {
  name: string,
  type: Reflect.ErasedType,
  hasDefault: bool,
  private: bool,
}
```

`private` reports the declaration's modifier, not whether the current caller has access.
`Reflect.shape` includes all declared fields. Its returned descriptions remain detached data:
modifying a descriptor does not change declaration visibility.

`Reflect.field` may read private fields, and `Reflect.makeRecord` may supply them, from any module.
Their existing type, tag, field-presence, and default checks still apply. Private does not mean
unchecked or arbitrarily typed.

The specification must distinguish source-level assignability from reflective compatibility:

> For record-field privacy, `Reflect.assignable` applies the existing closed-type compatibility
> rules without the source-level field visibility restriction. Its result is independent of the
> caller's module. The reflection readers, builders, and `Reflect.to` use that same reflective
> relation.

This is a deliberate clarification, not a new overload or caller-module parameter. The current
wording equating reflective compatibility with all source-level assignability checks must be amended
accordingly. Any existing interface-visibility wording should be reviewed for the same distinction
rather than silently folded into this change.

A direct source conversion can therefore be rejected while the corresponding reflective conversion
succeeds. For example, reflection can obtain an anonymous view that lists a private field; that view
can expose a writable alias. This is part of reflection's explicit escape hatch.

The promised boundary is ordinary checked access, not an absolute guarantee for every checked
program that invokes reflection. Protecting private state from reflection would require a different
reflection policy: a blanket denial could use declaration metadata, while selectively authorizing
callers would additionally need an authority mechanism. Neither is part of this proposal.

### Serialization policy

Do not change `Json.stringify` in this proposal. It remains value-directed and does not acquire an
automatic visibility filter. Making a field private must not silently be presented as a way to
redact credentials or other confidential values.

A public structural annotation also does not redact a value:

```ant
// account is an Account from the earlier example.
var label: { name: string } = account
Json.stringify(label) // token is still part of the underlying record
```

Construct an explicit export value when the output must contain selected fields:

```ant
type AccountOutput = { name: string }
Json.stringify(AccountOutput{ name: account.name })
```

Nested reference-valued fields need their own deliberate export policy; a shallow projection does
not sanitize their contents.

Reflection-based, type-directed codecs should default to omitting private fields when encoding and
leaving them to defaults when decoding. This is a recommended library policy, not a restriction on
reflection or a universal rule for all serializers.

Under that policy, a decoder must not fill a private field from input merely because the input names
it. It follows the codec's normal unknown-or-ignored-field policy. If an omitted private field lacks
a default, decoding fails unless an explicitly selected hook handles the type. A hook may validate
input and call the module's factory.

An explicitly opted-in codec can intentionally serialize or construct private state. Fully
encapsulated types commonly need such hooks for meaningful round trips. Omitting private fields does
not itself guarantee round-trip behavior or a stable wire format.

### Declaration files

Keep `private` a syntax error in `.d.ant` declaration files under their existing host-declaration
model. Host types that hide their state use the existing opaque form. This proposal does not turn
declaration files into a new distribution format for partially hidden Ant record implementations.

### Tools

`ant doc`, external completion, and hover omit inaccessible fields from the ordinary public API
presentation. Go to definition remains available.

The tools can report constructibility without guessing a factory function:

```text
// private fields; not constructible by a literal outside span.ant
```

When all private fields have defaults, a suitable comment is:

```text
// private fields with defaults
```

Public required fields must still be supplied. Neither comment claims that a function named `new`
exists.

The formatter and syntax highlighters recognize the contextual modifier. Existing field/function
name collision rules stay unchanged, including collisions involving private fields.

## Alternatives

**No private fields.** This leaves the factory and mutation boundary as documentation only.
Reflection's escape hatch does not eliminate the value of checking ordinary field access and
excluding internals from completion.

**Opaque exports.** A declaration-wide opaque export is worth considering separately, especially for
types with no public representation. Per-field privacy additionally supports mixed public data and
private state. An all-private record here is still reflectively transparent; it is not runtime
opacity.

**Type-scoped access.** This would complicate access by module helpers without matching Ant's
module-level function model. Use the module as the boundary.

**A naming convention or private-by-default fields.** Keep explicit `private` and public defaults.
Do not rename fields to change access or require visibility modifiers on every existing data record.

**Runtime enforcement.** This would be a different feature with a different threat model. It could
be designed without changing equality, so there is no need to claim that runtime privacy inherently
requires ignoring fields in comparison. It is rejected here because this proposal deliberately
preserves advisory checking and unrestricted reflection.

**Automatic JSON omission.** This is a possible future serializer policy, not a technical
impossibility under type erasure: declaration metadata could support it. It is not included because
it would change the established value-directed behavior and conflate two independently useful
policies.

**Same-type spread exceptions.** Do not add them initially. The declaration owner is better placed
to decide how the representation may be copied.

## Costs

The checker needs field-access checks at every access and conversion site, original declaration
ownership through inference and generic substitution, and context-correct assignability caching.
This is the main semantic implementation cost.

The parser, formatter, highlighting grammars, documentation generator, and LSP need the contextual
modifier and public API presentation. Reflection gains one metadata property and an explicit
statement about visibility-independent compatibility.

No per-access interpreter enforcement, hidden instance types, second nominal brand, new constructor
mechanism, or JSON behavior change is required.

Getter names still cannot collide with field names. A private field such as `endAt` and a function
`end` remain distinct names, under the existing call-resolution rules.

## Fit with other proposals

**annotations.md.** Serialization annotations remain codec policy. A future opt-in may include a
private field, and `@Skip` may omit public state. Neither changes source visibility.

**typed-data.md.** If `Type.build` and `Type.data` are adopted with the codec policy described in
the original draft, they should omit private fields by default and require a hook where omitted
required state cannot be constructed. Their behavior must be stated in that proposal, not inferred
from this modifier alone.

**frozen.md.** Preserve the original draft's intended compatibility: privacy is not a reason to omit
fields from freezing. The frozen proposal's own copying and immutability rules remain a separate
review; private does not itself mean frozen, immutable, or safe to copy.

## Open questions resolved in this revision

1. **External same-type spread?** No, when it would read inaccessible fields, including a spread
   with no overrides. Sharing an existing reference is unaffected.
2. **Reflection refuses private fields?** No. Document the escape hatch and make reflective
   compatibility independent of the calling module's field visibility.
3. **Document literal constructibility?** Yes. Report whether required private state prevents
   external literals; do not invent a factory name.

The implementation gate is the behavior below, especially contextual conversions and spread
provenance. This document proposes that behavior; it does not report an implementation passing the
tests.

## Spec changes

Section 2 gains the contextual `private` modifier. Section 4 gains field visibility, construction,
spread, and structural-widening rules, while retaining named identity and generic invariance.
Section 5 applies visibility to reads, calls, assignments, and destructuring. Section 10 extends the
established module/test visibility boundary to fields and records the declaration-file restriction.

Section 11 adds `Reflect.Field.private`, distinguishes reflective compatibility from source
visibility, and explicitly documents raw JSON and reflection exposure. Section 12 lists the new
checker errors while keeping ordinary runtime field behavior unchanged. `doc.md` and `lsp.md`
specify public field presentation and constructibility comments.

## Proposed regression matrix

These are acceptance criteria for a future implementation, not executed Ant tests.

| Case                                                                            | Expected result                                                        |
| ------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| Owner module reads, writes, destructures, and constructs private fields         | Accepted                                                               |
| Importing ordinary module performs those operations                             | Rejected                                                               |
| Import renaming or namespace access attempts to change the owner                | No additional access                                                   |
| Importing test file accesses fields under the existing test rules               | Accepted                                                               |
| Unrelated record has identical fields to an imported named record               | No implicit named-record conversion                                    |
| External literal omits private fields, all with defaults                        | Accepted                                                               |
| External literal omits a private field without a default                        | Rejected                                                               |
| External literal explicitly supplies a private field with a default             | Rejected                                                               |
| Repeated literal defaults create fresh arrays when the expression is `[]`       | Distinct arrays                                                        |
| Widening requests only public fields                                            | Accepted                                                               |
| External widening or generic bound requires a private field                     | Rejected                                                               |
| Owner deliberately exports a view containing a private field                    | Accepted; the view aliases the field                                   |
| A common union field is inaccessible in a relevant member                       | Rejected                                                               |
| Inference or nested function-type conversion would expose an inaccessible field | Same visibility rule; no bypass                                        |
| Outside code spreads a record with private fields, even into the same type      | Rejected                                                               |
| Anonymous public view is used as a spread source                                | Rejected under the existing operand rule                               |
| Public configuration is spread into a type with omitted private defaults        | Accepted                                                               |
| Spread source supplies a private destination field                              | Rejected, even if later overwritten                                    |
| Assignment shares an existing private-field record                              | Accepted; same record identity                                         |
| A permitted owner conversion is cached, then checked in an external module      | External conversion remains rejected                                   |
| `Reflect.shape` describes a private field                                       | Included, with `private: true`                                         |
| Returned reflection metadata is mutated                                         | Declaration and later descriptions unchanged                           |
| Reflective field read, construction, and compatible anonymous-view conversion   | Allowed by the documented reflective rules                             |
| The same `Reflect.assignable` call is made in different modules                 | Same result                                                            |
| Equality and hashing inspect private state                                      | Existing full-data semantics retained; hash collisions remain possible |
| Raw JSON receives a serializable record through a public structural view        | Actual private fields remain visible                                   |
| Default-policy codec sees an input key naming a private field                   | Does not populate it implicitly                                        |
| Default-policy codec omits a required private field without a hook              | Decode failure                                                         |
| Private field and type function have the same name                              | Existing clash error retained                                          |
| A declaration file uses `private` on a field                                    | Syntax error                                                           |

## Research notes and sources

The attached draft is the baseline. External sources verify comparisons and two important
pre-existing Ant constraints: named records already have nominal identity, and record spread already
excludes anonymous structural operands. The source-level/reflective compatibility distinction and
the stronger statement of limits are recommendations introduced by this revision. No Ant interpreter
or checker was executed for this review.

Local comparison experiments used TypeScript 5.8.3 and Go 1.23.2; these are the installed test
versions, not claims about the latest releases. A TypeScript record-shaped class with a
checker-private member retained that member in direct JSON, widened-view JSON, and runtime object
spread. A Go struct omitted its unexported member from JSON; reflection read its integer value while
reporting `CanInterface: false` and `CanSet: false`. Those experiments illustrate the documented
policy differences, not Ant behavior.

- **A1:** Ant specification, sections 1, 4, 10, 11, and 12:
  <https://github.com/ant-src/spec/blob/master/ant-lang.md>.
- **G1:** Go specification, exported identifiers and composite literals: <https://go.dev/ref/spec>.
- **G2:** Go `reflect`, especially `Value.CanSet`, `Value.Interface`, and kind-specific readers:
  <https://pkg.go.dev/reflect>.
- **G3:** Go `encoding/json`, struct field encoding: <https://pkg.go.dev/encoding/json>.
- **R1:** Rust Reference, struct expressions and functional update visibility:
  <https://doc.rust-lang.org/reference/expressions/struct-expr.html>.
- **T1:** TypeScript Handbook, classes and member visibility caveats:
  <https://www.typescriptlang.org/docs/handbook/2/classes.html>.
- **T2:** TypeScript Handbook, private/protected members in type compatibility:
  <https://www.typescriptlang.org/docs/handbook/type-compatibility.html>.
- **S1:** Serde's skip-serialization example: <https://serde.rs/attr-skip-serializing.html>.
- **S2:** Serde field attributes, including independent skip and default policies:
  <https://serde.rs/field-attrs.html>.

Sources were consulted on 2026-11-09. Public repository branches can change; the attached proposal
remains the basis for the design being revised.
