# Proposal: `ant code`

2026-11-19 · accepted

## Summary

Add `ant code`, a command that answers the questions an editor asks the language server, from a
terminal: where a name is declared, where it is used, what type it has, which declarations a module
has, and how to rename one. It has five subcommands, `def`, `refs`, `type`, `symbols`, and `rename`.
Each takes a location written as `file:line:col`, `file:line:name`, `file:Name`, or a bare qualified
name such as `Parser.expr`, and prints plain text by default or JSON with `-json`. `rename` writes
files, or prints a unified diff with `-diff`.

```sh
ant code def calc/eval.ant:145:parse
ant code refs calc/lexer.ant:Pos
ant code type logstat.ant:86:5
ant code symbols -match parse calc
ant code rename -diff calc/lexer.ant:Pos Span
ant code rename calc/lexer.ant:Pos Span
```

`ant code` runs the same analysis as `ant lsp`, in process, on the files as they are on disk. It
never runs Ant code. It adds no language rules. It changes one rule of `ant lsp`: references reach
the modules that import the declaring module, as a rename already does.

## Motivation

The language server answers these questions well, but only for a client that speaks the Language
Server Protocol. A coding agent, a shell script, a CI step, or a person in a terminal without an
editor plugin has `grep`, `sed`, and `ant doc`, and those can't tell a type from a field of the same
name, or an import from a local that shadows it. These problems came up while an agent wrote five
Ant programs:

1. **Renaming across modules was unsafe without the server.** Renaming the type `Pos` to `Span` in a
   four-module interpreter meant changing 29 places in three files. `sed 's/Pos/Span/'` would also
   hit `posOf` and comments, and `\bPos\b` would still hit text in strings. `ant lsp`'s rename did
   it exactly, but only through a 40-line JSON-RPC client written for the occasion.
2. **Inferred types are invisible.** After a pipeline such as
   `entries | Array.groupBy(...) | Map.entries() | Array.map(...)`, the only way to learn the type
   of the result, or of a step in the middle, is the editor. `ant doc` describes an exported
   declaration by name, not a private one and not the type at a use.
3. **References stop at the program.** `textDocument/references` on `Pos` found 10 uses, all in the
   declaring module, because a reference search covers the program rooted at the open file
   (`private/tools/lsp.md` section 4). The same position's rename found all 29. A caller asking "who
   uses this?" before changing it gets the shorter answer.
4. **Columns are hard to produce.** An agent reads line numbers from `grep -n` and from file views.
   Counting the column of a name in a line is error-prone, and a wrong column silently answers about
   a different name. `ant doc` already shows that looking up a declaration by name works better.
5. **A short answer and a complete answer look the same.** A search that skipped a file it could not
   parse returns fewer locations and no sign of it. A caller that acts on the list needs to know
   whether it is the whole list.

Each answer costs one process start and one analysis. That is slower than a running server, and fast
enough for a command typed or issued once per step.

## Prior art

| Tool                         | Shape                                                                                     | Positions                                      | What Ant takes                                                                     |
| ---------------------------- | ----------------------------------------------------------------------------------------- | ---------------------------------------------- | ---------------------------------------------------------------------------------- |
| `gopls` command line         | subcommands over the server's own code: `definition`, `references`, `rename`, `symbols` … | `file.go:line:col`, `file.go:#offset`          | One engine for the editor and the terminal. `rename -w` writes, `-d` prints a diff |
| `gorename` (retired)         | one command, one refactoring                                                              | `-from '"pkg".Name'` or `-offset file.go:#123` | Naming a declaration instead of pointing at it                                     |
| `rust-analyzer` command line | `symbols`, `search`, `ssr`, `diagnostics`, `scip`                                         | queries and patterns                           | Batch use of the server's analysis outside an editor                               |
| TypeScript `tsserver`        | a protocol only; scripted refactors go through libraries such as ts-morph                 | offsets                                        | A warning: without a command, scripts build their own clients                      |
| `ant doc`                    | signature and doc comment of a name                                                       | `Owner.name`, matched without regard to case   | Name syntax, output layout for signatures and docs                                 |
| `ant check`                  | errors as `path:line:column: message`, or one JSON object each                            | —                                              | The location format, the JSON fields of a location, the exit statuses              |
| `ant fmt`                    | rewrites files in place; `--check` changes nothing                                        | —                                              | Writing is the default, and how a file is replaced                                 |

Three lessons:

- **The command line should run the server's code, not a copy of it.** `gopls` keeps its command
  line in step with the editor at no extra cost, because each subcommand calls the handler the
  protocol calls. A second implementation would drift.
- **Scripts want names; editors want offsets.** `gorename -from` existed because a script knows what
  it wants to rename, not where the cursor would be. `ant code` takes both.
- **Writing and previewing are separate requests.** An agent that applies a rename should be able to
  review it first.

## Proposal

### Command

```
ant code def     [flags] <loc>          the declaration of a name
ant code refs    [flags] <loc>          every use of a declaration
ant code type    [flags] <loc>          the type, signature, and doc of a name or an expression
ant code symbols [flags] [path...]      the declarations of modules
ant code rename  [flags] <loc> <name>   rename a declaration and its uses
```

| Flag          | Subcommands | Meaning                                                              |
| ------------- | ----------- | -------------------------------------------------------------------- |
| `-json`       | all         | print JSON instead of text                                           |
| `-root dir`   | all         | the workspace: the files searched for importers, default `.`         |
| `-decl`       | `refs`      | include the declaration itself                                       |
| `-match text` | `symbols`   | only declarations whose name contains `text`, without regard to case |
| `-all`        | `symbols`   | include the fields of record types and the declarations in functions |
| `-diff`       | `rename`    | print the change as a unified diff and write nothing                 |
| `-force`      | `rename`    | rename even when the analysis is incomplete or a program has errors  |

Flags follow the subcommand and come before its operands, as with `ant test`.

The workspace is what an editor's workspace folder is to `ant lsp`: every `.ant` file under the
root, skipping entries that start with a dot and `node_modules` (`lsp.md` section 4). There is no
search for a Git root and no project file. A location may name a file outside the root; its
importers are still searched under the root only, and `rename` changes no file outside the root.

`ant code` reads files as they are on disk, each once per run, and never sees an editor's unsaved
text. It parses and checks; it evaluates no initializer, field default, function, or test.

### Locations

A `<loc>` takes one of four forms:

| Form             | Example                       | Means                                                              |
| ---------------- | ----------------------------- | ------------------------------------------------------------------ |
| `file:line:col`  | `calc/lexer.ant:3:13`         | the name at that position                                          |
| `file:line:name` | `calc/lexer.ant:3:Pos`        | the identifier `name` on that line                                 |
| `file:Name`      | `calc/parser.ant:Parser.expr` | a declaration of that module, by name                              |
| `Name`           | `Pos`, `Array.map`            | a top-level declaration of the workspace, or a prelude declaration |

Lines and columns count from 1, and a column counts bytes, as `ant check` prints them. A position
selects the name token that contains it. It is never moved to a nearby name: a position in
whitespace or a comment selects nothing, and a position past the end of the file or inside a UTF-8
character is a usage error.

A name is `Name` or `Owner.name`, the function `name` declared on `Owner` or a field of the record
type `Owner`, matched exactly. Unlike `ant doc`, case matters, since identifiers are case-sensitive
and a location must pick one binding. A by-name form finds private declarations as well as exports.

A location is ambiguous when it matches more than one binding: an identifier that appears twice on
the line with different bindings, a bare name declared in two modules, or `Owner.name` matching both
a function and a field. `ant code` then answers about none of them: it prints each candidate as a
`file:line:col` location with its line, and exits with status 1, so the caller can repeat the
request with one of them. A bare name never collides with the prelude, since a module cannot
redeclare a prelude name (section 10).

### Output

Text output is made of locations, one per line, in the form `ant check` and `grep -n` use:
`path:line:col: text`. Paths are relative to the current directory when the file lies under it, and
absolute otherwise. A declaration of the prelude has the location `prelude.d.ant:line:col`, the
read-only source `ant lsp` serves as `ant:/prelude.d.ant`. Standard output holds only the answer;
counts, warnings, and errors go to standard error.

With `-json`, standard output is one JSON object and a newline, described with each subcommand
below. A location in it has the fields of an `ant check -json` error without the message: `path`,
`line`, `column`, and `start` and `end`, each a `line` and a `column`, with `end` just past the
name. The object of a prelude location has `"prelude": true`. The same input gives the same output,
byte for byte. A later version may add fields; a client ignores those it does not know.

Every object of `refs` and `rename` carries `"complete": bool` (Completeness, below). A request that
fails prints, in place of its answer, an object with one field, `error`:

```json
{
  "error": {
    "code": "ambiguous",
    "message": "Pos names 2 declarations",
    "candidates": [
      {
        "path": "calc/lexer.ant",
        "line": 3,
        "column": 13,
        "start": { "line": 3, "column": 13 },
        "end": { "line": 3, "column": 16 }
      },
      {
        "path": "geo/point.ant",
        "line": 8,
        "column": 13,
        "start": { "line": 8, "column": 13 },
        "end": { "line": 8, "column": 16 }
      }
    ]
  }
}
```

| Code        | When                                                    | Extra field                                |
| ----------- | ------------------------------------------------------- | ------------------------------------------ |
| `usage`     | wrong flags or operands, or a position outside the file |                                            |
| `load`      | the location's file cannot be read or parsed            |                                            |
| `not-found` | the location selects nothing, or a name is not declared |                                            |
| `ambiguous` | the location matches more than one binding              | `candidates`: locations                    |
| `refused`   | the declaration or the new name cannot be renamed       |                                            |
| `check`     | a program a rename changes has errors                   | `errors`: the objects of `ant check -json` |
| `changed`   | a file changed between the analysis and the write       | `paths`                                    |
| `write`     | a file could not be written                             | `written` and `unwritten`: arrays of paths |

### `def`

Prints the location of the declared name, then its signature as `ant doc` prints it, on one line.
The declaration is the one `textDocument/definition` finds (`lsp.md` section 4): for a call through
`.`, the function it calls or its signature in the interface; for a field, the field in its record
type; for an import name, the declaration it imports; for the path of an `import`, the file.

```text
$ ant code def calc/eval.ant:145:parse
calc/parser.ant:147:13: func parse(tokens: Array[Token]): Result[Expr, ParseError]
```

JSON: `{ "location": Location, "signature": string }`.

### `refs`

Prints every use of the declaration, one location per line with the source line after it, in path
and then position order. The declaration itself is left out unless `-decl` is given. The search
covers the program rooted at the location's file and every module of the workspace whose imports
lead to the declaring module: the same set of modules a rename changes (Changes to `ant lsp`,
below). Test files are among them, and a test file may import a private declaration, so a private
top-level name is searched the same way. The answer is the same whether the location is the
declaration or one of its uses.

Uses are matched by binding, not by spelling: two parameters called `value`, or the fields `id` of
two record types, stay apart. An import alias is a binding of its own. From the alias, `refs` finds
the alias and its uses in that module. From the export, it finds every import and use, and those
written through an alias are marked `(as Name)`.

```text
$ ant code refs calc/lexer.ant:Pos
calc/eval.ant:2:24: import "./lexer.ant" { Pos, tokenize }
calc/eval.ant:11:49: export type EvalError = { message: string, pos: Pos }
...
calc/lexer.ant:5:42: export type Num = { value: float64, pos: Pos }
...
28 references in 3 files
```

JSON:
`{ "declaration": Location, "references": [{ "location": Location, "alias": string | null }], "complete": bool, "skipped": [string] }`.

### `type`

For a name, prints what hover shows (`lsp.md` section 4), as plain text: the kind and name with the
type the checker inferred, narrowed at that use, then the doc comment indented by four spaces. For a
function or a type it prints the signature as written. For a field it prints `name: Type` and the
record type it belongs to.

```text
$ ant code type logstat.ant:86:5
let topPaths: Array[Entry[string, int]]
```

For a position on a literal or an operator, where hover shows nothing, it prints the type of the
innermost expression that contains the position: on the `+` of `n + 1`, the type of the sum; on the
second `|` of a pipeline, the type of the pipeline up to and including that step.

```text
$ ant code type logstat.ant:84:36
Map[string, Array[Entry]]
```

A position on a keyword, in a comment, or in whitespace has no answer and exits with status 1, and
so does a name the checker could not resolve: `type` never prints a guessed type. JSON:
`{ "kind": string, "text": string, "doc": string, "location": Location }`, where `kind` is
`"expression"` for the second case and `location` is the name or the expression described.

### `symbols`

Prints the declarations of the modules under each path, a directory searched as the workspace is,
default the root. Each line is the location of the name, then the signature, which starts with the
kind of declaration. Functions on a type follow the type, and in a test file the tests follow, with
their full names as `ant test -run` matches them (`spec/test.md` section 1). Private top-level
declarations are listed with the exports, since `ant doc file.ant` already lists the exports alone;
an export's signature starts with `export`. `-all` adds the fields of record types and the
declarations inside functions.

```text
$ ant code symbols -match parse calc
calc/parser.ant:15:13: export type ParseError = { message: string, pos: Pos }
calc/parser.ant:31:6: func parsePrimary(p: Parser): Result[Expr, ParseError]
calc/parser.ant:147:13: export func parse(tokens: Array[Token]): Result[Expr, ParseError]
```

`symbols` needs a file to parse, not to check, and reads past syntax errors as `ant lsp` does: it
lists what it recovered, names the file on standard error, and exits with status 2. JSON:
`{ "symbols": [{ "location": Location, "kind": string, "name": string, "signature": string, "exported": bool, "container": string | null }] }`.

### `rename`

Renames the declaration at the location and every use `refs` reports, under the rules of
`textDocument/rename` (`lsp.md` section 4):

- It refuses a name that is a keyword, a reserved word, or a prelude name, or that is already bound
  in a scope the renamed binding reaches, in the declaring module or in an importer.
- It refuses to rename a prelude declaration, a namespace, a record field, an element of markup, or
  a lambda's `x`. A refusal says which of these it is; `rename` never falls back to replacing text.
- Renaming an export changes its name in each import and each use; a use through an alias keeps the
  alias. Renaming an alias changes that binding and its uses in the one module.
- A record entry or a destructuring written short keeps its field: renaming the variable `value` to
  `amount` turns `{ value }` into `{ value: amount }`.
- It changes only names. Comments, strings, and layout stay as they are, and a name inside a
  string's interpolation is a use like any other. A line whose layout `ant fmt` would now change is
  left for `ant fmt`.
- The same name is a rename that changes nothing, with status 0.

Renaming resolves member calls through the checker, so a use the checker can't type may be missed.
Before it writes, `rename` checks every program it changes, and refuses when any of them has an
error, printing the errors as `ant check` does. Errors in programs it does not change don't matter.
It refuses as well when the search is incomplete (Completeness, below). `-force` renames anyway in
both cases, with a warning on standard error.

Without `-diff`, `rename` writes each changed file and prints one line per file with its count of
edits. With `-diff`, it writes nothing and prints a unified diff of every file it would change.

```text
$ ant code rename calc/lexer.ant:Pos Span
calc/eval.ant: 5 edits
calc/lexer.ant: 10 edits
calc/parser.ant: 14 edits
```

JSON:
`{ "files": [{ "path": string, "edits": [{ "location": Location, "newText": string }] }], "written": bool, "complete": bool, "skipped": [string] }`,
with the edits of a file in source order, none overlapping, and `written` false under `-diff`.

A rename of an export covers the workspace and nothing else. Modules outside the root that import
the declaration are not found, and a name kept in a string, such as a key a program passes to
`Reflect`, is not changed.

### Writing files

A file is replaced as `ant fmt` replaces one: the new text goes to a temporary file beside it, which
is renamed into place with the file's mode, and a symbolic link named on the command line stays a
link. Files with no edits are not touched.

Before the first file is replaced, `rename` reads every file it is about to change again and
compares it with the text it analyzed. If one differs, it writes nothing and exits with status 1,
naming the file; running the command again analyzes the new text. `-diff` followed by a second run
without it is two analyses, not a saved plan.

Several files can't be replaced in one step. When a write fails after others succeeded, `rename`
lists the files it wrote and the files it did not, exits with status 2, and undoes nothing, since
another process may have changed the written files since. The check above catches a change it sees;
it locks nothing.

### Completeness

`refs` and `rename` find importers by reading the imports of every file of the workspace. A file
that can't be read or does not parse may import the declaring module without `ant code` knowing. The
answer is then incomplete:

- `refs` prints the uses it found, names each such file on standard error, and exits with status 2.
  Its JSON has `"complete": false` and the files in `skipped`.
- `rename` refuses, as above, unless `-force` is given.

A complete answer with no uses prints nothing and exits with status 0. It means no use within the
workspace, not no use anywhere.

### Exit status

| Status | When                                                                                       |
| ------ | ------------------------------------------------------------------------------------------ |
| `0`    | the request was answered, completely, or the files were written                            |
| `1`    | nothing was found, the location is ambiguous, a rename was refused, or a file changed      |
| `2`    | the usage is wrong, a file cannot be read, parsed, or written, or the answer is incomplete |

These are the statuses of `ant check`: 1 for an answer the caller may not want, 2 for a request that
could not be carried out in full. An empty list of references is an answer, with status 0.

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
ant code refs calc/lexer.ant:Pos                # 28 uses in 3 files: is this the right name?
ant code rename -diff calc/lexer.ant:Pos Span   # review the edits
ant code rename calc/lexer.ant:Pos Span
ant check calc && ant test calc
```

## Spec changes

- New `private/tools/code.md`, the contract of the command, beside `lsp.md` and `dap.md`: sections
  for the command, locations, output and its JSON, each subcommand, writing files, completeness, and
  the exit statuses, referring to `lsp.md` section 4 for what each answer contains.
- `private/tools/lsp.md` section 4: references reach importers (Changes to `ant lsp`, above).
- `docs/tools/code.md`, a reference page in the style of `docs/tools/doc.md`, with tested examples,
  one of them the agent's sequence above with `-json`.
- The usage text of `ant`, and the bash, zsh, and fish completions in `cmd/ant`.

`spec/ant-lang.md` does not change.

## Implementation sketch

Each feature of `internal/lsp` is a method on `*Server` that takes the protocol's parameter types:
`definition`, `references`, `hover`, `rename`, `documentSymbols`, and `workspaceSymbols`. `ant code`
builds a `Server` in process with the root as its workspace folder and the `utf-8` position
encoding, which the server already supports, opens the location's file as a document read from disk,
and calls the method. No protocol messages are exchanged.

A small exported layer in `internal/lsp` does what a client would otherwise do:

- turns a `<loc>` into a position: a column is used as it is, an identifier on a line is found among
  the line's identifier tokens, and a `Name` is found among the module's document symbols or the
  workspace symbols, with candidates collected for the ambiguity report;
- answers `type` on an expression from `check.Info.Types`, which holds the type of every checked
  expression, by way of the selection ranges the server already computes;
- applies a `WorkspaceEdit` to the files, from the last edit to the first, or renders it as a diff;
- renders hover markdown as plain text with `ant doc`'s layout;
- reports the workspace files the importer scan could not read or parse.

`references` gets the importer pass that `rename` already has (`references.go`, the loop over
`s.importers`), as one function both call, so the two can't disagree about what a use is. The
shorthand and alias rules above are what that code does today. `cmd/ant/code.go` parses the flags,
calls the layer, and prints.

Tests run each subcommand over a small workspace in `cmd/ant/testdata`, with the text and JSON
output as golden files, and reuse the server's existing tests for the answers themselves. The
workspace covers:

- the three-module rename of the motivation, checked and tested after it is applied, and the same
  count of references from the declaration and from a use;
- an importer nothing has opened, and a test file that imports a private declaration;
- an initializer that would write a file if it ran, with every subcommand run over it;
- a path with a space, a name after a multi-byte character, and CRLF line endings;
- each error code, among them a refused field rename that writes nothing, a file changed before the
  write, and a write that fails on the second file;
- an importer that does not parse, for `refs`, `rename`, and `rename -force`.

Moving the analysis out of `internal/lsp` into a package of its own, with the server and the command
as two adapters, is not needed for any of this, and can follow if the layer grows.

## Non-goals

- **A daemon.** `ant code` starts, answers, and exits. A long-lived process that caches the analysis
  between calls is what `ant lsp` is.
- **New refactorings.** `ant code` offers what the server offers. Renaming fields, moving a
  declaration to another module, extracting a function, and changing a signature are each a feature
  of the server first, and reach `ant code` through it.
- **Diagnostics.** `ant check [-json] [path...]` reports errors.
- **Formatting.** `ant fmt` formats; `rename` leaves layout to it.
- **Applying a saved plan.** The JSON of `rename -diff` describes a change; it is not input to a
  later run.
- **Search and replace by pattern**, as `rust-analyzer ssr` does. It is a larger design and may
  build on this one later.

## Later

- **Fixes.** `ant code fix <file.ant>`: apply the preferred code action of each error (`lsp.md`
  section 4), which states an inferred type, adds a record literal's missing fields, or imports a
  name one module exports. It waits for `ant check` to give its errors codes, so that a fix is
  chosen by code and not by message.
- **Callers.** `ant code callers <loc>`: the calls of a function, grouped by the function they occur
  in. It needs `textDocument/prepareCallHierarchy` and `callHierarchy/incomingCalls` in the server,
  which `lsp.md` section 5 leaves out today.
- **Field rename**, once the server can rename a field through record literals, destructuring,
  spreads, and structural types, and the preview can say what changes in `Json.stringify` output.
- **Batches.** A `-` location reads one request per line from standard input and answers each in
  turn over one analysis, for callers that ask many questions at once.

## Decisions

1. **Writing by default.** `rename` writes unless `-diff` is given, as `ant fmt` writes unless
   `--check` is given; the check before the write guards it.
2. **Checking before a rename.** `rename` refuses a program with errors, since a use the checker
   can't type may be missed; the refusal names `-force`.
3. **Checking after a rename** is left out until a capture the collision rules miss is found.
4. **Formatting after a write** is left to `ant fmt` (Non-goals).
5. **Bare names and the prelude.** `ant code refs File.write` searches every module of the
   workspace; the cost is paid only when asked for.
