# Proposal: field annotations

2026-11-05, rewritten 2026-11-26 · draft

## Summary

A field of a record type that a `type` declaration declares may carry field annotations: constant
values of declared types, written after the field's type and default with `@`. The language gives
them no meaning. A library reads back the one it declared, typed, through `Reflect.annotation`. The
case this is for is a validation library in the style of pydantic: the record type is the schema,
and the constraints sit on its fields.

```ant
import "./validate.ant" V

type Signup = {
  name: string @V.Check{ minLength: 3, maxLength: 32, pattern: /^[a-z0-9_]+$/ },
  age: int @V.Check{ min: 13 },
  email: string @V.Email,
  plan: string = "free" @V.Check{ oneOf: ["free", "pro"] },
}

var signup = try V.decode(text, type(Signup))   // Err: "$.age: at least 13, got 9"
```

```
Reflect.annotation[A](t: Reflect.ErasedType, field: string, a: Reflect.Type[A]): A | null
```

The spec calls them "field annotations" in full, every time, since it already says "annotation" for
a written type ("`ant run` discards them", the `needs-annotation` code).

## Motivation

### Validation

A program that reads data from outside, a request body, a config file, a form, wants two things: a
value of its own type, and a guarantee that the value is in range. `typed-json.md` gives the first
through `Json.decode`. The second is a list of constraints on single fields: at least 13, between 3
and 32 characters, matching a pattern, one of a few strings. Constraints are many, irregular, and
different for every field, so no policy for a whole call covers them, the way `key: Json.snakeCase`
covers renames.

The two libraries people reach for take opposite sides:

- **Zod** writes the schema as a value and derives the type from it with `z.infer`. Ant cannot
  compute a type from a value, so in Ant this style means writing every type twice, as a schema and
  as a record type, and keeping the two in step by hand; or giving up static types for validated
  data.
- **pydantic** takes the type as the schema and attaches constraints to fields,
  `age: Annotated[int, Field(ge=13)]`. Ant has the first half already: `Reflect.shape` takes a
  record type apart. It has no place for the second.

### What works today, and where it stops

- **Named types with a hook.** `typed-json.md` lists a `validate` hook on a named type under Later.
  With it, `type Port = uint16` with `Port.validate` is pydantic's `PositiveInt` or `EmailStr`. That
  is the right tool for a constraint a program reuses. For a one-off one, length 3 to 32 or age at
  least 13, it needs a named type per combination, and every use site converts with `Age(…)`.
- **A record hook.** `Signup.validate` checks fields that must agree, such as a password and its
  confirmation. Written as plain code for single fields, `V.min(s.age, 13)` receives only the value,
  so it cannot name the field in its error unless it is handed the name as a string.
- **A table keyed by `Reflect.Decl` and field name.** Works today (prototyped in a TOML codec:
  rename and skip, about 35 lines plus an `opts` parameter on every internal function). Field names
  are strings, so a typo is caught at run time; a field rename in Ant leaves the table stale; the
  table lives away from the type, is about as long as the type for validation, and a library that
  exports a type must also export a table per format.

### Codecs

Three codecs written over Reflect (YAML, TOML, XML) found the same need on a smaller scale:

- **Renames** where one field differs from the call's policy: a key such as `connection-max` that is
  not an Ant identifier.
- **XML attribute or child element.** The XML codec used `type Attr = string` as a marker, which
  works only for strings and changes the field's type for the whole program.
- **Skipping a field** that is state, not data.
- **Per-field format choices:** an inline table, CDATA, base64 or hex `Bytes`, a date layout.

All of these belong to one field of one declaration and are not part of the field's type: `age` is
an `int` wherever the program uses it, and only the validator cares that it is at least 13.

## Prior art

- **Go struct tags**, `` Age int `json:"age" validate:"min=13"` ``. After the field, which is where
  this proposal puts annotations, but a string that each library parses at run time: `valdiate:` or
  `min=1e` compiles.
- **Java Bean Validation, C# DataAnnotations, Kotlin.** `@Min(13)`, `[Range(13, 120)]`: typed,
  checked annotation types with arguments, written before the member, read through reflection. The
  nearest to this design in what an annotation is.
- **Rust serde attributes**, `#[serde(rename = "max_tokens")]`. Before the field, checked by the
  derive macro at compile time, since serde generates code instead of reflecting.
- **pydantic**, `Annotated[int, Field(ge=13)]` and `Field(alias=…)`: metadata through the type
  position, in a language where that does not change type equality.
- **Zod**, `z.object({ age: z.number().int().min(13) })`: the schema as a value, the type inferred.

## The design

### Syntax

A field annotation is `@` followed by one of:

- a record literal of a named record type, `@V.Check{ min: 13 }`;
- a call of a named primitive type's constructor, `@Toml.Key("connection-max")`;
- a bare type name, `@V.Email`, which means `V.Email{}` and needs a record type whose fields all
  have defaults.

It comes after the field's type and, where there is one, its default. A field may carry several,
separated by spaces:

```ant
type Pool = {
  connectionMax: int @Toml.Key("connection-max") @V.Check{ min: 1 },
  idleTimeout: int = 30 @Toml.Key("idle-timeout"),
  private cache: Map[string, int] = Map{} @Toml.Skip,
}
```

The field reads as plain Ant first: name, type, and default line up on the left, and what only a
library reads trails. The annotation comes after the default and not between the type and the
default, because `port: int @V.Check{ min: 1 } = 8080` reads as part of the type, which an
annotation is not. `private` comes first, as for any field.

`@` is a new token and starts nothing else. After a type or a default expression it can only begin
an annotation, so the grammar keeps one token of lookahead. A line break before `@` is allowed, as
anywhere inside the braces of a record type. `ant fmt` writes no space after `@` and one before it,
keeps the author's choice of one line or several, and indents an annotation on a line of its own as
a continuation line (`fmt.md` 4.5), so the field's name stays where it was. It aligns nothing.

### Where

Only on fields of the record body of a `type` declaration: exactly where a field default may stand
(section 4, Field defaults), for the same reason. An anonymous record type has no declaration to
hold metadata, and its equality is structural; a nested record type or a union member has no name a
literal could give. A field annotation in a declaration file is a syntax error until a host needs
one, as `private` is in `private-fields.md`.

No annotations on types, functions, or parameters. On an alias, `Shape` and `Circle | Rect` are one
type (section 4), so an annotation would make one type behave two ways. A record or named primitive
declaration has one tag and could carry metadata, and uses exist: an XML element name, the tag value
of a union member, one rename policy for all of a type's fields. But a table keyed by `Reflect.Decl`
has no field name to misspell, so the case against tables does not apply there, and a library keeps
that table.

### Checked

The annotation's type is a named record type or named primitive type in scope where the record type
is declared. It takes no type parameters, and it is one a type expression accepts (section 4, Types
as values), so it reaches no interface: `ant run` infers nothing and a record keeps only its tag, so
`@Range{ min: 0, max: 10 }` with `type Range[T] = { min: T, max: T }` would have no type argument to
be read back at. An unknown name, an argument or field of the wrong type, and two annotations of one
type on one field are compile errors. So `@V.Chek{ min: 13 }` does not compile, which a Go string
tag would accept.

A library that wants one constraint to apply to both kinds of number writes the field as a union,
`min: int | float64 | null = null`, since `13` is never a float (section 2, Literals).

### Constant

A field annotation is a constant, defined by a grammar production rather than by a checker rule, so
that a violation is a syntax error in both commands, as a local type's default that names a variable
is. A constant is one of:

- an integer, float, or duration literal, an integer or float literal with `-` before it;
- a string literal without `${}`, a raw string, a regular expression literal;
- `true`, `false`, `null`;
- an array literal of constants;
- a record literal of a named type whose entries are constants;
- a call of a type name with one constant argument: a named primitive's constructor,
  `Toml.Key("x")`, or a numeric conversion, `uint16(8080)`.

It names no variable or constant (`Math.PI`), calls no function, and has no operator but the `-` of
a negative number, so it never mentions a type parameter of the declaration either. It means the
same thing for every instantiation of a generic type. Map and set literals are left out until a
library needs one (open question 2); an array covers a list of allowed values, which matters because
two annotations of one type on one field are an error.

The rule covers what is written after `@`, not the defaults of the annotation's type. `@V.Email` is
`V.Email{}`, and a literal fills an omitted field from its default, which may read a variable or
call a function. That runs when the annotation is read, as `Reflect.makeRecord` runs defaults, and a
default that panics does so inside `Reflect.annotation`.

### No meaning

A field annotation does not change the type, its tag, equality, `String.from`, `Json.stringify`,
assignability, or a literal. Adding or removing one is invisible to every program that does not
reflect. In particular a record literal is never validated: `Signup{ name: "x", age: 5, ... }`
builds the record, and only a library that reads the annotations checks them.

So validation happens at the boundary, where data is parsed or where a program asks:
`V.decode(text, type(Signup))`, or `V.check(type(Signup), s)` on a value built in code. That is
Zod's guarantee, a value that came through `parse` is valid, and weaker than pydantic's, where
building the object validates it. It is the one that fits a language where a record literal runs no
code but its defaults.

### Reflection

```
Reflect.annotation[A](t: Reflect.ErasedType, field: string, a: Reflect.Type[A]): A | null
```

`Reflect.annotation` gives the annotation of type `A` on the field named `field` of the record type
`t`, or `null` when the field carries none of that type. It panics if `t` is not a named record type
or does not list the field, as `Reflect.field` does. `t` may be any instantiation of a generic type;
since an annotation cannot mention a type parameter, all instantiations read the same one.

Each call evaluates that one annotation afresh in the declaration's scope, as a default is evaluated
per literal, and nothing else: the field's other annotations are not evaluated. A record annotation
is mutable like any record, and a caller that changes one changes only its copy. A library that
reads the same field many times, a validator called once per value, caches by `Reflect.Decl`.

A lookup, not a list on `Reflect.Field`, for three reasons. `Reflect.shape` stays a function of the
type alone and does not evaluate code. A library reads only its own annotation types, so it never
needs to list a field's annotations, and the result comes back typed, with no loop over boxes and no
`Reflect.to`. And "two annotations of one type on one field" is exactly the uniqueness a lookup by
type needs.

The library side of the validator above:

```ant
// validate.ant
export type Check = {
  min: int | float64 | null = null,
  max: int | float64 | null = null,
  minLength: int | null = null,
  maxLength: int | null = null,
  pattern: Regex.Pattern | null = null,
  oneOf: Array[string] = [],
}

export type Email = {}

func rules(t: Reflect.ErasedType, field: string): Check {
  Reflect.annotation(t, field, type(Check)) ?? Check{}
}
```

A library should panic on its own annotation where it cannot apply, such as
`@V.Check{ minLength: 3 }` on an `int` field or `@Xml.Attribute` on a record-typed one, with a
message naming the declaration and field.

### Unchecked runs

`ant run` checks nothing (section 4, Checking and running), so each compile error has a run-time
side:

- **An unknown name** panics where the annotation is evaluated, inside `Reflect.annotation`, as a
  `type(T)` with an unknown name does. A name that is bound but is not a type panics there too.
- **An argument or field of the wrong type** is trusted, as `Reflect.valueOf` trusts its value.
- **Two annotations of one type** are the checker's alone; `Reflect.annotation` returns the first
  written.

## Fit with typed-json.md

The mechanism is here; the policy stays there. `typed-json.md` keeps its `key` call option as the
policy of a call, so a record of thirty fields for a snake_case API stays one option and not thirty
annotations, and declares `Json.Key` as an ordinary prelude type that overrides the policy for one
field. `Json.Skip` may follow. `tag` stays a call option for a root union and may also be a field
annotation `@Json.Tag("kind")`. Every other format walks the type with `Reflect.Value` and declares
and reads its own annotation types, as `Toml.Key` above.

`Json.stringify` walks values and reads no annotation: with `@Json.Key("max_tokens")` it still
writes `maxTokens`, where Go's `json.Marshal` honors the tag. Only a typed encoder renames. So this
proposal lands with or after `typed-json.md`, never before it: field annotations with no reader in
the prelude would promise Go programmers a rename that `Json.stringify` does not do.

## Types you do not own

A field annotation can only be written by the type's author. A library may also take an override
table keyed by `Reflect.Decl` and field name, which works today with no language change, and should
check the table's field names against `Reflect.shape` on first use, so that a typo fails on the
first call rather than silently. That is a library convention, not part of this proposal.

## Alternatives

- **Annotations before the field**, `@V.Check{ min: 13 } age: int`, as in Java and Rust. The first
  draft of this proposal. The field's name moves right or down, behind metadata only a library
  reads; on a line of its own an annotation turns the field into a continuation line under `fmt.md`
  4.5; and it raises the question of whether `private` or `@` comes first.
- **A checked block after the declaration**,
  `type Signup = { … } with { age: V.Check{ min: 13 }, … }`. The checker could check the field
  names, and the type stays bare. But every field name is written twice, and the constraints sit
  apart from their fields again.
- **Validation as code** in `Signup.validate`. Needs no syntax, and the checker checks `s.age`, but
  a helper gets only the value and cannot name the field in its error.
- **Named constrained types** for everything, `type Age = int` with a hook. See Motivation: right
  for reused constraints, one type per combination for the rest.
- **Per-library tables keyed by `Reflect.Decl` and field name.** See Motivation and Types you do not
  own.
- **Zod-style schema values.** Ant cannot derive a type from a value, so the type is written twice.
- **Go/Odin string tags.** Smallest syntax, but parsed by each library at run time and unchecked.
- **Metadata on types,** Python's `Annotated[int, Field(ge=13)]`. Either it changes type equality,
  or two equal types behave differently; both break section 4.
- **Marker types,** the status quo `type Attr = string`. Changes the field's type everywhere, covers
  one choice per field, and does not compose with a rename.
- **Per-type hooks,** Swift `CodingKeys`, Zig declarations on the struct. Checked and attached to
  the type, but a table of names per type is a worse spelling of the same metadata.

## Costs

- A token, one grammar position, the constant production, and one checker rule set mirroring field
  defaults.
- The interpreter keeps each annotation's syntax with the field, as it keeps defaults, and evaluates
  it in `Reflect.annotation`. `Reflect.shape` and `Reflect.Field` do not change.
- `ant fmt`, the LSP, the highlighting grammars of four editor repositories, and `ant doc` learn the
  form.

## Spec changes

- Section 2: the `@` token, and the constant production.
- Section 4: a "Field annotations" subsection after Field defaults.
- Section 10, Declaration files: a field annotation there is a syntax error.
- Section 11, Reflect: `Reflect.annotation`, and a line in the invariant for the value it returns.
- Section 12: how a field annotation is evaluated, and its rows in Runtime panics.
- `check.md`: the code each new error takes.
- `fmt.md`: spacing around `@`, and an annotation on a line of its own as a continuation line.

## Open questions

1. **Caching.** Each `Reflect.annotation` evaluates afresh, as a default does, and a library caches
   by `Reflect.Decl`. Should the interpreter instead evaluate each annotation once per run and hand
   out copies? Leaning: afresh, as written, until a measured validator says otherwise.
2. **Map and set literals** in a constant. Leaning: out until a library needs one.
3. **Listing.** A tool such as a schema generator that shows every annotation on a field, of types
   it does not know, cannot use a lookup by type. Leaning: no listing; `ant doc` reads the syntax,
   and a generator reads the annotation types it supports.
4. **A prototype first.** This proposal waits on `typed-json.md`. Before it is accepted, a
   validation library written today with named types, hooks, and a checked `Reflect.Decl` table
   shows how much a realistic signup or config type gains from annotations.
