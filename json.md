# Proposal: typed JSON

2026-11-28 · draft

## Summary

Three functions in `Json` read and write JSON by a type the program names.

```
Json.decode[T](text: string, t: Reflect.Type[T], strict: bool = false): Result[T, Error]
Json.decodeValue[T](value: Json.Value, t: Reflect.Type[T], strict: bool = false): Result[T, Error]
Json.encode[T](t: Reflect.Type[T], value: T, indent: string = ""): string
```

`Json.decode` reads text straight into records, arrays, maps, sets, named primitives, time values
and unions: it builds each record as a literal would, with its tag and its defaults, and says where
the text does not fit. `Json.encode` writes a value as its type describes it, so that `Json.decode`
reads it back. `Json.decodeValue` reads a `Json.Value` tree in place of text, for a part of a
document whose type depends on another part.

```ant
type Debug = {}
type Info = {}
type Level = Debug | Info

type Server = { host: string, port: int = 8080, tags: Array[string] = [] }
type Config = { name: string, level: Level = Info{}, servers: Array[Server] }

let text = `{
  "name": "prod",
  "level": "Debug",
  "servers": [{"host": "a.example"}, {"host": "b.example", "port": 9000}]
}`

let config = try Json.decode(text, type(Config))
println(config.servers[0])        // Server{ host: "a.example", port: 8080, tags: [] }
println(config.level is Debug)    // true

Json.decode(`{"name": "prod", "servers": [{"host": "a", "port": "9000"}]}`, type(Config))
// Err: $.servers[0].port: want int, got "9000"
```

Nothing in the language changes: no syntax, no checker rule, no new type. `Json.parse` and
`Json.stringify` stay as they are.

## Motivation

`stdlib.md` says what a program does today: "A decoder into records is a function per type", or a
walk of the type with `Reflect`. Both work, and both stop short.

- **A function per type** is ten lines for a record of two fields, `decodeServer` in
  `docs/guide/runtime-types.md`, and it changes with every field.
- **A walk over `Reflect`** is about forty lines that each program copies from that guide, and then
  owns: its union rule, its error wording, its missing cases. Two Ant programs that exchange JSON
  agree on the form of a union only if they copied the same decoder.
- **It is slow in Ant.** On a 1 MB config and 10 MB of records (measured 2026-11-04), the walk over
  `Reflect` took 114 ms and 1171 ms, where hand-written decoders took 29 ms and 320 ms and
  `Json.parse` itself 9 ms and 98 ms. Encoding was 92 ms and 999 ms against 5 ms and 75 ms for
  `Json.stringify`. The cost is one `Reflect.Value` box per node, a shape per type reached and the
  `Json.Value` tree in between, and a decoder inside the runtime needs none of them.
- **Integers above 2^53 are lost before a library sees them.** `Json.parse` reads every number as a
  `float64`, so `9007199254740993` arrives as `9007199254740992.0`, and no code over the tree can
  recover the digit. A 64-bit id needs a decoder that reads the text knowing the field is an `int`.
- **`Json.stringify` loses which record a value is.** It writes a record without its tag, so both
  members of `Level` above come out as `{}`: `Json.stringify(config)` has `"level":{}`, and
  `Error{ message: "x", kind: Error.NotExist{} }` has `"kind":{}`. No decoder can read that back. An
  enumeration is how Ant writes a field such as `level` (section 4, Tagged unions), so an ordinary
  config type runs into it.

The first four call for a decoder in the library. The last calls for an encoder that knows the type
of each slot, which a function of the value alone cannot know.

## Prior art

- **Go `encoding/json`.** `json.Unmarshal` walks the Go type by reflection. A member that names no
  field is ignored unless the decoder is told to refuse it (`DisallowUnknownFields`, or
  `RejectUnknownMembers` in v2). An integer field refuses `1.0`. A missing member leaves the zero
  value. Go has no sum types, so a part whose type depends on another member is kept as a
  `json.RawMessage` and decoded in a second step. This proposal takes the reflective walk, the
  default for unknown members and the second step.
- **Rust serde.** An enum is externally tagged by default: `{"Circle":{"radius":2.0}}`, and a
  variant with no data is `"Empty"`. Internal, adjacent and untagged forms are attributes, and
  untagged takes the first variant that deserializes. Swift's synthesized `Codable` writes an enum
  with associated values in the same shape. This proposal takes the default form.
- **kotlinx.serialization, System.Text.Json.** A discriminator member inside the object, `"type"` or
  `"$type"`. See Alternatives.
- **pydantic.** The type is the schema, as here. A field `x: int | None` with no default is
  required, as here. Its lax mode converts `"80"` to `80`; this proposal converts nothing.
- **Zod.** The schema is a value, which Ant cannot take a type from. `z.union` tries the members in
  order, `z.discriminatedUnion` reads a named member first, and unknown keys are dropped unless the
  schema is `.strict()`, which is where the parameter name comes from.
- **TypeScript.** `JSON.parse(text) as Config` checks nothing, and every number is a double.

## The design

### The functions

`Json.decode(text, t)` reads `text` as `Json.parse` does: one JSON text of RFC 8259, the same
nesting limit, an object that repeats a name refused. Every message `Json.parse` has for malformed
text, it has too. It returns `Ok` of a new value of type `T`, or `Err` with the first place the text
does not fit `t`. `Json.decode(text, type(Json.Value))` is `Json.parse(text)`.

`Json.decodeValue(value, t)` reads a tree by the same rules and builds new values throughout, so the
result shares nothing with `value`.

`Json.encode(t, value)` returns the text of `value` as `t` describes it. The type comes first, since
a call solves its type parameters left to right (section 4, Types as values), so
`Json.encode(type(Array[Shape]), [])` checks. `indent` is `Json.stringify`'s.

All three take the type as a value, so a generic function passes its own along:

```ant
type Page[T] = { items: Array[T], next: string | null = null }
type User = { id: int, login: string }

func items[T](body: string, t: Reflect.Type[T]): Result[Array[T], Error] {
  Json.decode(body, Reflect.arrayOf(t))
}

let page = try Json.decode(body, type(Page[User]))
```

### What has a JSON form

| Type                                                            | JSON                                                   |
| --------------------------------------------------------------- | ------------------------------------------------------ |
| `null`                                                          | `null`                                                 |
| `bool`                                                          | `true`, `false`                                        |
| a number type                                                   | a number (Numbers)                                     |
| `string`                                                        | a string                                               |
| a named primitive type                                          | as its base, without the tag                           |
| `Array[T]`, `Set[T]`                                            | an array                                               |
| `Map[K, V]`, where `K` is `string` or a named primitive over it | an object, its entries in order                        |
| a named record type                                             | an object of its fields in declaration order (Records) |
| a time type                                                     | its RFC 9557 string, and a `Zone` its identifier       |
| `Regex.Pattern`                                                 | its rendering as a string, `"/a+/i"`                   |
| a union                                                         | the form of the member (Unions)                        |

Every row but the last is what `Json.stringify` writes today, and `Json.decode` is its inverse:

- A named primitive is read as its base and given its tag: `"u-1"` at a `UserId` is `UserId("u-1")`.
- A set collapses members that repeat, as `Set.of` does.
- A time type is read by its `parse` function (`Time.parseInstant`, `Time.parsePlainDate`, and so
  on, and `Time.zone` for a `Zone`), so a `Duration` takes `"PT30S"` and `"30s"` both.
- A `Regex.Pattern` is read from its rendering, which is a literal that reads back to it (section
  11, Regex).
- Nothing is converted on the way: `"80"` is not an `int`, `80` is not a `string`, and `null` fits
  only a type that has `null` as a member.

No other type has a JSON form: `Bytes`, `Html`, `void`, a function type, an anonymous record type,
the sync, random and resource types, the four types of `Reflect`, and an opaque type of a host. A
type expression already cannot reach an interface (section 4, Types as values). An anonymous record
type is out because no value is ever made at one (section 11, Reflect, Builders), and `Bytes`
because `Json.stringify` refuses it (open question 3).

### Records

An object is read into a record type as a record literal with the same entries would be checked.

- A member that names a field is read at the field's type. Names match exactly, case included.
- A field the object leaves out takes its default, evaluated as a literal evaluates it: afresh for
  each record, in the scope of its declaration (section 4, Field defaults). A field with no default
  must be there, and its absence is `Err`: `$.servers[0]: missing field host`.
- A missing member and `null` are different things, as a missing entry and `null` are in a literal.
  `nick: string | null` must be given, as `null` or a string. `nick: string | null = null` may be
  left out. `port: int = 8080` may be left out, and `"port": null` is `Err`, since `null` is not an
  `int`.
- A member that names no field is skipped, so a server may add a field without breaking its clients.
  With `strict: true` it is `Err`, in every record the call reads:
  `$.servres: unknown field of Config`. That is the literal's own rule, and the one for a file a
  person edits.

A record is written as `Json.stringify` writes it: every field, in declaration order. No field is
left out for being `null` or equal to its default.

A field has the name the JSON has. `created_at` is an identifier, and so is a keyword such as
`type`, so `type Event = { type: string, created_at: Time.Instant }` reads
`{"type": "push", "created_at": "2026-11-28T09:00:00Z"}`. An object whose names are data, or are not
identifiers (`content-type`, `$ref`), is a `Map`.

### Numbers

A JSON number is read into a number type by its value.

- A float type takes any number, `float64` as `Json.parse` reads it. A finite number beyond the
  range of a `float32` is `Err`, as `Reflect.makeScalar` has it.
- An integer type takes a number whose value is an integer in the type's range. A number written
  with no fraction and no exponent is read digit by digit, so every `int64` and `uint64` reads back
  exactly: `Json.decode("9007199254740993", type(int))` is `Ok` of `9007199254740993`. A number
  written with either is read as a `float64` first, as `Json.parse` reads it, and fits where
  `Math.toInt` gives an integer in range: `1.0` is 1, `1e3` is 1000, and `80.5` fits no integer
  type.

`Json.decodeValue` has only the `float64` of the tree, so the second half of each rule is all of it.
This is the rule `Reflect.makeScalar` and the guide's decoder already follow.

Numbers are written as `Json.stringify` writes them: an `int` in full, a float with `.0` when
integral. A reader in JavaScript rounds an integer above 2^53, so a program that writes for one
keeps a 64-bit id in a `string` field.

### Unions

A union is read by what the JSON value is. Each member reads one of the six kinds of JSON value:

| Kind    | Members that read it                                              |
| ------- | ----------------------------------------------------------------- |
| `null`  | `null`                                                            |
| boolean | `bool`, a named primitive over it                                 |
| number  | a number type, a named primitive over one                         |
| string  | `string`, a named primitive over it, a time type, `Regex.Pattern` |
| array   | `Array`, `Set`                                                    |
| object  | `Map`, a record type                                              |

A union with at most one member for each kind needs nothing more. `string | null`, `int | string`,
`Array[string] | string` and `Config | null` are read by kind and written with no mark. `Json.Value`
is such a union, one member per kind, so a slot of that type takes any JSON as the tree.

Two members that read one kind cannot be told apart, since both are written alike:
`UserId | string`, `int | float64`, `Time.Instant | Time.PlainDate`, `Array[int] | Set[int]`,
`Config | Map[string, int]`. Such a union has no JSON form (see No JSON form). The rule cannot be
"the first member that fits": a union's members are a set, and `int | null` and `null | int` are one
type (section 4, Types as values).

### Tagged unions

Records are the exception, because a record has a name. In a union with two or more record members
the tag alone tells them apart (section 4, Tagged unions), so the JSON form of each carries it:

- a member with fields is an object of one member, whose name is the record's and whose value is the
  record as an object: `{"Circle":{"radius":2.0}}`;
- a member with no fields is its name as a string: `"Empty"`.

```ant
type Circle = { radius: float64 }
type Rect = { width: float64, height: float64 }
type Empty = {}
type Shape = Circle | Rect | Empty

Json.encode(type(Array[Shape]), [Circle{ radius: 2.0 }, Empty{}])
// [{"Circle":{"radius":2.0}},"Empty"]
Json.encode(type(Circle), Circle{ radius: 2.0 })      // {"radius":2.0}
Json.stringify(Circle{ radius: 2.0 })                 // {"radius":2.0}

Json.decode(`{"Rect": {"width": 1, "height": 2}}`, type(Shape))
// Ok{ value: Rect{ width: 1.0, height: 2.0 } }
Json.decode(`{"Square": {"side": 1}}`, type(Shape))
// Err: $: Shape has no member named "Square"
```

The form mirrors the literal: `Circle{ radius: 2.0 }` is the name and then the fields. It follows
from the slot's type and not from the value, so the same `Circle` is `{"radius":2.0}` in a field of
type `Circle` or `Circle | null`, and `{"Circle":{"radius":2.0}}` in a field of type `Shape`.

- **The name** is the declaration's, as `String.from` of its `Reflect.Decl` gives it (section 11,
  Reflect, Names): the name written after `type`, with its owner for a prelude type. `Ok[int]` is
  `"Ok"` and `Error.NotExist` is `"Error.NotExist"`. Two members with one name,
  `A.Circle | B.Circle` from two modules, have no JSON form.
- **Enumerations** fall out: `Level` is `"Debug"` or `"Info"`. The name is matched exactly. A
  document that spells it `"debug"` is read into a `string` or a named primitive over one.
- **Beside them.** Together the record members read the object kind, so the union holds no `Map`.
  When one of them has no fields they read the string kind too, and the union holds no other member
  that reads a string. `Shape | null` and `Shape | Array[Shape]` are fine.
- **Reading** an object here needs exactly one member, and `strict` does not bear on that: two
  members, none, or a name the union does not have are `Err`. A member with no fields is read from
  the string only, and one with fields from the object only, even when every field has a default:
  each value has one form.
- **Unions flatten**, so with `type Label = { text: string }`, `Shape | Label` is one union of four
  records, and `Shape | null` is tagged as `Shape` is.

`Result` and `Error` need nothing of their own:

```ant
let error = Error{ message: "no such file", kind: Error.NotExist{} }
Json.encode(type(Result[int, Error]), Result.err(error))
// {"Err":{"error":{"message":"no such file","kind":"Error.NotExist"}}}
```

The tag makes two things part of the format that were not before. Renaming a member type changes the
text, as renaming a field does. And a union that grows from one record to two changes the form of
the first: `Circle | null` is written bare and `Circle | Rect | null` tagged.

### Two steps

JSON from elsewhere marks its variants its own way, often with a member such as `"type": "push"`
beside the fields it selects. `Json.decode` does not guess at that. The program reads the fixed
part, keeps the rest as a tree, and reads the tree once it knows the type, which is what
`json.RawMessage` is for in Go:

```ant
type Request = { id: int | null = null, method: string, params: Json.Value = null }
type Rename = { path: string, to: string }

func handle(line: string): Result[void, Error] {
  let req = try Json.decode(line, type(Request))
  if req.method == "rename" {
    let p = try (Json.decodeValue(req.params, type(Rename)) | Result.wrapErr("rename"))
    try File.rename(p.path, p.to)
  }
  Result.ok(void)
}
```

`Json.decodeValue` is also how a tree from any other source becomes a record: one built in code, or
one that a parser for another format returns.

### Errors

A text that is not JSON is `Err` with `Json.parse`'s message. A text that does not fit the type is
`Err` with the path of the value, what the type wants and what the text has:

```
$.servers[1].port: want int, got "9000"
$.servers[1].port: want int, got 80.5
$.servers[1]: missing field host
$.servers[1].tags: want Array[string], got an object
$.level: Level has no member named "debug"
$.created: want Time.Instant, got "yesterday"
$.limits["max-size"]: want int, got null
```

The path starts at `$`, a field or a key that is an identifier follows a `.`, any other key is in
brackets as a JSON string, and an array index is in brackets. The type renders as `String.from`
renders it, an alias by its name. A scalar is shown as its JSON text, and an array or an object by
the word. The decoder stops at the first error, in the order of the text; a field left out is
reported when its object ends. `kind` is `null`.

### No JSON form

Whether a type has a JSON form depends on the type alone: every type it reaches has a row in the
table, and every union follows the two sections above. `Json.decode`, `Json.decodeValue` and
`Json.encode` panic on a type that has none, before reading anything, with a message that names the
type and the reason:

```
Json.decode: Upload.data: Bytes has no JSON form
Json.decode: UserId | string: UserId and string are both read from a JSON string
Json.encode: Shape | Map[string, int]: Circle and Map[string, int] are both read from a JSON object
```

A panic and not an `Err`, because the fault is in the program and no input changes it: the call
panics every time, so the first run of any test finds it. `ant check` does not report it, since the
type may arrive through a `Reflect.Type[T]` parameter. An implementation works a type out once and
keeps the result.

`Json.encode` also panics where `Json.stringify` does on a value: on `NaN` or an infinity, and on a
value that contains itself. Under `ant run` it trusts that `value` belongs to `t`, as
`Reflect.valueOf` does (section 11, Reflect, Unchecked runs).

### The round trip

For a type `t` with a JSON form and a value `v` that `Json.encode` writes,
`Json.decode(Json.encode(t, v), t)` is `Ok` of a value `==` to `v`. Two things are not kept. A
`Duration` reads back with its fraction spread, equal under `Time.compareDuration` and not under
`==`, as `stdlib.md` already says of its text. And sharing: two fields that held one array read back
as two arrays.

`Json.encode(t, v)` and `Json.stringify(v)` give the same text unless `t` reaches a union of two or
more records. So `Json.stringify` remains the way to write a value without naming its type, such as
a `Json.Value` or a map built on the spot, and `Json.encode` is the one to use when `Json.decode`
will read the text.

## Fit with other proposals

- **Field annotations** (`annotations.md`). Renaming one field, leaving one out, and a constraint
  such as "at least 13" belong to a field, and this proposal has no place to write them. A
  validation library there still walks `Reflect` itself; whether `Json.decode` should one day read a
  `Json.Key` annotation is a question for that proposal.
- **Private fields** (`private-fields.md`) recommends that a type-directed codec leave private
  fields out when writing and to their defaults when reading. `Json.encode` and `Json.decode` are
  such a codec and would follow that rule when it lands; until then every field is read and written.

## Alternatives

- **Leave it to libraries over `Reflect`.** The status quo. Motivation has what it costs: a copy per
  program, a walk nearly four times as slow as a hand-written decoder, and no exact integers.
- **A tree decoder only**, `Json.decodeValue` without `Json.decode`. One function fewer, and
  `Json.decodeValue(try Json.parse(text), t)` reads well. But the tree has lost every integer above
  2^53, and each document is built twice.
- **No tags: pick the record by its fields.** An object is the one member whose required fields it
  has. `Json.stringify` would stay the only encoder. But every member of an enumeration is `{}`, two
  members with the same fields cannot be told apart, and giving a field a default or adding one
  changes which objects fit which member.
- **The first member that decodes wins**, as serde's untagged form and `z.union` have it. A union
  has no order in Ant, the error for a value that fits nothing cannot say which member was meant,
  and every failed member is work thrown away.
- **A tag inside the object**, `{"$type":"Circle","radius":2.0}`, as System.Text.Json and
  kotlinx.serialization write it, and as TypeScript code narrows it. It needs a reserved name;
  `$type` cannot be a field, since it is not an identifier. And the tag may come after the fields,
  so the decoder reads ahead before it knows the type. The outer form needs neither. Open
  question 2.
- **Tags in `Json.stringify`.** It has only the value and cannot know that a slot is a union.
  Writing every record under its name would change the text of every program.
- **Go's integer rule**, where `1.0` is not an integer. Stricter, and exact with no second case. But
  `Json.decodeValue` cannot follow it, since the tree does not keep how a number was written, and
  the two functions would then disagree on one document.
- **A missing member as `null`** wherever the field's type allows it, as serde's `Option` and a Go
  pointer have it. Ant has no zero values (section 4, Field defaults), and `= null` in the
  declaration says the same thing in one place, for literals too.
- **Refusing unknown members by default**, as kotlinx.serialization and Jackson do. Open question 1.
- **Converting on the way in**: `"80"` to `80`, `1` to `true`, a name matched whatever its case.
  Each makes two texts mean one value, and section 1 has it that a value never changes type without
  a visible conversion.
- **Hooks**: a function on a type that the decoder calls, as Go's `UnmarshalJSON`. It needs a rule
  for how the runtime finds the function. Two steps cover the same ground with code the program can
  see.

## Costs

- Three signatures in `prelude.d.ant`. The checker types them as it types `Reflect.to` and
  `Reflect.valueOf` today.
- In the runtime, beside `Json.parse` and `Json.stringify`: the analysis of a type, kept per type; a
  decoder over the existing scanner that builds a record as `Reflect.makeRecord` does; and an
  encoder that is `Json.stringify`'s writer with the tag added. The scanner learns to keep an
  integer as written. None of it is written yet, so its size is not known.
- Documentation: three pages under `lib/Json`, and the "Typed decoding" section of
  `guide/runtime-types.md`, whose decoder stays as the example of a walk over `Reflect` for a format
  the library does not have.
- No change to `ant fmt`, the LSP or the editor grammars.

## Spec changes

- `stdlib.md`, Json: the three signatures, and this design in place of the sentences that send a
  program to a function per type or to `Reflect`.
- `stdlib.md`, the opening paragraph: the three functions join `Json.stringify` among the
  exceptions, since they panic on a type with no JSON form.
- `ant-lang.md`, section 8: `loadConfig` in `try` and Wrapping errors ends in
  `Json.decode(text, type(Config))` in place of `Config.fromJson(json)`.
- `ant-lang.md`, section 11, Reflect: the opening names `Json.decode` as the library's own policy
  for JSON. The rest stands: another format, or another policy for JSON, is still library code.

## Non-goals

- Renaming or leaving out single fields, and validation of ranges and patterns. See Fit with other
  proposals.
- Other conventions for a variant, such as `"type": "circle"`. Two steps.
- Reading a stream, or a file of one JSON text per line, without holding the text. A program splits
  the lines and decodes each.
- Comments, trailing commas and the other extensions of JSON5.
- A JSON Schema for a type. It is a walk over `Reflect.shape`.
- Other formats. A parser that returns a `Json.Value` gets records through `Json.decodeValue`, and
  nothing more is promised.

## Open questions

1. **Unknown members.** Skipped unless `strict: true`, as written, so that a client survives a
   server that adds a field. The other default catches a misspelled key in a config file without the
   author asking for it, and matches the record literal. Leaning: as written, since the first
   failure reaches production and the second is found while writing the config.
2. **The tag's form.** `{"Circle":{"radius":2.0}}` as written, or `{"$type":"Circle","radius":2.0}`.
   The second is closer to what TypeScript and OpenAPI clients expect; the first needs no reserved
   name and no reading ahead. Leaning: as written.
3. **`Bytes`.** Go writes `[]byte` as base64. `Json.stringify` refuses `Bytes` because the reader
   must know the encoding, and with a type on both sides the reader does. Leaning: leave it out
   until a program needs it, then standard base64 with padding in `Json.encode` and `Json.decode`
   only.
4. **`Json.decodeValue` now or later.** It is the smallest of the three, and without it a
   `Json.Value` field is a dead end for typed code. Leaning: now.
5. **`int | float64`.** Refused as written. Ant's own text tells the two apart, `1` and `1.0`, but
   JavaScript writes both as `1`. Leaning: refused; a field that takes both is a `float64`.
6. **Measure before accepting.** The benchmark behind the numbers in Motivation should run against a
   prototype of the decoder. The expectation to test is that it runs at or under the hand-written
   decoders, since the work per node is the scanner's plus one builder call.
