# Proposal: private fields

2026-11-07 · draft

## Summary

A field of a record type that a `type` declaration declares may be marked `private`. Its name is
then visible only in the module that declares the type, and in a test file that imports that module.
Outside, reading, writing, naming in a literal or a destructuring, spreading, and widening to an
anonymous record type that lists it are compile errors. Fields stay public by default, so every
existing type keeps its meaning. Like `export`, the rule belongs to the checker: `ant run` does not
enforce it, and `==`, `String.from`, `Json.stringify`, and reflection still see every field.

```ant
// pool.ant
export type Pool = {
  max: int = 10,
  private idle: Array[Conn] = [],
  private lock: Mutex = Mutex.new(),
}

export func Pool.get(p: Pool): Conn {
  p.lock.lock()
  defer p.lock.unlock()
  p.idle.pop() ?? Conn.open()
}
```

```ant
// main.ant
import "./pool.ant" { Pool }

var p = Pool{ max: 4 }        // ok: idle and lock take their defaults
var c = p.get()
p.idle.push(c)                // compile error: Pool.idle is private to pool.ant
```

A private field with no default means a literal outside the module cannot be written. The type's
functions then become the only way to build a value:

```ant
// span.ant
export type Span = { private start: int, private end: int }

export func Span.new(start: int, end: int): Result[Span, Error] {
  if start > end { return Result.err(Error{ message: "span: start ${start} after end ${end}" }) }
  Result.ok(Span{ start, end })
}

export func Span.length(s: Span): int { s.end - s.start }
```

## Motivation

Section 4, Field defaults, already describes the problem: "Fields that must agree, such as `min` and
`max`, stay required, or the type gets a function that checks them together." Today that function is
only a convention. Any module that holds a `Span` can write `s.end = 0`, and any module can write
`Span{ start: 9, end: 1 }`. The type's author cannot stop either, short of not exporting the type,
and then other modules cannot name it in a signature.

Three kinds of type need this:

- **Invariants.** `Span`, a sorted set, a non-empty list, a validated `Email`. A check in `new`
  means nothing if a literal or an assignment can skip it.
- **Internal state.** A pool's idle list, a cache's map, the mutex that guards them. Callers should
  not touch these, and the author should be free to change them without breaking anyone.
- **A stable API.** Every field of an exported type is part of the module's interface today.
  Renaming a field that holds internal bookkeeping breaks every importer that read it.

Today an author has two options, and neither is good:

- **Export the type with every field.** Documentation is then the only guard.
- **Keep the type private and export functions.** Other modules can hold the values but cannot name
  the type, so they cannot annotate a variable, a parameter, or a field with it. An exported
  function needs an exported type (section 6), so this works only inside one module.

A host can declare an opaque type (section 10, Declaration files), but Ant code cannot.

## Prior art

- **Go.** Unexported fields, decided by case, are visible to the whole package. Methods see them
  because they live in the same package, not because they are written inside the struct. Outside the
  package, a composite literal cannot name them, and `reflect` can read but not set them.
  `encoding/json` skips them.
- **Rust.** A field is private to its module unless marked `pub`. A struct literal outside the
  module is impossible if any field is private, which is how Rust gets constructors without a
  constructor feature. `#[non_exhaustive]` exists only to get the same effect without private
  fields.
- **TypeScript.** `private` is checked by the compiler and erased: at run time the field is an
  ordinary property, and `JSON.stringify` includes it. `#field` is the separate run-time form.
- **Swift.** `private` is scoped to the type's declaration and `fileprivate` to the file. The
  `fileprivate` level was added back because type scope proved too narrow for extensions and helpers
  in the same file.
- **Dart, Python.** A leading `_` makes a name library-private in Dart and private by convention in
  Python.
- **Zig.** Has no private fields. The argument for leaving them out is that a doc comment says
  enough and that hiding fields gets in the way of debugging and testing.

Go's and Rust's model fits Ant: the module is the unit of privacy, and functions on a type are
already declared in the type's module (section 6). TypeScript's model fits Ant's run time: the
checker enforces privacy and the interpreter does not, as with `export` (section 12).

## The design

**Syntax.** `private` before a field's name in the record body of a `type` declaration:

```ant
type Cache[K, V] = {
  limit: int,
  private entries: Map[K, V] = Map{},
  private order: Array[K] = [],
}
```

`private` is a keyword only in that position. It stays a name everywhere else, and in that position
when followed by `:`, so a field named `private` still works: `{ private: bool }`. The next token
decides, as it does for `type` (section 2), so the grammar keeps one token of lookahead. Code that
uses `private` as a name keeps working.

**Where.** Only the record body of a `type` declaration, the same places a field default may appear,
for the same reason: privacy belongs to a declaration, and an anonymous record type has none. In an
anonymous record type, a record type nested in another type, or a union member, `private` is a
syntax error. Local types may have private fields. They are never exported, but their values can
leave the module.

**Visibility.** A private field's name is visible in the module that declares the type, and in a
test file that imports that module, which sees private declarations already (section 10, Tests).
Everywhere else, the checker rejects every use of the name. Each of the following is a compile error
outside the module:

| Use                               | Example                                    |
| --------------------------------- | ------------------------------------------ |
| read, `?.` read, or call          | `s.end`, `s?.end`, `s.end()`               |
| assignment, compound assignment   | `s.end = 0`, `s.end += 1`                  |
| a literal entry, shorthand or not | `Span{ start: 1, end: 2 }`, `Span{ end }`  |
| a literal that omits it           | `Span{}`, when the field has no default    |
| a destructuring that names it     | `var { end } = s`                          |
| a spread that copies it           | `Other{ ...s }`, `Span{ ...s }`            |
| a spread that fills it            | `Span{ ...other }`, when `other` has `end` |
| widening to a type that lists it  | `var e: { end: int } = s`                  |
| a field common to a union         | `x.end` on `Span \| Range`                 |

The message names the field and the module, as an unexported import does:
`Span.end is private to span.ant`.

A literal outside the module may leave a private field out when it has a default, and the default
runs as usual, in the declaring module's scope (section 4, Field defaults). So `Pool{ max: 4 }`
above is fine, and whether outside code can write a literal is up to the author: give every private
field a default to allow it, or leave one without a default to forbid it. No other construct is
needed for this.

The spread rule is in both directions because a spread copies by name. Copying out of a value with
private fields, `Copy{ ...s }`, would read them into a type that may expose them. Copying into a
type, `Span{ ...other }`, would set them from a record the author never checked. A spread whose
operand's type has no private field and whose literal's type gets none from it is unaffected.

Widening follows the interface rule (section 4, Interfaces): visibility is checked where the checker
asks. Inside its module, a `Span` is assignable to `{ end: int }`, and outside it is not. A module
can still widen its own value and hand it out. That is a decision the author makes, like returning
`s.end`.

**What does not change.**

- **Inside the module.** A private field behaves exactly like any other field there.
- **Data.** Privacy applies to names and not to data, as section 11, Reflect, already says. `==`
  compares private fields, `String.from` and `Json.stringify` render them, a hash includes them, and
  `Record.same` is unaffected. Two `Span`s with different ends are not equal, and a value printed
  while debugging shows all of its state.
- **The run time.** `ant run` checks nothing. The interpreter keeps no per-field privacy, as it
  keeps no `export`, so an unchecked program reads a private field like any other.
- **Name clashes.** A function on a type still may not share a name with any of its fields, private
  ones included, since `x.f()` reads the field first at run time (section 5). See Costs.
- **Interfaces.** A field never satisfies an interface function, so nothing changes there.

**Reflection.** `Reflect.Field` gets `private: bool`:

```
type Reflect.Field = { name: string, type: Reflect.ErasedType, hasDefault: bool, private: bool }
```

`Reflect.shape` lists private fields and marks them. `Reflect.field` reads them and
`Reflect.makeRecord` sets them, from any module. This is what the Visibility paragraph of the
Reflect section already accepts for private types: an invariant that the type's functions keep is a
convention of the program's own code, not something reflection upholds. Codecs should skip private
fields by default, as Go's `encoding/json` does: leave them to their defaults when decoding, and
omit them when encoding. A codec cannot decode a type with a private field that has no default
unless the author registers a hook, and that is the right result: such a type is built through its
functions.

**Declaration files.** `private` is a syntax error in a declaration file. Every declaration there is
visible, and a host type that hides its state is opaque.

**Tools.**

- `ant doc` leaves private fields out of an exported type and ends its field list with the comment
  `// private fields`. Go's godoc does the same. From that line a reader can tell that the type may
  not be constructible outside the module.
- The LSP leaves private fields out of completion outside the module, and hover shows the type as
  `ant doc` does. Go to definition still works, as it does for any private name.
- `ant fmt` formats `private` like any other word before a field.

## Alternatives

- **No private fields; the status quo.** The Reflect section says privacy applies to names, not
  data, and that a type guards its invariant with functions on the type. This proposal keeps the
  first half: a private field is a private name. It makes the second half something the checker
  enforces rather than a convention. Leaving fields as they are keeps the language smaller, but
  there is still no way to write a constructor that cannot be bypassed.
- **Type-scoped privacy, as with classes.** Only functions declared on the type would see the field.
  But Ant's functions on a type are module-level functions, and helpers, other types' functions, and
  tests in the same module need the fields as well. Swift added `fileprivate` for this reason.
  Module scope also needs no new rule about which code counts as inside the type.
- **Naming, a leading `_` as in Dart.** It needs no new syntax and shows privacy at every use. But
  section 2 says case and spelling mean nothing about exports, a field would have to be renamed to
  change its visibility, and `_` alone already means discard.
- **Fields private by default, marked with `export`.** This matches how module members work, but
  every field of every exported data type would need `export`, and most exported records are plain
  data. Every existing program would also break.
- **Opaque exports.** A form like `export opaque type Span = {…}` would hide all fields at once. It
  is simpler but all or nothing: `Pool` above wants a public `max` beside private state. Marking
  every field `private` gives the same result.
- **Hiding at run time.** The interpreter would keep a privacy bit per field, and `String.from`,
  `Json.stringify`, and `==` would skip private fields. That costs the interpreter state the "Types
  are advisory" principle keeps out. It also makes `==` wrong for state that matters, and makes
  debugging output lie. Go's `fmt` prints unexported fields too.

## Costs

- **Getter names.** A getter cannot share its field's name. Go spells a field `end` and its getter
  `End`. Ant must use two names, such as a private `endAt` with a function `end`, or the reverse.
  Relaxing the clash rule for private fields would need the interpreter to know which fields are
  private, which the last alternative rules out.
- **The checker** gains the visibility check on field names, at every site in the table above. It
  already checks visibility for imports and for interface satisfaction.
- **The parser** gains one contextual keyword in one position. `ant fmt`, the highlighting grammars
  (tree-sitter, TextMate, IntelliJ), `ant doc`, and the LSP learn it.
- **Reflection** gains one field on `Reflect.Field`.

## Fit with other proposals

- **annotations.md.** Codecs skip private fields by default, so `@Skip` is needed mostly for public
  fields that are state. A private field that a codec should still handle could carry an annotation
  that opts it in. That belongs to the codec, not the language.
- **typed-data.md.** `Type.build` and `Type.data` follow the codec rule above: they skip private
  fields, and `build` fails on a type whose private field has no default.
- **frozen.md.** `freeze` copies every field, private ones included, since it copies data. A frozen
  value's private fields are as immutable as its public ones.

## Open questions

1. **Should a spread that copies out of a value be allowed when the literal's type is the same
   type?** `Span{ ...s, start: 0 }` outside the module would copy `end` unchanged into another
   `Span`. The value it copies passed the type's check already, but the new `start` did not. The
   draft forbids it, because changing one field can break an invariant (`start <= end`) as easily as
   setting all of them.
2. **Should `Reflect.makeRecord` refuse private fields from outside the module?** The interpreter
   does not know where a call came from, so this would have to be a checker rule on a value the
   checker cannot see. The draft says no and leaves the decision to codecs.
3. **The comment `ant doc` writes.** Should it also say when a literal is impossible outside the
   module, for example `// private fields; build with Span.new`? The tool cannot know which function
   builds the type, so the draft writes only `// private fields`.

## Spec changes

- Section 2: `private` as a contextual keyword, next to the rule for `type`.
- Section 4: a "Private fields" subsection after Field defaults. A sentence in Records about
  literals and spreads, one in Assignability about width subtyping, and the `min`/`max` paragraph of
  Field defaults pointing to the new subsection.
- Section 5: Calls through `.` and Destructuring name the visibility check.
- Section 10: Modules and Tests extend "private" to fields. Declaration files make `private` a
  syntax error.
- Section 11, Reflect: `private` on `Reflect.Field`, and the Visibility paragraph.
- Section 12: rows in the compile-error table.
- `doc.md`, `lsp.md`: rendering and completion.
