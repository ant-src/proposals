# Ant proposals

Design proposals for [Ant](https://github.com/ant-src/spec). A proposal describes a change before it
reaches the specification: the problem, prior art, the proposed API, and the spec sections it
touches. Nothing here is part of the language until it is accepted and written into the spec.

| Proposal                                 | Summary                                                             | Status |
| ---------------------------------------- | ------------------------------------------------------------------- | ------ |
| [assert.md](assert.md)                   | `assert(cond, message?)`: always on, shows compared values, narrows | Draft  |
| [ui.md](ui.md)                           | A client framework and server components on `Html`                  | Sketch |
| [parallel.md](parallel.md)               | `Parallel.map`, `each`, `run` and `race`: bounded, joined fan-out   | Draft  |
| [process.md](process.md)                 | `Os.run`, `output`, `start`, `wait`, `pipe`: running other programs | Draft  |
| [record-defaults.md](record-defaults.md) | `workers: int = 4`: record fields a literal may leave out           | Draft  |

## Process

1. Write `<name>.md` with a summary, motivation, prior art, the proposed API and semantics, the spec
   changes, non-goals and open questions.
2. Review it. Decisions go into the proposal itself, so the file records why the design is what it
   is.
3. Once accepted, the change is written into the spec and the proposal's status becomes Accepted. A
   rejected proposal stays here with its status set to Declined and a line on why.
