# Proposal: types as values

2026-11-03 · Anton

## Summary

Let a program pass a type to a function and let that function read the type and build values of it.
Three changes do this:

1. **`ant run` keeps type declarations whole.** A record tag keeps its field types, an alias keeps
   its body, and a generic declaration keeps its type parameters. Nothing is checked at run time;
   the bodies are only resolved when a program asks for them.
2. **A type expression, `type X`, is a value of the new prelude type `Type[T]`.** It names any type:
   `type Config`, `type Array[User]`, `type Page[User]`, `type (int | null)`.
3. **The `Type` namespace reads and builds.** `Type.info` describes a type as plain data.
   `Type.build` turns a format-neutral tree, `Type.Data`, into a checked value of the type, and
   `Type.data` turns a value back into a tree. A type can take over its own conversion by declaring
   `fromData`, `toData` or `validate` as functions on itself.

With these, decoding JSON into records is a few lines of library code:

```ant
type Server = { host: string, port: int = 8080, tags: Array[string] = [] }
type Config = { name: string, servers: Array[Server] }

var config = try Json.decode(text, type Config)
```

And so is every other format: CSV rows, environment variables, command-line flags, YAML, TOML, a
database row, a JSON Schema. Each one is written as an ordinary library, inside or outside the
standard library, against the same three functions.

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

Every field name is written three times, every default twice (once in the type, once in the
decoder), and the decoder drifts from the type as soon as someone adds a field. The checker cannot
help, because the decoder is correct Ant whatever it decodes.

### One decoder per format is the wrong shape

The same program usually reads its configuration from a file, overrides it from the environment, and
takes a few flags. That is three decoders per type today, and each new format (YAML, TOML, CSV, a
database row) is one more. The work is the same each time: walk the type, match it against what the
input has, convert, and report where it went wrong. Only the format differs.

### Why the language must change

Types are erased by `ant run` (section 4, Checking and running). A tag keeps its field names and
defaults and nothing else, so no library, built in or not, can learn that `port` is an `int`. And a
type is not a value: "It is an expression only as the head of a record literal, as the target of
`is`, as the left side of `.` naming one of its functions, or, for a named primitive type, as its
constructor." So there is no way to tell a decoder which type to build either.

A built-in `Json.decode` alone could cheat around both with an interpreter special case. This
proposal instead adds the smallest general mechanism, so that the next format does not need the next
special case, and so that format libraries can live outside the standard library.

## Prior art

| Language and library                            | How it decodes into a type                                                                                                  | What Ant takes                                                                                                                             |
| ----------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Go `encoding/json`, `reflect`                   | Run-time reflection on the struct type behind a pointer; struct tags rename fields; `UnmarshalJSON` takes over for one type | The call shape `decode(text, type)`, the rules for missing and unknown keys, and a hook a type declares on itself. Not struct tags         |
| Rust `serde`                                    | A derive macro generates code per type against a small, format-neutral data model; each format implements the model once    | The format-neutral tree. Every format converts to and from `Type.Data`, and only `Type.build` knows about types. Not code generation       |
| Python pydantic                                 | The class annotation is the schema, read at run time; `model_validate`, `field_validator`, `model_json_schema`              | The declaration is the schema; `validate` as a hook; schema export from the same description                                               |
| Python cattrs                                   | `structure(data, cls)` and `unstructure(value)` between classes and plain dicts and lists, format-independent               | Almost exactly `Type.build` and `Type.data`                                                                                                |
| TypeScript with Zod or Valibot                  | Types are erased, so the schema is a value and the type is computed from it (`z.infer`)                                     | Nothing. Computing a type from a value needs type-level functions and mapped types, which Ant does not have and this proposal does not add |
| Gleam `gleam/dynamic/decode`, Elm `Json.Decode` | Types are erased; decoders are composed by hand from combinators                                                            | The lesson that erasure alone pushes a language to hand-written decoders. That is where Ant is today                                       |
| Swift `Codable`, Kotlin `kotlinx.serialization` | The compiler synthesizes encode and decode per type; each format is an encoder and a decoder                                | The split between types and formats. Not synthesis, since `ant run` has no compile step                                                    |
| Java Jackson, .NET `System.Text.Json`           | Reflection, with annotations for names and converters, and source generation as an optional fast path                       | Reflection as the one path. Speed is not the goal of Ant's interpreter                                                                     |

Three lessons:

- **Separate the type from the format.** Serde and cattrs show that a small data model in the middle
  turns N formats times M types into N plus M. Go shows the cost of not doing it: `encoding/json`,
  `encoding/xml` and every third-party format reimplement the reflection walk.
- **Erased languages end up either with schemas as values or with hand-written decoders.** Zod needs
  type-level computation to get the type back; Gleam and Elm write decoders by hand. Keeping the
  declaration at run time, as Python does, avoids both.
- **A type needs a way out.** Every system above has a per-type override (`UnmarshalJSON`, a serde
  `impl`, a pydantic validator) for the shapes the default rules cannot express.

## The proposal

### 1. `ant run` keeps type declarations

A `type` declaration keeps its parsed body. For a record that is the type of each field, for an
alias the aliased type, and for a generic declaration its type parameters. Names in a body are
resolved lazily, in the scope of the declaration, the first time `Type.info`, `Type.build` or
`Type.data` reaches them, and the result is cached for the run. A local type resolves in its block,
as its literals do.

Nothing else changes for code that does not reflect. Annotations on variables, parameters and
results are still discarded, and nothing is checked when a value is created. In an unchecked
program, a body that names an unknown type panics when it is first resolved, as an unknown name
panics when it is reached.

### 2. Type expressions

`type` followed by a type, in any place an expression is expected, is a value of type `Type[T]`,
where `T` is that type.

```ant
var a = type Config                    // Type[Config]
var b = type Array[Server]             // Type[Array[Server]]
var c = type Page[User]                // a generic type with its arguments
var d = type Map[string, int]
var e = type (string | null)           // a union in parentheses
var f = type Json.Value                // a type reached through a namespace
```

The rules:

- **A union is written in parentheses.** `type` takes one type, and `|` after it is a pipe, so
  `type Config | describe()` pipes the type value into `describe`. That keeps the two meanings of
  `|` apart without a precedence rule.
- **A generic type needs its arguments.** `type Page` is a compile error, and a panic in an
  unchecked program, because `Page` alone is not a type.
- **A type parameter cannot be named.** Inside `func f[T](…)`, `type T` and `type Array[T]` are
  compile errors, because `T` is erased and the value would not know what it is. The function takes
  a `t: Type[T]` parameter instead, and its caller writes the type once. Under `ant run` the
  expression panics with `T is a type parameter; take a Type[T] parameter instead`.
- **A statement that starts with `type` is still a declaration.** A type expression as a statement
  would do nothing, so nothing is lost.
- **Type arguments are still never written at a call.** The type is an argument like any other, and
  `T` is solved from it: `Json.decode(text, type Config)` returns `Result[Config, Error]`.

A type value is immutable. `==` compares the resolved types, so `type Page[User] == type Page[User]`
is `true`, and two local types with the same name in different blocks are different types.
`String.from(type Page[User])` is `type Page[User]`. `Type[T]` is invariant, like every generic
type. It has no `is` test and appears once in a union.

### 3. Describing a type: `Type.info`

```
type Type.Info = Type.Scalar | Type.Named | Type.Record | Type.Array | Type.Map | Type.Set
               | Type.Union | Type.Other

type Type.Scalar = { name: string }                                   // "bool", "int64", "string", "null", "Bytes", "Time.Instant", …
type Type.Named  = { name: string, base: Type.Scalar }                // a named primitive type: UserId over string
type Type.Record = { name: string, args: Array[Type.Info], fields: Array[Type.Field] }
type Type.Field  = { name: string, type: Type.Info, hasDefault: bool }
type Type.Array  = { element: Type.Info }
type Type.Map    = { key: Type.Info, value: Type.Info }
type Type.Set    = { member: Type.Info }
type Type.Union  = { name: string, members: Array[Type.Info] }        // name is the alias, or "" when written out
type Type.Other  = { name: string }                                   // a function, sync object, resource, Html, Regex.Pattern, void, Type

Type.info[T](t: Type[T]): Type.Info
```

`Type.info` returns plain records that a program reads with `is` and field access. Aliases are
expanded, except that a union keeps the name of the alias it came from, so that a schema generator
can call it `Shape` rather than `Circle | Rect`. An anonymous record type is a `Type.Record` whose
name is `""`. A record's `name` includes its type arguments, `Page[User]`, and `args` lists them.

Each call builds a fresh graph, so a caller may change it without affecting anyone else. Within one
graph each named type is one record, so a recursive type is a cyclic graph:
`type Tree = { value: int, children: Array[Tree] }` gives a `Type.Record` whose `children` field is
an array of that same record. `Record.same` tells a program it has seen a node before.

`Type.info` is enough on its own for a schema generator, a `--help` text, a form, or the
documentation of an HTTP endpoint.

### 4. The data tree: `Type.Data`

```
type Type.Data = null | bool | int64 | uint64 | float64 | string | Bytes
               | Array[Type.Data] | Map[string, Type.Data] | Type.Tagged

type Type.Tagged = { name: string, value: Type.Data }                 // which member of a union, where it is ambiguous
```

`Type.Data` is the one shape every format converts to and from. It is a closed union, as
`Json.Value` is, so it needs no top type (section 4, Assignability, "There is no top type"). It has
the three number kinds that formats carry: JSON and YAML numbers are `float64`, while MessagePack,
CBOR and TOML keep integers. It has `Bytes` for binary formats. Time values travel as strings in the
form `String.from` gives them, which is also what `Json.stringify` writes.

A format library has two jobs: turn its parsed form into a `Type.Data`, and turn a `Type.Data` into
text. For JSON both are nearly the identity, since `Json.Value` is a subset of `Type.Data`.

### 5. Building and taking apart: `Type.build` and `Type.data`

```
Type.build[T](t: Type[T], data: Type.Data, coerce: bool = false,
              key: func(string): string | null = null): Result[T, Error]
Type.data[T](t: Type[T], value: T, key: func(string): string | null = null): Type.Data
```

`Type.build` walks the type and the data together and builds a value of `T`. It is the only way this
proposal gives to create a value from a type at run time, and the value it returns has been checked
against the type all the way down, so its static type is sound.

| Target type              | Accepts                                                                                                                       |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------- |
| `null`, `bool`, `string` | the same kind                                                                                                                 |
| an integer kind          | `int64` or `uint64` in range; a `float64` with no fraction, in range (so the JSON `8080` fits an `int`, and `8.5` does not)   |
| a float kind             | any number, rounded as the conversion to that kind rounds                                                                     |
| a named primitive type   | what its base accepts, then the tag: `"u-1"` into `UserId` is `UserId("u-1")`                                                 |
| `Bytes`                  | `Bytes`                                                                                                                       |
| a time type              | a string in the form that type's `parse` reads                                                                                |
| `Array[T]`               | an array, element by element                                                                                                  |
| `Set[T]`                 | an array, member by member; a repeated member counts once                                                                     |
| `Map[K, V]`              | a map; each key builds as `K`, which must be `string`, a named primitive over `string`, or an integer kind written in decimal |
| a named record type      | a map: each field from the key of its name, or its default when the key is missing                                            |
| a union                  | see below                                                                                                                     |
| `Type.Data`              | anything, kept as it is, for a part the program decodes later                                                                 |

A record ignores keys it has no field for, as Go does. A missing key with no default is an error,
including for a field typed `T | null`, since a record literal needs that field too; a field that
may be absent declares `= null`. A default is evaluated as in a literal.

A union picks its member by the kind of the data. When the data is a `Type.Tagged`, its `name`
chooses the member and its `value` is built as that member. Otherwise the members that accept the
data's kind are the candidates: integer data prefers an integer member and `float64` data a float
member, and after that any numeric member will do. One candidate is built. With none, the error
names what was expected. With several, which happens with two record members, since both accept a
map, the error is `ambiguous: Circle or Rect`, and the type needs a `Type.Tagged` in its data or a
`fromData` hook (below).

`coerce: true` is for formats whose every value is text: CSV, environment variables, query strings,
command-line flags. With it, a string also builds a number (as `String.toInt` and `String.toFloat`
read it), a `bool` (`1`, `t`, `true`, `0`, `f`, `false` in any case, as Go's `strconv.ParseBool`
reads it), and `null` from the empty string.

`key` maps a field name to the key the data uses, so `key: Type.snakeCase` reads `userId` from
`user_id`. The library provides `Type.snakeCase` and `Type.kebabCase`. One policy covers the whole
call; renaming one field is a hook's job.

**Errors** name the path from the root: `$.servers[2].port: expected int, got 8.5`,
`$.servers[0]: missing field host`, `$.mode: expected "fast" or "safe"`. The path uses `.name` for a
field or map key and `[i]` for an array index, as `stdlib.md` already shows for JSON. `Type.build`
stops at the first error, as Go does.

**Panics.** A type that no data can build, because it contains a function, a sync object, a
resource, `Html`, `Regex.Pattern`, `void`, a `Type`, or an anonymous record type (which has no tag
to give the value), is a program error, not a data error. `Type.build` checks the whole type before
reading the data and panics with the path to the part, such as
`Type.build: $.onSave: a function cannot be built`.

`Type.data` is the inverse: a record becomes a map with its fields in declaration order, a named
primitive its base, a time value its string, a set an array, integer kinds `int64` (unsigned ones
`uint64`), float kinds `float64`, and map keys their string form. A record member of a union that
`Type.build` could not tell apart from another is wrapped in a `Type.Tagged`, so
`Type.build(t, Type.data(t, v))` gives back a value `==` to `v` for every type it accepts. A cyclic
value panics with `cyclic value`, as `Json.stringify` does. In an unchecked program, a value that
does not fit `t` panics with its path.

### 6. Hooks: a type takes over its own conversion

Functions on types are fields of the type's tag at run time (section 6, Functions on types), so
`Type.build` and `Type.data` can find them. Three names are hooks:

```ant
func T.fromData(d: Type.Data): Result[T, Error]   // replaces the rules above for T
func T.toData(v: T): Type.Data                    // replaces them in Type.data
func T.validate(v: T): Result[T, Error]           // runs after T is built, either way
```

A hook applies wherever its type appears in the tree, and it may be declared on a record, a named
primitive type, or an alias, including an alias of a union. The checker requires a function on a
type with one of these names to have the hook's signature, so a function cannot become a hook by
accident.

A union that the rules cannot tell apart:

```ant
type Circle = { kind: string, radius: float64 }
type Rect = { kind: string, width: float64, height: float64 }
type Shape = Circle | Rect

func Shape.fromData(d: Type.Data): Result[Shape, Error] {
  if d is Map && d["kind"] == "circle" { return Result.ok(try Type.build(type Circle, d)) }
  if d is Map && d["kind"] == "rect" { return Result.ok(try Type.build(type Rect, d)) }
  Result.err(Error{ message: "expected a circle or a rect" })
}
```

An invariant:

```ant
func Server.validate(s: Server): Result[Server, Error] {
  if s.port < 1 || s.port > 65535 { return Result.err(Error{ message: "port ${s.port} is out of range" }) }
  Result.ok(s)
}
```

A hook's error gets the path where it ran: `$.servers[2]: port 70000 is out of range`. A hook on a
generic type is not called in this proposal, since it would need its type arguments (see Open
questions).

### 7. JSON on top

```
Json.decode[T](text: string, t: Type[T]): Result[T, Error]           // Type.build(t, Json.toData(try Json.parse(text)))
Json.encode[T](value: T, t: Type[T], indent: string = ""): string     // the inverse, through Type.data
Json.toData(v: Json.Value): Type.Data                                 // an object with a string "$type" becomes a Type.Tagged
Json.fromData(d: Type.Data): Json.Value                               // a Type.Tagged becomes an object with "$type"; integers become float64
```

A `Type.Tagged` in JSON is an object with a `"$type"` entry naming the member, merged into the value
when it is an object and beside a `"$value"` entry when it is not:
`{"$type": "Circle", "kind": "circle", "radius": 2}`. The key cannot clash with a field, since `$`
is not an identifier character. `Json.stringify` and `Json.parse` stay as they are; `Json.encode`
differs from `Json.stringify` only in tagging ambiguous union members and writing integer map keys
as strings, and in checking the value against the type.

The sentence of `stdlib.md` that defers typed decoding is replaced by these four functions.

## Writing a format library

The point of the design is that the next format is ordinary code. Two sketches:

A CSV reader with a header line, one record per row:

```ant
// Decodes CSV text with a header line into one T per row.
export func decodeRows[T](text: string, row: Type[T]): Result[Array[T], Error] {
  var rows = try Csv.parse(text)
  if len(rows) == 0 { return Result.ok([]) }
  var header = rows[0]
  var out: Array[T] = []
  for { index, value: cells } in Array.entries(Array.slice(rows, 1, len(rows))) {
    var fields: Map[string, Type.Data] = Map{}
    for { index: i, value: name } in Array.entries(header) { fields[name] = cells[i] }
    var item = try (Type.build(row, fields, coerce: true) | Result.wrapErr("row ${index + 2}"))
    Array.push(out, item)
  }
  Result.ok(out)
}
```

A `--help` text from an options record, using only `Type.info`:

```ant
// Lists the flags of an options record, one per line.
export func usage[T](options: Type[T]): string {
  var info = Type.info(options)
  if !(info is Type.Record) { panic("usage: options must be a record") }
  info.fields
    | Array.map({ "  --${Type.kebabCase(x.name)}${if x.hasDefault { "" } else { " (required)" }}" })
    | String.join("\n")
}
```

The same pattern gives environment variables (`Type.info` for the field names, `Type.build` with
`coerce: true`), YAML and TOML (their tree to `Type.Data`), and JSON Schema (a walk over
`Type.Info`).

## Spec changes

`ant-lang.md`:

- §4 Checking and running: a `type` declaration keeps its whole body, resolved on demand; the
  sentence "the rest of the body is discarded" goes.
- §4 Kinds: a row for `Type[T]`, kind type, no `is`, `==` on the resolved type, rendered as
  `type Page[User]`, once in a union.
- §4 Type declarations: the paragraph that says a type name has no value gains the type expression.
- §5 Expressions: a new subsection, Type expressions, with the rules of section 2 above.
- §6 Functions on types: the three hook names and their signatures.
- §11: a new subsection, Type, with `Type[T]`, `Type.Info` and its members, `Type.Data`,
  `Type.Tagged`, `Type.info`, `Type.build`, `Type.data`, `Type.snakeCase`, `Type.kebabCase`, and the
  build table. `Type` joins the prelude names of §10.
- §12 Values: a tag also keeps its body; a new kind, type. The panics table gains: an unknown name
  in a type body, a type parameter in a type expression, a generic type without arguments, a type
  `Type.build` cannot build, and a value `Type.data` cannot take apart.

`stdlib.md`: `Json.decode`, `Json.encode`, `Json.toData`, `Json.fromData` and the `$type`
convention; the deferral sentence removed.

`lsp.md` and `fmt.md`: hover on a type expression shows `Type[…]`; `fmt` formats the type after
`type` as it formats an annotation.

## Implementation

- **Parser.** `type` in expression position parses one type, with a parenthesized union.
- **Interpreter.** Keep the type syntax tree on each tag and constructor. Add a resolver from that
  tree, in the declaration's scope, to a run-time type descriptor with type arguments substituted,
  cached per tag and argument list. `Type.info`, `Type.build` and `Type.data` are Go functions over
  descriptors. Code that does not reflect pays nothing beyond the kept trees.
- **Checker.** Type the expression as `Type[T]`, reject type parameters and missing type arguments
  in it, and check hook signatures.
- **JavaScript backend.** Emit a descriptor per type declaration, built lazily on first use.
- **Editors.** Highlight `type` in expression position as a keyword; complete type names after it.
- **Tests.** The build table row by row, both directions; round trips for every buildable kind;
  recursive and generic types; local types; each union rule, including the ambiguous case; every
  hook, including on an alias; `coerce` and `key`; error paths; every panic; and an unchecked
  program whose type body names an unknown type.

## Non-goals

- **No dynamic field access.** `p["x"]` on a record stays a compile error. Values are built only
  through `Type.build`, which checks them.
- **No top type.** `Type.Data` is a closed union, as `Json.Value` is.
- **No run-time checking of ordinary code.** Annotations on variables, parameters and results are
  still discarded.
- **No per-field attributes.** There is no syntax for `json:"user_id"` or `Field(min=1)`; the `key`
  policy and the hooks cover the same ground.
- **No reified type parameters.** A generic function that needs its type takes a `Type[T]`.

## Alternatives considered

- **A built-in `Json.decode` only.** It solves JSON and nothing else, and the next format needs the
  same interpreter work again, in the standard library, since nobody else could write it.
- **Decoder combinators** (Gleam, Elm). They need no language change, but they are the hand-written
  decoder with better helpers: every field name and default is still written twice, and the decoder
  still drifts from the type. They remain possible as a library on `Json.Value` for anyone who wants
  them.
- **Schemas as values** (Zod). The type would have to be computed from the schema, which needs
  type-level functions and mapped types. Without them the schema loses its types.
- **Code generation or a derive macro** (serde, Codable). `ant run` parses and runs; it has no
  compile step to hang generation on, and generated files go stale.
- **The type from the expected type**, `var c: Config = try Json.decode(text)`. The checker knows
  `Config` there, but `ant run` discards the annotation, so the call would not know what to build.
- **A bare type name as the argument**, `Json.decode(text, Config)`. It reads well for one named
  record, but it cannot name `Array[Server]`, `Page[User]` or a union, and `Config` already means
  the tag and the namespace in expression position. The `type` keyword makes the one new meaning
  visible.
- **Full reflection** with field reads and writes by name, `Type.make` and the like. It needs a top
  type, and it gives up the guarantee that a reflected value was checked against its type.

## Open questions

1. **Hooks by name or by registration.** Names are discovered for free, and the signature check
   stops an accidental match from compiling, but a type cannot have a `validate` function that is
   not a hook. A registration call, such as `Type.hooks(type Shape, fromData: …)`, is explicit but
   needs a place to run.
2. **Hooks on generic types.** A `func Page.fromData[T](d: Type.Data, t: Type[Page[T]])` form would
   pass the type arguments in. Is that worth a second signature?
3. **Unknown keys.** Go ignores them, and so does this draft. Should `Type.build` take
   `strict: bool = false`, as Go's `DisallowUnknownFields` does?
4. **All errors or the first.** Pydantic reports every failing field; Go stops at the first. `Error`
   has only a message, so all errors would be one message with a line each.
5. **Map keys in `Type.Data`.** `Map[string, Type.Data]` fits JSON, YAML, TOML and environment
   variables. MessagePack and CBOR allow other keys. `Map[Type.Data, Type.Data]` would carry them,
   at the cost of a cast on every key in every other format.
6. **Time values in `Type.Data`.** TOML and YAML have dates of their own. Carrying them as strings
   keeps `Type.Data` small; a time member would keep them exact.
7. **Doc comments in `Type.Info`.** A `doc` on `Type.Record`, and doc comments on fields, which the
   language does not have yet, would make generated `--help` and schemas describe themselves. The
   interpreter would then keep the doc text too.
8. **The name `Type`.** A prelude name cannot be shadowed (section 3), so every program that
   declares its own `Type` breaks. `Reflect` is the alternative, at the cost of `Reflect[Config]`
   reading worse than `Type[Config]` in a signature.
9. **Frozen results.** If `frozen.md` is accepted, `Type.info` could return a frozen graph shared by
   every call instead of a fresh one.
