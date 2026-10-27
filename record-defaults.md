# Proposal: default values for record fields

2026-10-27 · Anton

## Summary

A field of a record type may declare a default, `workers: int = 4`, and a record literal may leave
out any field that has one. A field without a default must still be listed, and an unknown field is
still an error. The default is an expression evaluated each time a literal leaves its field out, in
the scope where the type is declared, as a parameter default is evaluated each time a call leaves
its parameter out (section 12, Call). There are no implicit zero values: `ant run` keeps no field
types, and many types have no zero. A default belongs to literals only; it changes nothing about the
type, its values, assignability or `==`.

```ant
type Server = {
  addr: string,
  workers: int = 4,
  verbose: bool = false,
  tags: Array[string] = [],
}

var s = Server{ addr: ":8080" }                 // workers: 4, verbose: false, tags: []
var t = Server{ addr: ":9090", verbose: true }
var u = Server{ workers: 8 }                    // compile error: missing field addr
```

## Motivation

Section 4 says a record literal "must list exactly the fields of its named type, which catches
typos." The typo half of that rule is worth keeping; the "exactly" half costs more as records grow.

1. **Wide records are verbose to build.** Configuration, options and request records have many
   fields, most of which nearly every caller leaves at one value. Each literal writes them all, so
   the one field that differs is hard to find among the ones that don't.
2. **Adding a field breaks every literal.** A new optional field (a timeout, a flag, a list of tags)
   must be added by hand at every literal in every module that builds the record, though none of
   them has an opinion about it. That makes widening a record an API break.
3. **Functions already have this.** A parameter with a default may be omitted (section 6, Calling),
   and HTML components lean on it ("defaults make them optional", section 5). A record is the other
   way to pass many named values, and the only one that can be stored, built up, or passed on whole,
   but it has no defaults.
4. **The workarounds repeat every field.** Two work today, and both are worse than the feature:

   ```ant
   // a constructor: each field written three times, and a record literal still lists all of them
   func Server.new(addr: string, workers: int = 4, verbose: bool = false, tags: Array[string] = []): Server {
     Server{ addr, workers, verbose, tags }
   }

   // a defaults record and a spread: the required field gets a placeholder, so forgetting it is not caught
   func Server.defaults(): Server { Server{ addr: "", workers: 4, verbose: false, tags: [] } }
   var s = Server{ ...Server.defaults(), addr: ":8080" }
   ```

   The constructor also turns a literal, which the checker and the reader both understand, into a
   call, and gives up the shorthand `Server{ addr }`.

## Prior art

| Language            | Field defaults                                                      | Fields left out of a literal                                                                                                       | When a default is evaluated                   | What a default may be                            |
| ------------------- | ------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------- | ------------------------------------------------ |
| Go                  | none                                                                | any field, which takes its type's zero value                                                                                       | —                                             | —                                                |
| Rust                | none on stable; RFC 3681 `age: i128 = 42` is unstable               | none, unless the literal ends in `..` (RFC) or `..Default::default()`                                                              | at compile time                               | a constant                                       |
| Zig                 | `a: i32 = 1`                                                        | fields with a default                                                                                                              | at compile time                               | a comptime value                                 |
| Swift               | `var port = 80` on a stored property                                | the memberwise initializer takes defaulted properties as optional arguments (SE-0242); an optional `var x: T?` is implicitly `nil` | per initialization                            | any expression                                   |
| Kotlin              | defaults on the data class constructor's parameters                 | parameters with a default, by name                                                                                                 | per call; may read earlier parameters         | any expression                                   |
| C#                  | property initializers; `required` (C# 11) forces a member to be set | members not marked `required`                                                                                                      | per construction                              | any expression                                   |
| Dart                | defaults on named constructor parameters; `required` marks the rest | parameters with a default                                                                                                          | at compile time                               | a constant                                       |
| Python dataclasses  | `port: int = 80`; `field(default_factory=list)`                     | fields with a default                                                                                                              | once, at class creation; a factory per object | anything but an unhashable value, such as a list |
| TypeScript          | none; `port?: number` makes a property optional                     | optional properties, which read as `undefined`                                                                                     | —                                             | —                                                |
| Elm, Haskell, Gleam | none                                                                | none; the idiom is updating a defaults record, `{ default \| port = 80 }`                                                          | —                                             | —                                                |

What Ant takes:

- **Opt in per field, not per type.** Go makes every field optional, and the community built
  `exhaustruct` to put back the check that a forgotten field is an error; C# added `required` for
  the same reason after property initializers made every member optional. Zig, Swift, Kotlin, Dart,
  Python and Rust's RFC let the author mark which fields may be omitted, and the rest stay required.
  A field added without a default still breaks every literal, which is right when every caller has
  to decide its value.
- **No zero values.** Go's zero values need a zero for every type. Ant has types with none: a
  record, a non-nullable union, a type parameter, a function, a sync object. And `ant run` keeps no
  field types (section 4, Checking and running), so it could not compute a zero at run time anyway.
  TypeScript's `?` is the nullable version of the same idea, and it makes the field's type
  `T | undefined` at every read, which Ant's narrowing would then have to remove at every use.
- **Evaluated per literal.** Python evaluates a function default once, so a list default is shared
  by every call, and dataclasses had to forbid mutable defaults and add `default_factory`. Swift,
  Kotlin, C# and JavaScript's class fields evaluate per object. Ant's parameter defaults already
  evaluate per call (section 12), and records are mutable reference values, so `tags = []` must be a
  new array each time.
- **Any expression.** Rust, Zig and Dart restrict defaults to constants because they have constant
  evaluation and want the default visible to the compiler. Ant has neither `const` nor compile-time
  evaluation, and its parameter defaults are ordinary expressions.
- **No marker at the literal.** Rust's RFC requires `..` at the end of a literal that uses defaults,
  so that every omission is visible where it happens. Ant's calls carry no such marker for omitted
  arguments, and a record literal is read the same way: the type declaration says which fields are
  optional.

## Syntax

In a `type` declaration whose body is a record type, a field may be followed by `=` and an
expression:

```
field = name ":" type [ "=" expression ]
```

Only a record type that is a `type` declaration's body takes defaults. In an anonymous record type
used as an annotation, `{ port: int = 80 }`, a default is a syntax error: it would not be the
declaration of anything a literal can name. The same holds for a record type nested inside another
type, such as the element type of an `Array`.

Neither tool changes. `ant fmt` already puts one space on each side of every `=` (fmt.md, the
spacing table). `ant doc` prints a type through its whole definition (doc.md, section 1), so the
defaults are part of the signature a reader sees.

## Semantics

1. **Omitting a field.** A record literal must give a value to every field of its type that has no
   default. A field that has a default may be given a value by an entry or a spread, or left out.
   Every entry still names a field the type lists.
2. **Evaluation.** A literal evaluates its entries and spreads in order, as today. Then, for each
   field still without a value, in declaration order, the field's default is evaluated and stored. A
   default is evaluated only when it is used: a field given a value, by an entry or by a spread,
   never evaluates its default. Each evaluation is fresh, so `tags: Array[string] = []` gives each
   record its own array. Fresh evaluation is not a copy: `tags: Array[string] = sharedTags` reads
   the variable each time and shares the one array it holds, as a parameter default would.
3. **Scope.** A default sees what its type's body sees, plus the values of the declaring module: the
   module's top-level names, its imports, the prelude, and the types visible where the type is
   declared, local types of enclosing blocks included, since each of those is one tag for the whole
   run (section 12, Names and scopes). It does not see the literal's scope, so a literal in another
   module that builds `Geometry.Config{}` evaluates `Geometry`'s defaults, which may use names
   `Geometry` does not export. It does not see the other fields of the record.
4. **What a default may not do.** A default of a local type may not name a `var`, a function bound
   with `var`, or a parameter of an enclosing block or function: the type has one tag for the whole
   run, so its default cannot close over the variables of one run of the block. A default lies
   outside every boundary (section 5, `return`, `break`, `continue`), as a library's top-level
   initializer does, so `return`, `try`, `break` and `continue` in it, outside a lambda, have
   nothing to leave. All of these are syntax errors, found by the parser and resolver, so `ant run`
   rejects them too and never evaluates a default against a top-level name that a local one
   shadowed.
5. **Types.** The checker checks each default against its field's declared type, with that type as
   the expected type, so `[]`, `Map{}` and `null` need no annotation. In a generic record, the type
   parameters are rigid while checking the default, as in a generic function's body: in
   `type Bag[T] = { items: Array[T] = [] }` the default is fine for every `T`, and
   `type Box[T] = { value: T = 0 }` is a compile error.
6. **Inference.** An omitted field does not take part in solving a literal's type arguments.
   `Bag{ items: [1] }` is a `Bag[int]`; `Bag{}` fixes nothing, so it needs an expected type, as `[]`
   does: `var b: Bag[int] = Bag{}`.
7. **The type is unchanged.** A default says what a literal fills in, nothing more. Every value of
   `Server` still has all four fields, so reading `s.workers` needs no narrowing, an anonymous
   record type `{ workers: int }` still accepts a `Server`, and `==`, `String.from`,
   `Json.stringify` and destructuring see every field as before.
8. **Unchecked runs.** Rules 1 to 3 also hold at run time. The tag keeps, beside the field names,
   the default expression of each field that has one, so `ant run` fills in omitted fields without
   any field type. A literal that leaves out a field with no default still panics.
9. **Failures.** A default runs at the literal, so a failure in it shows there. A default that reads
   a top-level `var` of its module before that `var`'s initializer has run panics, as a function
   that reads it would (section 10): in the module below, `var early = Config{ host: "x" }` placed
   above `var defaultPort` panics at the literal. A default that builds its own type,
   `type Node = { next: Node | null = Node{} }`, recurses; evaluating a default counts as a call, so
   it panics when the goroutine's call depth exceeds the runtime's limit, as unbounded recursion
   does (section 12, Runtime panics).

```ant
// config.ant
var defaultPort = 8080

export type Config = {
  host: string,
  port: int = defaultPort,
  retries: int = 3,
  headers: Map[string, string] = Map{},
}

// main.ant
import "./config.ant" { Config }

var c = Config{ host: "example.com" }            // port: 8080, read from config.ant's scope
var d = Config{ ...c, retries: 5 }               // every field from c; no default runs
c.headers["accept"] = "text/html"                // d.headers is the same map as c.headers
var e = Config{ host: "example.com" }            // e.headers is a new, empty map
```

A record whose fields all have defaults can be built with an empty literal, `Config{}`, which reads
as "the defaults". A spread now converts to a type with more fields when the extra ones have
defaults: `Point3{ ...p }` fills `z` from `z: float64 = 0.0`, so widening a record no longer breaks
the conversions either.

## When to use a default

A default suits a field that callers set independently of the others: a timeout, a flag, a list that
starts empty. It does not suit fields that must agree. Zig's documentation warns about this case:
with `min: int = 0, max: int = 100`, a caller who sets only `max: -1` builds a record that breaks
the invariant `min <= max` without writing either value wrong. Such fields stay required, or the
type gets a function, `Range.new(min, max)`, that checks them together.

A field every caller should think about stays without a default even when most callers would write
the same value. Leaving the default off is how a new field makes every literal fail to check until
someone decides.

## Considered and rejected

- **Go's zero values.** Omitting any field, filled with `0`, `""`, `false`, `null` or an empty
  collection. Rejected because a type with no zero would need a second rule (always list it), the
  interpreter would need field types it does not keep, and a forgotten field would no longer be an
  error. The `ant/TODO.md` sketch proposed this; the open questions there are what led to declared
  defaults.
- **Implicit `null` for nullable fields.** Letting `nickname: string | null` default to `null`
  without writing `= null`, as Swift does for an optional stored property in its memberwise
  initializer. It is the most common default, but it would tie two separate facts together: that
  `null` is a valid value and that omission is allowed. Keeping them apart gives one rule for every
  field, and `= null` is seven characters.
- **A marker at the literal.** Rust's `Server{ addr: ":80", .. }`. It shows the reader that fields
  were left out, and it keeps a way to demand an exhaustive literal: one without `..` must still
  list every field, so a caller who wants to hear about a new defaulted field gets a compile error.
  But it adds syntax to routine construction, Ant's calls have no such marker for omitted arguments,
  and `...` already means a spread in a literal, so the two spellings would sit one dot apart.
- **Defaults that read other fields.** `end: int = start + 10`. A parameter default may read earlier
  parameters, but a record's fields are not a scope, and fields whose values depend on each other
  are the coupled fields of "When to use a default": a caller who sets `end` alone can still break
  `start <= end`. A function on the type checks them together, with no new rules.
- **Constant-only defaults.** Ant has no constant evaluation, and a constant rule would differ from
  parameter defaults for no gain.
- **A lint or flag that requires every field.** An author who wants every caller to choose a value
  leaves the default off. The choice is per field, in the declaration.

## Spec changes

- **Section 4, Checking and running.** "A record tag also keeps the names of its fields in
  declaration order" becomes "the names of its fields in declaration order and the default
  expression of each field that has one".
- **Section 4, Records.** Add the default syntax and the example from the summary. "A record literal
  must list exactly the fields of its named type" becomes "A record literal must give every field of
  its named type that has no default, and may give the others; an entry for a field the type does
  not list is a compile error, which catches typos." The paragraph on what the interpreter keeps
  adds the defaults. Add rules 2, 3 and 5 to 7.
- **Section 4, Type declarations.** Add rule 4 for local types.
- **Section 5, `return`, `break`, `continue`.** A record default is, like a library's top-level
  initializer, outside every boundary.
- **Section 4, Inference.** Add rule 6.
- **Section 12, Expressions, record literal.** "Every name the tag lists must have received a value"
  becomes "then every field the tag lists that has received no value takes its default, evaluated in
  declaration order in the scope of rule 3; a field without a default must have received a value."
  Rule 9's call-depth sentence goes with it.
- **Section 12, Values.** The tag row adds "and the default expression of each field that has one".
- **Section 12, Runtime panics.** The record literal row reads "a field the tag lists is left
  without a value and has no default".
- **`ant/TODO.md`.** Drop "Default values for records". Flags keeps its open question: defaults give
  a checked record of flag values a declaration, but filling it from `Os.args()` still needs a way
  to iterate a record's fields.

## Non-goals

- Optional fields whose values may be absent at run time. A default fills the field; the value is
  always complete.
- Defaults for function types, union types, or anything other than a record field.
- Iterating a record's fields or reading its defaults as values. `Flag.parse` into a record and a
  typed `Json.decode` both want this; it is a separate proposal.
- Constructors. Functions on types (section 6) remain the way to validate or derive fields.

## Open questions

None at this draft. The exhaustive-literal marker and defaults that read other fields are rejected
above; either can be reopened if programs show a need.
