# Add `Array.countBy`

```ant
func Array.countBy[T, K](xs: Array[T], key: func(T): K): Map[K, int]
```

Use `countBy` to count occurrences per key. Today this takes two steps, `xs.groupBy(key).map({ x.value.length() })`, and
builds an array for every group. `countBy` can compute the same result in one pass, keeping one counter per distinct
key.

```ant
let failures = log.lines()
| Array.filter({ x.endsWith(" 401") })
| Array.countBy({ x.fields()[0] })
// Map{ "203.0.113.9": 3, "192.0.2.77": 2 }
```

`countBy` calls `key` once per element, from left to right. It returns a new map using the same key equality and
first-seen order as `groupBy`. An empty array produces an empty map.

`countWhere(predicate)` answers “how many elements match?” with one integer. `countBy(key)` answers “how many elements
have each key?” with a map of counts.
