# Proposal: typed data

2026-11-03 · Anton

## Summary

A follow-up to `reflection.md`, which adds the mechanism: `type(T)` as a value of type `Type[T]`,
`Type.shape` to take a type apart, and `Dynamic` boxes with readers and builders that keep every
value sound. That proposal deliberately has no policy, so every format would write its own walk over
types: union choice, missing and unknown fields, defaults, error paths. This proposal is the shared
layer on top, so that a format only converts its own syntax:

- **`Type.Data`**, a closed, format-neutral tree: null, bool, three number kinds, string, `Bytes`,
  arrays and string-keyed maps.
- **`Type.build(t, data)`** turns a tree into a checked value of `T`, and **`Type.data(t, v)`**
  turns a value back into a tree. Both carry one policy, written down once: how unions are chosen,
  what a missing or unknown key means, how numbers and text convert, and how errors name their path.
- **Hooks**: a record or a named primitive type can take over its own conversion with `fromData` and
  `toData`, and check itself with `validate`.
- **`Json.decode` and `Json.encode`** on top, as four lines of library each.

```ant
type Server = { host: string, port: int = 8080, tags: Array[string] = [] }
type Config = { name: string, servers: Array[Server] }

var config = try Json.decode(text, type(Config))
```

A format that agrees with this policy is a converter between its syntax and `Type.Data`: YAML, TOML,
CSV rows, environment variables, command-line flags, query strings. A format that does not, such as
XML with attributes or JSON with base64 `Bytes`, walks the type with `reflection.md`'s `Dynamic`
directly and keeps full control. The two paths are the point: serde's lesson that a shared data
model turns N formats times M types into N plus M, without taking away the lower layer.

Nothing here works without `reflection.md`, and nothing here changes it.

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

### Reflection alone moves the problem into every format

With `reflection.md`, that decoder becomes one generic walk, its JSON sketch, about 80 lines. But
YAML, TOML, CSV, environment variables and flags each need the same walk again, and each makes the
same decisions slightly differently: which member of `int | string` a `"3"` is, whether a missing
`T | null` field is `null`, whether an unknown key is an error, how an error names its path. A user
who reads a config from a file and overrides it from the environment would meet two policies for one
type. Reflection is right to leave policy out; something above it has to put one policy in.

### A real script

`~/dev/language-forge/post_language.ant` posts generated languages to Reddit. About 60 of its 359
lines are JSON plumbing: four helpers that walk `Json.Value` (`field`, `text`, `textOr`,
`textList`), a 12-line `decodeLanguage`, and nested loops over the Anthropic, OpenAI and Reddit
replies. With this proposal they become small reply types and one call each:

```ant
type Block = { type: string, text: string = "" }
type AnthropicReply = { stopReason: string | null = null, content: Array[Block] }

var reply = try Json.decode(r.body, type(AnthropicReply), key: Type.snakeCase)
var text = reply.content | Array.filter({ x.type == "text" }) | Array.map({ x.text }) | String.concat()
```

Unknown keys in the replies are ignored, and an error reads `$.content[2].text: expected string`.

## Prior art

| Library                                      | What it does                                                                              | What this proposal takes                                                                   |
| -------------------------------------------- | ----------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| Rust `serde`                                 | One data model between every type and every format; `#[serde(tag = "kind")]` for unions   | The shared tree, and the internally tagged union                                           |
| Python cattrs                                | `structure(data, cls)` and `unstructure(value)` between classes and plain dicts and lists | Almost exactly `Type.build` and `Type.data`                                                |
| Python pydantic                              | Decoding by the declared class; `model_validate`, `field_validator`; strict and lax modes | `validate` as a hook; `coerce` as lax mode for text-only sources                           |
| Go `encoding/json`                           | Ignores unknown keys; fills missing ones with zero values; `UnmarshalJSON` per type       | Ignoring unknown keys, and a hook a type declares on itself. Not zero values: Ant has none |
| Go `strconv.ParseBool`, `flag`, env decoders | Text-only sources parse numbers and booleans from strings                                 | The exact boolean spellings under `coerce`                                                 |
| Go `encoding/json/v2` experiment             | Stricter defaults: case-sensitive names, duplicate names rejected                         | A reminder that this policy is a choice, which is why it sits above `reflection.md`        |

## The proposal

### 1. The data tree: `Type.Data`

```
type Type.Data = null | bool | int64 | uint64 | float64 | string | Bytes
               | Array[Type.Data] | Map[string, Type.Data]
```

`Type.Data` is the one shape the formats on this path convert to and from. It is a closed union, as
`Json.Value` is, so it needs no top type. It has the three number kinds that formats carry: JSON and
YAML numbers are `float64`, while MessagePack, CBOR and TOML keep integers. It has `Bytes` for
binary formats. Time values travel as strings in the form `String.from` gives them, which is also
what `Json.stringify` writes.

Every `Json.Value` is a `Type.Data` value at run time, but the checker still needs a copy, because
`Array[Json.Value]` is not an `Array[Type.Data]`; `Json.toData` is that copy.

### 2. Building: `Type.build`

```
Type.build[T](t: Type[T], data: Type.Data, coerce: bool = false,
              key: func(string): string | null = null, tag: string | null = null): Result[T, Error]
```

`Type.build` walks the type, with `Type.shape`, and the data together, and builds the value with
`reflection.md`'s builders, so the value it returns is sound for the same reasons theirs are.

| Target type              | Accepts                                                                                                                     |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------- |
| `null`, `bool`, `string` | the same kind                                                                                                               |
| an integer kind          | `int64` or `uint64` in range; a `float64` with no fraction, in range (so the JSON `8080` fits an `int`, and `8.5` does not) |
| a float kind             | any number, rounded as the conversion to that kind rounds                                                                   |
| a named primitive type   | what its base accepts, then the tag: `"u-1"` into `UserId` is `UserId("u-1")`                                               |
| `Bytes`                  | `Bytes`                                                                                                                     |
| a time type              | a string in the form that type's `parse` reads                                                                              |
| `Array[T]`               | an array, element by element                                                                                                |
| `Set[T]`                 | an array, member by member; a repeated member counts once                                                                   |
| `Map[K, V]`              | a map; each key builds as `K` from its string (see below)                                                                   |
| a named record type      | a map: each field from the key of its name, or its default when the key is missing                                          |
| a union                  | the one member that accepts the data (see below)                                                                            |

**Records.** A record ignores keys it has no field for, as Go does. A missing key with no default is
an error, including for a field typed `T | null`, since a record literal needs that field too; a
field that may be absent declares `= null`. This differs from Go, which fills a missing field with
its zero value, because Ant has no zero values. A default is evaluated as in a literal, which
`Dynamic.makeRecord` already does.

**Map keys.** `K` must be `string`, an integer kind, or a named primitive over one of them. An
integer key is read from its decimal string, so `{"1": "a"}` builds a `Map[int, string]`, as in Go.

**Unions.** An Ant value always names its member by kind or tag (section 4, Unions), so a union is
never ambiguous on the value side, and `Type.data` never has to choose. Data loses tags, though:
`UserId("a")` and `"a"` are both the string `"a"`. So `Type.build` decides by the type, never by
trying members until one works. Each member reads one data kind: a record or a map reads a map, an
array or a set an array, a string, a time type or a named primitive over `string` a string, and so
on. A union is buildable when no two members read the same kind, with three refinements:

- **Numbers.** A union may hold at most one integer kind and one float kind. Integer data goes to
  the integer member and `float64` data to the float member, and when only one of them is there, it
  takes any number it accepts. So `int | string` reads the JSON `3` as `3`, and `int | float64`
  reads `3` as `3.0` from JSON, whose numbers are `float64`, but as `3` from TOML.
- **Records with a tag.** Several record members may share the map kind when the call passes
  `tag: "kind"`. The map's `kind` entry then names the member: it must equal a member's declared
  name, after the `key` policy, so with `key: Type.snakeCase` the value `"circle"` selects `Circle`.
  The members must have distinct names and no field named like the tag. A map member cannot share
  the map kind with a record, tag or not.
- **Text under `coerce`.** A string member, when there is one, takes every string. Without one, at
  most one member besides `null` may read text: `int | null` builds `42` from `"42"` and `null` from
  `""`.

A union that breaks these rules, such as `UserId | string` or `Circle | Rect` without a tag, cannot
be built, which is a program error (see Panics). A program that needs another shape decodes that
part into a `Type.Data` field and finishes it by hand, declares a hook on the record that holds it,
or walks it with `Dynamic`. `Type.Data` and `Json.Value` are buildable under these rules, so a field
of either type keeps its subtree for later, at the cost of a copy.

**`coerce: true`** is for formats whose every value is text: CSV, environment variables, query
strings, command-line flags. With it, a string also builds a number (as `String.toInt` and
`String.toFloat` read it), a `bool` (`1`, `t`, `T`, `TRUE`, `true`, `True`, `0`, `f`, `F`, `FALSE`,
`false`, `False`, which is the list Go's `strconv.ParseBool` reads), and `null` from the empty
string.

**`key`** maps a field name to the key the data uses, so `key: Type.snakeCase` reads `userId` from
`user_id`. The library provides `Type.snakeCase` and `Type.kebabCase`. One policy covers the whole
call; renaming one field is a hook's job. Field annotations, below, may replace this option.

**Errors** name the path from the root: `$.servers[2].port: expected int, got 8.5`,
`$.servers[0]: missing field host`, `$.shape: kind "hexagon" names no member of Circle | Rect`. The
path uses `.name` for a field or map key and `[i]` for an array index, as `stdlib.md` already shows
for JSON. `Type.build` stops at the first error, as Go does.

**Panics.** A type that `Type.build` cannot build is a program error, not a data error: one whose
shape is `Type.Func` or `Type.Opaque` (other than `Bytes` and the time types), `Type.Void`,
`Type.Never`, an anonymous record type (which `Dynamic.makeRecord` refuses), a map key of another
type, or a union that breaks the rules above. `Type.build` checks the whole type, with these
options, before it reads the data, and panics with the path to the part, such as
`Type.build: $.onSave: a function cannot be built`. The checker reports the same thing as a compile
error wherever the type is known statically, so `Json.decode(text, type(Handler))` fails `ant check`
when `Handler` holds a function.

### 3. Taking apart: `Type.data`

```
Type.data[T](t: Type[T], value: T, key: func(string): string | null = null,
             tag: string | null = null): Type.Data
```

`Type.data` is the inverse, with the same options, and reads the value with `reflection.md`'s
readers. A record becomes a map with its fields in declaration order, plus the tag entry when it is
a member of a union and `tag` is given; a named primitive becomes its base, a time value its string,
a set an array, integer kinds `int64` (unsigned ones `uint64`), float kinds `float64`, and map keys
their string form.

Since values are never ambiguous, `Type.data` accepts every union, including ones `Type.build`
cannot build: `UserId | string` encodes fine and only fails to decode. It panics on the other types
`Type.build` panics on, and on a cyclic value with `cyclic value`, as `Json.stringify` does.

For every type `Type.build` accepts, `Type.build(t, Type.data(t, v))` with the same options gives
back a value `==` to `v`, except where `v` holds a `NaN`, which is never `==` to anything.

### 4. Hooks: a type takes over its own conversion

Three function names on a record or a named primitive type are hooks:

```ant
func T.fromData(d: Type.Data): Result[T, Error]   // replaces the rules above for T
func T.toData(v: T): Type.Data                    // replaces them in Type.data
func T.validate(v: T): Result[T, Error]           // runs after T is built, either way
```

The functions on a type are fields of its tag at run time (section 6, Functions on types), so
`Type.build` and `Type.data` find them there. The rules:

- **`fromData` and `toData` come as a pair.** A type that declares one declares the other, so that
  `Type.data` writes what `fromData` reads and round trips hold. The checker enforces the pair.
- **Signatures are checked.** A function on a type with one of these names must have the hook's
  signature, so a function cannot become a hook by accident.
- **Only nominal types have hooks.** An alias is interchangeable with its body (section 4), so a
  hook on `Shape` would fire where a program wrote `Shape` and not where it wrote `Circle | Rect`,
  for the same type. Unions use `tag`, or a hook on the record that holds them.
- **A hook applies wherever its type appears in the tree**, at any depth, and only there. The call's
  options do not pass into a hook; a hook that calls `Type.build` passes its own.
- **A hook's error gets the path where the hook ran.** When the message itself starts with a path,
  as one from an inner `Type.build` does, the two are joined: `$.servers[2]` and `$.port: …` give
  `$.servers[2].port: …`.

Hooks are format-neutral: one `Port.fromData` serves JSON, YAML and the environment alike. A format
that needs a per-format override, such as JSON writing `Time.Instant` differently from YAML, keeps a
codec table keyed by type value on the `Dynamic` path, as `reflection.md` sketches.

A legacy format that stores a port as a string:

```ant
type Port = uint16

func Port.fromData(d: Type.Data): Result[Port, Error] {
  if d is string {
    var n = String.toInt(d) ?? return Result.err(Error{ message: "expected a port, got ${d}" })
    if n < 0 || n > 65535 { return Result.err(Error{ message: "port ${n} is out of range" }) }
    return Result.ok(Port(uint16(n)))
  }
  Result.ok(try Type.build(type(uint16), d) | Port)
}

func Port.toData(p: Port): Type.Data { int64(p) }
```

An invariant:

```ant
func Server.validate(s: Server): Result[Server, Error] {
  if s.port < 1 || s.port > 65535 { return Result.err(Error{ message: "port ${s.port} is out of range" }) }
  Result.ok(s)
}
```

A hook on a generic type is not called in this proposal, since it would need its type arguments (see
Later).

### 5. JSON on top

```
Json.decode[T](text: string, t: Type[T], key: func(string): string | null = null,
               tag: string | null = null): Result[T, Error]                       // Type.build(t, Json.toData(try Json.parse(text)), …)
Json.encode[T](t: Type[T], value: T, indent: string = "", key: func(string): string | null = null,
               tag: string | null = null): string                                 // Json.stringify(Json.fromData(Type.data(t, value, …)), indent)
Json.toData(v: Json.Value): Type.Data                                             // a copy
Json.fromData(d: Type.Data): Json.Value                                           // integers become float64; panics on Bytes
```

JSON gets no conventions of its own: a union of records is tagged with whatever key the program
passes, as serde's `#[serde(tag = "kind")]` does, so `{"kind": "circle", "radius": 2}` decodes with
`tag: "kind", key: Type.snakeCase`. `Json.stringify` and `Json.parse` stay as they are.
`Json.encode` differs from `Json.stringify` in calling `toData` hooks, writing the tag entry,
writing integer map keys as strings, and checking the value against the type. A JSON that carries
`Bytes` as base64 is a different policy and walks the type with `Dynamic` instead.

The sentence of `stdlib.md` that defers typed decoding is replaced by these four functions.

## Writing a format library

A format that accepts this policy is a converter. Two sketches:

A CSV reader with a header line, one record per row:

```ant
// Decodes CSV text with a header line into one T per row. A short row leaves the
// remaining fields to their defaults.
export func decodeRows[T](text: string, row: Type[T]): Result[Array[T], Error] {
  var rows = try Csv.parse(text)
  if len(rows) == 0 { return Result.ok([]) }
  var header = rows[0]
  var out: Array[T] = []
  for { index, value: cells } in Array.entries(Array.slice(rows, 1, len(rows))) {
    var fields: Map[string, Type.Data] = Map{}
    for { index: i, value: name } in Array.entries(header) {
      if i < len(cells) { fields[name] = cells[i] }
    }
    var item = try (Type.build(row, fields, coerce: true) | Result.wrapErr("row ${index + 2}"))
    Array.push(out, item)
  }
  Result.ok(out)
}
```

A `--help` text from an options record, with `reflection.md`'s `Type.shape` alone:

```ant
// Lists the flags of an options record, one per line.
export func usage[T](options: Type[T]): string {
  var s = Type.shape(Type.erase(options))
  if !(s is Type.Record) { panic("usage: options must be a record") }
  s.fields
    | Array.map({ "  --${Type.kebabCase(x.name)}${if x.hasDefault { "" } else { " (required)" }}" })
    | String.join("\n")
}
```

The same pattern gives environment variables (`Type.shape` for the field names, `Type.build` with
`coerce: true`), YAML and TOML (their tree to `Type.Data`), and JSON Schema (a walk over
`Type.shape`).

## Under consideration: field annotations

Not part of the proposal yet; written down to decide later. The `key` option renames every field of
a call by one policy, which is too blunt: a program that talks to an API wants `maxTokens` in Ant
and `max_tokens` on the wire, and the name belongs next to the field, not at every call. Field
annotations would put it there. They would also answer `reflection.md`'s open question 5, Field
metadata, so the syntax and its reflection belong there, and the annotations this layer honors
belong here.

### Typed annotations

An annotation is a value of a declared type, a named primitive or a record, written before a field
with `@` and the type's constructor call or literal:

```ant
type Language = {
  name: string,
  tagline: string,
  @Type.Key("key_features") keyFeatures: Array[string],
  @Type.Key("code_example") codeExample: string,
  @Type.Tag("kind") shape: Shape,
}
```

The annotation types are ordinary declarations. The prelude would declare the ones `Type.build` and
`Type.data` honor, and any library declares its own:

```ant
type Type.Key = string            // the key the field reads and writes, in every format
type Type.Tag = string            // the entry that names the member of a union of records
type Description = string         // a JSON Schema library's own annotation
```

A library reads an annotation back by its type, which keeps the result typed without a top type:

```
Type.annotation[A](record: Type.Erased, field: string, a: Type[A]): A | null
```

```ant
var desc = Type.annotation(t, "keyFeatures", type(Description))   // Description | null
```

The rules this would need:

- **Checked like any expression.** An unknown annotation type is a compile error, and so are
  arguments of the wrong type and two annotations of one type on one field. The checker catches
  `@Type.Kee(…)`, which a string tag never would.
- **Constant.** An annotation is a literal, a constructor call or a record literal of literals, and
  names no variable, as a field default may name no variable of its block. The interpreter evaluates
  it once, when the type is first resolved.
- **On fields only.** An annotation on a `type` declaration would bring back the problem of hooks on
  aliases: `Shape` and `Circle | Rect` are one type, and an annotation on `Shape` would make them
  behave differently. A field annotation belongs to the field, whatever its type.
- **Grammar.** `@` is a new token, and it only starts an annotation, before a field name in a record
  type, so the grammar still needs one token of lookahead.

### What it would change in this proposal

- `@Type.Key` replaces the `key` option, and `Type.snakeCase` and `Type.kebabCase` go, so that a
  renamed field has one obvious spelling.
- `@Type.Tag` on a field replaces the `tag` option. A root value that is itself a union of records
  then has no field to carry the tag; it is either wrapped in a record or keeps a call option for
  the root alone.
- Later candidates: `@Type.Skip` for a field that is never read or written, and descriptions for
  schemas and `--help`, unless field doc comments come first.

### Alternatives

- **Go struct tags**, `` keyFeatures: Array[string] `json:"key_features"` ``. A string that each
  library parses at run time, so a misspelt key or option fails silently, and each format has its
  own tag key, so one rename is written once per format.
- **Free-form annotations**, `@json("key_features")`, with `json` a bare word. Shorter, but as
  unchecked as a tag: `@jsno(…)` compiles, two libraries can claim one word, and the arguments have
  no type.
- **Per-format keys**, `@Json.Key` and `@Yaml.Key`, as Go has. Only for formats that need different
  names for one field, which is rare; a format library could later pass its own annotation type to
  `Type.build` to override `@Type.Key`.
- **Field names as the wire names**, `max_tokens: int`. It works today and needs nothing, but the
  wire's spelling leaks into every use in Ant.

## Costs

- **A policy in the standard library.** Every rule in section 2 is a decision that some format will
  disagree with. The `Dynamic` path is the answer for those formats, but users will meet this policy
  first and expect every format to follow it.
- **Hooks reserve three names.** A function on a type named `fromData`, `toData` or `validate` must
  have the hook's signature. A type cannot have a `validate` that is not a hook.
- **A second walk beside `Dynamic`'s.** If `Type.build` is native, it duplicates in Go what Ant code
  over `Dynamic` could do. The reference semantics are the Ant ones, and a test keeps the two in
  step (see Implementation).

## Spec changes

`ant-lang.md`, on top of `reflection.md`'s changes:

- §6 Functions on types: the three hook names, their signatures, the pair rule, and nominal types
  only.
- §11, the Type subsection: `Type.Data`, `Type.build`, `Type.data`, `Type.snakeCase`,
  `Type.kebabCase`, the build table and the union rules.

`stdlib.md`: `Json.decode`, `Json.encode`, `Json.toData`, `Json.fromData`; the deferral sentence
removed.

`doc.md`: `ant doc` lists the hooks of a type with its other functions.

## Implementation

- **Reference.** Write `Type.build` and `Type.data` first as Ant code over `reflection.md`'s
  `Dynamic`, extending its JSON sketch to `Type.Data`. That pins the semantics down in terms of the
  lower layer and proves that layer sufficient. Calling hooks from Ant needs `Dynamic.call` or a
  codec table (`reflection.md`, open question 4); the reference uses a table built from the tags.
- **Native.** Then implement both in Go over the same descriptors, since every format on this path
  goes through them. `reflection.md` expects the Ant walk to be several times slower than
  hand-written decoders; native code removes the box per node. Run the reference and the native
  version against each other on generated values of generated types.
- **Checker.** Check hook signatures and pairs; report types that `Type.build` cannot build where
  they are known.
- **JavaScript backend.** The Ant reference runs there unchanged, and a native port is optional.
- **Tests.** The build table row by row, both directions; round trips for every buildable kind, with
  and without `key` and `tag`; recursive, generic and local types; every union rule and the panic of
  each broken one; a union that encodes but does not decode; every hook, the pair rule and nested
  error paths; `coerce`.
- **Benchmarks.** `Json.decode` against a hand-written decoder on a 1 MB configuration and a 10 MB
  array of records, for both the reference and the native version, recorded here before acceptance.

## Non-goals

- **No mechanism.** Type values, shapes, boxes and builders are `reflection.md`'s.
- **No policy for every format.** A format that disagrees with section 2 uses `Dynamic`.
- **No per-field attributes in this draft.** Field annotations above are a candidate to decide
  separately.

## Alternatives considered

- **Policy in the mechanism.** An earlier draft of this proposal, as `reflect.md`, put type values,
  type descriptions and `Type.build` in one design. It could not express a format whose rules
  differ, such as JSON with base64 `Bytes` or XML attributes, because `Type.build` was the only way
  to make a value. `reflection.md`'s policy-free `Dynamic` underneath fixes that, and this proposal
  became the layer above it.
- **Every format on `Dynamic` alone.** Sound and flexible, but each format repeats the walk and its
  decisions, and a program that reads one type from a file and from the environment meets two
  policies.
- **A built-in `Json.decode` only.** It solves JSON and nothing else, and the next format needs the
  same work again.
- **Trying union members in order**, as `reflection.md`'s sketch does as a placeholder. The data
  then decides which member a value is, so `"3"` into `int | string` depends on member order, and
  adding a member can change what old data decodes to. Deciding by type makes the choice visible
  when the type is written.
- **Hooks on aliases.** They made `Shape` and `Circle | Rect` build differently although section 4
  makes them the same type, and the run time would have had to keep alias identity. `tag` covers the
  common case without either.
- **A `"$type"` key in JSON.** It claimed a key in every JSON document, including ones that use
  `$type` for something else, as Newtonsoft.Json payloads do, and it was JSON's alone. The
  format-neutral `tag` option replaces it.
- **A tagged node in `Type.Data`.** With `tag` deciding records and the union rules deciding
  everything else by type, no union is left that the data must disambiguate.

## Later

- **Hooks on generic types**, which would need a signature that receives the type arguments, such as
  `func Page.fromData[T](d: Type.Data, t: Type[Page[T]])`.
- **A strict mode** for unknown keys, as Go's `DisallowUnknownFields`.
- **Every error at once**, as pydantic reports, as one message with a line per error.
- **Non-string map keys and time values in `Type.Data`**, for MessagePack and CBOR keys and for TOML
  and YAML dates.
- **Omitting defaults on encode**, `omitempty` in spirit, once `reflection.md` exposes defaults (its
  open question 6).

## Open questions

1. **Hooks by name, by registration, or by interface.** Names are discovered for free, and the
   signature check stops an accidental match from compiling. A registration call, such as
   `Type.hooks(type(Port), fromData: …)`, is explicit but needs a place to run. `interfaces.md`
   lists these hooks as candidate prelude interfaces with checked signatures; whichever it decides
   for accidental satisfaction should hold here too.
2. **Tag values.** This draft matches the tag entry to the member's declared name after the `key`
   policy. A program whose discriminator values are not derived from its type names, such as `"c"`
   for `Circle`, needs a hook on the record that holds the union. Is a map from values to members,
   `tags: Map{ "c": type Circle, … }`, worth adding?
3. **Field annotations.** Adopt the typed `@` annotations above in place of `key` and `tag`, keep
   the call options, or both?
4. **Namespace.** This draft puts `Type.Data`, `Type.build` and `Type.data` in `reflection.md`'s
   `Type` namespace, which adds no prelude name but mixes the policy layer into the mechanism's
   namespace. A namespace of its own, such as `Data.build`, reads well but claims a common name that
   no program could then declare.
