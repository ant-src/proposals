# Ant proposals

Design proposals for [Ant](https://github.com/ant-src/spec). A proposal describes a change before it
reaches the specification: the problem, prior art, the proposed API, and the spec sections it
touches. Nothing here is part of the language until it is accepted and written into the spec.

| Proposal                       | Summary                                                                    | Status |
| ------------------------------ | -------------------------------------------------------------------------- | ------ |
| [ui.md](ui.md)                 | A client framework and server components on `Html`                         | Sketch |
| [frozen.md](frozen.md)         | `freeze` and `Frozen[T]`: deep immutable copies, enforced at run time      | Draft  |
| [coverage.md](coverage.md)     | Statement and branch coverage for `ant test` and `ant run`, in LCOV        | Draft  |
| [reflect.md](reflect.md)       | Types as values: `type X`, `Type.info`, `Type.build`; JSON decoding on top | Draft  |
| [reflection.md](reflection.md) | `type T` values, `Type.shape`, and `Dynamic` boxes that keep their type    | Draft  |
| [interfaces.md](interfaces.md) | Interfaces satisfied by functions on types, dispatched on the value's tag  | Draft  |

## Process

1. Write `<name>.md` with a summary, motivation, prior art, the proposed API and semantics, the spec
   changes, non-goals and open questions.
2. Review it. Decisions go into the proposal itself, so the file records why the design is what it
   is.
3. Once accepted, the change is written into the spec and implemented, and the proposal is removed:
   the spec is the record from then on, and git history keeps the proposal. A rejected proposal
   stays here with its status set to Declined and a line on why.
