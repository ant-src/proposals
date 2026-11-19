# Proposal: `ant code`

2026-11-18 · draft

## Summary

Add `ant code` for semantic source navigation and refactoring from a terminal. Its first commands
list symbols, inspect types and declarations, find definitions and references, and rename symbols.
Every command has readable terminal output and a structured JSON form. Queries never execute Ant
code. Rename prints a preview unless `--write` is given.

```sh
ant code symbols src/
ant code symbols --query reserve .
ant code inspect src/api.ant:24:13
ant code definition src/api.ant:24:13
ant code references --root . src/api.ant:24:13
ant code rename --root . src/store.ant:22:19 reserveStock
ant code rename --root . --write src/store.ant:22:19 reserveStock
```

The command and the language server use one semantic implementation. This proposal adds a terminal
interface and makes workspace reference searches complete; it does not change Ant's syntax, checking
rules, or execution semantics.

## Motivation

Writing five Ant programs across CSV reporting, JSON log analysis, deployment planning, binary
telemetry, and concurrent HTTP reservations exposed a gap between the tools available in an editor
and those available through a shell.

- Inspecting types and navigating code required an LSP client with initialization, document
  notifications, message framing, and response handling. An agent running commands needed to write
  that client before using the semantic operations.
- `ant doc` helped when the name of an exported declaration was known. It intentionally does not
  describe private declarations or the inferred type at a particular use.
- Finding references to the planner's exported function at its declaration returned only that
  declaration. The same server's rename operation found 12 edits across the declaration, a script,
  and a test file. A reference query from the script saw that program's references but missed the
  test callers.
- Function rename worked and its edits passed checking and tests. Renaming a record field was
  explicitly unsupported. A terminal interface must distinguish the operations it can perform from
  the ones that need further semantic implementation.

These observations were made against implementation revision
`a97cc3d53ef5fb6fb7824a8ad6f6c7e928202853`. They motivate a small, callable interface over the
existing semantic tools, with explicit search scope and reliable failure reporting.

## Prior art

The
[Language Server Protocol](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/)
already defines hover, symbols, definitions, references, and rename. Ant implements these
operations. The protocol serves an editor session; shell tools benefit from an invocation that opens
the source, performs one operation, prints the answer, and exits.

[gopls](https://pkg.go.dev/golang.org/x/tools/gopls) also exposes semantic operations through its
command-line interface. The useful precedent is sharing a language-aware implementation between
terminal and editor clients.

Within Ant, `ant doc` establishes compact declaration output, `ant check` establishes source
positions and diagnostics, and `ant fmt` establishes explicit file operations. `ant code` follows
those conventions while adding JSON results and a preview for semantic edits.

## Command interface

```text
ant code symbols [flags] [path...]
ant code inspect [flags] <file:line:column>
ant code definition [flags] <file:line:column>
ant code references [flags] <file:line:column>
ant code rename [flags] <file:line:column> <new-name>
```

Flags follow the subcommand and precede its operands, as with `ant test`. `--` ends flags. Both
`-name` and `--name` spell a flag; the examples use the latter. `ant code --help` lists commands,
and `ant code <command> --help` describes that command's operands, flags, output, and exit statuses.

| Flag                 | Commands     | Meaning                                                |
| -------------------- | ------------ | ------------------------------------------------------ |
| `--json`             | All          | Emit one JSON result on standard output                |
| `--root <directory>` | All          | Workspace boundary; defaults to the current directory  |
| `--query <text>`     | `symbols`    | Filter qualified names by a case-insensitive substring |
| `--declaration`      | `references` | Include the declaration in the locations               |
| `--write`            | `rename`     | Apply the computed edits instead of only showing them  |

An unknown flag, missing operand, extra operand, or flag used with another command is a usage error.
With no paths, `symbols` lists the workspace root. Explicit paths and target filenames are resolved
relative to the current directory, not relative to `--root`. The root never changes the process's
working directory.

### Positions

A position is a filename followed by `:line:column`. The last two colon-separated components are
positive decimal integers; everything before them is the filename, so drive letters and colons in a
filename remain possible. A filename containing spaces is quoted as one shell argument.

Lines and columns start at 1. A column counts UTF-8 bytes from the start of the line, matching Ant's
parser and CLI diagnostics. It is not a display column or a count of UTF-16 code units. Tabs occupy
one byte. LF, CRLF, and CR each end one line. JSON ranges also carry zero-based byte offsets into
the original source. Their ends are exclusive.

A position outside the file or inside a UTF-8 character is rejected, not clamped to a nearby name.
The position immediately after the last character of a line is valid but may have no target. Symbol
operations select the name token containing the position. They do not guess a nearby symbol from
whitespace or comments. `definition` also accepts a position in an import path.

### Workspace and source discovery

The root is an explicit directory, canonicalized to an absolute path. There is no Git-root search or
project manifest discovery. Directory searches walk recursively in lexical order, including scripts,
libraries, `_test.ant` files, and `.d.ant` files. They skip entries beginning with `.` and
directories named `node_modules`; they do not consult `.gitignore` or follow symbolic links found
during the walk.

An explicitly named file or directory may be a symbolic link and is resolved to its target. Imports
are resolved as the language specifies, even when their targets are outside the root or in a
directory excluded from discovery. Canonical paths identify modules and deduplicate locations.

The root bounds reference searches and writes. `symbols` may list explicitly named paths outside it;
the root supplies its search directory only when no paths are given. `inspect` and `definition` may
read a target outside it and follow its imports. `references` may inspect an external declaration to
find its uses inside the root. It reports no uses outside that root; `--declaration` may
additionally include the external declaration itself. Rename requires both the selected declaration
and every proposed edit to be inside the root. A local import alias may be renamed even when its
imported declaration is outside.

The CLI reads files as they exist on disk. It never sees an editor's unsaved buffers. All analysis
within one invocation uses captured source bytes, including when a file is reached through several
programs. It never evaluates initializers, field defaults, functions, or test bodies.

## Operations

### `symbols`

List declarations in the selected files. Each item has a qualified name, kind, signature as written,
and declaration location. Private declarations are included. The initial scope is top-level
functions, types, interfaces, bindings, declared members, and top-level tests. A destructuring lists
each bound name. Function-local declarations and parameters are inspected by position rather than
included in this listing.

An owner's members are qualified, such as `Store.reserve` or `Route.p95`. Query matching applies to
that qualified name, not to documentation or function bodies. Symbols are ordered by canonical file
path and declaration position. Duplicate input paths do not duplicate entries.

This operation needs parsing, not successful type checking. An unannotated binding is listed without
inventing a type; `inspect` supplies its inferred type. A parse failure makes the listing
incomplete, but declarations recovered without ambiguity may still be returned.

### `inspect`

Describe the selected symbol or expression. Return its source range, type at that use, declaration
signature, documentation, and definition location where these exist. The type at a use includes flow
narrowing. The signature describes the declaration; it is not replaced with an inferred signature or
presented as the narrowed type.

A symbol token selects that symbol. Otherwise, an operator or literal selects the innermost checked
expression containing it. For example, inspecting `n` after a null guard reports its narrowed type;
inspecting `+` in `n + 1` reports the addition's result type. Expression inspection reuses the
checker's recorded expression types and extends the current symbol-oriented hover lookup.

A private declaration is inspectable like an export. An import path returns the resolved module
location and its module documentation. A known symbol without documentation has an empty doc string.
An expression without a declaration has a null symbol. Failure to resolve a name must not be
rendered as a successful inspection with an invented type.

### `definition`

Return the selected symbol's declaration location. Named imports and their aliases lead to the
original declaration. A module namespace or import path leads to the imported file's start. Prelude
definitions use the virtual document URI `ant:/prelude.d.ant`.

A resolved built-in that has no source declaration yields an empty location list. A position with no
symbol or import path yields `no-target`. A name that cannot be resolved yields `unresolved`. The
result represents static resolution; it does not predict a function value's runtime target.

### `references`

Find uses of the selected binding in the workspace, including declarations only with
`--declaration`. Match resolved symbol identity, not spelling. Distinct parameters called `value`
and fields of unrelated named record types called `id` must remain distinct. If the checker cannot
establish a unique field binding, report `unresolved` rather than choosing one by spelling.

The search includes unopened files, tests, and importers of the declaring module. A test may import
a private declaration, so being private does not establish that a top-level symbol has no callers.
The result must be the same when selected at a declaration or an unaliased use of that declaration.

For example, these three files contain five locations for `double`: its declaration, two imports,
and two calls.

```ant [math.ant]
export func double(n: int): int { n * 2 }
```

```ant [main.ant]
import "./math.ant" { double }

println(double(21))
```

```ant [math_test.ant]
import "./math.ant" { double }

test "double" { assert(double(3) == 6) }
```

`ant code references --declaration math.ant:1:13` finds all five when these files are in the root.
Without `--declaration`, it finds four. The word in the test's name is a string and is not a
reference. Selecting the call at `main.ant:3:9` produces the same result.

An explicit import alias is a binding of its own. Selecting its declaration or a use searches that
local alias. Selecting the original exported declaration finds imports and uses of the export,
including uses through aliases; those occurrences carry `viaAlias: true`. This distinction also
governs rename.

Discovery first parses the workspace's imports and builds reverse dependency edges. It then analyzes
the programs needed to resolve uses of the target, including transitive importers where values or
types can flow through another module. The same declaration is identified across separate checker
runs by canonical source location and kind, not by an in-memory object pointer. Results are
deduplicated by URI and source range and sorted in that order.

A file that cannot be scanned or parsed may hide an importer. Such a search is incomplete, even if
the known programs produced useful results. Diagnostics name the affected files. A successful empty
list means no references were found within the declared scope, not that no references exist outside
the workspace.

### `rename`

Compute edits for the selected binding and a new identifier. The default output is a unified diff;
`--json` returns the edit plan. `--write` computes and applies the same operation. A write
invocation recomputes from current disk contents; it does not replay a previous preview.

The initial supported targets are declared variables and constants, functions, named types,
interfaces, parameters, type parameters, and import bindings whose identities are unambiguous. The
prelude, implicit lambda `x`, test names, namespace declarations, and record fields are unsupported
targets. A module import binding is renamable; that does not rename its file or create a language
namespace declaration.

This initial support follows existing semantic rename capabilities. Named record-field rename is
separate work: constructors, destructuring, structural types, and serialized field names need a
contract of their own. It remains discoverable through `symbols`, `inspect`, and `definition`, and
an attempted rename returns `unsupported-target` with the reason. The CLI must not silently turn
that request into a textual replacement.

Rename obeys these rules:

- Validate the new identifier using Ant's keyword, reserved-word, and prelude-name rules. Reject
  `_`. An unchanged name is a successful no-op.
- Update declarations and resolved uses, including type expressions, qualified calls, and named
  arguments when a parameter is renamed.
- Renaming an import alias changes that binding and its local uses. Renaming an export changes the
  export name in imports and unaliased uses; explicit aliases and their uses retain their names.
- Preserve the external name of a shorthand field when its bound variable changes: renaming local
  `value` to `amount` changes `{ value }` to `{ value: amount }`. Apply the corresponding rule to
  destructuring. The selected binding determines which side changes.
- Detect collisions and accidental capture in every affected scope and importer, including private
  imports from tests. Check both existing uses of the target and other bindings that the new name
  could shadow.
- Preserve comments, documentation, string contents, unrelated identifiers, line endings, and
  surrounding formatting. Code inside string interpolation is a normal semantic use. Do not run the
  formatter as a side effect.

A rename requires complete workspace discovery and successful analysis of affected programs. It must
fail before writing if it cannot establish those conditions. The edited snapshots are parsed,
resolved, and checked before a plan is accepted, with binding checks ensuring that uses still name
their intended declarations. A type check alone does not prove that a newly shadowed name kept its
meaning. Existing type errors in an affected program block rename; unrelated type errors need not.

Scope is explicit: renaming an export updates its uses inside the chosen root. Consumers outside
that root may require a corresponding change. This is reported with the target's export status, not
inferred to be a complete migration of every consumer.

Rename preserves binding relationships. Names visible through reflection, printed type tags, and
stack traces may change with the declaration. It does not find or update names stored as strings.

## Results, diagnostics, and exit status

### Terminal output

`symbols`, `definition`, and `references` print one location per line, followed by the symbol name
or a short source excerpt. Paths are relative to the current directory where possible. `inspect`
prints the signature, type at the use, definition location, and doc comment. An empty successful
list prints nothing. Rename previews print a unified diff; `--write` prints the changed paths.

Errors and analysis diagnostics go to standard error. An incomplete result is explicitly labeled
there and has a nonzero exit status. Terminal styling follows `ant doc`: no color for redirected
output, `NO_COLOR`, or a dumb terminal. Text output is intended for people; JSON is the scripting
contract.

### JSON output

`--json` writes one object followed by a newline, with no progress messages or color codes on
standard output. It uses the same envelope for success, partial results, and failures:

```json
{
  "schemaVersion": 1,
  "command": "definition",
  "scope": {
    "kind": "program",
    "root": "file:///work"
  },
  "complete": true,
  "result": {
    "locations": [
      {
        "uri": "file:///work/math.ant",
        "range": {
          "start": { "line": 1, "column": 13, "offset": 12 },
          "end": { "line": 1, "column": 19, "offset": 18 }
        }
      }
    ]
  },
  "diagnostics": [],
  "errors": []
}
```

This example locates `double` in `export func double(n: int): int { n * 2 }`. Real files use
absolute, canonical file URIs; virtual prelude locations use the URI given above. Ranges always
refer to the captured, unmodified source bytes. JSON coordinate fields follow this proposal's
one-based byte columns, not LSP's coordinate conventions.

`scope.kind` is `files` for `symbols`, `program` for `inspect` and `definition`, and `workspace` for
`references` and `rename`. A `files` scope additionally lists the canonical explicit input URIs,
with the root as the input when no paths were given. `complete` means analysis covered the declared
scope sufficiently for the operation. It does not mean the workspace has no unrelated errors or that
a write succeeded. Missing relevant information makes it false; the tool never silently truncates a
result. `result` is null when there is no useful result. For a usage error before the scope can be
established, `scope` is null and `complete` is false.

| Command      | `result` fields                                                               |
| ------------ | ----------------------------------------------------------------------------- |
| `symbols`    | `symbols`: an array of symbol records                                         |
| `inspect`    | `location`, `type`, `symbol`, and `module`; fields that do not apply are null |
| `definition` | `locations`: declaration locations, possibly empty                            |
| `references` | `symbol` and `locations`; each occurrence adds `isDeclaration` and `viaAlias` |
| `rename`     | `symbol`, `newName`, `sources`, `files`, and `written`                        |

A symbol record has `name`, `qualifiedName`, `kind`, `exported`, `signature`, `doc`, `declaration`,
and `aliasOf`. Names are strings, `exported` is a boolean, and `declaration` is a location or null
for a symbol without a source declaration. `qualifiedName` includes an owner where there is one,
such as `Store.reserve`. `signature` is a string or null where no signature applies; `doc` is an
empty string when undocumented. `aliasOf` is the imported declaration's location for an explicit
alias and null otherwise. Kinds include `variable`, `constant`, `function`, `type`, `interface`,
`parameter`, `type-parameter`, `namespace`, `import`, `field`, and `test`.

For inspection, `type` is the type at the use as a string or null when no value or type is selected.
`module` is an object with `location` and `doc` when inspecting an import path, and null otherwise.
Type and signature fields are Ant text for display, not a separate structured type language.

For rename, `sources` lists the captured source URIs and SHA-256 digests used to compute the plan.
`files` lists each edited URI and its edits; each edit has an original-source range and `newText`.
Edits must not overlap. `written` lists successfully replaced file URIs and is empty in a preview.
The plan contains no editable result when analysis is incomplete or rename is refused.

Locations and file lists are sorted by URI and start offset. Edits are emitted in ascending source
order and can be applied in reverse order. Identical inputs produce identical results; the envelope
contains no timestamps or elapsed durations. Clients must accept additional object fields. A
breaking change to existing fields or coordinate rules increments `schemaVersion`.

`diagnostics` contains source-analysis messages with `phase`, `severity`, `code`, `location`, and
`message`. Phases are `read`, `parse`, `load`, `resolve`, and `check`; severities are `error` and
`warning`. A location has a null range for a file-level error, and is null only when no file is
known. Diagnostic codes are nullable until the checker supplies stable codes; this command does not
invent codes by matching message text.

`errors` contains command failures with `code`, `message`, and a `relatedLocations` array where
applicable. Initial codes include `usage`, `invalid-position`, `invalid-name`, `no-target`,
`unresolved`, `unsupported-target`, `collision`, `outside-root`, `incomplete-analysis`,
`changed-source`, `read-failed`, and `write-failed`. In JSON mode ordinary failures use this
envelope instead of also printing a human report. If standard output itself cannot be written,
standard error reports that failure.

| Status | Meaning                                                                                         |
| ------ | ----------------------------------------------------------------------------------------------- |
| `0`    | Complete query, valid rename preview, successful write, or a successful empty/no-op result      |
| `1`    | No target, unresolved symbol, semantic refusal, changed source, or incomplete semantic analysis |
| `2`    | Invalid usage or position/name syntax, or an I/O, source parsing/loading, or write failure      |

Status 2 outranks status 1. Read-only commands may return useful partial results with status 1 or 2.
Rename returns no applicable edits on refusal or incomplete analysis. A preview that would change
files exits 0: changes are the requested result, not a failed check.

## Writing files

Before the first replacement, validate all edit ranges, verify root boundaries and destination
permissions, prepare replacement files, and recheck the captured source digests. A detected change
to any source used in analysis aborts with `changed-source` and no replacements. A later invocation
recomputes the operation. No files are changed merely to make the workspace checkable.

Write each changed file through a temporary sibling and atomic replacement, preserving its file
mode. An explicitly selected symbolic link remains a link; its canonical target is the file being
edited. Unchanged files are not rewritten. No interactive prompt, Git operation, or automatic commit
is part of `--write`.

Multiple files cannot be replaced atomically as one portable filesystem operation. A write failure
after some replacements exits 2 and reports the files already written and those still pending. It
does not report success or automatically roll back over possible concurrent edits. A process crash
may likewise leave a partially applied rename. Source checks detect observed concurrent changes;
they do not provide a filesystem transaction or lock out other writers. These limits belong in the
command's documentation and failure tests.

## Shared implementation

Move the reusable semantic operations into `internal/code`, with the CLI and LSP as adapters. The
shared layer accepts a source provider and workspace boundary and returns typed Go results using
byte offsets. The CLI supplies disk snapshots; LSP supplies disk sources with open-buffer overlays.
Each adapter converts coordinates and renders its own output.

The existing checker already records expression types, identifier and member bindings, call
signatures, and declaration metadata in `check.Info`. Existing LSP code supplies target lookup,
declaration descriptions, symbols, collision checks, and rename edits. Extract those operations
incrementally with their tests. The shared layer must not depend on JSON-RPC messages or an LSP
`Server` instance.

Reference search needs a behavioral fix as part of this work. Its current implementation searches
the program rooted at the requested document, while rename separately analyzes workspace importers.
Build one workspace reference operation and use it for both commands and their LSP equivalents.
Preserve the alias distinction and name-collision checks when doing so. Both clients must benefit
from the fix; neither should maintain a separate approximation of symbol identity.

Start with an invocation-local cache of parsed files and analyzed roots. A daemon, persistent index,
or public Go API is not required to ship these commands. Measure repeated-command costs on real
workspaces before choosing a persistent service contract.

## Delivery and acceptance

1. **Extract query operations and add the CLI.** Ship `symbols`, `inspect`, and `definition` with
   text and JSON output. Test private declarations, inferred and narrowed types, expression types,
   imported names, declaration files, and prelude locations. Include an initializer that would write
   a file or panic if executed, and verify that queries do neither.
2. **Unify workspace references.** Add reverse-import discovery, source-based identity across
   checker runs, partial-result diagnostics, and `references`. Fix the LSP behavior at the same
   time. The planner regression must return 12 locations with `--declaration`, or 11 without it,
   from either its declaration or an unaliased use. The three-file example above must yield five
   with `--declaration`. Test unopened importers and private imports in tests.
3. **Expose rename preview and write.** Cover supported target kinds, explicit aliases, shorthand
   preservation, parameter names at call sites, and collisions in importers. Apply the planner's
   12-edit rename across three files, check the result, and run its tests. Verify unsupported fields
   produce a structured refusal and no writes.
4. **Verify the command contract.** Test paths with spaces and colons, UTF-8 before a target, CRLF,
   invalid positions, discovery exclusions, external imports, empty results, parse/type errors, and
   deterministic JSON. Test changed-source detection, a no-op rename, and injected failures before
   and during file replacement. Document the commands and add shell completion.

The five commands, both output forms, complete references, and guarded writes form the initial
delivery. The tests for extracted operations should run against the shared layer, with adapter tests
checking equivalent CLI and LSP results under equivalent source snapshots and roots.

## Specification and documentation changes

- Add `code.md` to the specification repository and link it from its README. Specify command syntax,
  positions, discovery, operation semantics, JSON fields, exit statuses, and write behavior.
- Update the command overview associated with language section 4, Checking and running, and section
  10, Modules. State that these operations read and check source without running it.
- Add a `tools/code.md` documentation page, with runnable fixtures showing navigation and a rename
  from preview to checking and testing. Add `ant code` to CLI help and shell completions.
- Document the shared reference scope and alias behavior for LSP users. The current checkout's
  implementation refers to LSP specification material not present in its spec tree; the CLI contract
  should be self-contained and the editor documentation should state the same semantics.
- Add an agent-oriented example that discovers a symbol, inspects it, finds its uses, and consumes a
  rename plan as JSON. It should require no protocol client or third-party package.

The language's advisory type checking remains unchanged. `ant code` checks what it needs for a
semantic query or edit; `ant run` and `ant test` retain their existing behavior.

## Non-goals

The initial command does not run tests or debugging sessions, evaluate expressions, replace
`ant check`, format source, search documentation by task, or introduce a project manifest. It does
not add extract-function, move-file, signature-changing, record-field, or serialization
refactorings. It does not read unsaved editor buffers, replay saved edit plans, or operate a
background daemon. These features can be proposed separately with their own observed use cases and
contracts.

## Open questions

- **Selecting by name.** A future form such as `--symbol Store.reserve --file store.ant` could avoid
  locating a column for a top-level declaration. Positions remain the initial form because they also
  identify local bindings and distinguish shadowed names. Review whether named selection is valuable
  enough to include now without introducing ambiguous resolution rules.
- **Reusing an approved preview.** The initial `--write` recomputes from disk. A future plan token
  or saved-plan application could bind a write to exactly the previously reviewed snapshots and
  edits. It would need versioning, stale-source rejection, and a defined recovery model; this
  proposal does not treat its JSON previews as executable input.
- **Record fields.** The log analyzer's `p95` to `p95Ms` case establishes a practical need. A
  follow-up should decide which named and structural fields can be renamed soundly and how a preview
  describes the change to `Json.stringify` output without pretending to migrate external consumers
  or rewriting unrelated string keys.
