# Proposal: `ant code`

2026-11-18 · draft

## Summary

Add `ant code`, a command that answers the questions an editor asks the language server, from a
terminal: where a name is declared, where it is used, what type it has, which declarations a module
has, and how to rename one. It has six subcommands, `def`, `refs`, `type`, `symbols`, `rename`, and
`fix`. Each takes a location written as `file:line:col`, `file:line:name`, `file:Name`, or a bare
qualified name such as `Parser.expr`, and prints plain text by default or JSON with `-json`.
`rename` and `fix` write files, or print a unified diff with `-diff`.

`ant code` runs the same analysis as `ant lsp`, in process, on the files as they are on disk. It
adds no language rules. It changes one rule of `ant lsp`: references reach the modules that import
the declaring module, as a rename already does.

## Motivation

The language server answers these questions well, but only for a client that speaks the Language
Server Protocol. A coding agent, a shell script, a CI step, or a person in a terminal without an
editor plugin has `grep`, `sed`, and `ant doc`, and those can't tell a type from a field of the same
name, or an import from a local that shadows it. These problems came up while an agent wrote five
Ant programs (`private/agent-study/README.md`):

1. **Renaming across modules was unsafe without the server.** Renaming the type `Pos` to `Span` in a
   four-module interpreter meant changing 29 places in three files. `sed 's/Pos/Span/'` would also
   hit `posOf` and comments, and `\bPos\b` would still hit text in strings. `ant lsp`'s rename did
   it exactly, but only through a 40-line JSON-RPC client written for the occasion.
2. **Inferred types are invisible.** After a pipeline such as
   `entries | Array.groupBy(...) | Map.entries() | Array.map(...)`, the only way to learn the type
   of the result is hover. Inlay hints show it in an editor and nowhere else.
3. **References stop at the program.** `textDocument/references` on `Pos` found 10 uses, all in the
   declaring module, because a reference search covers the program rooted at the open file
   (`private/tools/lsp.md` section 4). The same position's rename found all 29. A caller asking "who
   uses this?" before changing it gets the shorter answer.
4. **Columns are hard to produce.** An agent reads line numbers from `grep -n` and from file views.
   Counting the column of a name in a line is error-prone, and a wrong column silently answers about
   a different name. `ant doc` already shows that looking up a declaration by name works better.
5. **The mechanical fixes are editor-only.** The server can import a name, state an inferred type,
   and add a record literal's missing fields. A terminal user retypes them.

Each answer costs one process start and one analysis. That is slower than a running server, and fast
enough for a command typed or issued once per step.

## Prior art

| Tool                         | Shape                                                                                             | Positions                                      | What Ant takes                                                                     |
| ---------------------------- | ------------------------------------------------------------------------------------------------- | ---------------------------------------------- | ---------------------------------------------------------------------------------- |
| `gopls` command line         | subcommands over the server's own code: `definition`, `references`, `rename`, `symbols`, `fix`, … | `file.go:line:col`, `file.go:#offset`          | One engine for the editor and the terminal. `rename -w` writes, `-d` prints a diff |
| `gorename` (retired)         | one command, one refactoring                                                                      | `-from '"pkg".Name'` or `-offset file.go:#123` | Naming a declaration instead of pointing at it                                     |
| `rust-analyzer` command line | `symbols`, `search`, `ssr` (structural search and replace), `diagnostics`, `scip`                 | queries and patterns                           | Batch use of the server's analysis outside an editor                               |
| TypeScript `tsserver`        | a protocol only; scripted refactors go through libraries such as ts-morph                         | offsets                                        | A warning: without a command, scripts build their own clients                      |
| Pyright                      | `--outputjson` for diagnostics; no navigation from the command line                               | —                                              | JSON output beside text                                                            |
| `ant doc`                    | signature and doc comment of a name                                                               | `Owner.name`, matched without regard to case   | Name syntax, output layout for signatures and docs                                 |

Three lessons:

- **The command line should run the server's code, not a copy of it.** `gopls` keeps its command
  line in step with the editor at no extra cost, because each subcommand calls the handler the
  protocol calls. A second implementation would drift.
- **Scripts want names; editors want offsets.** `gorename -from` existed because a script knows what
  it wants to rename, not where the cursor would be. `ant code` takes both.
- **Writing and previewing are separate requests.** `gopls rename -w` and `-d` let a caller look
  before it writes. An agent that applies a rename should be able to review it first.

## Proposal

### Command

```
ant code def     [flags] <loc>          the declaration of a name
ant code refs    [flags] <loc>          every use of a declaration
ant code type    [flags] <loc>          the type, signature, and doc of a name
ant code symbols [flags] [path...]      the declarations of modules
ant code rename  [flags] <loc> <name>   rename a declaration and its uses
ant code fix     [flags] <file.ant>     apply the mechanical fixes for a module's errors
```

| Flag          | Subcommands     | Meaning                                                              |
| ------------- | --------------- | -------------------------------------------------------------------- |
| `-json`       | all             | print JSON instead of text                                           |
| `-root dir`   | all             | the workspace: the files searched for importers, default `.`         |
| `-diff`       | `rename`, `fix` | print the change as a unified diff and write nothing                 |
| `-force`      | `rename`        | rename even when a changed program has errors                        |
| `-decl`       | `refs`          | include the declaration itself                                       |
| `-match text` | `symbols`       | only declarations whose name contains `text`, without regard to case |
| `-all`        | `symbols`       | include private declarations and the declarations inside functions   |

The workspace is what an editor's workspace folder is to `ant lsp`: every `.ant` file under the
root, skipping entries that start with a dot and `node_modules` (`lsp.md` section 4). A location may
name a file outside the root; its importers are still searched under the root only.

### Locations

A `<loc>` takes one of four forms:

| Form             | Example                       | Means                                                              |
| ---------------- | ----------------------------- | ------------------------------------------------------------------ |
| `file:line:col`  | `calc/lexer.ant:3:13`         | the name at that position                                          |
| `file:line:name` | `calc/lexer.ant:3:Pos`        | the identifier `name` on that line                                 |
| `file:Name`      | `calc/parser.ant:Parser.expr` | a declaration of that module, by name                              |
| `Name`           | `Pos`, `Array.map`            | a top-level declaration of the workspace, or a prelude declaration |

Lines and columns count from 1, and a column counts bytes, as `ant check` prints them. A name is
`Name` or `Owner.name`, the function `name` declared on `Owner` or a field of the record type
`Owner`, matched exactly. Unlike `ant doc`, case matters, since identifiers are case-sensitive and a
location must pick one binding.

A location is ambiguous when it matches more than one binding: an identifier that appears twice on
the line with different bindings, a bare name declared in two modules, or `Owner.name` matching both
a function and a field. `ant code` then names none of them: it prints each candidate as a
`file:line:col` location with its line, and exits with status 1, so the caller can repeat the
request with one of them. A bare name never collides with the prelude, since a module cannot
redeclare a prelude name (section 10).

### Output

Text output is made of locations, one per line, in the form `ant check` and `grep -n` use:
`path:line:col: text`. Paths are relative to the current directory when the file lies under it, and
absolute otherwise. A declaration of the prelude has the location `prelude.d.ant:line:col`, the
read-only source `ant lsp` serves as `ant:/prelude.d.ant`; `ant code def -json` reports it with
`"prelude": true`.

JSON output is one object, described with each subcommand below. Locations in it are objects with
`path`, `line`, `col`, `endLine`, and `endCol`, the end exclusive.

### `def`

Prints the location of the declared name, then its signature as `ant doc` prints it, on one line.
The declaration is the one `textDocument/definition` finds (`lsp.md` section 4): for a call through
`.`, the function it calls or its signature in the interface; for a field, the field in its record
type; for an import name, the declaration it imports.

```text
$ ant code def calc/eval.ant:145:parse
calc/parser.ant:147:13: func parse(tokens: Array[Token]): Result[Expr, ParseError]
```

JSON: `{ "location": Location, "signature": string, "prelude": bool }`.

### `refs`

Prints every use of the declaration, one location per line with the source line after it, in path
and then position order. The declaration itself is left out unless `-decl` is given. The search
covers the program rooted at the location's file and every module of the workspace whose imports
lead to the declaring module: the same set of modules a rename changes (Changes to `ant lsp`,
below). Uses written through an import alias are included, marked `(as Name)`.

```text
$ ant code refs calc/lexer.ant:Pos
calc/eval.ant:2:24: import "./lexer.ant" { Pos, tokenize }
calc/eval.ant:11:49: export type EvalError = { message: string, pos: Pos }
...
calc/lexer.ant:5:42: export type Num = { value: float64, pos: Pos }
...
28 references in 3 files
```

The count goes to standard error, so standard output holds only locations. JSON:
`{ "declaration": Location, "references": [{ "location": Location, "alias": string | null }] }`.

### `type`

Prints what hover shows (`lsp.md` section 4), as plain text: the kind and name with the type the
checker inferred, narrowed at that use, then the doc comment indented by four spaces. For a function
or a type it prints the signature as written. For a field it prints `name: Type` and the record type
it belongs to.

```text
$ ant code type logstat.ant:86:5
let topPaths: Array[Entry[string, int]]
```

A position on a literal, an operator, or a keyword has no answer, as it has none in hover, and exits
with status 1. JSON: `{ "kind": string, "text": string, "doc": string }`.

### `symbols`

Prints the declarations of the modules under each path, a directory searched as the workspace is,
default the root. Each line is the location of the name, then the signature, which starts with the
kind of declaration. Functions on a type follow the type, and in a test file the tests follow, with
their full names as `ant test -run` matches them (`spec/test.md` section 1). By default only
exported declarations and tests are listed. `-all` adds private declarations, the fields of record
types, and the declarations inside functions.

```text
$ ant code symbols -match parse calc
calc/parser.ant:15:13: type ParseError = { message: string, pos: Pos }
calc/parser.ant:147:13: func parse(tokens: Array[Token]): Result[Expr, ParseError]
```

JSON:
`{ "symbols": [{ "location": Location, "kind": string, "name": string, "signature": string, "exported": bool, "container": string | null }] }`.

### `rename`

Renames the declaration at the location and every use `refs` reports, under the rules of
`textDocument/rename` (`lsp.md` section 4). It refuses a name that is a keyword, a reserved word, or
a prelude name, or that is already bound in a scope the renamed binding reaches, and it refuses to
rename a prelude declaration, a namespace, a record field, or a lambda's `x`. A refusal prints the
reason and exits with status 1, writing nothing.

Renaming resolves member calls through the checker, so a use the checker can't type may be missed.
Before it writes, `rename` checks every program it changes, and refuses when any of them has an
error, printing the errors as `ant check` does. `-force` renames anyway.

Without `-diff`, `rename` writes each changed file in full, through a temporary file renamed into
place, and prints one line per file with its count of edits. It changes only the names: a line whose
layout `ant fmt` would now change is left for `ant fmt`. With `-diff`, it writes nothing and prints
a unified diff of every file it would change.

```text
$ ant code rename calc/lexer.ant:Pos Span
calc/eval.ant: 5 edits
calc/lexer.ant: 10 edits
calc/parser.ant: 14 edits
```

JSON: `{ "files": [{ "path": string, "edits": [{ "location": Location, "newText": string }] }] }`,
the same with and without `-diff`.

### `fix`

Applies the code actions of `lsp.md` section 4 that are marked preferred, one per error of the
module: stating the type of a `var` or `let` or the return type of a function where an error asks
for an annotation, adding the fields a record literal leaves out, and importing an undefined name
that exactly one module of the workspace exports. An error with no preferred action is left alone,
and `fix` lists it. The missing-fields fix inserts placeholders, so the module still has errors
after it, now at the values to fill in.

`fix` writes as `rename` does, and `-diff` previews the same way. It exits with status 0 when it
changed something or found nothing to change, and 1 when errors remain that it could not fix.

### Exit status

| Status | When                                                                                 |
| ------ | ------------------------------------------------------------------------------------ |
| `0`    | the request was answered, or the files were written                                  |
| `1`    | nothing was found, the location is ambiguous, a rename was refused, or errors remain |
| `2`    | the usage is wrong, or a file cannot be read or parsed                               |

A file that does not parse is status 2 for every subcommand but `symbols` and `fix`, which read past
syntax errors as `ant lsp` does.

### Changes to `ant lsp`

`textDocument/references` reaches the importers of the declaring module, as rename does: for an
export, a top-level declaration a test file can import, or a parameter of a top-level function, it
adds the uses in every module of the workspace whose imports lead to the declaring module. A caller
then gets one answer to "who uses this?", from the editor and from `ant code refs`, and it matches
the edits a rename would make. `textDocument/documentHighlight` is unchanged, since it covers one
document.

## Example

The rename from the motivation, as an agent would carry it out:

```sh
ant code refs calc/lexer.ant:Pos            # 28 uses in 3 files: is this the right name?
ant code rename -diff calc/lexer.ant:Pos Span   # review the edits
ant code rename calc/lexer.ant:Pos Span
ant check calc/calc.ant && ant test calc
```

## Spec changes

- New `private/tools/code.md`, the contract of the command, beside `lsp.md` and `dap.md`: sections
  for the command, locations, output, each subcommand, and the exit statuses, referring to `lsp.md`
  section 4 for what each answer contains.
- `private/tools/lsp.md` section 4: references reach importers (Changes to `ant lsp`, above).
- `docs/tools/code.md`, a reference page in the style of `docs/tools/doc.md`, with tested examples.
- The usage text of `ant`, and the bash, zsh, and fish completions in `cmd/ant`.

`spec/ant-lang.md` does not change.

## Implementation sketch

Each feature of `internal/lsp` is a method on `*Server` that takes the protocol's parameter types:
`definition`, `references`, `hover`, `rename`, `documentSymbols`, `workspaceSymbols`, and the code
actions. `ant code` builds a `Server` in process with the root as its workspace folder and the
`utf-8` position encoding, which the server already supports, opens the location's file as a
document read from disk, and calls the method. No protocol messages are exchanged.

A small exported layer in `internal/lsp` does what a client would otherwise do:

- turns a `<loc>` into a position: a column is used as it is, an identifier on a line is found among
  the line's identifier tokens, and a `Name` is found among the module's document symbols or the
  workspace symbols, with candidates collected for the ambiguity report;
- applies a `WorkspaceEdit` to the files, from the last edit to the first, or renders it as a diff;
- renders hover markdown as plain text with `ant doc`'s layout.

`references` gets the importer pass that `rename` already has (`references.go`, the loop over
`s.importers`), shared between the two. `cmd/ant/code.go` parses the flags, calls the layer, and
prints. Tests run each subcommand over a small workspace in `cmd/ant/testdata`, with the text and
JSON output as golden files, and reuse the server's existing tests for the answers themselves.

## Non-goals

- **A daemon.** `ant code` starts, answers, and exits. A long-lived process that caches the analysis
  between calls is what `ant lsp` is.
- **New refactorings.** `ant code` offers what the server offers. Renaming fields, moving a
  declaration to another module, extracting a function, and changing a signature are each a feature
  of the server first, and reach `ant code` through it.
- **Diagnostics.** `ant check` reports errors. Running the checker over a directory and printing its
  errors as JSON is a change to `ant check`, proposed separately.
- **Formatting.** `ant fmt` formats; `rename` and `fix` leave layout to it.
- **Search and replace by pattern**, as `rust-analyzer ssr` does. It is a larger design and may
  build on this one later.

## Later

- **Callers.** `ant code callers <loc>`: the calls of a function, grouped by the function they occur
  in. It needs `textDocument/prepareCallHierarchy` and `callHierarchy/incomingCalls` in the server,
  which `lsp.md` section 5 leaves out today.
- **Field rename**, once the server can rename a field through record literals, destructuring,
  spreads, and structural types.
- **Batches.** A `-` location reads one request per line from standard input and answers each in
  turn over one analysis, for callers that ask many questions at once.

## Open questions

1. **Checking before a rename.** Refusing to rename a program with errors is safe, but an agent
   often renames halfway through a change. The alternative is to rename anyway and print a warning
   that uses may be missed. `-force` covers both, so the question is only the default.
2. **Formatting after a write.** A rename can lengthen a line past what `ant fmt` would keep on one
   line. `rename` could run `ant fmt` on each file it changes when that file was formatted before.
3. **The name.** `ant code` groups read and write operations under one word, as `ant doc` names what
   it shows. `ant query` fits the reads but not `rename`; `ant refactor` fits `rename` but not
   `def`.
4. **Bare names and the prelude.** `ant code refs File.write` searches the workspace for calls of a
   prelude function, which is useful when changing how a program handles a library's errors. It
   costs an analysis of every module of the workspace, where a declaration in a module costs only
   the modules that import it.
