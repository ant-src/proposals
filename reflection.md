# Proposal: reflection

2026-11-03 · Anton

## Summary

Add type values and a `Reflect` namespace, so that a program can learn a type at run time and read
and build values of it. This is the base that typed decoders and encoders for JSON, YAML, XML and
other formats are written on. This proposal specifies no format.

- **`type T` is an expression.** In expression position, `type` followed by a type evaluates to a
  run-time description of that type. The checker gives it the static type `Type[T]`.
- **`Reflect.info` describes a type one level at a time** as a closed union of records: a number of
  some width, an array of some element type, a record with these fields, a union of these members.
- **`Reflect.Value` pairs a value with its type**, so code that learns a type at run time can read
  and build values of it. Building takes parts as `Reflect.Value`s and checks their types, not
  their contents.
- **`Reflect.as` brings a value back into static types.** It succeeds when the checker would accept
  the same step: assignment, or narrowing with `is`. There is no top type and no unchecked cast.
- **The interpreter keeps a little more.** A record tag also keeps the written type of each field,
  and an alias tag keeps its body. Both are resolved on demand in the scope of the declaration, as
  field defaults already are. No other evaluation rule reads them, and `ant run` still checks no
  type.

With this in place, a later proposal can add `Json.decode` as library code:

```ant
type Server = { host: string, port: int = 8080, tags: Array[string] = [] }
type Config = { name: string, servers: Array[Server], debug: bool = false }

var config = try Json.decode(text, type Config)   // Result[Config, Error]; omitted fields take their defaults
```

## Motivation

`stdlib.md` says of JSON: "A decoder into records is a function per type: narrow the tree with `is`,
build the record, and propagate errors with `try`, passing the path along." For the `Config` above
that is two functions and about forty lines. The field defaults get written a second time, every
new field touches the decoder, and a typo in a key string compiles. YAML, XML, TOML, environment
variables, command-line flags, query strings and database rows each need the same functions again:
N formats times M types, all written by hand.

The library cannot write those functions for you. `func decode[T](text: string): T` has nothing to
work with at run time, because section 1 says "No evaluation rule consults a written type", and the
interpreter keeps only tags and field names. An empty array has no element type, a `Box[int]` record
does not remember `int`, and a field declared `Time.Instant | null` is only a name.

Goals:

1. `Json.decode(text, type Config)` returns `Result[Config, Error]`, and a checked program gets a
   real `Config`.
2. Anyone can write a format in Ant. The built-in JSON decoder gets no powers that a YAML or TOML
   library written by a user lacks.
3. Field defaults apply when the input leaves a field out, with no second copy of them.
4. Checked programs stay sound. There is no top type, no unchecked cast, and no way to get a value
   that its static type does not describe.
5. `ant run` stays an interpreter of values. Ordinary evaluation consults no type.

## Prior art

| Approach                                 | Examples                                                                                                                  | What Ant takes                                                                                                                                                                                                                         |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Run-time types on every value            | Go `reflect`, Java and Kotlin reflection (Jackson, Gson), C# `System.Reflection`                                          | The shape of the API: a type value, a kind switch, a value handle (`reflect.Value`) that reads and builds. Not the cost: Ant's values keep only their kind and tag, and types are not attached to them                                 |
| Type descriptions as a tagged union      | Zig `@typeInfo`, which returns `std.builtin.Type`                                                                         | `Reflect.Info`: one closed union that a program narrows with `is`, one level at a time                                                                                                                                                 |
| Annotations kept and resolved lazily     | Python `typing.get_type_hints`, PEP 649 and PEP 749 (deferred annotations, Python 3.14); pydantic and msgspec built on it | The storage model: keep the written field types, resolve them in the declaration's scope only when asked. Python made the same change so that annotations cost nothing until read                                                     |
| Typed dynamic box                        | Haskell `Data.Dynamic` with `Typeable` (`fromDynamic` returns `Maybe a`); Rust `Box<dyn Any>` with `downcast`             | `Reflect.Value` and `Reflect.as`: a value of statically unknown type can only come back through a checked step, so there is no top type to assign to                                                                                   |
| Schemas as values, in an erased language | TypeScript zod, io-ts, Effect Schema; Elm `Json.Decode`; Gleam decoders                                                   | The confirmation that an erased language needs a value at run time. Ant gets that value from the declaration, so the schema is not written twice                                                                                       |
| Compile-time code generation             | Rust serde derive, Swift `Codable`, kotlinx.serialization, C# `System.Text.Json` source generators, OCaml `ppx_deriving`  | The data-model idea from serde, as a later library choice (see How formats use it). Not the mechanism: Ant has no build step and no macros, and the checker never generates code                                                       |
| Type-directed transforms                 | TypeScript typia and ts-runtime-checks, which compile a type into a validator                                             | Nothing. They need a compiler pass that `ant run` does not have                                                                                                                                                                        |
| Struct tags and attributes               | Go `json:"name,omitempty"`, Rust `#[serde(rename)]`, Java `@JsonProperty`, C# `[JsonPropertyName]`                        | Not yet. This proposal leaves field options to the formats (open question 1)                                                                                                                                                           |

Three lessons shape the proposal:

- **An erased language needs the type as a value, passed explicitly.** TypeScript's ecosystem shows
  this clearly: everything that validates at run time takes a schema object. Reified generics, where
  the runtime passes `T` behind the scenes, would make every generic call carry types and would make
  `ant run` depend on the checker. Ant keeps the value explicit, `type Config`, and builds it from
  the declaration that already exists.
- **Soundness comes from types, not from inspecting contents.** A deep check that an array holds
  only `int`s does not make it an `Array[int]`, because another name can still push a string into
  it. That is the reason section 4 makes type arguments invariant. `Reflect.Value` therefore carries
  a type, and every step that changes the type is one the checker already allows. Haskell's
  `Typeable` and Rust's `Any` work the same way.
- **Formats are libraries.** Go's `encoding/json` and `encoding/xml` are ordinary packages built on
  `reflect`. Go's `encoding/json/v2` (an experiment in Go 1.25) was written without any new language
  support. Ant should be able to do the same: if reflection is enough for JSON, it is enough for
  every format a user writes.

### Where languages are going

1. **Annotations stay, but cost nothing until they are read.** Python 3.14 evaluates annotations
   lazily (PEP 649), so a module pays for them only when a library such as pydantic asks. Ant's field
   defaults already work this way, and field types follow them.
2. **Reflection is moving to compile time where there is a compiler.** .NET's source generators and
   Swift's `Codable` replace run-time reflection so that trimmed and ahead-of-time builds stay small.
   Ant has no such build, and its interpreter already has every declaration in memory, so run-time
   reflection costs it little. A future compiler could still specialize a decoder for a known type
   without changing anything this proposal specifies.
3. **Typed boundaries replace `any`.** Go's `any` plus a type assertion is the old model. TypeScript's
   `unknown`, Haskell's `Dynamic` and Rust's `Any` all make the value useless until a checked step
   recovers its type. Ant has no top type at all (section 4, Assignability), and `Reflect.Value` is
   not one: nothing is assignable to it.

### Fit with Ant

| Principle or decision                   | Fit     | Why                                                                                                                                                                    |
| --------------------------------------- | ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Types are advisory                      | Partial | The interpreter keeps written field types and alias bodies. Only `type` and `Reflect` read them, and nothing is checked during ordinary evaluation. The principle's wording changes |
| No top type                             | Strong  | `Reflect.Value` is a box that is entered explicitly and left through a checked step, not a type that everything is assignable to                                       |
| Unions are distinguishable at run time  | Strong  | This is what makes `Reflect.member` and `Reflect.as` total. Every value in a union belongs to exactly one member, and `is` can tell which one                          |
| Errors are values, panics are bugs      | Strong  | Reflection panics only on a decoder bug, such as building a record with a field of the wrong type. Bad input is the decoder's `Result`                                 |
| Field defaults                          | Strong  | `Reflect.record` fills in omitted fields exactly as a literal does, so "missing in the input" and "omitted in the literal" mean the same thing                          |
| Minimal syntax                          | Good    | One new expression, `type T`, using a keyword that already exists and the type grammar that already exists. The parser keeps one token of lookahead                  |
| One obvious way                         | Good    | One way to name a type at run time and one value handle. The hand-written decoder of `stdlib.md` stays possible but is no longer the only way                         |
| Interpreter stays simple                | Good    | Tags gain a pointer to syntax the parser already built. Descriptors are resolved lazily. No existing evaluation path changes                                          |
| Library close to Go                     | Good    | The design follows `reflect`: `Type`, `Value`, kinds, `New`/`MakeSlice`/`MakeMap`. What Ant leaves out is mutation through reflection and calling functions reflectively |

## The proposal

### Type values

In expression position, the keyword `type` followed by a type is a **type expression**. Its value is
a description of that type:

```ant
var t = type Config
var list = type Array[Config]
var maybe = type Time.Instant | null
var handler = type func(Http.Request): Http.Reply
```

- **Syntax.** `type` takes the whole type to its right, as `return` takes the whole expression to
  its right and as a function type's result extends as far as it can. So `type int | null` is the
  union, and a type value used in a pipe is parenthesized: `(type Config) | describe()`. The type is
  written in the type grammar of section 4, and every name in it resolves as it would in an
  annotation at that point. At the start of a statement `type` still begins a declaration, so the
  parser needs only one token of lookahead. A lambda that returns a type value writes
  `{ (type Config) }`.
- **Static type.** `type T` has the type `Type[T]`, a prelude type that the checker knows. `T` must
  be a valid type under every rule of section 4, the union rules included. `Type[T]` is invariant in
  `T`, because a `Type[T]` is used both to build `T` values and to accept them.
- **Type parameters have no value.** `T` may not mention a type parameter of an enclosing function
  or type. This is a compile error that says to take a `Type[T]` parameter, since the interpreter
  never knows what a type parameter stands for. A generic helper receives its type the same way it
  receives any other argument:

```ant
func fetchJson[T](url: string, t: Type[T]): Result[T, Error] {
  var response = try Http.get(url)
  Json.decode(response.body, t)
}

var users = try fetchJson("https://example.com/users", type Array[User])
```

- **Evaluation.** Evaluation resolves every name in the type: a prelude type name, a tag in scope, a
  namespace path such as `Time.Instant` or `Geometry.Point`. It also applies type arguments. It
  checks nothing the checker checks, and it reads no field type until something asks for one (see
  What the interpreter keeps). A name that resolves to no type panics, as an unknown name in `is`
  does.
- **Equality and rendering.** Two type values are `==` when they describe the same type. Aliases are
  expanded, unions are compared as sets of members, records are compared by tag and type arguments,
  and anonymous record types by their fields. A pair of types already under comparison counts as
  equal, so recursive aliases compare in finite time, as cyclic values do. `==` is consistent with
  hashing, so a type value can be a map key, for example to cache a decoding plan per type.
  `String.from` writes the type in the notation of section 4 with aliases expanded:
  `Array[Config]`, `Time.Instant | null`.

The prelude adds four functions on `Type` for composing a type from a type parameter. Only the
built-in constructors are covered, because a generic record would need a type operator that Ant
does not have:

```
Type.array[T](t: Type[T]): Type[Array[T]]
Type.map[K, V](k: Type[K], v: Type[V]): Type[Map[K, V]]
Type.set[T](t: Type[T]): Type[Set[T]]
Type.nullable[T](t: Type[T]): Type[T | null]
```

### What the interpreter keeps

Section 4 says a record tag keeps "the names of its fields in declaration order and the default of
each field that has one; the rest of the body is discarded." Under this proposal it also keeps the
written type of each field, and an alias tag keeps its body. Both are the syntax the parser has
already built, kept together with the scope of the declaration, which the tag already holds for its
defaults. They are resolved the first time `Reflect` asks for them, in that scope, with the type
arguments of the instantiation being described. A local type resolves in its block's types, as its
defaults do (section 4, Type declarations).

Nothing else reads them. A record literal, a field read, `==`, `is`, a call and every other rule of
section 12 behave exactly as before, and a program that never evaluates `type` pays only for the
memory of syntax it has already parsed.

### Describing a type

`Reflect.Type` is a type value whose static type is unknown. `Type[T]` is assignable to
`Reflect.Type` for every `T`, as a named record type is assignable to an anonymous one that it fits.
The value is the same and only its static type is wider. There is no step back: a program that
needs to know it holds a particular type compares, `r == type Config`, and then uses `type Config`
itself.

`Reflect.info` describes one level of a type:

```
type Reflect.Info = Reflect.Bool | Reflect.Number | Reflect.String | Reflect.Null | Reflect.Void
                  | Reflect.Never | Reflect.Bytes | Reflect.Array | Reflect.Map | Reflect.Set
                  | Reflect.Record | Reflect.Tagged | Reflect.Union | Reflect.Func | Reflect.Opaque

type Reflect.Bool = {}
type Reflect.Number = { integer: bool, signed: bool, bits: int }    // int8 … uint64, float32, float64
type Reflect.String = {}
type Reflect.Null = {}
type Reflect.Void = {}
type Reflect.Never = {}
type Reflect.Bytes = {}
type Reflect.Array = { element: Reflect.Type }
type Reflect.Map = { key: Reflect.Type, value: Reflect.Type }
type Reflect.Set = { element: Reflect.Type }
type Reflect.Record = { name: string | null, args: Array[Reflect.Type], fields: Array[Reflect.Field] }
type Reflect.Field = { name: string, type: Reflect.Type, hasDefault: bool }
type Reflect.Tagged = { name: string, base: Reflect.Type }          // a named primitive type
type Reflect.Union = { members: Array[Reflect.Type] }
type Reflect.Func = { params: Array[Reflect.Type], result: Reflect.Type }
type Reflect.Opaque = { name: string }                              // Html, Regex.Pattern, the time, sync, random and resource types

Reflect.info(t: Reflect.Type): Reflect.Info
Reflect.assignable(from: Reflect.Type, to: Reflect.Type): bool      // the rule of section 4, Assignability
```

- An alias is described by its body, so `Reflect.info(type Shape)` is a `Reflect.Union` and
  `Reflect.info(type Result[int, Error])` is the union of `Ok[int]` and `Err[Error]`. `int` and
  `byte` are `int64` and `uint8`.
- A record's `fields` are in declaration order. Their types are given at the record's type
  arguments, so the field `value: T` of `Box[int]` is described as `int`. `name` is the declared
  name, or `null` for an anonymous record type, whose `fields` are the fields the type lists.
- A union's `members` are flattened and deduplicated, in the order written.
- Children are `Reflect.Type` handles, which are described only when `Reflect.info` is called on
  them, so a recursive type such as `Json.Value` or `Tree` has a finite description at every level.
- `Reflect.Info` is a closed union, like `Html.Node`. A kind added to the language later will break
  exhaustive `is` chains over it, on purpose.

### Values of a run-time type

`Reflect.Value` is an opaque prelude type: a value together with a `Reflect.Type` that describes it.
The invariant, which every function below keeps, is that **in a checked program the value belongs
to the type**.

```
Reflect.valueOf[T](t: Type[T], v: T): Reflect.Value       // v with the type t
Reflect.typeOf(v: Reflect.Value): Reflect.Type
Reflect.as[T](v: Reflect.Value, t: Type[T]): T | null     // the value as a T, when the checker would allow the step
```

`valueOf` takes the type first, so the type fixes `T` and the value is then checked against it:
`Reflect.valueOf(type Shape, circle)` checks, and the `Value` has the type `Shape`. With the value
first, `circle` would fix `T` to `Circle` and the invariant `Type[T]` would reject `type Shape`.

`Reflect.as(v, t)` returns the value when either of these holds, and `null` otherwise:

- the type of `v` is assignable to `t`, by the rule of section 4, Assignability; or
- the type of `v` is a union, or an anonymous record type, and an `is` test on the value narrows it
  to a type assignable to `t`, exactly as `if x is Circle { x }` would narrow it in the checker
  (section 5, `is`).

So `as` never inspects the contents of a container. `Reflect.valueOf(type Array[int], [])` cannot
come back out as an `Array[string]`, although the empty array would pass any check of its elements.
That is the invariance argument of section 4 applied at run time. A `T` that includes `null` cannot
tell a held `null` from a failed step. `Reflect.info` tells the two apart, as `Map.has` does for a
map.

**Reading.** Each function returns new `Value`s that hold the same element, field or payload values
as the original, without copying them. Each new `Value` is typed by what the type of `v` says about
that part.

```
Reflect.member(v: Reflect.Value): Reflect.Value            // for a union type, v at the member its value belongs to; v otherwise
Reflect.unwrap(v: Reflect.Value): Reflect.Value            // a named primitive's payload, at its base type; panics on another type
Reflect.fields(v: Reflect.Value): Array[Reflect.Value]     // a record's fields, in the order its type lists them; panics on another type
Reflect.elements(v: Reflect.Value): Array[Reflect.Value]   // an array's elements or a set's members, in order; panics on another type
Reflect.entries(v: Reflect.Value): Array[Entry[Reflect.Value, Reflect.Value]]   // a map's entries, in order; panics on another type
```

`member` is total because of the union rule of section 4: every value of a union carries the
discriminator of exactly one member. An unchecked program can build a union that breaks that rule,
and then `member` panics when the value fits two members. A leaf is read with `as`:
`Reflect.as(v, type string)`. `String.from(v)` renders the held value, which is how an encoder
writes a number of any width.

**Building.** Each function returns a new `Value` of the type `t`. A part whose type is not
assignable to the slot it fills, or a `t` of the wrong kind, panics. Those are bugs in the code that
builds, not errors in its input.

```
Reflect.number(t: Reflect.Type, text: string): Reflect.Value | null   // the number text spells, as t's kind; null when that kind cannot hold it exactly
Reflect.array(t: Reflect.Type, elements: Array[Reflect.Value]): Reflect.Value
Reflect.set(t: Reflect.Type, members: Array[Reflect.Value]): Reflect.Value
Reflect.map(t: Reflect.Type, entries: Array[Entry[Reflect.Value, Reflect.Value]]): Reflect.Value
Reflect.record(t: Reflect.Type, fields: Map[string, Reflect.Value]): Reflect.Value   // omitted fields take their defaults
Reflect.wrap(t: Reflect.Type, payload: Reflect.Value): Reflect.Value  // a named primitive over the payload
```

- `number` reads a JSON number: an optional `-`, digits, an optional fraction and an optional
  exponent. An integer kind takes the exact decimal value when it is an integer in range, so `"1e3"`
  and `"1.0"` give an `int` and `"1.5"` gives `null`. A float kind rounds to the nearest value, and
  gives `null` beyond its range, as `String.toFloat` does. Other spellings, such as hexadecimal or
  `_` separators, are the format's job to rewrite first.
- `record` evaluates as a record literal of section 12 does: an unknown name panics, a field left
  out takes its default, evaluated afresh in the scope of the declaration, and a field left out with
  no default panics. A decoder looks at `Reflect.Field.hasDefault` first and reports the missing
  field as an error. A `t` that is an anonymous record type panics, since every record value has a
  name (section 4, Records).
- `array`, `set`, `map` and `record` build new containers. Nothing in `Reflect` writes to a value
  that already exists.
- A leaf of a known type is `Reflect.valueOf(type string, s)`. A slot of a union type takes a part
  of any member type, since the member is assignable to the union.

### Example: a JSON decoder

This is the sketch a later `Json` proposal would start from, over today's `Json.Value`. It shows that
the API is enough to write the decoder, not how JSON should handle union representation or names
(see How formats use it).

```ant
export func decode[T](text: string, t: Type[T]): Result[T, Error] {
  var tree = try Json.parse(text)
  var v = try decodeAt(tree, t, "$")
  Result.ok(Reflect.as(v, t) ?? panic("decoder built a value of the wrong type"))
}

func decodeAt(json: Json.Value, t: Reflect.Type, path: string): Result[Reflect.Value, Error] {
  var info = Reflect.info(t)
  if info is Reflect.Union {
    for m in info.members {
      var r = decodeAt(json, m, path)
      if r is Ok { return r }
    }
  } else if info is Reflect.Null {
    if json == null { return Result.ok(Reflect.valueOf(type null, null)) }
  } else if info is Reflect.Bool {
    if json is bool { return Result.ok(Reflect.valueOf(type bool, json)) }
  } else if info is Reflect.String {
    if json is string { return Result.ok(Reflect.valueOf(type string, json)) }
  } else if info is Reflect.Number {
    if json is float64 {
      var n = Reflect.number(t, String.from(json))
      if n != null { return Result.ok(n) }
    }
  } else if info is Reflect.Tagged {
    return Result.ok(Reflect.wrap(t, try decodeAt(json, info.base, path)))
  } else if info is Reflect.Array {
    if json is Array {
      var out: Array[Reflect.Value] = []
      for e in Array.entries(json) {
        Array.push(out, try decodeAt(e.value, info.element, "${path}[${e.index}]"))
      }
      return Result.ok(Reflect.array(t, out))
    }
  } else if info is Reflect.Record {
    if json is Map && info.name != null {
      var fields: Map[string, Reflect.Value] = Map{}
      for f in info.fields {
        if Map.has(json, f.name) {
          fields[f.name] = try decodeAt(json[f.name], f.type, "${path}.${f.name}")
        } else if !f.hasDefault {
          return Result.err(Error{ message: "${path}: missing field ${f.name}" })
        }
      }
      return Result.ok(Reflect.record(t, fields))
    }
  }
  Result.err(Error{ message: "${path}: expected ${t}" })
}
```

The matching encoder reads instead of building:

```ant
func encodeAt(v: Reflect.Value): string {
  var info = Reflect.info(Reflect.typeOf(v))
  if info is Reflect.Union { encodeAt(Reflect.member(v)) }
  else if info is Reflect.Tagged { encodeAt(Reflect.unwrap(v)) }
  else if info is Reflect.Record {
    var parts = Array.map(Array.entries(Reflect.fields(v)), {
      "${Json.stringify(info.fields[x.index].name)}:${encodeAt(x.value)}"
    })
    "{${String.join(parts, ",")}}"
  } else if info is Reflect.Array {
    "[${Reflect.elements(v) | Array.map(encodeAt) | String.join(",")}]"
  } else if info is Reflect.String { Json.stringify(Reflect.as(v, type string) ?? "") }
  else { String.from(v) }            // numbers, bool, null; a real encoder handles every case
}
```

### Errors and panics

Reflection follows section 8. Input a decoder cannot accept is the decoder's `Result`, and the
decoder decides that by looking at `Reflect.info` before it builds anything. A `Reflect` function
panics only when its caller has a bug: building with a part of the wrong type, `fields` on a value
that is not a record, `record` without a required field. Each of these is a value-dependent panic
in section 12's last column, like an index out of range. A checked program can reach them, but only
through a bug in reflective code. It cannot reach them through a type error in ordinary code.

## How formats use it

This proposal stops at reflection. The format proposals that follow it decide:

- **Names.** A field named `userId` and a key `user_id`. The simplest form is a naming policy given
  per call. Per-field names need either an option map keyed by `Reflect.Type` or field annotations
  (open question 1).
- **Unions.** `Circle | Rect` has no single JSON form. Untagged (try each member in order, as the
  sketch does), internally tagged (`"type": "Circle"`, from `Reflect.Record.name`) and externally
  tagged (`{"Circle": {...}}`) can all be built from `info` and `member`. Serde offers all three;
  JSON should choose a default.
- **Exact numbers.** `Json.Value` holds numbers as `float64`, so an `int64` above 2^53 loses digits
  before `Reflect.number` sees it. A typed decoder should read the number text straight from the
  input, which `Reflect.number` already accepts.
- **Time values, `Bytes` and other opaque types.** A format decides their text: an RFC 9557 string
  for a time value, as `Json.stringify` writes today, and base64 or a refusal for `Bytes`. The
  format recognizes them with `t == type Time.Instant`.
- **XML.** XML needs to know whether a field is an attribute, an element or text. This is the
  strongest case for field annotations. Without them, an XML library takes options per call,
  keyed by type and field name.
- **A shared data model.** One design is a written-out union `Data`, which every format parses into
  and which one generic function decodes into any type. Serde has shown that this gives N + M code
  instead of N × M. It can be written in Ant on top of this proposal, and the JSON proposal should
  decide whether the standard library provides it.

## Spec changes

- **Section 1, Principles.** "Types are advisory" becomes: "`ant check` reads annotations and
  `ant run` checks none of them. The interpreter keeps a type's name, as a tag, a record's field
  names in order, and the written types of fields and of alias bodies, which only a `type`
  expression and the `Reflect` functions read. No other evaluation rule consults a written type."
- **Section 4.** Checking and running, and Type declarations: what a tag keeps. Kinds: a row for
  `Type[T]` and `Reflect.Type` (kind `type`, no `is`, `==` as above, rendered as the type) and one
  for `Reflect.Value` (no `is`, `==` on the type and the value, rendered as its value). Both may
  appear at most once in a union, like the sync types. Assignability: `Type[T]` is assignable to
  `Reflect.Type`. A new subsection "Type values" with the syntax, the static type and the
  type-parameter rule.
- **Section 5.** The type expression, its precedence, and the statement-start rule.
- **Section 11.** `Type` and its four functions in the prelude. A new `Reflect` section with the
  listing, the invariant, and the rules for `as`, `member`, `number` and `record`.
- **Section 12.** Values: the tag kind gains the written field types or the alias body, and new
  kinds are added for type values and `Reflect.Value`. Expressions: how `type T` is evaluated.
  Runtime panics: a type expression naming something that is not a type, in the checker's column,
  and the `Reflect` misuse cases, in the last column.
- **`stdlib.md`, Json.** "Typed decoding through reflection is deferred" points here until the JSON
  proposal replaces it. `Json.stringify` already panics on "a type", so it needs no change.
- **`fmt.md`, `lsp.md`.** `ant fmt` formats a type expression as it formats an annotation. Hover
  shows `Type[Config]`, and completion after `type` offers types.

## Implementation

- **Parser.** In expression position, `type` parses a type with the existing type grammar and
  produces a node that holds it. The record and alias declarations keep the type syntax they
  already parse instead of discarding it.
- **Interpreter.** A tag gets the field type syntax next to `Defaults`, which already carries the
  declaration's scope. A type value is a descriptor node: a primitive kind, a container with child
  descriptors, a tag with type arguments, a union, a function, or an unresolved alias that expands
  when `info` asks. Descriptors can be hash-consed so that `==` and `assignable` on equal types are
  pointer comparisons and decoding stays linear in the input. `Reflect.Value` is a pair. The
  builders reuse the record-literal path, so defaults run exactly as in a literal.
- **Checker.** `Type[T]` and `Reflect.Type` as types, synthesis of `Type[T]` for a type expression,
  the type-parameter rule, and the one assignability rule. `Reflect.as`, `valueOf` and the `Type`
  functions are ordinary generic signatures in `prelude.d.ant`.
- **JavaScript backend.** The same descriptors and the same pair. Nothing depends on JavaScript's
  own reflection.
- **Tests.** Every `Info` case, generic records at their arguments, local types, recursive aliases
  (`Json.Value`, `Tree`), `==` and hashing on equal types written differently, `as` succeeding and
  failing for each rule (assignability, narrowing, invariance of containers, the empty-array case),
  `member` on every union kind, defaults through `record`, panics on wrong parts, and the JSON sketch
  round-tripping the `Config` above. Benchmarks: decoding a large array of records, against the
  hand-written decoder of today.

## Non-goals

- **Writes through reflection.** No setting a field or pushing an element through a `Reflect.Value`.
  Decoders build new values, and field defaults cover what Go's "decode into a filled struct" is
  used for.
- **Calling functions reflectively,** listing a type's functions, or listing a module's
  declarations.
- **The type of an arbitrary value.** A value carries its kind and tag, not its type. `[]` has no
  element type to report, so there is no `Reflect.typeOf(value)`. The type always comes from a
  `type` expression.
- **Run-time type checking of ordinary code.** `ant run` still checks no annotation. Only the
  `Reflect` builders compare types, and only types to types.
- **Hooks by name.** A type does not customize its own encoding through a function declared on it,
  just as it cannot customize `String.from` (section 4, Generics).
- **Doc comments at run time.** The interpreter does not keep them (open question 4).

## Alternatives considered

- **Code generation** (`ant gen` writes one decoder function per type, like serde derive or Go's
  easyjson). It needs no runtime change and the generated code is fully checked. But Ant has no
  build step ("a program is the files it imports"), generated files go stale, and every format would
  need its own generator. A future compiler can do the same specialization behind this API instead.
- **Reified generics.** The checker would pass `T` to `decode[T]` implicitly. That breaks the rule
  that `ant run` discards annotations, makes unchecked programs behave differently from checked
  ones, and puts a cost on every generic call.
- **The tag as the type value.** `Json.decode(text, Config)` with `Config` evaluating to its tag. A
  tag cannot express `Array[Config]`, `Config | null` or `Box[int]`, and aliases have no runtime
  identity (section 5, `is`).
- **A dynamic value without a type** (Go's `reflect.Value` over `any`). Without a type to carry,
  getting a value back into static types needs a check of its contents, which is unsound for
  mutable containers, as shown above. Ant would also need a top type that it deliberately does not
  have.
- **Schema combinators only** (Elm, zod). This needs no language change, and it is what `stdlib.md`
  recommends today. The schema repeats the type declaration, and unlike zod, Ant cannot compute the
  static type from the schema, so the two can drift apart.
- **Decoding into a prototype value.** A decoder that copies the shape of an example value. An empty
  array has no element type and a field holding `null` does not say what else it could hold.

## Open questions

1. **Field annotations.** Per-field names, `omitempty`, XML attributes and flattening all want data
   on a field. The options are: none, with the format taking a per-call option map keyed by
   `Reflect.Type` and field name; a narrow syntax such as `userId: string @name("user_id")`, kept by
   the tag and exposed as `Reflect.Field.annotations`; or Go-style string tags, which are known to
   be fragile. This draft proposes none, until the JSON and XML proposals show what they need.
2. **Encoding without a type.** `Json.stringify` walks a value without its type today, and an encoder
   built on this proposal needs the type as well: `encode(type Config, config)`. A
   `Reflect.dynamic(v)` could give a `Value` typed by what the value itself shows: exact for
   primitives, tagged primitives and non-generic records, and unknown element types elsewhere. That
   would need an "unknown" type in `Reflect.Info`.
3. **Alias names.** `info` expands aliases, so a JSON Schema generator cannot name `Json.Value` in
   `$defs`. One option is `Reflect.alias(t): string | null` for a type that was written as an alias.
4. **Doc comments.** Schema and CLI-help generators want field docs. Keeping them costs memory in
   every program, so the option is to keep them only when `ant doc` or a flag asks.
5. **`Type[T]` and `Reflect.Type` as two names.** One generic `Type[T]` with an existential form
   (`Type[?]`) would be one name instead of two, but it would add a type-system feature that Ant
   otherwise does not need.
6. **Function types.** `Reflect.Func` describes a signature but no `Reflect` function builds a
   function value. Should `Reflect.Func` stay in `Info`, for documentation and schema tools, or
   become `Reflect.Opaque`?
