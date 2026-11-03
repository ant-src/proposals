# Proposal: types as values

2026-11-03 · Anton

## Summary

Let a program pass a type to a function, and let that function read the type and build values of it.
Three changes do this:

1. **`ant run` keeps type declarations whole.** A record tag keeps its field types, an alias keeps
   its body, and a generic declaration keeps its type parameters. Nothing is checked at run time;
   the bodies are only resolved when a program asks for them.
2. **A type expression, `type X`, is a value of the new prelude type `Type[T]`.** It names any type:
   `type Config`, `type Array[User]`, `type Page[User]`, `type (int | null)`.
3. **The `Type` namespace reads and builds.** `Type.info` describes a type as plain data.
   `Type.build` turns a format-neutral tree, `Type.Data`, into a checked value of the type, and
   `Type.data` turns a value back into a tree. A record or a named primitive type can take over its
   own conversion by declaring `fromData` and `toData`, and check itself by declaring `validate`.

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

### What it changes in the principles

Section 1 says: "**Types are advisory.** `ant check` reads annotations and `ant run` discards them.
The interpreter keeps only a type's name, as a tag, and a record's field names in order. No
evaluation rule consults a written type." This proposal breaks the last sentence on purpose, and the
principle is reworded to say where:

> **Annotations are advisory.** `ant check` reads them and `ant run` discards them, and no
> evaluation rule consults one. A type is consulted only where a program passes it as a value, with
> `type X`.

Ordinary code stays exactly as erased as it is today. Reflection happens only at a visible `type X`,
which is the same kind of rule as "No implicit coercion": a value changes type only through a call
that says so, and `Type.build` is that call.

## Prior art

| Language and library                            | How it decodes into a type                                                                                                  | What Ant takes                                                                                                                             |
| ----------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Go `encoding/json`, `reflect`                   | Run-time reflection on the struct type behind a pointer; struct tags rename fields; `UnmarshalJSON` takes over for one type | The call shape `decode(text, type)`, ignoring unknown keys, and a hook a type declares on itself. Not struct tags, and not zero values     |
| Rust `serde`                                    | A derive macro generates code per type against a small, format-neutral data model; each format implements the model once    | The format-neutral tree, and the internally tagged union (`#[serde(tag = "kind")]`). Not code generation                                   |
| Python pydantic                                 | The class annotation is the schema, read at run time; `model_validate`, `field_validator`, `model_json_schema`              | The declaration is the schema; `validate` as a hook; schema export from the same description                                               |
| Python cattrs                                   | `structure(data, cls)` and `unstructure(value)` between classes and plain dicts and lists, format-independent               | Almost exactly `Type.build` and `Type.data`                                                                                                |
| TypeScript with Zod or Valibot                  | Types are erased, so the schema is a value and the type is computed from it (`z.infer`)                                     | Nothing. Computing a type from a value needs type-level functions and mapped types, which Ant does not have and this proposal does not add |
| Gleam `gleam/dynamic/decode`, Elm `Json.Decode` | Types are erased; decoders are composed by hand from combinators                                                            | The lesson that erasure alone pushes a language to hand-written decoders. That is where Ant is today                                       |
| Swift `Codable`, Kotlin `kotlinx.serialization` | The compiler synthesizes encode and decode per type; each format is an encoder and a decoder                                | The split between types and formats. Not synthesis, since `ant run` has no compile step                                                    |
| Java Jackson                                    | Reflection, with annotations for names, converters and polymorphic type ids                                                 | Reflection as the one path                                                                                                                 |
| .NET `System.Text.Json`                         | Reflection, or source generation as an optional fast path                                                                   | Nothing beyond Jackson's lesson. Speed is not the goal of Ant's interpreter                                                                |

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
resolved lazily, in the scope of the declaration, the first time `Type.info`, `Type.build`,
`Type.data` or `==` on a type value reaches them, and the result is cached for the run. A local type
resolves in its block, as its literals do. Resolution is safe from any goroutine, including several
at once on the same type.

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
  the type as a parameter instead, and its caller writes it (see Generic functions below). Under
  `ant run` the expression panics with `T is a type parameter; take a Type parameter instead`.
- **A statement that starts with `type` is still a declaration.** So a block whose value is a type
  expression ends in `(type Config)`, in parentheses. A bare type expression as a statement would do
  nothing, so nothing else is lost.
- **Type arguments are still never written at a call.** The type is an argument like any other, and
  `T` is solved from it: `Json.decode(text, type Config)` returns `Result[Config, Error]`.

A type value is immutable. `==` compares the resolved types, with aliases expanded, so
`type Page[User] == type Page[User]` is `true`, `type Id == type (int | string)` is `true` for
`type Id = int | string`, and two local types with the same name in different blocks are different
types. A type value can be a map key or a set member. `String.from(type Page[User])` is
`type Page[User]`. `Type[T]` is invariant, like every generic type. It has no `is` test and appears
once in a union.

### 3. Generic functions

A generic function that needs a type takes it as a parameter, typed with as much of the type as it
knows, and the caller writes the whole type:

```ant
func fetchPage[T](url: string, page: Type[Page[T]]): Result[Page[T], Error] {
  var reply = try Http.get(url)
  Json.decode(reply.body, page)
}

var users = try fetchPage("https://example.com/users", type Page[User])   // T = User
```

`T` is solved from the argument, as for any generic parameter. This covers envelopes, wrappers and
containers without any way to build `Type[Page[T]]` from a `Type[T]` inside the function, which this
proposal does not add (see Later).

### 4. Describing a type: `Type.info`

```
type Type.Info = Type.Scalar | Type.Named | Type.Record | Type.Array | Type.Map | Type.Set
               | Type.Union | Type.Other

type Type.Scalar = { name: string }                                   // "bool", "int64", "string", "null", "Bytes", "Time.Instant", …
type Type.Named  = { name: string, base: Type.Scalar }                // a named primitive type: UserId over string
type Type.Record = { name: string, args: Array[Type.Info], fields: Array[Type.Field] }
type Type.Field  = { name: string, info: Type.Info, hasDefault: bool }
type Type.Array  = { element: Type.Info }
type Type.Map    = { key: Type.Info, value: Type.Info }
type Type.Set    = { member: Type.Info }
type Type.Union  = { name: string, members: Array[Type.Info] }        // the alias as written, Result[Config, Error], or ""
type Type.Other  = { name: string }                                   // a function, sync object, resource, Html, Regex.Pattern, void, Type

Type.info[T](t: Type[T]): Type.Info
```

`Type.info` returns plain records that a program reads with `is` and field access. Aliases are
expanded. A union keeps the name of the alias it was written as, with its type arguments, so that a
schema generator can call it `Shape` or `Result[Config, Error]` rather than listing the members; the
name is a label only, and nothing else in this proposal depends on it. An anonymous record type is a
`Type.Record` whose name is `""`. A record's `name` includes its type arguments, `Page[User]`, and
`args` lists them.

Each call builds a fresh graph, so a caller may change it without affecting anyone else. Within one
graph each named type is one record, so a recursive type is a cyclic graph:
`type Tree = { value: int, children: Array[Tree] }` gives a `Type.Record` whose `children` field is
an array of that same record. `Record.same` tells a program it has seen a node before.

`Type.info` is enough on its own for a schema generator, a `--help` text, a form, or the
documentation of an HTTP endpoint.

### 5. The data tree: `Type.Data`

```
type Type.Data = null | bool | int64 | uint64 | float64 | string | Bytes
               | Array[Type.Data] | Map[string, Type.Data]
```

`Type.Data` is the one shape every format converts to and from. It is a closed union, as
`Json.Value` is, so it needs no top type (section 4, Assignability, "There is no top type"). It has
the three number kinds that formats carry: JSON and YAML numbers are `float64`, while MessagePack,
CBOR and TOML keep integers. It has `Bytes` for binary formats. Time values travel as strings in the
form `String.from` gives them, which is also what `Json.stringify` writes.

A format library has two jobs: turn its parsed form into a `Type.Data`, and turn a `Type.Data` into
text. For JSON both are a copy that changes nothing at run time, since every `Json.Value` is also a
`Type.Data` value; the checker still needs the copy, because `Array[Json.Value]` is not an
`Array[Type.Data]`.

### 6. Building: `Type.build`

```
Type.build[T](t: Type[T], data: Type.Data, coerce: bool = false,
              key: func(string): string | null = null, tag: string | null = null): Result[T, Error]
```

`Type.build` walks the type and the data together and builds a value of `T`. It is the only way this
proposal gives to create a value from a type at run time, and the value it returns has been checked
against the type all the way down, so its static type is sound.

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
its zero value, because Ant has no zero values. A default is evaluated as in a literal.

**Map keys.** `K` must be `string`, an integer kind, or a named primitive over one of them. An
integer key is read from its decimal string, so `{"1": "a"}` builds a `Map[int, string]`, as in Go.

**Unions** are decided by the type, so that the data never makes the choice ambiguous. Each member
reads one data kind: a record or a map reads a map, an array or a set an array, a string, a time
type or a named primitive over `string` a string, and so on. A union is buildable when no two
members read the same kind, with three refinements:

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

A union that breaks these rules, such as `UserId | string` or `Circle | Rect` without a tag, is a
program error, not a data error, and panics like the types in Panics below. A program that needs
another shape decodes that part into a `Type.Data` field and finishes it by hand, or declares a hook
on the record that holds it. `Type.Data` and `Json.Value` are buildable under these rules, so a
field of either type keeps its subtree for later, at the cost of a copy.

**`coerce: true`** is for formats whose every value is text: CSV, environment variables, query
strings, command-line flags. With it, a string also builds a number (as `String.toInt` and
`String.toFloat` read it), a `bool` (`1`, `t`, `T`, `TRUE`, `true`, `True`, `0`, `f`, `F`, `FALSE`,
`false`, `False`, which is the list Go's `strconv.ParseBool` reads), and `null` from the empty
string.

**`key`** maps a field name to the key the data uses, so `key: Type.snakeCase` reads `userId` from
`user_id`. The library provides `Type.snakeCase` and `Type.kebabCase`. One policy covers the whole
call; renaming one field is a hook's job.

**Errors** name the path from the root: `$.servers[2].port: expected int, got 8.5`,
`$.servers[0]: missing field host`, `$.shape: kind "hexagon" names no member of Circle | Rect`. The
path uses `.name` for a field or map key and `[i]` for an array index, as `stdlib.md` already shows
for JSON. `Type.build` stops at the first error, as Go does.

**Panics.** A type that no data can build is a program error, not a data error: one that contains a
function, a sync object, a resource, `Html`, `Regex.Pattern`, `void`, a `Type`, an anonymous record
type (which has no tag to give the value), a map key of another type, or a union that breaks the
rules above. `Type.build` checks the whole type, with these options, before it reads the data, and
panics with the path to the part, such as `Type.build: $.onSave: a function cannot be built`. The
checker reports the same thing as a compile error wherever the type is known statically, so
`Json.decode(text, type Handler)` fails `ant check` when `Handler` holds a function.

### 7. Taking apart: `Type.data`

```
Type.data[T](t: Type[T], value: T, key: func(string): string | null = null,
             tag: string | null = null): Type.Data
```

`Type.data` is the inverse of `Type.build`, with the same options. A record becomes a map with its
fields in declaration order, plus the tag entry when it is a member of a tagged union; a named
primitive becomes its base, a time value its string, a set an array, integer kinds `int64` (unsigned
ones `uint64`), float kinds `float64`, and map keys their string form. It panics on the types
`Type.build` panics on, on a cyclic value with `cyclic value`, as `Json.stringify` does, and, in an
unchecked program, on a value that does not fit `t`, with its path.

For every type both accept, `Type.build(t, Type.data(t, v))` with the same options gives back a
value `==` to `v`, except where `v` holds a `NaN`, which is never `==` to anything.

### 8. Hooks: a type takes over its own conversion

Functions on types are fields of the type's tag at run time (section 6, Functions on types), so
`Type.build` and `Type.data` can find them. Three names are hooks, on a record or a named primitive
type:

```ant
func T.fromData(d: Type.Data): Result[T, Error]   // replaces the rules above for T
func T.toData(v: T): Type.Data                    // replaces them in Type.data
func T.validate(v: T): Result[T, Error]           // runs after T is built, either way
```

The rules:

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

A legacy format that stores a port as a string:

```ant
type Port = uint16

func Port.fromData(d: Type.Data): Result[Port, Error] {
  if d is string {
    var n = String.toInt(d) ?? return Result.err(Error{ message: "expected a port, got ${d}" })
    if n < 0 || n > 65535 { return Result.err(Error{ message: "port ${n} is out of range" }) }
    return Result.ok(Port(uint16(n)))
  }
  Result.ok(try Type.build(type uint16, d) | Port)
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

### 9. JSON on top

```
Json.decode[T](text: string, t: Type[T], key: func(string): string | null = null,
               tag: string | null = null): Result[T, Error]                       // Type.build(t, Json.toData(try Json.parse(text)), …)
Json.encode[T](value: T, t: Type[T], indent: string = "", key: func(string): string | null = null,
               tag: string | null = null): string                                 // Json.stringify(Json.fromData(Type.data(t, value, …)), indent)
Json.toData(v: Json.Value): Type.Data                                             // a copy
Json.fromData(d: Type.Data): Json.Value                                           // integers become float64; panics on Bytes
```

JSON gets no conventions of its own: a union of records is tagged with whatever key the program
passes, as serde's `#[serde(tag = "kind")]` does, so `{"kind": "circle", "radius": 2}` decodes with
`tag: "kind", key: Type.snakeCase`. `Json.stringify` and `Json.parse` stay as they are.
`Json.encode` differs from `Json.stringify` in calling `toData` hooks, writing the tag entry,
writing integer map keys as strings, and checking the value against the type.

The sentence of `stdlib.md` that defers typed decoding is replaced by these four functions.

## Writing a format library

The point of the design is that the next format is ordinary code. Two sketches:

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

## Costs

- **A second type resolver.** The interpreter gains name lookup in a declaration's scope, alias
  expansion and type-argument substitution, beside the checker's. It needs no inference and no
  assignability, so it should be a few hundred lines of Go, but it is a second place where Ant's
  type language is implemented, and the two must agree. Tests that run the same reflected types
  through `ant check` and `ant run` keep them honest.
- **Memory.** Every declaration keeps its type syntax tree for the run, reflected or not.
- **A principle reworded.** Section 1's "No evaluation rule consults a written type" becomes "no
  evaluation rule consults an annotation", as Motivation says. The interpreter is no longer purely
  type-blind, which is the main cost against keeping it simple.
- **Breaking names.** `Type` joins the prelude, and a prelude name cannot be shadowed (section 3),
  so a program that declares its own `Type` must rename it. A function on a type named `fromData`,
  `toData` or `validate` must now have the hook's signature.

## Spec changes

`ant-lang.md`:

- §1 Principles: "Types are advisory" becomes "Annotations are advisory", as worded in Motivation.
- §4 Checking and running: a `type` declaration keeps its whole body, resolved on demand; "the rest
  of the body is discarded" goes.
- §4 Records: "it keeps no field type" goes.
- §4 Kinds: a row for `Type[T]`, kind type, no `is`, `==` on the resolved type, rendered as
  `type Page[User]`, once in a union.
- §4 Type declarations: the paragraph that says a type name has no value gains the type expression.
- §5 Expressions: a new subsection, Type expressions, with the rules of section 2 above, and the
  generic-function idiom of section 3.
- §6 Functions on types: the three hook names, their signatures, the pair rule, and nominal types
  only.
- §11: a new subsection, Type, with `Type[T]`, `Type.Info` and its members, `Type.Data`,
  `Type.info`, `Type.build`, `Type.data`, `Type.snakeCase`, `Type.kebabCase`, the build table and
  the union rules. `Type` joins the prelude names of §10.
- §12 Values: a tag also keeps its body; a new kind, type. The panics table gains: an unknown name
  in a type body, a type parameter in a type expression, a generic type without arguments, a type
  `Type.build` cannot build, and a value `Type.data` cannot take apart.

`stdlib.md`: `Json.decode`, `Json.encode`, `Json.toData`, `Json.fromData`; the deferral sentence
removed.

`lsp.md`, `fmt.md`, `doc.md` and `dap.md`: hover shows `Type[…]` for a type expression; `fmt`
formats the type after `type` as it formats an annotation; `ant doc` lists the hooks of a type with
its other functions; the debugger shows a type value as `String.from` renders it.

## Implementation

- **Parser.** `type` in expression position parses one type, with a parenthesized union.
- **Interpreter.** Keep the type syntax tree on each tag and constructor. Add the resolver from that
  tree, in the declaration's scope, to a run-time type descriptor with type arguments substituted,
  cached per tag and argument list under a lock. `Type.info`, `Type.build` and `Type.data` are Go
  functions over descriptors. Code that does not reflect pays nothing beyond the kept trees.
- **Checker.** Type the expression as `Type[T]`; reject type parameters and missing type arguments
  in it; check hook signatures and pairs; report types that `Type.build` cannot build where they are
  known.
- **JavaScript backend.** Emit a descriptor per type declaration, built lazily on first use.
- **Editors.** Highlight `type` in expression position as a keyword; complete type names after it.
- **Tests.** The build table row by row, both directions; round trips for every buildable kind, with
  and without `key` and `tag`; recursive, generic and local types; every union rule and the panic of
  each broken one; every hook, the pair rule and nested error paths; `coerce`; resolution from many
  goroutines at once; an unchecked program whose type body names an unknown type; and the same
  reflected types under `ant check` and `ant run`.

## Non-goals

- **No dynamic field access.** `p["x"]` on a record stays a compile error. Values are built only
  through `Type.build`, which checks them.
- **No top type.** `Type.Data` is a closed union, as `Json.Value` is.
- **No run-time checking of ordinary code.** Annotations on variables, parameters and results are
  still discarded.
- **No per-field attributes.** There is no syntax for `json:"user_id"` or `Field(min=1)`; the `key`
  policy, `tag` and the hooks cover the same ground.
- **No reified type parameters.** A generic function that needs its type takes it as a parameter.

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
- **Hooks on aliases.** An earlier draft let `func Shape.fromData` decide a union of records. It
  made `Shape` and `Circle | Rect` build differently although section 4 makes them the same type,
  and the run time would have had to keep alias identity. `tag` covers the common case without
  either.
- **A `"$type"` key in JSON.** An earlier draft had `Json` mark union members with `"$type"`. It
  claimed a key in every JSON document, including ones that use `$type` for something else, as
  Newtonsoft.Json payloads do, and it was JSON's alone. The format-neutral `tag` option replaces it.
- **A tagged node in `Type.Data`.** With `tag` deciding records and the union rules deciding
  everything else by type, no union is left that the data must disambiguate, so `Type.Data` needs no
  node of its own for it.
- **`type Page[T]` inside a generic function**, filled in from a `Type[T]` argument. It would make
  the run time read parameter annotations to know which argument holds `T`, which is more erasure
  broken, not less. The caller writing the whole type, as in section 3, covers the same programs.
- **The name `Reflect`.** It would break fewer programs than `Type`, but `Reflect[Config]` reads
  worse than `Type[Config]` in every signature that takes one, and Ant is young enough to take the
  break.

## Later

- **Constructors on type values**, such as `Type.array(t)` and `Type.nullable(t)`, for a generic
  function that gets a `Type[T]` and needs a `Type[Array[T]]`. Wait for a program that needs one.
- **Hooks on generic types**, which would need a signature that receives the type arguments, such as
  `func Page.fromData[T](d: Type.Data, t: Type[Page[T]])`.
- **Doc comments in `Type.Info`**: a `doc` on records, and on fields once fields can have doc
  comments, so that generated `--help` text and schemas describe themselves.
- **A strict mode** for unknown keys, as Go's `DisallowUnknownFields`.
- **Every error at once**, as pydantic reports, as one message with a line per error.
- **Non-string map keys and time values in `Type.Data`**, for MessagePack and CBOR keys and for TOML
  and YAML dates.
- **A shared, frozen `Type.info` graph**, if `frozen.md` is accepted.

## Open questions

1. **Hooks by name or by registration.** Names are discovered for free, and the signature check
   stops an accidental match from compiling, but a type can no longer have a `validate` function
   that is not a hook. A registration call, such as `Type.hooks(type Port, fromData: …)`, is
   explicit but needs a place to run.
2. **Tag values.** This draft matches the tag entry to the member's declared name after the `key`
   policy. A program whose discriminator values are not derived from its type names, such as `"c"`
   for `Circle`, needs a hook on the record that holds the union. Is a map from values to members,
   `tags: Map{ "c": type Circle, … }`, worth adding to `Type.build`?
