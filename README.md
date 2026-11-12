# Ant proposals

Design proposals for [Ant](https://github.com/ant-src/spec). A proposal describes a change before it
reaches the specification: the problem, prior art, the proposed API, and the spec sections it
touches. Nothing here is part of the language until it is accepted and written into the spec.

| Proposal                               | Summary                                                                          | Status   |
| -------------------------------------- | -------------------------------------------------------------------------------- | -------- |
| [ui.md](ui.md)                         | A client framework and server components on `Html`                               | Sketch   |
| [frozen.md](frozen.md)                 | `freeze` and `Frozen[T]`: deep immutable copies, enforced at run time            | Draft    |
| [coverage.md](coverage.md)             | Statement and branch coverage for `ant test` and `ant run`, in LCOV              | Draft    |
| [typed-data.md](typed-data.md)         | `Type.Data`, `Type.build`, hooks and `Json.decode`, on top of `reflection.md`    | Draft    |
| [annotations.md](annotations.md)       | Typed `@` annotations on record fields, read through `Reflect`                   | Draft    |
| [private-fields.md](private-fields.md) | `private` record fields, visible only in the declaring module, checker-enforced  | Draft    |
| [flags.md](flags.md)                   | A typed `Flags` library: parsing, help and shell completion from one declaration | Draft    |
| [flag.md](flag.md)                     | A lightweight `Flag` namespace: Go's `flag` rules, typed by each flag's default  | Draft    |
| [flags-v2.md](flags-v2.md)             | Typed `Flags.Key[T]` declarations over Go's `flag` rules and usage text          | Draft    |
| [buffer.md](buffer.md)                 | `Buffer`, a mutable byte sequence beside `Bytes`, and binary integer readers     | Draft    |
| [error.md](error.md)                   | A prelude function `error(message)`; `Result.fail` landed instead                | Declined |

## Process

1. Write `<name>.md` with a summary, motivation, prior art, the proposed API and semantics, the spec
   changes, non-goals and open questions.
2. Review it. Decisions go into the proposal itself, so the file records why the design is what it
   is.
3. Once accepted, the change is written into the spec and implemented, and the proposal is removed:
   the spec is the record from then on, and git history keeps the proposal. A rejected proposal
   stays here with its status set to Declined and a line on why.
