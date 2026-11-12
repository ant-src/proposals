# Proposal: `Buffer` and binary integers

2026-11-12 · draft

## Summary

Add `Buffer`, a mutable, growable sequence of bytes, beside the immutable `Bytes`, and add functions
that read and write fixed-size integers and floats in both. `Bytes` stays as it is: the value that
I/O gives and takes, that compares by contents, and that shares memory with text. `Buffer` is where
a program computes bytes: it holds one byte per byte, `buf[i] = x` writes in place, and
`Buffer.toBytes` hands the result on.

```ant
// A frame: a big-endian length, then the payload masked with a 4-byte key.
func frame(payload: Bytes, key: Bytes): Bytes {
  let buf = Buffer.new()
  buf.pushUint32(uint32(len(payload)))
  buf.extend(payload)
  for i in Array.range(4, len(buf)) {
    buf[i] = Bits.xor(buf[i], key[(i - 4) % 4])
  }
  buf.toBytes()
}

// And back: the length, then that many bytes.
func payloadOf(frame: Bytes): Bytes {
  let n = int(frame.uint32(0))
  frame.slice(4, 4 + n)
}
```

`Array[byte]` stays legal and stops being the way to build bytes.

## Motivation

`Bytes` is immutable, and the spec sends a program that computes bytes to `Array[byte]`: "A program
that computes bytes one at a time builds an `Array[byte]` and converts it." An array holds a value
per element, so that path is expensive in memory, and a change to one byte of existing `Bytes` goes
through `Bytes.toArray` and `Bytes.fromArray`, two copies of the whole.

Measured on ant master of 2026-11-12, on an Apple M-series laptop:

| Program                                                            | Time   | Peak memory |
| ------------------------------------------------------------------ | ------ | ----------- |
| An empty loop of 4 million iterations, for scale                   | 0.71 s | 22 MB       |
| Encode 4 MB a byte at a time: `push`, then `Bytes.fromArray`       | 0.92 s | 259 MB      |
| Mask 4 MB with a key into `Array.filled`, then `Bytes.fromArray`   | 0.96 s | 356 MB      |
| Read a million big-endian `uint32` with `b[i]` and arithmetic      | 0.52 s | 28 MB       |
| Change one byte of 1 MB, 200 times: `toArray`, assign, `fromArray` | 3.36 s | 161 MB      |

Three things follow.

- **Memory, not speed, is the problem.** A program that builds 4 MB holds 65 to 90 bytes per byte at
  its peak. The loop itself costs what any Ant loop costs.
- **A change in place is linear.** One byte of a megabyte takes 17 ms. A program that patches a
  checksum into a header, draws into a pixel buffer, or masks a frame pays that per change, or keeps
  the data as an array for its whole life and pays the memory.
- **Integers are spelled out by hand.** A `uint32` is four index reads, four conversions and three
  multiplications, and a signed or little-endian one is easier to get wrong. The `Bytes` decision
  deferred readers "until a program needs them"; every binary format needs them.

## Prior art

| Language | Immutable          | Mutable                  | Integers                                   |
| -------- | ------------------ | ------------------------ | ------------------------------------------ |
| Python   | `bytes`            | `bytearray`              | `int.from_bytes`, `struct`                 |
| Rust     | `Bytes`, `&[u8]`   | `BytesMut`, `Vec<u8>`    | `u32::from_be_bytes`, `Buf`/`BufMut`       |
| Java     | `ByteString` (lib) | `byte[]`, `ByteBuffer`   | `ByteBuffer.getInt`, `order()`             |
| JS       | none               | `Uint8Array`             | `DataView.getUint32(offset, littleEndian)` |
| Go       | `string`           | `[]byte`, `bytes.Buffer` | `binary.BigEndian.Uint32`, `AppendUint32`  |
| Swift    | `Data` (value)     | `Data` (copy on write)   | `withUnsafeBytes`, `load(as:)`             |

Python and Rust have exactly this pair, and both convert between the two explicitly. Go has one
mutable type and uses `string` where it needs an immutable key; Ant's `Bytes` is stored as a Go
string already. For integers, Go's `encoding/binary` gives the semantics, and `DataView` gives the
shape that fits Ant: an offset and a flag, where Go slices and picks a byte-order value.

## The design

### The type

`Buffer` is a kind of its own, built in as `Bytes` is: declared in the prelude, not a primitive, and
taken by the same language forms.

- `len(buf)` is the number of bytes, and `for x in buf` yields each as a `byte`. The loop reads the
  length once, when it starts, as a loop over an array does.
- `buf[i]` reads a `byte` and `buf[i] = x` writes one; both panic when `i` is outside
  `0 ..= len(buf) - 1`. `x` is a `byte`, so an `int` is converted as anywhere: `buf[i] = byte(n)`.
- `buf is Buffer` tests the kind. A `Buffer` is never a `Bytes`, nor the reverse.
- `==` compares contents, as it does for arrays. A `Buffer` and a `Bytes` are of different kinds and
  so unequal. A `Buffer` can be a map key under the rule for arrays: changing it afterwards is a
  program error.
- It renders as `Buffer(89 50 4e 47)`. `Json.stringify` panics on it, as on `Bytes`.
- For reflection it is opaque, as `Bytes` is.

There is no literal and no capacity. A buffer grows as an array does, by amortized doubling, and a
program that knows the size asks for it with `Buffer.new(n)` and assigns by index.

### Functions

```
Buffer.new(size: int = 0): Buffer                                // size zero bytes; panics if size < 0 or above 2^31 − 1
Buffer.from(b: Bytes): Buffer                                    // a copy
Buffer.toBytes(buf: Buffer): Bytes                               // a copy; buf is unchanged
Buffer.length(buf: Buffer): int
Buffer.slice(buf: Buffer, from: int, to: int): Bytes             // a copy; panics as Array.slice does
Buffer.push(buf: Buffer, x: byte): void                          // mutates, as do the functions below
Buffer.extend(buf: Buffer, src: Bytes | Buffer): void            // appends src as it was at the call
Buffer.copy(buf: Buffer, at: int, src: Bytes | Buffer): void     // overwrites from at; panics if at or at + len(src) is outside 0 ..= length
Buffer.fill(buf: Buffer, x: byte, from: int = 0, to: int | null = null): void   // to: the length; panics as slice does
Buffer.truncate(buf: Buffer, n: int): void                       // keeps the first n bytes; panics if n is outside 0 ..= length
```

`Buffer.copy` does not grow the buffer, as Go's `copy` does not; `extend` does. `truncate(0)`
empties a buffer for reuse. Text goes in as its bytes: `buf.extend(String.toBytes(s))`, which need
not copy `s`.

Everything else stays on `Bytes`. Searching, `startsWith`, hashing, encoding, and every read and
write of a file, a connection or a command take and give `Bytes`, and a program with a `Buffer`
calls `toBytes` first. That is one copy at memory speed, and it keeps each library function at one
signature.

### Integers and floats

```
Bytes.uint8(b: Bytes, at: int): uint8
Bytes.uint16(b: Bytes, at: int, little: bool = false): uint16
Bytes.uint32(b: Bytes, at: int, little: bool = false): uint32
Bytes.uint64(b: Bytes, at: int, little: bool = false): uint64
Bytes.int8 … Bytes.int64                                         // two's complement
Bytes.float32(b: Bytes, at: int, little: bool = false): float32  // IEEE 754 bits
Bytes.float64(b: Bytes, at: int, little: bool = false): float64

Buffer.uint8 … Buffer.float64                                    // the same readers, on a Buffer

Buffer.pushUint16(buf: Buffer, v: uint16, little: bool = false): void   // appends 2 bytes
Buffer.setUint16(buf: Buffer, at: int, v: uint16, little: bool = false): void   // overwrites 2 bytes
…                                                                // push and set for each of the ten kinds
```

A reader takes the bytes from `at` and panics when `at` or `at + size` is outside `0 ..= length`.
`set` panics the same way and never grows the buffer; `push` appends. The order is big-endian unless
`little` is true, since network formats are big-endian and a file format names its order once, in a
constant the program passes along. A float is read and written by its bits, so a NaN keeps its
payload.

The value's type carries the width, so there is no size argument and no checked narrowing inside the
library: a program holding an `int` writes `buf.pushUint32(uint32(n))`, and the conversion panics
when `n` does not fit, as every conversion does.

`uint8` and `int8` are there for uniformity: `b.uint8(i)` is `b[i]`, and `b.int8(i)` saves a
conversion that would otherwise panic above 127.

### Concurrency

A `Buffer` is shared between goroutines by reference, as an array is, and section 9 applies:
unsynchronized use from two goroutines is a data race and a program error, and memory stays safe. A
racy read of a byte sees some byte that was stored there. A racy multi-byte read or write may see or
leave a mix of two values' bytes, as a racy read of two array elements may. A racy `push`, `extend`
or `truncate` leaves the length before or after it, and an index that raced with a `truncate` reads
the byte or panics with the ordinary index error.

`Bytes` stays the type to send across goroutines: nothing can change it.

### What becomes of `Array[byte]`

Nothing changes for it. It remains an array of `byte` values with every `Array` function, and
`Bytes.fromArray` and `Bytes.toArray` remain: `fromArray` is how a constant is written, and
`toArray` is the way to `map`, `filter` or `sort` bytes. The spec's advice changes, and the
checker's message for `b[i] = x` on `Bytes` points to `Buffer.from(b)`.

## Spec changes

- **Section 4, Types.** `Buffer` beside `Bytes` in Primitives, the kinds table, and the lists for
  `is`. Records, arrays, maps, sets and `Buffer` are the mutable kinds.
- **Sections 6 and 8.** `for` and `len` take a `Buffer`.
- **Section 9, Data races.** The paragraph above.
- **Section 11.** A `Buffer` section after `Bytes`; the readers in `Bytes`; the sentence on building
  an `Array[byte]` replaced; rendering; `Json.stringify`.
- **Reflection.** `Buffer` in the opaque list.
- **Run-time semantics.** Index read and write, `==`, `is`, `for`, and the panics table.
- **`stdlib.md`.** No change.

## Implementation

`value.Buffer` follows `value.Array`: an atomic pointer to a header holding a `[]byte`, and a mutex
taken by the functions that change the length. An index read or write loads the header and touches
one byte; growth publishes a new header, so a racing reader holds a whole slice, old or new. The
integer functions are `encoding/binary` calls. The interpreter gains one case in each of index read,
index write, `len`, `for`, `==`, hashing, rendering and `is`; the checker allows the same forms. No
existing path changes, and arrays keep their one representation.

Expected, not yet measured: the two building programs above at about 1 to 2 bytes per byte, the
one-byte change in constant time, and a `uint32` read several times faster as one call. A per-byte
loop gains little in time, since the loop is the cost.

For the JavaScript backend both `Bytes` and `Buffer` are a `Uint8Array`, the second with a length
and room to grow, and the integer functions are `DataView` calls.

## Non-goals

- **I/O into a buffer.** No `File.readInto(buf)`. A read gives fresh `Bytes`, and Go's reuse of one
  slice across reads is an optimization Ant leaves to the runtime.
- **A cursor.** No read position in `Bytes` or `Buffer`. A parser keeps its offset in a variable.
- **A format language.** No `struct.pack("<IH", …)`. Typed calls say the same and are checked.
- **Variable-length integers, bit fields.** Written in Ant with `Bits` until a program shows the
  need.
- **Typed arrays.** No `Array[int32]` stored compactly. The interpreter does not know an array's
  element type (section 4, Checking and running), and this proposal does not change that.

## Alternatives considered

- **Store `Array[byte]` compactly.** No new type, and existing programs get cheaper. But the
  interpreter never knows `T`, so an array would have to switch representation on what is stored in
  it, and every `Array` function would have a second path. That is hidden machinery in the part of
  the runtime that is meant to stay simple.
- **Make `Bytes` mutable.** One type, as in Go. It gives up `Bytes` as a map key, the sharing with
  strings that makes `String.fromBytes` and an HTTP `body` beside its `bytes` free, and safe passing
  between goroutines, and it goes against `frozen.md`.
- **A write-only builder, with `Bytes.set` and `Bytes.xor` returning copies.** It covers encoding. A
  change in place stays linear, and each bulk operation someone needs becomes a library function.
- **Readers only.** Worth having alone, and the first step of the implementation, but it leaves both
  memory rows of the table as they are.

## Fit with other proposals

- **`frozen.md`.** `freeze` of a `Buffer` should give `Bytes`, the immutable form that exists
  already, so `Frozen[Buffer]` is `Bytes`.
- **`typed-data.md`.** `Type.Data` keeps `Bytes`. A `Buffer` field is opaque and so refused, as a
  channel is; a decoder gives `Bytes`.

## Open questions

1. **A hand-over without a copy.** `Buffer.take(buf): Bytes` could return the contents and leave the
   buffer empty, as `BytesMut::freeze` and `strings.Builder.String` do. A goroutine that raced with
   it could then write into bytes a `Bytes`, and perhaps a string or a map key, already holds. The
   draft has the copy only; `take` can be added if the copy shows up in a profile.
2. **The name.** `Buffer` is Go's and Node's word, but `File.buffered` uses it for something else.
   `ByteArray` and `Bytes.Builder` are the alternatives.
3. **`Bytes | Buffer` in the library.** `File.writeChunk`, `Crypto.sha256` and the like could take
   either and save the `toBytes`. The draft says no, for one signature per function.
4. **Byte order as a flag.** `little: bool` reads well at a call with a named argument and badly as
   a bare `true`. A pair of tags, or separate `uint32le` functions, are the alternatives.
