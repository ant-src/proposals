# Proposal: HTML expressions

2026-10-27 · Anton

## Summary

Add HTML as an expression. `<p>Hello, {name}</p>` is an expression of the new built-in type `Html`,
an immutable value that holds markup. A string interpolated with `{...}` is escaped; an `Html` value
is inserted as it is. Components are ordinary functions that return `Html` and are called with
ordinary calls. `<>...</>` is a fragment. `<script>` and `<style>` hold literal text and take no
interpolation, nor do `on*`, `style`, and `srcdoc` attributes, and an interpolated URL attribute is
checked for its scheme. `Html.raw(s)` is the one way to turn a string into markup. White space that
holds a line break collapses as a browser would collapse it. The grammar keeps its one token of
lookahead: `<` begins an element by the same rule that makes `/` begin a regular expression.

## Motivation

Ant has `Http.serve`, and a handler that answers a browser builds HTML. Today it does so with
template strings:

```ant
func row(u: User): string {
  `<tr><td>${u.name}</td><td>${u.email}</td></tr>`
}
```

1. **Injection by default.** `${u.name}` renders the string as it is. A user named
   `<script>…</script>` runs code in every visitor's browser. The fix is to remember an escape
   function at every interpolation, and the program that forgets one still checks and runs.
2. **Double escaping.** Once `row` returns a string, a caller cannot tell markup from text. Escaping
   it again turns `<tr>` into `&lt;tr&gt;`; not escaping text built elsewhere is the first defect.
   The type `string` carries no answer, so every program invents a convention.
3. **No structure.** A missing `</td>` or a misspelled closing tag is a string like any other.
   Neither command notices, and the browser repairs it in its own way.
4. **No library fixes it alone.** A builder library, `Html.el("p", [], [Html.text(name)])`, can be
   written today and is safe, but nobody wants to write a page in it, so the template string wins.
   Every language that made HTML both safe and pleasant did it with syntax or a macro.

## Prior art

| Approach                      | Examples                                                                           | What Ant takes                                                                                        |
| ----------------------------- | ---------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Markup as an expression       | JSX/TSX (React, Preact, Solid); templ (Go); Rust `maud`, `rsx`                     | The syntax: elements, `{...}` interpolation, fragments `<>`, self-closing tags                        |
| XML literals                  | Scala 2, VB.NET, E4X in JavaScript                                                 | A warning. Scala 3 and JavaScript dropped them: they were tied to an XML tree type and XML namespaces |
| Contextual autoescaping       | Go `html/template`, Google Closure Templates                                       | The goal, not the method. Ant forbids the hard contexts instead of parsing CSS and JavaScript         |
| Escape by default, raw opt-in | Svelte `{@html}`, React `dangerouslySetInnerHTML`, Go `template.HTML`, Rails `raw` | `Html.raw`: one named, greppable function                                                             |
| Components as functions       | templ, React function components, Elm `Html` functions                             | Components are plain functions with named arguments; no props object, no protocol                     |
| Builder DSLs                  | kotlinx.html, Elm, Clojure Hiccup, Python `dominate`                               | Nothing. Safe, but verbose enough that people fall back to strings                                    |
| Tagged templates              | Lit ``html`...` ``                                                                 | Nothing. The markup is a string the compiler cannot see into                                          |
| Safe URL attributes           | Go `html/template` (`#ZgotmplZ`), Closure (`about:invalid#zClosurez`)              | Replacing a URL with an unsafe scheme instead of failing                                              |

Three lessons shape the proposal:

- **Escaping must be the default and the bypass must have a name.** Every framework that escapes by
  default is safe in practice; every one that made escaping opt-in, PHP and early Rails among them,
  produced a generation of XSS. The bypass is searchable in each: `dangerouslySetInnerHTML`,
  `{@html}`, `template.HTML`.
- **The value is markup, not a tree.** JSX needs a virtual DOM because it renders on the client;
  Scala's XML literals died partly because they built a tree nobody wanted. A server renders to
  bytes. A type that wraps markup is all the safety needs.
- **Contexts are where escaping breaks.** Escaping is correct for text and quoted attributes. It is
  not correct inside `<script>`, inside `style`, or in an `onclick` attribute, and Go needs a CSS
  and JavaScript lexer to handle them. Few programs need to interpolate there, so Ant forbids it.

## HTML expressions

### Where an element starts

`<` begins an HTML expression when the token before it, skipping line breaks, is not an operand, as
`/` begins a regular expression (section 2, Literals): after `=`, `(`, `[`, `,`, `{`, `:`, `return`,
`??`, or an operator, and at the start of a file or of an interpolation. After an operand, an
identifier, a literal, `)`, `]`, or `}`, `<` stays the comparison.

`<` also begins an HTML expression when it is the first token of a line where a line break ends a
statement: in a block, a lambda, or a script's top level. Today such a line is a syntax error, since
no statement starts with `<` and a line break before `<` does not continue an expression (section 2,
Line endings). Inside parentheses and brackets, where line breaks are ignored, the first rule alone
applies, so `(total` followed by `< limit)` on the next line is still a comparison.

In both cases the `<` must be followed directly, with no white space, by an ASCII letter, which
starts a tag name, or by `>`, which starts a fragment. A `<` in such a place followed by anything
else, as in `var x = < 3`, is a syntax error that says a comparison needs a left operand, not an
error about a malformed tag. The check reads one character ahead in the lexer and adds no lookahead
to the grammar.

```ant
var big = n < limit                          // comparison: the token before < is an identifier
var title = <h1>{post.title}</h1>            // an element: the token before < is =

func card(post: Post): Html {
  var date = Time.format(post.date, "yyyy-MM-dd")
  <article>{date}</article>                  // an element: the first token of a line in a block
}
```

No program that is valid today changes meaning: every `<` that would begin an element follows a
token after which `<` was a syntax error. Ant writes type arguments in square brackets, so there is
no `<T>` to collide with.

### Elements

An element is a start tag, content, and an end tag that names the same element, or a self-closing
tag:

```ant
<p class="lead">Hello, {name}.</p>
<img src="/logo.png" alt="Ant" />
```

- A name starts with an ASCII letter and continues with ASCII letters, digits, and `-`, so custom
  elements such as `my-widget` work. Case is kept as written, for SVG's `foreignObject` and
  `viewBox`.
- A **void element**, one of `area base br col embed hr img input link meta source track wbr`, is
  written self-closing and renders without the slash: `<br />` gives `<br>`. `<br></br>` is a syntax
  error.
- Any other element written self-closing renders with an end tag: `<div />` gives `<div></div>`,
  which is also what SVG wants for `<circle />`.
- An end tag that names another element, an element never closed, and an end tag with nothing to
  close are syntax errors, in `ant check` and `ant run` alike.
- `<!--`, `<!doctype`, and `<?` are syntax errors. A comment in markup is `{/* ... */}`, which
  renders nothing (see White space). The doctype is written where the page becomes a response (see
  Serving a page).

The element is an expression of type `Html`. Its value is the markup, built from left to right:
attributes in order, then content in order (section 12, Expressions).

### Text and interpolation

Between tags, text is markup written by the author and is kept as written, apart from the white
space rule below. `&amp;` stays `&amp;`, so the author writes entities as in an `.html` file. Three
characters are special: `<` begins a tag, `{` begins an interpolation, and `}` alone is a syntax
error. A literal brace is `&#123;` and `&#125;`, or `{"{"}`.

`{expr}` interpolates any expression; braces nest inside it as in `${...}` of a template string, so
lambdas and record literals can appear there. The value goes in by its kind:

| Value          | Renders as                                                      |
| -------------- | --------------------------------------------------------------- |
| `Html`         | its markup, unchanged                                           |
| `string`       | the string escaped: `&`, `<`, `>`, `"`, and `'` become entities |
| a number       | `String.from(n)`, which needs no escaping                       |
| `null`, `void` | nothing                                                         |
| an array       | each element in order by these rules, with nothing between them |

The checker accepts a union of these types, and arrays of them at any depth. Any other type is a
compile error, and a panic at run time: a record, a map, a function, a `bool`, and a named primitive
type. `String.from` of a record would put `Point{ x: 1.0 }` on a page, which is a bug and not a
rendering, and `UserId("u-1")` likewise; write `{string(id)}`. A `bool` is excluded for the same
reason: `true` on a page is almost always a mistake.

`void` renders nothing so that an `if` without `else` is the conditional:

```ant
<nav>
  <a href="/">Home</a>
  {if user != null { <a href="/account">{user.name}</a> }}
</nav>
```

An array renders its elements so that `Array.map` is the loop. The lambda's `{` follows `,`, so it
is a lambda (section 5, Blocks), and the `<` after it begins an element:

```ant
<ul>
  {Array.map(tasks, { <li class={if x.done { "done" } else { "open" }}>{x.title}</li> })}
</ul>
```

`for` is a statement of type `void` and is not used to render; there is no special loop syntax.

### Attributes

An attribute is one of:

- `name="text"`: literal text, written by the author and kept as written. `"` ends it; entities work
  as in content. There is no interpolation inside the quotes, and single quotes are not accepted.
- `name={expr}`: a value. A `string` is escaped as in content; a number renders with `String.from`;
  `true` writes the bare name, a boolean attribute; `false` and `null` leave the attribute out. Any
  other type is a compile error and a panic.
- `name` alone: a boolean attribute, as `name={true}`.

A name starts with an ASCII letter or `_` and continues with letters, digits, `-`, `_`, `:`, and
`.`, so `data-id`, `aria-label`, `hx-get`, and `xlink:href` work. An attribute written twice on one
element is a syntax error. Attributes render in written order, separated by one space, each value in
double quotes.

```ant
<input type="checkbox" name={field} checked={task.done} disabled />
<td class={if n < 0 { "negative" } else { "positive" }}>{n}</td>
```

Joining values is an ordinary expression: ``class={`btn btn-${kind}`}``. There is no class list
syntax and no attribute spread (see Open questions).

### Unsafe contexts

Escaping makes a string safe as text and as a quoted attribute value. It does not make it safe as
JavaScript, as CSS, or as a URL. The rules below take those cases away from interpolation, by name,
so the parser enforces them with no knowledge of types.

1. **`<script>` and `<style>`** hold literal text up to their end tag. Neither `{` nor `<` is
   special inside, so JavaScript and CSS are written as in an `.html` file, braces and all. Their
   attributes follow the ordinary rules, so `<script src={bundle} defer />` works.
2. **Attributes that run code** take only literal text: every name that starts with `on`, `style`,
   and `srcdoc`. `onclick={handler}` and `style={css}` are syntax errors whose message names the
   alternative: a `data-` attribute that a script reads, or a class.
3. **URL attributes**, `href`, `src`, `action`, `formaction`, `cite`, `poster`, and `xlink:href`,
   check an interpolated value at run time. A relative URL, or one whose scheme is `http`, `https`,
   `mailto`, or `tel`, is kept. Any other scheme, `javascript:` and `data:` among them, is replaced
   by `about:invalid#ant`: the link goes nowhere, and the page still renders. The scheme is read as
   a browser reads it, after removing leading and trailing white space and every tab and line break,
   and ignoring case. Literal text is never checked.
4. **`<title>` and `<textarea>`** hold text only, as HTML parses them. Their content may interpolate
   strings and numbers but no `Html`, and child elements are a syntax error.
5. **`<pre>`** keeps its white space (see White space).

Passing data to a script goes through an attribute, which is escaped like any other:

```ant
<div id="chart" data-points={Json.stringify(points)}></div>
<script>
  const points = JSON.parse(document.getElementById("chart").dataset.points)
</script>
```

### Raw HTML

`Html.raw(s)` returns `s` as markup, unescaped. It is the one way to make `Html` from a string, so a
review for injection is a search for `Html.raw`. Its use is markup a program trusts: the output of a
sanitizer, a Markdown renderer, a cached page, an SVG file shipped with the program.

```ant
<article>{Html.raw(Sanitize.html(post.body))}</article>
```

There is no constructor: `Html` is not a named primitive type over `string`, because the constructor
`Html(s)` would be the bypass under a harmless name.

### Fragments

`<>...</>` is an `Html` value with no element around it. It holds content like an element and takes
no attributes. It is how a function returns several siblings, and how a block of markup is passed as
an argument.

```ant
func head(title: string): Html {
  <>
    <meta charset="utf-8" />
    <title>{title}</title>
  </>
}
```

An empty fragment `<></>` is the empty `Html`.

### White space

Inside content, outside `<pre>`, `<textarea>`, `<script>`, and `<style>`:

- A run of white space on one line is kept as written.
- A run of white space that contains a line break is removed when it is the first or the last thing
  in the content, and otherwise becomes one space.
- An interpolation that holds only comments is removed first, and the runs on either side of it then
  count as one run.

This is the browser's own rule, collapsing white space to one space, applied when the expression is
built, with the indentation of the source dropped at the edges. So the source can be indented to
match the Ant code around it, and the output carries no indentation. Unlike JSX, a line break
between a word and a tag stays a space, so `Hello` on one line and `<b>{name}</b>` on the next reads
`Hello <b>ada</b>`, as it would in an `.html` file.

```ant
<ul>
  <li>a</li>
  <li>b</li>
</ul>
// <ul><li>a</li> <li>b</li></ul>
```

Inside `<pre>` and `<textarea>` the text is kept exactly, line breaks and indentation included, as
in a template string. Inside `<script>` and `<style>` it is kept exactly too.

### Components

A component is a function that returns `Html`. There is no component tag, no props type, and no
children protocol: parameters are the props, named arguments name them, defaults make them optional,
and a parameter of type `Html` takes children.

```ant
func card(title: string, children: Html, footer: Html | null = null): Html {
  <section class="card">
    <h2>{title}</h2>
    {children}
    {if footer != null { <footer>{footer}</footer> }}
  </section>
}

func taskList(tasks: Array[Task]): Html {
  card(title: "Tasks", children: <ul>
    {Array.map(tasks, { <li>{x.title}</li> })}
  </ul>)
}
```

Since `Html` is a value, every Ant construct composes with it: a component is passed as a function
value, stored in a map, returned from an `if`, built in a goroutine, and tested with `==`. An
exported component states its return type, `: Html`, as every exported function does.

## The `Html` type

`Html` is built in, like `Bytes`: a kind of its own, immutable, with the functions of a namespace.

```
Html.raw(s: string): Html              // s as markup, unescaped; the one bypass
```

| Type   | Kind | `is`   | `==` compares | `String.from`                                                      | In a union |
| ------ | ---- | ------ | ------------- | ------------------------------------------------------------------ | ---------- |
| `Html` | html | `Html` | the markup    | the markup at the top level; `Html.raw("<p>…</p>")` in a container | once       |

`String.from(h)` is the markup, so `println(page)` prints it and a template string embeds it:
`` `<!doctype html>${page}` `` is the response body. That turns the markup back into a `string`,
which is the right direction: a `string` never becomes `Html` again without `Html.raw`. `len`,
indexing, and `for` do not take `Html`; it is not text to take apart.

The same expression with the same values gives the same bytes every time, so a test compares a
component's output with `==` against `Html.raw` of the expected markup, or its `String.from` against
a string.

### Serving a page

```ant
func layout(title: string, children: Html): Html {
  <html lang="en">
    <head>
      <meta charset="utf-8" />
      <title>{title}</title>
      <link rel="stylesheet" href="/app.css" />
    </head>
    <body>{children}</body>
  </html>
}

func handle(req: Http.Request): Http.Reply {
  var page = layout(title: "Tasks", children: taskList(tasks))
  Http.Reply{
    status: 200,
    headers: Map{ "Content-Type": "text/html; charset=utf-8" },
    body: `<!doctype html>${page}`,
  }
}
```

## Example

A form with an error, a list, a conditional, an inline script, and a component:

```ant
type Field = { name: string, label: string, value: string, required: bool, error: string | null }

func field(f: Field): Html {
  <label>
    {f.label}
    <input name={f.name} value={f.value} required={f.required} />
    {if f.error != null { <span class="error">{f.error}</span> }}
  </label>
}

func signup(fields: Array[Field], next: string): Html {
  <>
    <form method="post" action={next}>
      {Array.map(fields, field)}
      <button type="submit">Sign up</button>
    </form>
    <script>
      document.querySelector("form").addEventListener("submit", (e) => {
        if (!confirm("Send?")) { e.preventDefault() }
      })
    </script>
  </>
}
```

With `next` taken from a query string, `action={next}` cannot become `javascript:alert(1)`, and a
field value of `"><script>` stays text. `{Array.map(fields, field)}` passes the component as a
function value. The fragment matters: without it the `<form>` and the `<script>` are two statements,
and `signup` returns only the `<script>`. Both are `Html`, so the checker cannot see the mistake
(see Open questions).

## Dynamic semantics

`ant run` keeps everything above that does not mention a type. An HTML expression evaluates its
attributes and content left to right and appends to the markup:

- literal text and attribute text as written, after the white space rule, which the parser applies;
- a string escaped, a number through `String.from`, `null` and `void` as nothing, an array element
  by element, an `Html` as it is;
- a boolean attribute by its value, and a URL attribute after its scheme check.

A value of another kind, in content or in an attribute, and an `Html` inside `<title>` or
`<textarea>`, is a kind panic like the others of section 12. The checker rules these out.

An `Html` value is a string with another kind, so the interpreter builds it with a string builder
and the JavaScript backend with string concatenation and one escape function; neither needs a tree.

## `ant fmt`

The code inside `{...}` is formatted as an expression, as in a template string (fmt.md 4.9).
Attributes take one space between them and none around `=`, and `/>` takes one space before it.
Lines of markup are reindented to the block they are in, which the white space rule makes safe,
since a run with a line break means the same at any indentation. Text, and everything inside
`<pre>`, `<textarea>`, `<script>`, and `<style>`, is kept byte for byte.

## Considered and rejected

- **Component tags**, `<Card title="x">…</Card>` calling `Card(title: "x", children: …)`. JSX tells
  a component from an element by its first letter. In Ant case means nothing (section 2), functions
  are named in lower case, and an element name cannot be resolved without the checker. An ordinary
  call costs a few characters and needs no rule.
- **`Html` as `type Html = string`.** The constructor would be the raw bypass, and every `string(h)`
  would expose markup as a string that looks like text.
- **Contextual autoescaping** in scripts, styles, and event handlers, as Go does. It needs a
  JavaScript and CSS lexer in the language and still has edge cases a decade on. Forbidding the
  contexts costs one `data-` attribute.
- **A node tree** with element, attribute, and text values, to transform or diff. Server rendering
  needs markup. A tree can be a library later; the syntax would not change.
- **Builder functions or a tagged template** as the only form. The first is too verbose to compete
  with template strings, and the second hides the markup from the parser.
- **Rendering every value with `String.from`**, as a template string does. Records and booleans on a
  page are bugs, and a named primitive would render as its constructor call.

## Spec changes

None until accepted. The changes would be:

| Where                                 | Change                                                                                                               |
| ------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| `spec/ant-lang.md` §2, Literals       | Where `<` begins an HTML expression, next to the rule for `/`                                                        |
| §2, Line endings                      | A line that starts with `<` in a block begins an HTML expression                                                     |
| §4, Kinds                             | A row for `Html`                                                                                                     |
| §5, new subsection `HTML expressions` | Elements, text, interpolation, attributes, unsafe contexts, fragments, white space, components                       |
| §11, new subsection `Html`            | `Html.raw`; the paragraph saying `String.from` is called implicitly only in template strings also names HTML content |
| §11, String                           | `String.from` of `Html`                                                                                              |
| §12, Expressions and Runtime panics   | The evaluation rule; rows for a bad kind in content or an attribute, and `Html` in `<title>` or `<textarea>`         |
| `spec/fmt.md`                         | A section 4.10 on HTML expressions                                                                                   |

The fixed lists, void elements, raw text elements, URL attributes, and URL schemes, follow the
WHATWG HTML standard, and the spec names them in one place so that a later version of HTML is one
edit.

## Non-goals

- **Client-side rendering**: a DOM, events, state, or hydration. The JavaScript backend gets the
  same strings; a client framework is a separate design.
- **Validating HTML**: content models, such as a `<div>` inside a `<p>`, and known element or
  attribute names. HTML changes, and the browser is the judge.
- **Template files** read at run time. Templates are Ant code and are checked like it.
- **XML and XHTML** output, and namespaces beyond keeping `xlink:href` as written.
- **Escaping for other formats**: CSS, JavaScript, SQL, shell. Each needs its own rules.
- **Complete protection from injection.** The rules make the common mistakes impossible: text and
  attribute values are escaped, code contexts take no interpolation, and URL attributes refuse
  dangerous schemes. They do not make every page safe. `Html.raw` passes anything through, a URL
  with a safe scheme can still point somewhere hostile, `srcset` and `<meta http-equiv="refresh">`
  are not checked, and literal text the author writes is trusted. The spec should describe the
  feature as escaping by default, never as preventing injection.

## Open questions

- [ ] `{expr}` or `${expr}`? `{}` is JSX and shorter. `${}` is what Ant's template strings use, and
      a literal `{` in text would need no escape. This proposal takes `{}`.
- [ ] Two sibling elements at the end of a block are two statements, and only the last one is the
      result (see Example). Should an HTML expression followed directly by another on the next line
      be a syntax error, asking for `<>...</>`?
- [ ] Should `null` render nothing? It makes `{user?.name}` quiet when it might be a bug. The
      alternative renders only `void` as nothing and makes `null` a compile error until `?? ""`.
- [ ] Should `class` also take an array, `class={["card", if selected { "selected" }]}`, its strings
      joined with one space and its `null` and `void` elements dropped? It is a convenience for the
      most common computed attribute, but it gives one attribute a rule of its own; today the same
      is ``class={`card ${if selected { "selected" } else { "" }}`}`` or `String.join`.
- [ ] Attribute spread, `<input {...attrs} />` with `attrs: Map[string, string | bool | null]`, for
      components that forward attributes. Names would be checked at run time, and the unsafe names
      of rule 2 refused.
- [ ] Should `style={...}` take numbers or a checked value, for the common progress bar,
      ``style={`width: ${pct}%`}``? Today it is a syntax error.
- [ ] `srcset`, a list of URLs: check each, or forbid interpolation?
- [ ] Should `Http.Reply.body` accept `Html` and set `Content-Type` to `text/html; charset=utf-8`
      and add the doctype when the markup starts with `<html`?
- [ ] Attribute names with `@` and a leading `:`, for Alpine.js and Vue: allow, or leave to literal
      `Html.raw`?
- [ ] `Html.raw` or a louder name, `Html.unsafe` or `Html.trusted`?
- [ ] Should text check that every `&` begins a valid character reference? HTML tolerates a bare
      `&`, so the check would catch typos only.
