# Proposal: field annotations

2026-11-05 · draft

## Summary

A field of a record type that a `type` declaration declares may carry annotations: constant values
of declared types, written before the field with `@`. The language gives them no meaning. Reflection
hands them to library code, boxed at their own types, so a codec reads the annotations it declared
and ignores the rest.

```ant
import "./toml.ant" Toml
import "./xml.ant" Xml

type Pool = {
  @Toml.Key("connection-max") connectionMax: int,
  @Toml.Key("idle-timeout") idleTimeout: int = 30,
  @Toml.Skip cache: Map[string, int] = Map{},
}

type Item = {
  @Xml.Attribute id: int,
  @Xml.Attribute lang: string | null = null,
  @Xml.CData note: string = "",
  name: string,
}
```

```
type Reflect.Field = { name: string, type: Reflect.ErasedType, hasDefault: bool,
                       annotations: Array[Reflect.Value] }
```

## Motivation

Three codecs written over Reflect (YAML, TOML, XML) found four things no type could say:

- **Renames.** TOML and YAML keys such as `connection-max` are not Ant identifiers.
- **XML attribute or child element.** The XML codec used `type Attr = string` as a marker, which
  works only for strings and changes the field's type for the whole program.
- **Skipping a field** that is state, not data.
- **Per-field format choices:** an inline table, CDATA, base64 or hex `Bytes`, a date layout.

All four belong to one field of one declaration. They are not part of the field's type: `id` is an
`int` wherever the program uses it, and only the XML codec cares that it is an attribute.

## The design

**Syntax.** `@` followed by an annotation: a record literal `@Toml.Field{ key: "x", inline: true }`,
a named primitive's constructor call `@Toml.Key("x")`, or a bare type name `@Xml.Attribute`, which
means `Xml.Attribute{}` and needs a record type whose fields all have defaults. A field may carry
several, one per line or on one line. `@` is a new token and starts nothing else, so the grammar
keeps one token of lookahead.

**Where.** Only on fields of the record body of a `type` declaration: exactly where a field default
may stand (section 4, Field defaults), for the same reason. An anonymous record type has no
declaration to hold metadata, and its equality is structural; a nested record type or a union member
has no name a literal could give. No annotations on types, functions, or parameters: `Shape` and
`Circle | Rect` are one type (section 4), so an annotation on an alias would make one type behave
two ways.

**Checked.** The annotation's type is a named record type or named primitive type in scope; an
unknown name, a wrong argument, and two annotations of one type on one field are compile errors. So
`@Toml.Kee("x")` does not compile, which a Go string tag would accept.

**Constant.** An annotation is built from literals, record literals, and constructor calls of
constants; it names no variable and calls no function. It may not mention a type parameter of the
declaration. So it means the same thing for every instantiation of a generic type and on every run.

**No meaning.** An annotation does not change the type, its tag, equality, `String.from`,
`Json.stringify`, assignability, or a literal. Adding or removing one is invisible to every program
that does not reflect.

**Reflection.** `Reflect.shape` fills `Reflect.Field.annotations` with the field's annotations in
the order written, each boxed at its own type. Since `shape` builds a fresh description on each call
(Reflect section), the annotations are evaluated afresh too, as a default is per literal; a record
annotation is mutable like any record, and a caller that changes one changes only its copy. A codec
finds its own by type value, which the Reflect section already compares:

```ant
let KeyT = Reflect.erase(type(Key))

func keyOf(f: Reflect.Field): string {
  for a in f.annotations {
    if Reflect.type(a) == KeyT { return String.from(Reflect.to(a, type(Key))) }
  }
  f.name
}
```

A codec should panic on its own annotation where it cannot apply, such as `@Xml.Attribute` on a
record-typed field, with a message naming the declaration and field.

## Fit with typed-data.md

The mechanism is here; the policy stays there. `typed-data.md` declares `Type.Key` (and later
`Type.Skip`) as ordinary prelude types and honors them in `Type.build` and `Type.data`; its `key`
call option and `Type.snakeCase` go, as its "Under consideration" section already plans. A format on
the `Type.Data` path inherits renames for free; a format on the `Reflect.Value` path declares and
reads its own. Its open question 3 is answered: annotations replace `key`; `tag` stays as a call
option for a root union and may also be an annotation on a field.

## Types you do not own

An annotation can only be written by the type's author. A codec may also take an override table
keyed by `Reflect.Decl` and field name, which works today with no language change. That is a library
convention, not part of this proposal.

## Alternatives

- **No annotations; per-codec options keyed by `Reflect.Decl` and field name.** Works today
  (prototyped in a TOML codec: rename and skip, about 35 lines plus an `opts` parameter on every
  internal function). Field names are strings, so a typo is caught only at run time and only when
  the codec reaches that record; a field rename in Ant leaves the table stale; the table lives away
  from the type and every call site passes it; a library that exports a type must also export a
  table per format.
- **Go/Odin string tags.** Smallest syntax, but parsed by each library at run time, unchecked, and
  one string per format key.
- **Metadata on types,** Python's `Annotated[int, Field(alias=…)]`. Either it changes type equality,
  or two equal types behave differently; both break section 4.
- **Marker types,** the status quo `type Attr = string`. Changes the field's type everywhere, covers
  one choice per field, and does not compose with a rename.
- **Per-type hooks,** Swift `CodingKeys`, Zig declarations on the struct. Checked and attached to
  the type, but Reflect has no way to call a function on a type, and a table of names per type is a
  worse spelling of the same metadata.

## Costs

- A token, one grammar position, one checker rule set mirroring field defaults.
- The interpreter keeps each annotation's syntax with the field, as it keeps defaults, and evaluates
  it in `Reflect.shape`.
- `ant fmt`, the LSP, the highlighting grammars, and `ant doc` learn the form.

## Spec changes

- Section 4: a "Field annotations" subsection after Field defaults.
- Section 11, Reflect: `annotations` on `Reflect.Field`, and a sentence in Shapes.
- Grammar: the `@` token.
