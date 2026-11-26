# Proposal: typed JSON

2026-11-03, rewritten 2026-11-26 · Anton

## Summary

Two functions in `Json` that read JSON text into a declared type and write a value of one back, and
one helper for key names:

```
Json.decode[T](text: string, t: Reflect.Type[T], key: (func(string): string) | null = null,
               tag: string | null = null): Result[T, Error]
Json.encode[T](t: Reflect.Type[T], value: T, indent: string = "", key: (func(string): string) | null = null,
               tag: string | null = null): string
Json.snakeCase(name: string): string
```

```ant
type Server = { host: string, port: int = 8080, tags: Array[string] = [] }
type Config = { name: string, servers: Array[Server] }

var config = try Json.decode(text, type(Config))
```

The type is the schema. Both functions carry one policy, written down once in `stdlib.md`: how a
union is chosen, what a missing or unknown key means, how numbers are read, and how an error names
its path. Both are native. Nothing in the language changes: no syntax, no reserved names, no checker
rule.

Reflection (`ant-lang.md`, section 11, Reflect) already makes typed decoding possible as library
code, and it stays the way to write every other format. This proposal is the standard library doing
it once, for JSON.

## Motivation

### Decoding is hand-written today

`stdlib.md` says: "A decoder into records is a function per type: narrow the tree with `is`, build
the record, and propagate errors with `try`, passing the path along so an error can say
`$.servers[2].port`." For the two types above that is:

```ant
func Server.fromJson(v: Json.Value, path: string): Result[Server, Error] {
  if !(v is Map) { return Result.err(Error{ message: "${path}: expected an object" }) }
  var host = v["host"]
  if !(host is string) { return Result.err(Error{ message: "${path}.host: expected a string" }) }
  var port = 8080
  var p = v["port"]
  if p != null {
    if !(p is float64) { return Result.err(Error{ message: "${path}.port: expected a number" }) }
    port = Math.toInt(p) ?? return Result.err(Error{ message: "${path}.port: expected an integer" })
  }
  var tags: Array[string] = []
  // ... and the same again for each tag, then Config, then every type it reaches
  Result.ok(Server{ host, port, tags })
}
```

Every field name is written three times, every default twice, and the decoder drifts from the type
as soon as someone adds a field.

### The generic walk exists, and is slow

With reflection that decoder is one generic walk over `Reflect.shape`, about 65 lines of Ant. A
program that wants typed JSON today copies those lines, and takes its policy from whoever wrote the
copy. Measured on 2026-11-04, on a 1 MB configuration and a 10 MB array of records, the walk takes
3.7 to 3.9 times as long as a hand-written decoder: one `Reflect.Value` box per node, a shape record
per type reached, and the `Json.Value` tree in between. It also reads every number through
`Json.parse`, so an `int` above 2^53 arrives rounded.

A native decoder reads the text along the type: no boxes, no tree in between, and an integer read
from its digits.

### A real script

`~/dev/language-forge/post_language.ant` posts generated languages to Reddit. About 60 of its 359
lines are JSON plumbing: four helpers that walk `Json.Value` (`field`, `text`, `textOr`,
`textList`), a 12-line `decodeLanguage`, and nested loops over the Anthropic, OpenAI and Reddit
replies. With this proposal they become small reply types and one call each:

```ant
type Block = { type: string, text: string = "" }
type AnthropicReply = { stopReason: string | null = null, content: Array[Block] }

var reply = try Json.decode(r.text(), type(AnthropicReply), key: Json.snakeCase)
var text = reply.content | Array.filter({ x.type == "text" }) | Array.map({ x.text }) | String.concat()
```

Unknown keys in the replies are ignored, and an error reads `$.content[2].text: expected string`.

## Prior art

| Library                                  | What it does                                                                         | What this proposal takes                                                           |
| ---------------------------------------- | ------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------- |
| Go `encoding/json`                       | Decodes by the declared type; ignores unknown keys; reads an integer from its digits | All three. Not zero values for missing keys: Ant has none                          |
| Go `encoding/json/v2`, stable in Go 1.27 | Stricter defaults: case-sensitive names, duplicate names rejected                    | The same defaults: keys match exactly, and a duplicate name is an error            |
| Rust `serde`                             | `#[serde(tag = "kind")]` for unions; `rename_all = "snake_case"` for a type's fields | The internally tagged union, and one key policy for a call                         |
| Python pydantic                          | The declared class is the schema; `model_validate_json` reads text straight into it  | The type as the schema. Not validation, which `annotations.md` leaves to a library |

## The design

### 1. Decoding: `Json.decode`

`Json.decode` reads one JSON text, as `Json.parse` defines it, and builds a value of `T`. A text
that `Json.parse` refuses, `Json.decode` refuses with the same message, a repeated name and nesting
beyond the limit included. The value it returns is one that the builders of section 11 could have
built, so it is sound for the same reasons theirs are.

| Target type            | Accepts                                                                                  |
| ---------------------- | ---------------------------------------------------------------------------------------- |
| `null`                 | `null`                                                                                   |
| `bool`                 | `true` or `false`                                                                        |
| `string`               | a string                                                                                 |
| an integer kind        | a number written with no fraction and no exponent, in range                              |
| a float kind           | any number, rounded to that kind; a finite number beyond its range is an error           |
| a named primitive type | what its base accepts, then the tag: `"u-1"` into `UserId` is `UserId("u-1")`            |
| a time type            | a string that the type's parser reads: `Time.parseInstant` for `Time.Instant`, and so on |
| `Array[T]`             | an array, element by element                                                             |
| `Set[T]`               | an array, member by member; a repeated member counts once                                |
| `Map[K, V]`            | an object; each key builds as `K` from its string (see below)                            |
| a named record type    | an object: each field from the key of its name, or its default when the key is missing   |
| a union                | the one member that accepts the value (see below)                                        |

**Numbers.** An integer type reads the digits of the number, as Go does, so every `int64` and
`uint64` is exact, where `Json.parse` rounds above 2^53. A number written with a fraction or an
exponent, such as `8080.0` or `1e3`, is not an integer, as in Go. A float type takes any number and
rounds it as `Reflect.makeScalar` does.

**Records.** A record ignores keys it has no field for, as Go does. A missing key with no default is
an error, including for a field typed `T | null`, since a record literal needs that field too; a
field that may be absent declares `= null`. This differs from Go, which fills a missing field with
its zero value, because Ant has no zero values. A default is evaluated as in a literal. A JSON
`null` is a value like any other: a field takes it only when its type has `null`, and it never
stands for a missing key.

**Map keys.** `K` must be `string`, an integer kind, or a named primitive over one of them. An
integer key is read from its decimal string, so `{"1": "a"}` builds a `Map[int, string]`, as in Go.

**Unions.** An Ant value always names its member by kind or tag (section 4, Unions), so a union is
never ambiguous on the value side, and `Json.encode` never has to choose. JSON loses tags, though:
`UserId("a")` and `"a"` are both the string `"a"`. So `Json.decode` decides by the type, never by
trying members until one works. Each member reads one kind of JSON value: `null`, a boolean, a
number, a string (`string` and the time types), an array (an array or a set), or an object (a record
or a map). A named primitive reads what its base reads. A union can be decoded when no two members
read the same kind, with two refinements:

- **Numbers.** A union may hold one integer kind and one float kind. A number written with no
  fraction and no exponent goes to the integer member, and any other number to the float member.
  When only one of them is there, it takes every number it accepts. So `int | string` reads `3` as
  `3`, and `int | float64` reads `3` as `3` and `3.0` as `3.0`, which is how `Json.encode` writes
  the two.
- **Records with a tag.** Several record members may share the object kind when the call passes
  `tag: "kind"`. The object's `kind` entry then names the member: it must be a string equal to a
  member's declared name, after the `key` policy, so with `key: Json.snakeCase` the value `"circle"`
  selects `Circle`. The members must have distinct names after the policy, and no field whose key is
  the tag. A map member cannot share the object kind with a record, tag or not.

A union that breaks these rules, such as `UserId | string` or `Circle | Rect` without a tag, cannot
be decoded, which is a program error (see Panics). A program that needs another shape declares that
part as a `Json.Value` field and finishes it by hand, or walks the type with `Reflect.Value`.
`Json.Value` is itself a union these rules accept, and decoding into it gives what `Json.parse`
gives.

**`key`** maps a field name to the key the JSON uses, so `key: Json.snakeCase` reads `userId` from
`user_id`. One policy covers the whole call; renaming one field is the job of a field annotation
(see Fit with other proposals). `Json.snakeCase` puts `_` before an uppercase letter that follows a
lowercase letter or a digit, and before the last letter of an uppercase run when a lowercase letter
follows it, then lowercases with `String.toLower`: `maxTokens` is `max_tokens`, `userID` is
`user_id`, `HTTPServer` is `http_server`, and `sha256Sum` is `sha256_sum`.

**Errors** name the path from the root: `$.servers[2].port: expected int, got 8.5`,
`$.servers[0]: missing field host`, `$.shape: kind "hexagon" names no member of Circle | Rect`. The
path uses the keys of the JSON: `.name` for a key that is an identifier, `["a b"]` for any other,
written as `Json.stringify` writes the string, and `[i]` for an array index. `Json.decode` stops at
the first error, as Go does.

**Panics.** A type that cannot be decoded is a program error, not a data error: a function, `void`,
`never`, an opaque type other than the time types, an anonymous record type (which
`Reflect.makeRecord` refuses), a map key of another type, a union that breaks the rules above, or a
record two of whose fields `key` gives one key. `Bytes` is among the opaque types: JSON carries it
only in an encoding its reader must know, which is why `Json.stringify` refuses it too.
`Json.decode` checks the whole type, with the call's options, before it reads the text, and panics
with the path to the part, such as `Json.decode: $.onSave: a function cannot be decoded`.

### 2. Encoding: `Json.encode`

`Json.encode` is the inverse, with the same options. It accepts exactly the types `Json.decode`
does, checked the same way before anything is written, so every text it writes reads back. A record
becomes an object with its fields in declaration order, each under its key. Where the slot a record
sits in is a union with several record members, the tag entry comes first. A named primitive is
written as its base, a set as an array, and a map as an object, an integer key as its decimal
string. Numbers, strings, time values, and `indent` are as `Json.stringify` writes them, so an
integer is its digits and `3.0` stays `3.0`. It panics where `Json.stringify` does on a value: a
`NaN`, an infinity, or a cyclic value.

For every type both accept, `Json.decode(Json.encode(t, v), t)` with the same options is `Ok` with a
value `==` to `v`. The one exception is a `Time.Duration`, which parses back with its fraction
spread (`stdlib.md`, Time).

`Json.stringify` stays as it is, and stays the function that writes any value without its type.
`Json.encode` differs from it in applying `key`, writing the tag entry, writing integer map keys as
strings, and refusing a type that would not read back.

### 3. Unchecked runs

`ant run` checks nothing (section 4, Checking and running). `Json.decode` needs no check, since it
builds from the type value alone. `Json.encode` trusts that `value` is a `T`, as `Reflect.valueOf`
does, and panics where a part's kind does not fit its slot.

## Other formats

Every other format walks the type with `Reflect.shape` and `Reflect.Value` and keeps its own policy,
as the Reflect section says. Three codecs written that way, as tests in the `ant` repository, show
why the policy is not shared. YAML reads a scalar the way its target type asks, so `yes` into a
`string` is `"yes"`. TOML has dates and no `null`. XML has attributes. What the three share is the
shape of the walk, which is the small part of each: about 130 of the TOML codec's 730 lines decode
along the type.

## Fit with other proposals

- **`annotations.md`.** A field annotation `@Json.Key("max_tokens")` overrides the `key` policy for
  one field, and `Json.Skip` and `Json.Tag` may follow. They are ordinary prelude types that
  `Json.decode` and `Json.encode` read. That proposal lands with or after this one.
- **`private-fields.md`.** A private field is left out by `Json.encode` and left to its default by
  `Json.decode`, as that proposal recommends for a typed codec. A record with a private field that
  has no default cannot be decoded, which joins the list under Panics.

## Costs

- **A policy in the standard library.** Every rule in section 1 is a decision that some JSON API
  disagrees with. `Json.parse` and `Reflect.Value` are there for those, but users will meet this
  policy first.
- **A second writer beside `Json.stringify`.** Two functions turn a value into JSON, and a reader
  has to learn which one a program means.
- **Native code.** A type-directed reader and writer in Go, beside the ones `Json.parse` and
  `Json.stringify` have. Its size is reported with the implementation.

## Spec changes

`stdlib.md`, Json: the three functions, the table, the union rules, errors, and panics. The sentence
that leaves typed decoding to library code keeps saying so for other formats.

`ant-lang.md`: nothing.

`ant/docs`: a page each for `Json.decode`, `Json.encode`, and `Json.snakeCase`.

## Implementation

- **Native.** Both functions are Go code over the type descriptors reflection already has. A number
  stays text until its target type is known. Whether the decoder walks the text once or an internal
  tree is not observable.
- **The type check is remembered** per type and options, so a call in a loop pays for it once.
- **Reference.** The Ant walk over `Reflect.Value`, with this policy, stays in the tests. A test
  runs it and the native functions on generated values of generated types.
- **Tests.** The table row by row, both directions; round trips with and without `key` and `tag`;
  recursive, generic and local types; every union rule and the panic of each broken one; integers at
  both ends of each kind's range; error paths.
- **Benchmarks.** `Json.decode` against the hand-written decoder and the Ant walk on the two files
  of the 2026-11-04 measurement, recorded here before acceptance.

## Non-goals

- **No shared layer for other formats.** See Alternatives considered.
- **No language change.** Hooks are under Later for that reason.
- **No change to `Json.parse`, `Json.stringify`, or `Json.Value`.**

## Alternatives considered

- **A shared data tree for every format.** The previous draft, `typed-data.md`, had a format-neutral
  tree `Type.Data`, `Type.build` and `Type.data` between it and any type, and JSON as a converter on
  top. It was dropped for four reasons:
  - The YAML, TOML and XML codecs above each need a policy the tree cannot carry.
  - JSON did not fit it either. Through `Json.Value` an integer above 2^53 was rounded on the way
    in, and every integer was written as a float, `8080.0`, on the way out.
  - The walk it shared is the small part of a codec.
  - Go has no such layer: `encoding/json` sits on `reflect`.

  What the tree did fit is flat text: CSV rows, environment variables, flags. See Later.

- **Library code over reflection only.** The state today. It is three to four times slower, every
  program copies the walk, and each copy has its own policy.
- **Options on `Json.stringify` in place of `Json.encode`.** `Json.stringify` walks a value, and a
  value does not say that the slot it sits in is a union, so it cannot know where a tag entry
  belongs.
- **Trying union members in order.** The data then decides which member a value is, so `"3"` into
  `int | string` depends on member order, and adding a member can change what old data decodes to.
  Deciding by type makes the choice visible when the type is written.
- **A `"$type"` key.** It claimed a key in every JSON document, including ones that use `$type` for
  something else, as Newtonsoft.Json payloads do. The `tag` option names the key per call.
- **Integers by value,** so that `8080.0` fits an `int`. Go refuses it, JavaScript never writes it,
  and reading by spelling is the same rule that sends `3` and `3.0` to the two members of
  `int | float64`.
- **`Bytes` as base64,** as Go writes a `[]byte`. `Json.stringify` already refuses `Bytes`; a field
  that carries binary is a `string`, and the program converts it with `Encoding.fromBase64`.

## Later

- **Hooks.** A record or a named primitive type that takes over its own conversion, with a
  `fromJson` and `toJson` pair, and checks itself with `validate`. Hooks need a language rule,
  reserved function names with checked signatures (section 6) or prelude interfaces (section 4,
  Interfaces), which is why they are not here. Until then a `Json.Value` field keeps a subtree for
  code to finish, and validation is a function called on the decoded value.
- **A strict mode** for unknown keys, as Go's `DisallowUnknownFields`.
- **Every error at once**, as pydantic reports, as one message with a line per error.
- **Line and column** in an error about a value, as `Json.parse` gives for malformed text.
- **A compile error for a type that cannot be decoded.** It needs a checker rule for two library
  functions, one that depends on the `tag` argument and cannot see through a generic wrapper.
- **Omitting fields on encode**, `omitempty` in spirit, as a predicate the caller gives. A declared
  default is not a zero value, so nothing compares a field with its default.
- **Flat text sources.** Environment variables, flags and CSV rows are a map of strings and a table
  of spellings for numbers and booleans. If a second library function needs those rules, they are
  written once, for that function and this one.

## Open questions

1. **Tag values.** This draft matches the tag entry to the member's declared name after the `key`
   policy. A program whose discriminator values are not derived from its type names, such as `"c"`
   for `Circle`, has to keep that part as a `Json.Value`. Is a map from values to members worth
   adding?
2. **Shipping without hooks.** Go's `encoding/json` had `UnmarshalJSON` from the start. Is a first
   version with `Json.Value` fields as the only escape enough, or do hooks have to land with it?
