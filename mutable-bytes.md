<!-- status: draft -->

# Mutable Bytes

## Motivation

`Bytes` is a compact but immutable sequence. Editing a file today requires converting it to
`Array[byte]` and back, allocating an array of boxed values. For large files, `File.readChunk`
also returns a fresh value for each chunk.

Make `Bytes` the one type for binary data: compact, mutable, and usable directly for I/O.
Do not add `ByteArray`, `Buffer`, pointers, or a second binary collection.

## Proposal

`Bytes` becomes a mutable, fixed-length sequence of `byte`, backed by contiguous memory.
It remains a distinct runtime kind from `Array` and `string`.

```ant
let data = try File.readBytes("input.bin")

for i in 0 .. len(data) {
  if data[i] == byte(0x41) {
    data[i] = byte(0x42)
  }
}

try File.writeBytes("output.bin", data)
```

### Semantics

- `b[i]` returns a `byte`; `b[i] = value` changes that byte. Indices are bounds-checked, and
  writes require `byte` (no implicit conversion from `int`).
- `let` prevents rebinding, not mutation. Assigning `let other = b` shares the same storage,
  just as assigning an array does.
- Length is fixed. There is no append, resize, or capacity API. `Bytes.concat` makes a new value.
- `Bytes.slice` returns an **independent copy**, not a view. `Bytes.toArray` and
  `Bytes.fromArray` also copy.
- `String.toBytes`, `String.fromBytes`, and `String.fromBytesLossy` produce independent
  storage. Mutating bytes must never change a `string`, or vice versa.
- `==` and hashing stay content-based. `Bytes` remains valid as a `Map` key or `Set`
  member; changing it after insertion is a program error, as with mutable array keys.
- Concurrent access follows Ant's existing data-race rule: unsynchronized mutation is a
  program error, but must not corrupt memory. A byte read observes a whole byte, never
  invalid storage.

### API additions

```ant
Bytes.new(size: int): Bytes                          // zero-filled; panics on invalid size
Bytes.replaceByte(b: Bytes, old: byte, new: byte,
                  from: int = 0, to: int | null = null): int
```

`replaceByte` mutates the half-open range `[from, to)` (`null` means `len(b)`) and returns
the number of replacements. Invalid ranges panic as other in-memory collection operations
do. It executes natively rather than calling an Ant function for each byte.

Existing `Bytes` functions keep their names and results, except that `slice` now copies.
No new literal syntax is needed.

### Reusable file I/O

```ant
File.readInto(f: File.Handle, bytes: Bytes,
              ctx: Context | null = null): Result[int | null, Error]
File.writeChunk(f: File.Handle, bytes: Bytes,
                to: int | null = null): Result[void, Error]
```

`readInto` reads directly into the existing buffer. It returns a positive byte count,
`null` on EOF, and `0` for an empty buffer without reading. A short read is **not** EOF;
only the returned prefix is valid. The next read uses the same handle and preserves its
existing read-ahead and cancellation guarantees. An I/O error is `Err`, not a panic.
The operation does not retain the caller's buffer after returning.

`writeChunk` retains its existing behavior; optional `to` writes only the first `to`
bytes (`null` means all). An invalid `to` is `Err`. It completes or reports a write error
before returning and does not retain the mutable buffer. Existing callers are unchanged.

### Example: constant-memory binary replacement

```ant
let input = try File.open("input.bin")
defer { File.close(input) }

let output = try File.create("output.bin")
defer { File.close(output) }

let data = Bytes.new(256 * 1024)

while true {
  let n = try input.readInto(data)
  if n == null { break }

  data.replaceByte(byte(0x41), byte(0x42), to: n)
  try output.writeChunk(data, to: n)
}

try output.close()
```

This allocates one reusable data buffer, modifies its contents in place, and writes only
its filled prefix. There is no `Bytes`/`Array` conversion or per-byte Ant callback.
The implementation should not allocate a new data buffer on each successful iteration;
OS I/O and internal buffering may still copy bytes.

## Compatibility

This is a **semantic breaking change**, though read-only source code largely stays valid:

- `Bytes` is no longer safe to share for concurrent mutation without synchronization.
- Slicing and string conversions can require copies where the old implementation shared memory.
- Existing code that hashes a `Bytes` value and then changes it becomes incorrect, as with
  mutable array keys.
- `File.readBytes`, `File.readChunk`, encoding, hashing, and crypto functions keep returning
  or accepting `Bytes`; they must not retain unprotected mutable storage after returning.

## Implementation and verification

- Represent `Bytes` as a compact, GC-managed byte buffer, not `Array` slots.
- Provide synchronization sufficient for Ant's memory-safety guarantee under data races.
  Native block operations should synchronize once per operation, not once per element.
- Test aliasing, independent slices, string isolation, key hashing, invalid indices,
  concurrent access, short reads, EOF, cancellation, and close/write errors.
- Benchmark whole-file edits and 256 KiB streaming against the current
  `Bytes -> Array[byte] -> Bytes` path and an equivalent Go `[]byte` loop. Measure
  throughput, peak memory, and allocations per chunk before claiming speedups.

## Non-goals

Resizable byte arrays, zero-copy mutable views, ownership types, raw pointers, packed
numeric arrays, and multi-byte replacements across chunk boundaries. Those can be
separate proposals if real programs need them.

## References

- [Ant language specification — Bytes](https://github.com/ant-src/spec/blob/master/ant-lang.md)
- [Ant standard library — File](https://github.com/ant-src/spec/blob/master/stdlib.md)
- [Current `Bytes` runtime representation](https://github.com/ant-src/ant/blob/master/value/bytes.go)
