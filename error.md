# Proposal: `error()` Prelude Function

2026-11-11 · declined: a prelude name cannot be shadowed, so `error` would stop being a variable
name everywhere; [result-fail.md](result-fail.md) proposes `Result.fail` for the same need.

## Motivation

Returning a standard error is unnecessarily verbose:

```ant
return Result.err(Error{ message: "invalid quantity" })
```

Introduce `error()` as a convenience function:

```ant
return error("invalid quantity")
```

## Definition

Add to the prelude:

```ant
func error[T](message: string): Result[T, Error] {
  return Result.err(Error{ message })
}
```

`T` is inferred from the expected result type.

## Example

```ant
func parseOrder(row: Row): Result[Order, Error] {
  let quantity = row.quantity.toInt()

  if quantity == null {
    return error("order ${row.id}: quantity must be an integer")
  }
  if quantity <= 0 {
    return error("order ${row.id}: quantity must be positive")
  }

  Result.ok(Order{ id: row.id, quantity })
}
```

## Rules

- `error(message)` constructs `Result[T, Error]`; it does not throw or return early.
- `Result.err(value)` remains available for arbitrary error types.
- `Error{ message }` remains available for constructing an error value.
- No new syntax, implicit conversions, or changes to `try`.
- `error` becomes a prelude name, subject to existing shadowing rules.
