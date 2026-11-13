# Proposal: mutable `Bytes`

2026-11-13 · draft 2

## Decision

Make `Bytes` a packed mutable sequence with the same reference semantics as `Array`.
Assignment and argument passing share the sequence, including its length. `push` and `extend`
mutate it; `slice` produces an independent copy. Capacity stays an implementation detail.

Use Go as a reference for packed storage and bulk byte operations. This proposal deliberately
departs from Go's slice-value semantics. It adds no `Buffer`, copy on write, or freezing mechanism.

```ant
let a = Bytes.fromArray([1, 2, 3])
let b = a

a.push(byte(4))
assert(len(b) == 4)                         // both names refer to one collection

let part = a.slice(0, 2)
part[0] = byte(9)
assert(a[0] == byte(1))                     // slicing copies
```

The current spec routes byte-by-byte construction through `Array[byte]`, which stores general
values in slots. Mutable `Bytes` supplies packed working storage for encoders, packets, images,
and other binary data while preserving Ant's existing collection model. [Ant specification][spec]

## Values and identity

- A `Bytes` value refers to one growable collection. Assignment, arguments, returns, container
  storage, and channels share that collection. Growth and truncation are visible through every
  alias, subject to the normal synchronization rules.
- `let` fixes a binding and permits mutations of its collection, as it does for arrays. Rebinding
  a `var` points that variable at another collection; it does not redirect existing aliases.
- `len(b)` counts bytes. `b[i]` reads a `byte`; `b[i] = v` writes a `byte`. Both panic unless
  `0 <= i < len(b)`. Numeric conversions remain explicit and checked.
- There is no nil byte sequence. An empty collection is a `Bytes`; absence is `Bytes | null`.
- Lengths are at most `2^31 - 1`. Negative sizes, invalid bounds, and growth past that limit panic.
  Validate bounds and resulting sizes before modifying the collection; check arithmetic before
  overflow or conversion to host indices.
- Every operation that constructs a collection returns an independent mutable object, even for an
  empty result. An empty mutable collection must never be a shared singleton.

`==` continues to compare length and byte contents. Add `Bytes.same` to compare collection
identity, matching `Array.same`. Distinct empty collections are equal but are not the same object.

## API

Add these functions and redefine `slice`:

```ant
Bytes.new(length: int = 0): Bytes
Bytes.push(b: Bytes, value: byte): void                         // mutates
Bytes.extend(b: Bytes, src: Bytes): void                        // mutates
Bytes.pop(b: Bytes): byte | null                               // mutates
Bytes.truncate(b: Bytes, length: int): void                     // mutates
Bytes.slice(b: Bytes, from: int, to: int): Bytes                // copies
Bytes.clone(b: Bytes): Bytes                                   // copies
Bytes.same(a: Bytes, b: Bytes): bool                            // identity
Bytes.fill(b: Bytes, value: byte,
           from: int = 0, to: int | null = null): void          // mutates
Bytes.copy(dst: Bytes, src: Bytes, at: int = 0,
           from: int = 0, to: int | null = null): int           // mutates dst
```

Keep `length`, `concat`, `indexOf`, `startsWith`, `endsWith`, `fromArray`, and `toArray` with their
existing signatures. Searching reads the current contents. `fromArray` retains its checked
conversion from arrays of integers. `fromArray`, `toArray`, and `concat` copy their contents;
`concat` produces an independent collection even with one input. `Array[byte]` stays an ordinary
array. There is no implicit conversion between it and `Bytes`.

### Allocate, grow, and shrink

`new(n)` creates `n` zero bytes. `new()` creates an empty collection.

`push` appends one byte. `extend` appends the source's initial contents once, in order. In
particular, `b.extend(b)` doubles the original sequence; it does not keep visiting bytes it just
appended. An empty source does nothing. These are sequential aliasing guarantees, not atomic
snapshots under concurrent mutation.

`pop` removes and returns the last byte, or returns `null` when empty. `truncate(n)` keeps the first
`n` bytes and requires `0 <= n <= len(b)`. It cannot grow the collection. `truncate(0)` empties it;
the implementation may retain storage for reuse. Later growth initializes each newly visible byte
from the appended input, so removed bytes never become visible through reslicing.

Growth may allocate internally, but the collection's identity stays the same. Implementations
should provide amortized O(1) work per appended byte across repeated appends.

### Slice and clone

`slice(b, from, to)` requires `0 <= from <= to <= len(b)` and copies that half-open range into a new
collection. `clone(b)` copies the whole collection. Neither result shares writable storage with
its input. There is no reslicing beyond the length or capacity-limiting argument.

A slice of a small region does not keep the original allocation alive. Copying costs work
proportional to the selected size and may matter in bulk workloads even when the language is
interpreted. Parsers that repeatedly consume prefixes should keep an offset into the original
bytes instead of repeatedly copying the remaining suffix.

### Fill and copy

`fill(b, value, from, to)` overwrites the selected range. A null `to` means the captured length.
Require `0 <= from <= to <= len(b)`. It changes no length. Use `b.fill(byte(0))` to zero all bytes;
use `b.truncate(0)` to empty the collection.

`copy(dst, src, at, from, to)` overwrites an existing destination range without growing it:

- `at` is the destination offset; require `0 <= at <= len(dst)`.
- `from` and `to` select the source range; null `to` means the captured source length.
  Require `0 <= from <= to <= len(src)`.
- Copy `min(to - from, len(dst) - at)` bytes and return that count. Other destination bytes remain
  unchanged. Validate all bounds before writing, including when the copied count will be zero.
- Self-copy and overlap work as if the selected source bytes were copied before being overwritten.
  The implementation need not allocate a temporary buffer. This does not promise atomicity with
  respect to other goroutines.

```ant
let b = Bytes.fromArray([1, 2, 3, 4])
let n = Bytes.copy(b, b, at: 1, from: 0, to: 3)
assert(n == 3)
assert(b == Bytes.fromArray([1, 1, 2, 3]))
```

Explicit bounds are necessary because `dst.slice(...)` would create a copy. The copied-count and
overlap behavior follow Go's `copy`; the range arguments replace shared slices. [Go built-ins][go-builtins]

## Language behavior

**Iteration.** `for x in b` evaluates its receiver once, then follows that collection as an array
loop does. Each step reads the current byte at the next index and stops when that index is outside
the current length. Appended bytes may be visited; truncation can end the loop. Rebinding `b`
during the loop does not switch the collection being visited.

**Map keys and set members.** Keep Ant's mutable-key rule. A stored key's length and contents must
not change through any alias while it is in the map or set. This includes `push`, `extend`, `pop`,
and `truncate`, as well as indexed writes. Cloning isolates the caller's aliases but does not make
the clone immutable. Rebinding a variable alone does not change the stored key.

**Kinds and reflection.** Keep the existing bytes kind, `is Bytes`, opaque reflection shape,
`Bytes(89 50 4e 47)` rendering, and rejection by `Json.stringify`. Boxing and unboxing with
reflection preserve the collection reference; neither operation copies or freezes it. Extend
`Reflect.same` to use byte-collection identity, as `Bytes.same` does. Content equality stays `==`.

## Strings and library boundaries

Strings remain immutable, valid Unicode text:

- `String.toBytes(s)` returns an independent writable UTF-8 copy.
- `String.fromBytes(b)` copies the current bytes into one immutable snapshot, validates that exact
  snapshot, and returns it or `null`. It must snapshot **before** validation.
- `String.fromBytesLossy(b)` decodes an independent snapshot.

Concurrent writes may influence which bytes are copied; the copy is not an atomic snapshot of
all contents. Validation of the completed copy preserves the string invariant. Later byte writes
cannot change an accepted string.

HTTP request and response text and bytes, and command output's text and byte fields, represent
independent snapshots of the received data. Mutating the byte collection does not update the text.
Remove the current promise that valid UTF-8 lets these fields share one allocation.
[Ant I/O specification][io-spec]

Existing binary APIs continue to use `Bytes`. Returned chunks, decoded data, and digests must be
independent of storage that later runtime operations reuse. Copy a scratch or pooled buffer before
exposing it, or transfer its ownership and stop reusing that allocation.

Read-only standard-library consumers, such as hashing, encoding, and file writes, do not mutate
input. Synchronous calls finish accessing it before returning; a buffered writer copies data it
retains. An I/O API retaining input beyond the call snapshots it when accepting it unless an
explicit contract transfers ownership. These rules do not change the sharing behavior of ordinary
assignment or channels. [Go Writer contract][go-io]

## Concurrency and implementation

Extend Ant's section 9 rules to byte elements and collection length. Unsynchronized conflicting
accesses are data races. A race must not corrupt storage metadata, invalidate a captured
allocation, or turn a checked access into an out-of-bounds host access. A scalar read observes a
byte stored at that location; bulk operations and conversions need not be atomic.

On the Go backend, use a shared byte-collection object with packed `[]byte` storage. Follow the
array runtime's structure: atomically publish immutable headers and serialize operations that
change length. Each operation uses one coherent captured header per object for both bounds checks
and storage access. The header is immutable; its backing bytes are mutable. Do not read a mutable
slice header concurrently with an unsynchronized update. [Array runtime][array-runtime]

An operation racing with growth may finish against an old allocation, and its write may be lost.
The old allocation remains alive for that operation. Runtime locks protect structural integrity;
programs still synchronize shared mutation. Do not promise atomic whole-collection snapshots or
turn an old allocation into immutable text merely by replacing its header: an in-flight writer
may still hold it.

The implementation audit must cover:

- **Content equality and hashing.** A pointer representation must not introduce pointer equality
  for `==`. Hash the current contents without a persistent cached hash. If a raw slice appears
  inside a runtime interface, audit all Go comparisons and map keys, including equality's
  `pair{a, b}` cycle map, which currently checks the left operand's kind first. [Equality runtime][equality-runtime]
- **String sharing.** Remove unsafe conversions that share writable bytes with strings. Perform
  one copy and validate that exact result.
- **Read scratch storage.** `value.Bytes(h.scratch[:n])` currently copies because `Bytes` is a Go
  string. A replacement wrapper must explicitly copy that reused storage. [File runtime][file-runtime]
- **Empty objects and host APIs.** Allocate independent collection objects for new empty results.
  Host constructors should copy by default; any ownership-adopting path must be explicit and used
  only when the caller gives up the storage.

## Scope and adoption

This proposal changes `Bytes` mutability, identity, slicing, and collection operations. It keeps
one byte type throughout the standard library. General freezing, numeric binary codecs, shared
views, and capacity/reservation APIs are separate decisions. Numeric helpers must not promise
NaN-payload preservation without a backend-independent contract.

Reusable-buffer I/O is a companion proposal. Since `slice` copies, that design should let read and
write operations select a range directly when avoiding a temporary matters. It must separately
specify partial reads, EOF, cancellation, and destination lifetime. This proposal adds no
`File.readInto` declaration or new file-error behavior.

Update the spec's bytes definition, mutability and identity rules, index assignment, iteration,
mutable-key guidance, reflection, string conversions, I/O ownership text, prelude declarations,
and panic conditions. Extend the library's mutating-function convention to permit a copied-byte
count from `Bytes.copy`. Update runtime dispatch, host mappings, debugger inspection, and tests.

Validate shared growth, independent slices and empty results, self-extension, overlapping copy,
iteration during mutation, identity through reflection, string stability, and retained read chunks.
Measure construction, patching, copying, and native bulk operations against representative current
programs. Neither design has been prototyped here; no allocation count or speedup is claimed.

[spec]: https://github.com/ant-src/spec/blob/bfd3099ec4092de0992c826a8e6eeac80d6fce50/ant-lang.md
[io-spec]: https://github.com/ant-src/spec/blob/bfd3099ec4092de0992c826a8e6eeac80d6fce50/stdlib.md
[go-builtins]: https://pkg.go.dev/builtin#copy
[go-io]: https://pkg.go.dev/io#Writer
[array-runtime]: https://github.com/ant-src/ant/blob/8637fe37d46b5c7b47f5c86d553977c362796fa7/value/array.go
[equality-runtime]: https://github.com/ant-src/ant/blob/8637fe37d46b5c7b47f5c86d553977c362796fa7/value/equal.go
[file-runtime]: https://github.com/ant-src/ant/blob/8637fe37d46b5c7b47f5c86d553977c362796fa7/internal/lib/file.go
