# Proposal: scripts without `main`

2026-10-25 · Anton

## Summary

Drop `func main()`. A module may hold statements at the top level, between and after its declarations, and `ant run` executes them in source order as the last step of initialization. A module with at least one top-level statement is a **script**. A script cannot be imported and cannot export, so a program has exactly one script, the file given to `ant run`, and every other module stays a library: declarations whose `var` initializers may still run code, as today. A script's top-level `var`s narrow like locals. `func`, `type`, `import` and `export` are unchanged, and no new keyword or syntax is added.

```ant
// hello.ant
println("hello, world")
```

## Motivation

1. **Every program starts with a wrapper.** Hello world is `func main() { println("hello, world") }`. Every example in the spec, the stdlib and `ant/examples` wraps its real code in a function whose only job is to be called. 404 `.ant` files in the `ant` repository declare `main`.
2. **Ant already runs top-level code.** A top-level `var` initializer is an arbitrary expression: it can call functions, print, open files and start goroutines, and it runs at load time in source order (section 10). The module body is a script already, restricted to the `var x = ...` shape. `main` is the one statement it is allowed to call implicitly.
3. **The REPL already does it.** `ant.Session.Eval` evaluates statements at the top level. A file and a REPL session read the same code differently today.
4. **The JS backend wants it.** An ES module is a list of statements. Compiling `main` means emitting a call to it; compiling a script means emitting the script.
5. **`main` is a special case to specify.** Section 10 needs a paragraph for its name, its signature, and its absence (exit status 2), and sections 9 and 12 end the program when it returns. (Outside the spec, the Go host API has `Start`, "initialize without calling main", next to `Run`.)

## Prior art

| Approach | Examples | Notes |
| --- | --- | --- |
| Statements in every module | JavaScript (ES modules), Python, Ruby, Lua, Perl, PHP | Importing a module runs its body once. Each grows an "am I the entry point?" test for files that are both a program and a library: Python `if __name__ == "__main__"`, Ruby `if __FILE__ == $0`, Deno and Bun `import.meta.main` |
| Statements in one entry file only | Swift (`main.swift`), C# 9 top-level statements (one file per project), F# (last file of the project) | The entry file is the program; every other file is declarations. No dual-use idiom is needed |
| `main` without boilerplate | Java 25 compact source files (JEP 512), Kotlin `fun main()` with no parameters | Shrinks the ceremony but keeps the function |
| `main` required | Go (with `init()`), Rust, Zig, Dart, Gleam, Odin | Package level is declarations only |
| Separate script dialect | Kotlin `.main.kts`, Elixir `.exs`, Scala CLI | Two file kinds with two rule sets |

Three lessons shape the proposal:

- **Dual-use files are the cost of statements everywhere.** `__name__` and `import.meta.main` exist because importing a file can run a program. A rule that a script is never imported removes the question instead of answering it.
- **Languages that added top-level code late restricted it to one file.** C# and Swift (and F#) chose "statements in the entry file only"; nobody needed more.
- **Strict first.** "A script cannot be imported" can be relaxed to "any module may have statements" later without breaking a program. The reverse breaks every library that grew a statement.

## The proposal

### Modules and scripts

A module is a sequence of `import` declarations followed by **items**: `var`, `func` and `type` declarations, optionally `export`ed, and statements. A statement is anything a block may hold that is not a declaration: an expression, an assignment, `if`, `for`, `while`, `select`, `go`, `defer`. Declarations and statements interleave freely.

A module with at least one top-level statement is a script. The test is syntactic, so the parser decides it, and both commands enforce the two rules that follow from it:

- **A script cannot be imported.** An `import` that resolves to a script is a load error: `ant run` exits with status 2 and `ant check` reports it. The error is reported at the import and names the imported module's first top-level statement, since a library without exports becomes a script the moment it gains a stray statement, and the importer is where that shows.
- **A script cannot export.** `export` in a script is a syntax error, reported once, at the first top-level statement: "a module with exports cannot have top-level statements".

Libraries are not side-effect free: a `var` initializer may still call anything, as it does today (Motivation, point 2). The rule keeps program logic out of imports; it does not make initialization pure, and closing that gap is not part of this proposal.

`ant run` accepts a library too. It initializes it and exits with status 0, which is a quick way to see that a module loads.

### Evaluation

Initialization is unchanged: imports depth first in source order, each module once, `func` and `type` hoisted. A script's items then run in source order, `var` initializers and statements alike. This is the existing rule for top-level `var`s, extended to statements:

- A function body may name any top-level `var`; reading or assigning one before its initializer has run panics. Assigning is new to the rule: `x = 1` above `var x = 0` is a compile error, and `ant run`, which does not check, needs a panic for it. The same write is already reachable today from a function called by an earlier initializer, and section 12 does not define it.
- A top-level statement or initializer that refers directly, outside a nested function body, to a `var` declared below it is a compile error.
- The value of a top-level expression statement is discarded.
- A `var` declared inside a top-level `if`, `for` or other block is local to that block, as anywhere else. Only top-level `var`s are fields of the module record.

```ant
//! Counts the lines of the files given as arguments.

func count(path: string): Result[int, Error] {
  var text = try File.read(path)
  Result.ok(len(String.lines(text)))
}

var total = 0
for path in Os.args() {
  var n = count(path)
  if n is Ok {
    total += n.value
    println(`${n.value} ${path}`)
  } else {
    println(`cannot read ${path}`)
  }
}
println(`${total} total`)
```

### Ending a script

The program ends when the script's last item has run, whatever other goroutines are doing, exactly as it ends today when `main` returns. The goroutine that initializes the modules and runs the script is the **main goroutine**; nothing else about goroutines changes.

- **`defer`** at the top level registers on the script's body. Deferred blocks run when the last item has run or when a panic unwinds the main goroutine, in reverse order, before the program exits. `Os.exit` still skips them.
- **`return`** at the top level is already a syntax error: it is outside any boundary (section 5), and nothing here changes that. A script that needs to leave early either calls `Os.exit(status)` or puts its logic in a function and calls it; `func main() { ... }` followed by `main()` is an ordinary program, and the name has no special meaning.
- **`try`** and **`?? return`** at the top level are syntax errors too, by the same rule: section 8 already forbids `try` in a top-level initializer outside a lambda, and a statement is no different. `var text = try File.read(path)` in a script does not parse; the example above moves the `try` into a function. Whether a top-level `try` on an `Err` should end the program instead is an open question.
- **Exit status** is `0` when the last item has run, `1` when the main goroutine panics, and `2` when the program could not be loaded: a syntax error, an import cycle, a missing module, or an import of a script.

### Narrowing

Section 7 never narrows a top-level `var`, because any call may reassign it. In a script that would make the most natural code fail to check:

```ant
var text = File.read("notes.txt")
if text is Ok {
  println(len(String.lines(text.value)))    // compile error today: a top-level var never narrows
}
```

A script cannot be imported, so its top-level `var`s are assigned only by its own code. The checker can then treat the script's top-level code as the body of a function and its `func` declarations as nested bodies, and the existing closure rule applies unchanged:

- In top-level code, a script's `var` narrows like a local, unless some nested body assigns it, a `func` declaration included.
- Inside a `func`, a script's `var` starts at its declared type, as an outer local does inside a nested body today, with no exception. Section 7 lets a local that is never assigned after its declaration keep its narrowing inside nested bodies; a `func` declaration does not get that exception, because it is hoisted and has no position: it can be called before the test that narrowed the variable has run. A lambda, a `go` block or a `defer` block written in top-level code is created where it stands and keeps the exception as it does in any function.

Libraries keep the current rule: they have no top-level code to narrow in, and their `var`s may be reassigned by any call into the module.

### Tooling

- **`ant fmt`** lays out the top level as a broken block at indentation zero already (fmt.md 4.3). The rule that a file of top-level statements is a syntax error goes away.
- **The library tables.** `ant/tests/stdlib/*.ant` are files of top-level statements, each line checked against its `// =>` comment, and fmt.md names them as the files it rejects. They parse as scripts once this lands, so `ant fmt .` formats them. Formatting keeps every line break and aligns runs of trailing comments (fmt.md 3, 4.8), so the tables stay one call per line; they are formatted once when this lands and kept formatted, like every other file.
- **`ant doc`** documents a script's module doc (`//!`), which says what the program does and how to run it. A script has no exports to list.
- **Host API** (implementation, not spec). `Load` no longer fails for a missing `main`. `Run` initializes every module, the script last, and returns as it does today. `Start` initializes too, so `Start` on a script runs the script; a host that wants to call functions loads a library. `Instance.Func` and `Value` are unchanged.
- **REPL.** `Session` already evaluates statements; a file and a session now accept the same items.
- **JS backend.** A script compiles to an ES module's top-level code, a library to a module of declarations.

### Migration

Ant is unreleased, so `main` is removed rather than deprecated. The rewrite is mechanical: delete `func main(): void {` and its closing brace and unindent the body. A `main` whose body uses `return` keeps the function and gains a `main()` call at the end of the file. A declared `main` that nothing calls is an ordinary unused function, so a program that is not migrated runs its initializers and exits 0 without doing its work; the migration script must cover every file in `ant` and every example in `spec`.

## Spec changes

- **Section 3 (Variables).** Extend "a top-level initializer may refer directly … only to the `var`s above it" to top-level statements.
- **Section 4 (Checking and running).** "A syntax error anywhere stops the program before `main`" becomes "before any module is initialized".
- **Section 5 (`return`, `break`, `continue`).** Unchanged; the rule that `return` outside any boundary is a syntax error already covers top-level statements.
- **Section 7 (Narrowing).** Add the script rule above to the paragraph on top-level `var`s, including why a `func` declaration does not get the never-assigned exception.
- **Section 8 (`try`).** "In a top-level initializer outside a lambda" becomes "in a top-level initializer or statement outside a lambda".
- **Section 9 (Goroutines).** "The program exits when the main goroutine returns" becomes "when the script's last item has run". Define the main goroutine as the one that initializes the modules.
- **Section 10 (Modules).** Replace "Nothing else appears at the top level" with the item list and the definition of a script, add the two script rules, replace the paragraph on `main` with the evaluation and ending rules, and update the exit statuses. "`ant run` stops before `main`" on an import cycle becomes "before initializing any module".
- **Section 12 (Dynamic semantics).** Names and scopes: a script's top level is the module scope, and a `var` declared in a top-level `if`, `for` or other block belongs to that block's scope. Deferred blocks and panics: a script's body is the main goroutine's root block. Goroutines: "the program ends when `main` returns" becomes "when the script ends". Runtime panics: the `var` row becomes "read or assigned before its initializer has run".
- **Section 13 (Example)** and every example in `stdlib.md`: drop the `main` wrapper.
- **fmt.md section 1.** Drop "so a file with top-level statements is a syntax error here too".
- **fmt.md section 5.** Drop "It formats modules only. A file of top-level statements, such as the library tables in the test corpus, is a syntax error".
- **doc.md.** "A program's main module" becomes "A script".

## Alternatives considered

- **Statements in every module, as in JavaScript.** Importing a script would run it, and Ant would need a `__name__`-style test for dual-use files. Script narrowing would be unsound, since another module could call a function that reassigns a `var`. It remains possible later: lifting the import ban breaks no program.
- **Keep `main` and allow statements too.** Two ways to write a program, against "one obvious way", and the order between top-level statements and `main` would need a rule.
- **Keep `main`, shorten it.** Already short (`func main() { ... }`); what costs is the wrapper and the indentation, not the signature.
- **A script command, `ant script file.ant`, or a `.ants` extension.** Two file kinds and two rule sets for what the parser can tell from the file.

## Non-goals

- **Top-level `return` or an exit value.** `Os.exit` and a called function cover early exit.
- **A shebang line.** Useful for scripts and independent of this proposal; it belongs in section 2.
- **Implicit `args`.** `Os.args()` stays the one way to read the command line.
- **Printing the last expression.** That is the REPL's job, not `ant run`'s.

## Open questions

Decided after review:

| Question | Decision |
| --- | --- |
| Does the program end when the script's last item has run, even with goroutines running? | Yes, today's rule: the main goroutine ending ends the program |
| Where is an import of a script reported? | At the import, naming the imported module's first top-level statement |

Still open:

- [ ] Should `export` in a script be a syntax error, as proposed, or meaningless? The error is what makes a stray statement in a library visible in the library itself, rather than only at its importers.
- [ ] Should a top-level `try` on an `Err` end the program, printing the error and exiting with status 1, as Swift's top-level `try` does? It is the script-friendly choice; strict-first keeps it a syntax error until scripts show the need, and lifting the error later breaks nothing.
- [ ] Should the checker reject a `func main` that nothing calls during the migration? The review leans yes: a silent exit 0 that skips the program's work is the worst failure. The checker has no warnings, so it would be a temporary compile error in the implementation, not in the spec, removed once `ant` and `spec` are migrated.
