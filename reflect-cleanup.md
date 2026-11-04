# Proposal: Reflect cleanup

2026-11-04 · Anton · Accepted

## Summary

Five corrections to the Reflect section of the specification (`ant-lang.md`, section 4, Types as
values, and section 11, Reflect), each found while implementing it or writing its tests:

1. **Shapes carry no `name`.** `Reflect.Record`, `Reflect.Named`, and `Reflect.Opaque` lose the
   field. A `Reflect.Decl` renders as the declared name, and `String.from(t)` renders the type, so
   the field said one of those two things in each shape, and not the same one.
2. **`Reflect.Named` becomes `Reflect.Tagged`**, after the kind "tagged primitive" in the table of
   Kinds, the builder `Reflect.makeTagged`, and the reader `Reflect.payload`.
3. **The union rule for records says what it can mean.** An anonymous record member takes any
   record, because no named record type may stand beside it (section 4, Unions). The current text
   describes a choice that never arises.
4. **Bodiless prelude types are testable.** Section 10 says a `type` with no body is one `is` cannot
   test; the prelude's own `Bytes`, `Html`, `Regex.Pattern`, and time types are tested by `is`, as
   the table of Kinds says. The checker followed the sentence rather than the table and rejected
   `Bytes | Mutex`, since it gave `Bytes` and `Html` no discriminator. A declaration kind for the
   two replaces the name matching that fixed it.
5. **One discriminator.** The runtime resolves a union to a member with the checker's own
   discriminator keys, instead of a second table of kinds and tags written for values.
6. **`Reflect.default` goes.** It evaluates a field default, effects included, each time it is
   called, and its one use, an encoder that leaves out fields equal to their defaults, is wrong
   whenever a default is not a literal. Decoding never needs it: `Reflect.Field.hasDefault` says a
   field may be left out and `Reflect.makeRecord` fills it. Omitting fields on encode is format
   policy and moves to `typed-data.md`.

A program sees two renames, one dropped field, and one dropped function. Everything else is text and
implementation.

## Motivation

- The implementation of `Reflect.shape` sets `Reflect.Record.name` to the bare declared name, `Box`,
  and `Reflect.Opaque.name` to the rendered type, `Channel[int]`, because that is what the spec says
  for each. Both are derivable: `String.from(s.decl)` is the first and `String.from(t)` the second.
  In `Reflect.Record` the two nullable fields `name` and `decl` are `null` together, so a decoder
  tests one and ignores the other.
- `Reflect.Named` is the shape of a named primitive type, but a named record type is a
  `Reflect.Record`, and the checker's own `Named` is any declared type. The runtime calls the values
  "tagged primitives", builds them with `Reflect.makeTagged`, and reads them with `Reflect.payload`.
  The shape should use the same word.
- The invariant in section 11 says "a record whose tag no named member has goes to the anonymous
  member". Section 4 forbids an anonymous record type in a union alongside a named record type, so
  the only unions with an anonymous member are like `{ name: string } | null | int`, where every
  record goes to it. The sentence is true but suggests a resolution step that does not exist.
- The checker treated `Bytes` and `Html` as opaque declarations with no discriminator, like `Mutex`,
  so `Bytes | Mutex` and `Bytes | Html` were compile errors although the table of Kinds gives each
  its own kind and `is` test. The fix in `ant` matches the names `Bytes` and `Html` in
  `types.Discriminator`, as the checker already matches `Time.` and `Regex.Pattern` when it declares
  them. The runtime has the same special case: `types.Decl.Tag` holds either the tag of a declared
  type or, for these two, a kind.
- `Reflect.default(t, name)` runs the default of one field afresh, as a literal would. With
  `type Job = { name: string, id: int = next() }`, two calls return 1 and 2 and advance the counter,
  so `Reflect.field(d, "id") == Reflect.default(t, "id")` is false for a job whose `id` is 7, and an
  encoder written as section 11 suggests runs every default of every record it writes, effects
  included; a default that reads a top-level `var` before its initializer ran, or one that builds
  its own type, panics inside the encoder. The review of the original proposal made the same point
  about the "`omitempty` in spirit" advice and asked for it to be removed. Without that use the
  function has no caller: a decoder leaves a field out when `hasDefault` is set and
  `Reflect.makeRecord` evaluates the default where a literal would. Go's `omitempty` is zero-value
  based and serde's `skip_serializing_if` is a predicate the caller writes; neither evaluates a
  declared default.
- The runtime's `Reflect.valueOf` resolves a union by its own list: a kind per primitive, a tag per
  record or tagged primitive, the time type's name, a sync object's name compared as a string, and
  the four Reflect types by their tags. The checker decides which unions are legal with
  `types.Discriminator`, keyed by kind, tag, or shape class. Two tables that must agree are one too
  many.

## Prior art

| Language | What it does                                                                                               | What this proposal takes                                          |
| -------- | ---------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| Go       | `reflect.Type` has `Name()`, `""` for an unnamed type, and `String()`, which renders; `Kind()` is separate | The split: a name belongs to a declaration, a rendering to a type |
| Zig      | `std.builtin.Type.Struct` carries fields and no name; `@typeName(T)` renders any type                      | Shapes without a name field                                       |
| Rust     | `std::any::type_name::<T>()` renders; structural reflection has no name of its own                         | The same                                                          |

## The proposal

### Shapes without a name

```
type Reflect.Record = { decl: Reflect.Decl | null, args: Array[Reflect.ErasedType], fields: Array[Reflect.Field] }
type Reflect.Tagged = { decl: Reflect.Decl, args: Array[Reflect.ErasedType], base: Reflect.ErasedType }
type Reflect.Opaque = { decl: Reflect.Decl, args: Array[Reflect.ErasedType] }
```

`decl` is `null` exactly for an anonymous record type. The declared name of a type is
`String.from(s.decl)`; the type itself, arguments included, is `String.from(t)` of the type value
the shape came from, which the caller holds. A shape renders as
`Reflect.Record{ decl: Server, args: [], fields: [...] }`.

```ant
type Server = { host: string, port: int = 8080 }

var t = Reflect.erase(type(Array[Server]))
var s = Reflect.shape(t)
if s is Reflect.Array {
  var e = Reflect.shape(s.element)
  if e is Reflect.Record && e.decl != null {
    println("${t} holds ${e.decl} with ${len(e.fields)} fields")      // Array[Server] holds Server with 2 fields
  }
}
```

A decoder that refuses anonymous record types tests `s.decl == null` where the sketch tested
`s.name == null`.

### `Reflect.Tagged`

`Reflect.Named` is renamed `Reflect.Tagged` in the `Reflect.Shape` union, the listing, and the
prose, which calls the case "tagged primitives". `Reflect.makeTagged` and `Reflect.payload` keep
their names.

### The anonymous member

The reader rule reads: "When that type is a union, the reader resolves it to the member the child's
value belongs to. The union rules guarantee there is exactly one, found from the value's kind or tag
alone (section 4, Unions): an anonymous record member takes any record, since no named record type
may stand beside it, and a value of a type `is` cannot test goes to the one such member." The same
wording goes into the `Reflect.valueOf` entry of `prelude.d.ant`.

### Bodiless types in the prelude

Section 10, Declaration files, says of `type Regex.Pattern`: "its values come only from the
functions declared on it, and `is` cannot test them, as for the synchronization types". That holds
for a host's declaration file. The prelude's own bodiless types are the exception the table of Kinds
already lists: `Bytes` and `Html` are kinds of their own, and `Regex.Pattern` and the time types
carry a tag, so `is` tests all of them by name. The sentence gains that exception and points at the
table.

In the checker, a declaration kind for `Bytes` and `Html` says what they are instead of their names:
an opaque type whose values carry a kind, which `is` tests by name and which counts once in a union.
`types.Discriminator` then gives them `kind:Bytes` and `kind:Html` without looking at the name. The
runtime builds the same declarations, and `types.Decl.Tag` holds a tag and nothing else.

### One discriminator

`Reflect.valueOf`, the readers, and `Reflect.default` resolve a union to a member by computing the
value's discriminator keys in the form `types.Discriminator` uses for types, and taking the member
with that key: a primitive, `null`, or `void` by kind; an array, map, set, or function by its class;
a record by its tag's declaration, then by the class `record` for an anonymous member; a tagged
primitive, time value, or pattern by its tag's declaration. A value with no key, a sync object or a
value of one of the four Reflect types, goes to the one member that has none, which the union rules
allow at most once. The runtime then consults the same rule that admitted the union.

### Dropping `Reflect.default`

`Reflect.default` leaves the listing, with its three panic cases, and the paragraph that defines it
and the sentence that recommends comparing fields with their defaults leave section 11. A field
default is evaluated in two places only, as before this section existed: a record literal and
`Reflect.makeRecord`, which fills an omitted field "as a literal evaluates it". An encoder that
wants to leave fields out takes a predicate from its caller, as serde's `skip_serializing_if` does,
which is the policy `typed-data.md` decides; its "omitting defaults on encode" option, which it
built on `Reflect.default`, becomes that predicate or goes.

```ant
type Job = { name: string, id: int = next(), tags: Array[string] = [] }

var s = Reflect.shape(Reflect.erase(type(Job)))
if s is Reflect.Record {
  for f in s.fields { println("${f.name} may be left out: ${f.hasDefault}") }
}
var made = Reflect.makeRecord(Reflect.erase(type(Job)), Map{ name: Reflect.valueOf(type(string), "x") })
// name may be left out: false, id: true, tags: true; made has a fresh id and a new tags array
```

## Spec changes

- **Section 11, Reflect.** The listing: the three shapes above, `Reflect.Tagged` in `Reflect.Shape`,
  and no `Reflect.default`. The bullets "Records", "Named primitives", and "Opaque": no `name`; the
  declared name is the `decl` rendered, the type is the type value rendered. The paragraph on
  `Reflect.default` and the omit-defaults sentence go; the "Records" bullet keeps "`hasDefault` says
  whether a literal, and so `Reflect.makeRecord`, may leave the field out". The invariant's reader
  bullet: the anonymous-member sentence above. The example that narrows with `s is Reflect.Named`
  uses `Reflect.Tagged`.
- **Section 10, Declaration files.** The bodiless-type bullet gains the prelude's exception and a
  pointer to the table of Kinds.
- **Section 4** and **section 12** change nothing: the kinds, the union rules, and the panics stay.

## Implementation

- `prelude.d.ant`: the three types, `Reflect.Tagged`, and their docs; `Reflect.default` and its doc
  go; `TestPreludeTypesMatchLibrary` keeps the tags in step.
- `internal/interp`: `FieldDefault`, exported for `Reflect.default` alone, goes back to being the
  private helper `fillDefaults` calls; `FillDefaults` and `HasDefault` stay for
  `Reflect.makeRecord`.
- `internal/lib/reflect.go`: the tags and `shape`, which drop the name values; union resolution by
  discriminator keys, which needs the declaration a tag was given, read without building it, since a
  tag never reached by a type expression cannot name a member.
- `internal/types`: a declaration kind for `Bytes` and `Html`, set where `resolveTypeDecl` sets the
  time types' kind, with the kind in `Decl.Base`; `Discriminator` reads it; `Decl.Tag` typed as
  `*value.Tag`. The runtime's `builtinDecl` builds the same kind.
- Tests: `tests/stdlib/reflect.ant` renderings and its `Reflect.default` block, which goes;
  `tests/run/reflect_defaults.ant`, which keeps the `makeRecord` cases (fresh evaluation, a generic
  record at two type arguments, a local type) and loses the `Reflect.default` ones;
  `tests/run/lib/typed.ant` (`s.decl == null`); a `tests/check` case that `Bytes | Mutex` and
  `Bytes | Html` are unions.
- `proposals/typed-data.md`: the "omitting defaults on encode" option becomes a caller predicate or
  goes; `Reflect.Opaque` keeps its name.

## Not changed

- **`Reflect.Decl`** was considered for removal: a runtime kind that serves codec tables keyed per
  generic declaration. Without it nothing says that `Box[int]` and `Box[string]` come from one
  declaration, since a name collides across modules and a type value needs its arguments. With
  `name` gone it also carries the declared name, so the shapes lose a field rather than keep two. It
  stays.
- **The explicit `Reflect.erase`.** The friction is real: `==` needs assignable operands, so
  comparing tokens of two types needs `Reflect.erase` on both sides, and a function that returns a
  local type's token returns a `Reflect.ErasedType`. It is the price of no implicit conversion, paid
  at the edges only, and stays as decided.
- **How the interpreter finds a type's names.** The resolver gives the head of every named type in a
  type expression or a declaration body a synthetic identifier, resolved like a variable, so the
  interpreter evaluates `Geometry.Point` as `is` would. A lookup by name through the environments at
  run time would be a second mechanism; the walk at load is cheap. It stays.

## Open questions

None.
