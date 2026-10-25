# Proposal: scripts without `main`

2026-10-25 · Anton · Accepted

## Summary

Drop `func main()`. A module may hold statements at the top level, between and after its declarations, and `ant run` executes them in source order as the last step of initialization. A module with at least one top-level statement is a **script**. A script cannot be imported and cannot export, so a program has exactly one script, the file given to `ant run`, and every other module stays a library: declarations whose `var` initializers may still run code, as today. A script's top-level code is the body of an implicit function that the runtime calls: `return`, `try` and `?? return` work in it, an `Err` it returns ends the program with status 1 and the error on standard error, and its top-level `var`s narrow like locals. `func`, `type`, `import` and `export` are unchanged, and no new keyword or syntax is added.

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

A script's top-level code is the body of an implicit function, the **script body**, which only the runtime calls. It is a boundary in the sense of section 5, so everything a function body allows, the script allows, by the existing rules:

- **`return`** leaves the script early. Its deferred blocks run, which `Os.exit` skips.
- **`try`** on an `Err` returns that `Err` from the script, and **`?? return`** returns the value after it.
- **`defer`** registers on the script body. Deferred blocks run when the body ends, by its last item, by `return`, or by a panic unwinding the main goroutine, in reverse order, before the program exits.

```ant
var text = try File.read(path)
var user = find(id) ?? return Result.err(Error{ message: `no user ${id}` })
if len(Os.args()) == 0 {
  println("usage: count FILE...")
  return
}
```

The script body differs from a function body in one way: **its last expression is discarded.** Only a `return`, written or reached through `try` or `?? return`, gives it a result. Otherwise a script ending in `File.write(...)` would exit 1 when the write failed, and appending a `println` would hide that failure again.

The checker allows a script to return `void` or an `Err`: a bare `return`, `try`, `return Result.err(...)`, or `?? return` with such a value. `return 5` is a compile error. At run time the result decides the exit:

| The script body… | Exit status | Standard error |
| --- | --- | --- |
| runs its last item, or returns a value that is not an `Err` | `0` | nothing |
| returns an `Err` | `1` | `String.from` of its `error` and a newline |
| panics | `1` | the message and a stack trace, as today |
| could not be loaded: a syntax error, an import cycle, a missing module, or an import of a script | `2` | the error |

`try` and `Result.unwrap` both end a script on an `Err`, but they say different things. `try` is an expected failure, reported as the error alone after the deferred blocks run; `unwrap` is a bug, reported as a panic with its trace. That is the split section 1 makes between errors and panics.

A bare `x ?? return` exits with status 0, which is what it says. A script that means failure returns an `Err`.

Libraries have no script body: `return`, `try` and `?? return` in a library's top-level initializer stay syntax errors, as today. `func main() { ... }` followed by `main()` is an ordinary program, and the name has no special meaning.

### Narrowing

Section 7 never narrows a top-level `var`, because any call may reassign it. In a script that would make the most natural code fail to check:

```ant
var text = File.read("notes.txt")
if text is Ok {
  println(len(String.lines(text.value)))    // compile error today: a top-level var never narrows
}
```

A script cannot be imported, so its top-level `var`s are assigned only by its own code. The script body is a function body, so they are its locals for narrowing, the script's `func` declarations count as nested bodies, and the existing closure rule applies unchanged:

- In top-level code, a script's `var` narrows like a local, unless some nested body assigns it, a `func` declaration included.
- Inside a `func`, a script's `var` starts at its declared type, as an outer local does inside a nested body today, with no exception. Section 7 lets a local that is never assigned after its declaration keep its narrowing inside nested bodies; a `func` declaration does not get that exception, because it is hoisted and has no position: it can be called before the test that narrowed the variable has run. A lambda, a `go` block or a `defer` block written in top-level code is created where it stands and keeps the exception as it does in any function.

Libraries keep the current rule: they have no top-level code to narrow in, and their `var`s may be reassigned by any call into the module.

### Tooling

- **`ant fmt`** lays out the top level as a broken block at indentation zero already (fmt.md 4.3). The rule that a file of top-level statements is a syntax error goes away.
- **The library tables.** `ant/tests/stdlib/*.ant` are files of top-level statements, each line checked against its `// =>` comment, and fmt.md names them as the files it rejects. They parse as scripts once this lands, but they stay out of the formatted corpus: the harness reads them one input per line, and a line that starts with a regular expression literal is its own input there, while the whole file would read it as a division continuing the line before.
- **`ant doc`** documents a script's module doc (`//!`), which says what the program does and how to run it. A script has no exports to list.
- **Host API** (implementation, not spec). `Load` no longer fails for a missing `main`. `Run` initializes every module, the script last, and returns as it does today. `Start` initializes too, so `Start` on a script runs the script; a host that wants to call functions loads a library. `Instance.Func` and `Value` are unchanged.
- **REPL.** `Session` already evaluates statements; a file and a session now accept the same items.
- **JS backend.** A script compiles to an ES module's top-level code, a library to a module of declarations.

### Migration

Ant is unreleased, so `main` is removed rather than deprecated. The rewrite is mechanical: delete `func main(): void {` and its closing brace and unindent the body. A `main` whose body uses `return` keeps the function and gains a `main()` call at the end of the file. A declared `main` that nothing calls is an ordinary unused function, so a program that is not migrated runs its initializers and exits 0 without doing its work; the migration script must cover every file in `ant` and every example in `spec`.

## Spec changes

- **Section 3 (Variables).** Extend "a top-level initializer may refer directly … only to the `var`s above it" to top-level statements.
- **Section 4 (Checking and running).** "A syntax error anywhere stops the program before `main`" becomes "before any module is initialized".
- **Section 5 (`return`, `break`, `continue`).** The script body is a boundary. `return` outside any boundary, which now means in a library's top-level initializer outside a lambda, stays a syntax error.
- **Section 7 (Narrowing).** Add the script rule above to the paragraph on top-level `var`s, including why a `func` declaration does not get the never-assigned exception.
- **Section 8 (`try`).** `try` in a script body returns from it; "in a top-level initializer outside a lambda" becomes "in a library's top-level initializer outside a lambda".
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

- **An exit status from `return`.** `return 3` is a compile error; `Os.exit` is the one way to choose a status other than `0` and `1`.
- **A shebang line.** Useful for scripts and independent of this proposal; it belongs in section 2.
- **Implicit `args`.** `Os.args()` stays the one way to read the command line.
- **Printing the last expression.** That is the REPL's job, not `ant run`'s.

## Open questions

Decided after review:

| Question | Decision |
| --- | --- |
| Does the program end when the script's last item has run, even with goroutines running? | Yes, today's rule: the main goroutine ending ends the program |
| Where is an import of a script reported? | At the import, naming the imported module's first top-level statement |
| `export` in a script? | A syntax error, as proposed |
| Reject an uncalled `func main` during migration? | No. Ant is new; the migration script rewrites every program, and the checker gets no rule for the name |

| `try` at the top level? | The script body is an implicit function: `return`, `try` and `?? return` work, an `Err` result exits 1 with the error, the last expression is discarded, and a script returns `void` or an `Err`. Precedent: Rust's `main() -> Result`, Zig's `main() !void`, C#'s synthesized `Main` |
| How does a returned `Err` print? | `String.from` of the error, no special case for a `message` field |
