# Proposal: mutable `Bytes`

2026-11-13 · draft 2, revised after review

## Summary

Make `Bytes` a mutable, growable sequence of bytes that is shared by reference, as an array is, and
add functions that read and write fixed-size integers and floats in it. There is still one byte
type: what I/O gives and takes is what a program computes in. `b[i] = x` writes in place, `push` and
`extend` grow, and `slice` copies.

```ant
// A frame: a big-endian length, then the payload masked with a 4-byte key.
func frame(payload: Bytes, key: Bytes): Bytes {
  let b = Bytes.new()
  b.pushUint32(uint32(len(payload)), Bytes.BigEndian{})
  b.extend(payload)
  var i = 4
  while i < len(b) {
    b[i] = Bits.xor(b[i], key[(i - 4) % 4])
    i += 1
  }
  b
}

// And back: the length, then that many bytes.
func payloadOf(frame: Bytes): Bytes {
  let n = int(frame.uint32(0, Bytes.BigEndian{}))
  frame.slice(4, 4 + n)
}
```

This is the alternative that `buffer.md` describes and sets aside. The two proposals share their
motivation and their integer functions, and differ in the type: `buffer.md` adds `Buffer` beside an
immutable `Bytes`, and this one makes `Bytes` the mutable type. The section Against `Buffer` says
why.

## Motivation

`Bytes` is immutable, and the spec sends a program that computes bytes to `Array[byte]`: "A program
that computes bytes one at a time builds an `Array[byte]` and converts it." An array holds a value
per element. `buffer.md` measured that path on ant master of 2026-11-12, on an Apple M4 Pro:

| Program                                                            | Time   | Peak memory |
| ------------------------------------------------------------------ | ------ | ----------- |
| A loop of 4 million iterations that only adds, for scale           | 0.40 s | 22 MB       |
| Encode 4 MB a byte at a time: `push`, then `Bytes.fromArray`       | 0.92 s | 259 MB      |
| Mask 4 MB with a key into `Array.filled`, then `Bytes.fromArray`   | 0.96 s | 356 MB      |
| Read a million big-endian `uint32` with `b[i]` and arithmetic      | 0.52 s | 28 MB       |
| Change one byte of 1 MB, 200 times: `toArray`, assign, `fromArray` | 3.36 s | 161 MB      |

Building holds 65 to 90 bytes per byte at its peak; a change in place copies the whole twice; and
every binary integer is spelled out by hand. A fix needs packed storage that can be written, and
readers and writers for integers. Where the packed storage lives is the question this proposal
answers differently.

## Prior art

| Language | Bytes type              | Shared or copied           | Immutable bytes      |
| -------- | ----------------------- | -------------------------- | -------------------- |
| Go       | `[]byte`                | header copied, data shared | `string`             |
| JS       | `Uint8Array`            | shared by reference        | none                 |
| Java     | `byte[]`                | shared by reference        | none in the platform |
| Kotlin   | `ByteArray`             | shared by reference        | none in the platform |
| Python   | `bytearray` and `bytes` | shared by reference        | `bytes`              |
| Rust     | `Vec<u8>` and `Bytes`   | owned, moved               | `Bytes` (lib)        |

Most languages have one mutable byte type and nothing else in the platform. Go is the closest
reference for packed storage and bulk operations, and this proposal follows its `copy` and
`encoding/binary`, but not its slices: a Go slice is a value whose length is its own, so `append` on
one alias may or may not be seen by another. Ant's arrays are shared with their length, and `Bytes`
follows the arrays.

## The design

### The type

A `Bytes` value refers to one growable collection, as an array does. Assignment, arguments, return
values, storage in a container and a channel send all share it, and growth and truncation are seen
through every alias. `let` fixes the binding, not the contents.

- `len(b)` is the number of bytes, and `for x in b` yields each as a `byte`. The loop evaluates `b`
  once and goes by index while the index is below the current length, as a loop over an array does:
  bytes appended during it are visited, a truncation ends it, and rebinding `b` does not switch the
  collection it walks.
- `b[i]` reads a `byte` and `b[i] = x` writes one; both panic when `i` is outside
  `0 ..= len(b) - 1`. `x` is a `byte`, so an `int` is converted as anywhere: `b[i] = byte(n)`.
- `==` compares length and contents, as now. `Bytes.same` compares identity, as `Array.same` does,
  and `Reflect.same` reports identity for `Bytes` too. Two empty collections are equal and not the
  same.
- A `Bytes` stays a map key under the rule for arrays: changing it afterwards, by `b[i] = x` or by
  any function that changes the length, is a program error.
- There is no null `Bytes` and no shared empty one: every function that makes a collection makes a
  new one. Absence is `Bytes | null`.
- `is Bytes`, rendering as `Bytes(89 50 4e 47)`, the panic in `Json.stringify` and the opaque shape
  in reflection stay as they are. Boxing into a `Reflect.Value` and out shares the collection.

There is no literal and no capacity. A collection grows by amortized doubling, and a program that
knows the size asks for it with `Bytes.new(n)` and assigns by index. A collection holds at most 2^31
− 1 bytes, and a function that would grow it past that panics before changing anything.

### Functions

New, beside `length`, `concat`, `indexOf`, `startsWith`, `endsWith`, `fromArray` and `toArray`,
which keep their signatures:

```
Bytes.new(size: int = 0): Bytes                                 // size zero bytes; panics if size < 0 or above 2^31 − 1
Bytes.clone(b: Bytes): Bytes                                    // a copy
Bytes.same(a: Bytes, b: Bytes): bool                            // identity
Bytes.push(b: Bytes, x: byte): void                             // mutates, as do the functions below
Bytes.extend(b: Bytes, src: Bytes): void                        // appends src as it was at the call
Bytes.pop(b: Bytes): byte | null                                // removes and returns the last byte; null when empty
Bytes.truncate(b: Bytes, n: int): void                          // keeps the first n bytes; panics if n is outside 0 ..= length
Bytes.fill(b: Bytes, x: byte, from: int = 0, to: int | null = null): void   // to: the length when null; panics as Array.slice does
Bytes.copy(dst: Bytes, at: int, src: Bytes, from: int = 0, to: int | null = null): void   // panics unless the range fits in dst
```

`Bytes.slice` keeps its signature and now copies, as `Array.slice` does. `concat`, `fromArray` and
`toArray` copy, and `concat` of one part is a new collection.

`extend` appends `src` as it was at the call, so `b.extend(b)` doubles `b`. `truncate(0)` empties a
collection for reuse, and the bytes it drops are never visible again: growth writes each new byte
from what is appended. `fill(b, byte(0))` zeroes without changing the length.

`copy` overwrites `to − from` bytes of `dst` from `at`, and never grows it; `extend` grows. It
panics, before writing, when `from` and `to` are not a range of `src` as for `Array.slice`, or when
`at + (to − from)` is above `len(dst)`. Overlap works as if the source range were read before any
byte is written, so `b.copy(1, b, 0, 3)` shifts three bytes right by one. The range arguments take
the place of Go's slicing, since `src.slice(...)` would copy first.

```ant
let b = Bytes.fromArray([1, 2, 3, 4])
b.copy(1, b, 0, 3)
assert(b == Bytes.fromArray([1, 1, 2, 3]))
```

### Integers and floats

```
type Order = BigEndian | LittleEndian

Bytes.uint8(b: Bytes, at: int): uint8
Bytes.uint16(b: Bytes, at: int, order: Bytes.Order): uint16
Bytes.uint32(b: Bytes, at: int, order: Bytes.Order): uint32
Bytes.uint64(b: Bytes, at: int, order: Bytes.Order): uint64
Bytes.int8 … Bytes.int64                                         // two's complement
Bytes.float32(b: Bytes, at: int, order: Bytes.Order): float32    // IEEE 754
Bytes.float64(b: Bytes, at: int, order: Bytes.Order): float64

Bytes.pushUint16(b: Bytes, v: uint16, order: Bytes.Order): void          // appends 2 bytes
Bytes.setUint16(b: Bytes, at: int, v: uint16, order: Bytes.Order): void  // overwrites 2 bytes
…                                                                // push and set for each of the ten kinds
```

`Bytes.BigEndian` and `Bytes.LittleEndian` are empty records, and `Bytes.Order` is their union, as
`Time.RoundingMode` is a union of `Time.HalfExpand` and the other modes. The order is required, as
in Go's `binary.BigEndian.Uint32`: every call says which order it reads, and no reader of the code
has to remember a default. A one-byte function takes none. A format names its order once and passes
it along:

```ant
let order = Bytes.LittleEndian{}
let width = b.uint32(16, order)
let height = b.uint32(20, order)
```

When the language gains enums, `Bytes.Order` becomes one, and calls change only in spelling.

These are `buffer.md`'s functions on the one type. A reader takes the bytes from `at` and panics
when `at` or `at + size` is outside `0 ..= length`; `set` panics the same way and never grows;
`push` appends. The value's type carries the width, so `b.pushUint32(uint32(n))` panics in the
conversion when `n` does not fit. Thirty names, and the generic `pushNumber` that would replace
twenty of them is rejected for the reason `buffer.md` gives: the width would follow whatever type
the argument happens to have.

A float is written from its IEEE 754 bits and read back into them. Whether a NaN keeps its payload
is not promised, since a backend that holds a float in a JavaScript number may canonicalize it.

### Strings and the standard library

Strings stay immutable, valid Unicode, so they no longer share memory with `Bytes`:

- `String.toBytes(s)` gives a new collection with a copy of the UTF-8.
- `String.fromBytes(b)` copies the bytes, validates the copy, and returns it or `null`. It validates
  what it copied, not `b`, so a write racing with the call cannot put invalid UTF-8 into a string.
  `String.fromBytesLossy` copies the same way.
- An HTTP request's and response's `body` and `bytes`, and a command's text and byte output, are
  independent copies: writing into `bytes` does not change `body`. The spec's promise that they
  share one allocation is removed, so a text body is held twice.

Every library function that gives `Bytes` gives a new collection that nothing else holds: a read
chunk, a decoded value, a digest. A runtime that reads into a reused scratch buffer copies out of it
or stops reusing it. A function that takes `Bytes` and only reads it, hashing, encoding or writing,
leaves it untouched and is done with it when it returns; one that keeps it past the call, a buffered
writer, copies what it keeps. This is Go's `io.Writer` contract, and it does not change the sharing
of ordinary assignment and channel sends.

### Concurrency

A `Bytes` is shared between goroutines by reference, as an array is, and section 9 applies:
unsynchronized use from two goroutines is a data race and a program error, and memory stays safe. A
racy read of a byte sees some byte that was stored there. A racy multi-byte read, write, `copy` or
conversion may see or leave a mix of two states. A racy `push`, `extend`, `pop` or `truncate` leaves
the length before or after it; an index that raced with a `truncate` reads the byte or panics with
the ordinary index error; and a write that raced with growth may land in the old storage and be
lost.

What changes for programs is that `Bytes` is no longer the type to hand to another goroutine without
a thought. A program that sends bytes and keeps writing sends `b.clone()`, as it would an array.

### What becomes of `Array[byte]`

Nothing changes for it. `Bytes.fromArray` remains the way a constant is written, and `Bytes.toArray`
the way to `map`, `filter` or `sort` bytes. The spec's advice changes, and there is no conversion
between the two other than these.

## Compatibility

No existing program writes into `Bytes`, so every program means what it meant. What changes is cost
and the embedding API:

- **`slice` copies.** A parser that takes `b.slice(i, len(b))` again and again was linear and
  becomes quadratic. It keeps an offset into the original instead, which the integer readers take
  anyway.
- **Text conversions copy.** `String.toBytes`, `String.fromBytes` and HTTP and command output each
  copy once, at memory speed.
- **The host API.** `Bytes` is passed to Go as a string type today and becomes a type of its own.
  Host constructors copy, and a constructor that adopts a caller's `[]byte` is explicit about it.

## Spec changes

- **Section 4, Types.** `Bytes` moves from the immutable kinds to the mutable ones; the sentence
  "Strings, `Bytes`, time values, and …" loses `Bytes`.
- **Section 5, Expressions.** Index assignment on `Bytes`; `for` over `Bytes` follows the current
  length.
- **Section 9, Data races.** The paragraph above.
- **Section 11.** The `Bytes` section: the type is mutable and shared, the new functions, the
  integer functions, the sentence on building an `Array[byte]` replaced, the last paragraph on not
  copying removed. `Bytes.same` beside `Array.same`; `Reflect.same` covers `Bytes`. The rule that a
  `mutates` function returns `void`, the removed element or whether it changed anything already
  covers `pop`. `String.toBytes`, `fromBytes` and `fromBytesLossy` copy.
- **Section 12, Dynamic semantics.** The table of Values, index write, `for`, and the table of
  runtime panics.
- **`stdlib.md`.** The ownership paragraph above; HTTP and command `body` and `bytes` no longer
  share.

## Implementation

`value.Bytes` follows `value.Array`: an atomic pointer to an immutable header holding a `[]byte`,
and a mutex taken by the functions that change the length. An operation loads one header and uses it
for both its bounds check and its access, so a race never turns a checked index into an
out-of-bounds one; growth publishes a new header, and the old storage lives as long as an operation
holds it. The integer functions are `encoding/binary` calls.

What needs care:

- **Equality and hashing.** `==` compares contents, never the pointer, and a hash is computed from
  the current contents each time. Equality's cycle map in `value/equal.go` keys on pairs of values,
  and a pointer there must not stand in for equality.
- **Strings.** Every unsafe conversion that shares memory between a string and `Bytes` goes; each
  becomes one copy, validated after copying.
- **Scratch storage.** `internal/lib/file.go` makes `Bytes` from a reused scratch slice, which is a
  copy today only because `Bytes` is a Go string. It copies explicitly.
- **Tools.** The debugger's inspector, the host API's table of Go types, and the docs. Highlighting
  does not change.

Expected, not yet measured: the building programs at about 1 to 2 bytes per byte, the one-byte
change in constant time, and a `uint32` read several times faster as one call. A prototype that
reruns the table, and measures the new copies in `slice` and the text conversions, comes before
acceptance.

For the JavaScript backend `Bytes` is a `Uint8Array` with a length and room to grow, and the integer
functions are `DataView` calls.

## Non-goals

- **Immutable bytes.** None until `frozen.md`; see Fit with other proposals.
- **Views.** No slice that shares storage with its source. Offsets and the range arguments of `copy`
  do that work.
- **Capacity.** No `reserve` or `capacity`; `Bytes.new(n)` is the one size hint.
- **I/O into existing bytes.** No `File.readInto(b)`. That is its own proposal, which has to settle
  partial reads, end of file, cancellation and how long the runtime may hold `b`.
- **A cursor, a format language, variable-length integers.** As in `buffer.md`.

## Against `Buffer`

`buffer.md` keeps `Bytes` immutable and adds `Buffer`. Its case is real, and this proposal pays each
cost it names:

- **Sharing with text.** `Buffer` keeps `slice` and the string conversions free. Here they copy, and
  a text body is held twice. Copy on write would avoid that, at the price of a runtime invariant
  that memory a string shares is never written. This proposal takes the copies: they are linear, at
  memory speed, and visible in the spec, where copy on write is hidden machinery.
- **Values nobody else can change.** A digest, a key, or bytes sent to another goroutine are safe
  today without a copy. Here they are in the position of arrays, which Ant already asks programs to
  handle, under the same section 9 and the same map-key rule.
- **Reversibility.** `buffer.md` says adding `Buffer` can be revisited and mutable `Bytes` cannot.
  But `Buffer` cannot be taken away once programs use it either: if `Bytes` later became mutable,
  the language would keep two mutable byte types for good.

What one type buys: nothing to learn twice and no conversion between two; no `toBytes` copy before
every write, hash or send; one signature per library function without `Bytes | Buffer`; and about
ten fewer names, since the readers exist once. Go, JavaScript, Java and Kotlin all run their binary
I/O on one mutable type.

## Other alternatives

- **Store `Array[byte]` compactly.** Rejected as in `buffer.md`: the interpreter never knows `T`, so
  every `Array` function would gain a second path.
- **Go's slice semantics.** A `Bytes` as a value whose length is its own, with `slice` sharing
  storage. It keeps slicing free, but `push` on one alias may or may not show through another, which
  is the most common slice bug in Go and not how Ant's arrays behave.
- **Byte order as a flag**, `little: bool = false`, as `DataView` has it. It reads badly as a bare
  `true`, and a default hides the order from the call. Separate `uint32le` and `uint32be` functions
  say it at the call but double the thirty names.
- **A clamping `copy`.** Go's `copy` copies `min` of the two lengths and returns the count. Ant
  panics on bounds everywhere else, and a returned count needs a fourth case in the `mutates` rule.

## Fit with other proposals

- **`frozen.md`.** It lists `Bytes` among the kinds that are immutable already, so that
  `Frozen[Bytes]` is `Bytes`. That line changes: `freeze(b)` of a `Bytes` is a copy whose writes
  panic, as for an array, and `Frozen[Bytes]` is how a program says that bytes will not change. It
  also gives back the immutable bytes this proposal takes away.
- **`typed-data.md`.** `Type.Data` keeps `Bytes`, now mutable, as it keeps arrays.
- **`buffer.md`.** The competing proposal; one of the two is taken.

## Open questions

1. **Sharing HTTP `body` and `bytes`.** Holding a text body twice may matter for large responses.
   Making one of the two a function, computed on demand, would avoid it at the cost of an API
   change.
