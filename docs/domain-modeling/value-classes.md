---
hide:
  - toc
---

# Value Classes

## Problem

A value class is a type whose identity is defined by its contents, not by which object it happens to be in memory. Think of a point on a coordinate plane: two points at (3, 4) are the same point regardless of how many `Point` objects you've allocated. The value *is* the coordinates. A color, a money amount, a date: these are all values. Two instances with the same fields should be equal, hashable to the same bucket, and printable the same way.

Pony doesn't give you any of that for free. A bare class has identity semantics: each instance is a distinct object, and the language has no built-in notion of "these two things hold the same data, so treat them as equal."

```pony
class val Point
  let x: I64
  let y: I64

  new val create(x': I64, y': I64) =>
    x = x'
    y = y'

actor Main
  new create(env: Env) =>
    let a = Point(3, 4)
    let b = Point(3, 4)

    // Won't compile — Point has no eq method
    // if a == b then env.out.print("equal") end

    // Identity comparison compiles, but it answers "are these the
    // same object?" not "are these logically the same?"
    if a is b then
      env.out.print("same object")
    else
      env.out.print("different objects")  // always prints this
    end
```

The `==` operator requires the type to implement `Equatable`, which `Point` doesn't. The `is` operator compares identity (whether two references point to the same object in memory), which isn't what you want for values. And `Map` requires its keys to be both `Hashable` and `Equatable`, so you can't use a `Point` as a map key either.

## Solution

Turn `Point` into a proper value class by implementing four interfaces: `Equatable` for structural equality, `Hashable` so it works with `Map` and `Set`, `Comparable` for ordering, and `Stringable` for printing. `Equatable` and `Hashable` are the essential pair. `Comparable` and `Stringable` are optional but come up often enough that they're worth covering together.

Start with `Equatable`. The default `eq` method compares identity, so you need to override it to compare field values instead:

```pony
class val Point is Equatable[Point]
  let x: I64
  let y: I64

  new val create(x': I64, y': I64) =>
    x = x'
    y = y'

  fun eq(that: box->Point): Bool =>
    (x == that.x) and (y == that.y)
```

The `eq` method takes `box->Point`, which is a read-only view of the other point. This is a viewpoint adaptation: `box->` means "read the `Point` through a `box` reference." Since `Point` is a `val` class, `box->val` resolves to `val`, so you can read the other point's fields just fine.

Now `==` compares contents. But to use `Point` as a `Map` key, you also need `Hashable`. The `Hashable` interface requires a single method, `hash`, that returns a `USize`:

```pony
use "collections"

class val Point is (Equatable[Point] & Hashable)
  let x: I64
  let y: I64

  new val create(x': I64, y': I64) =>
    x = x'
    y = y'

  fun eq(that: box->Point): Bool =>
    (x == that.x) and (y == that.y)

  fun hash(): USize =>
    x.hash() xor (y.hash() << 1)
```

The hash function combines the hashes of both fields. Every primitive numeric type in Pony already has a `hash` method that applies good bit mixing, so you don't need to worry about mixing individual field hashes. What you do need to worry about is how you combine them.

Plain XOR (`x.hash() xor y.hash()`) is a bad choice. It's symmetric: `Point(1, 2)` and `Point(2, 1)` would get the same hash. Worse, any point where both coordinates are equal (`Point(5, 5)`) would hash to zero, since any value XORed with itself is zero. Adding a bit shift (`<< 1`) before the XOR makes the combination order-dependent and avoids the self-cancellation problem.

For types with more than two fields, chain the combination. Each field gets a different shift to keep the hashes distinct:

```pony
// For a hypothetical Color class with r, g, b fields:
fun hash(): USize =>
  r.hash() xor (g.hash() << 1) xor (b.hash() << 2)
```

There's one rule that must never be violated: **if two values are equal, they must have the same hash.** The reverse isn't required (different values can share a hash; that's just a collision), but if `a == b` and `a.hash() != b.hash()`, hash-based collections like `Map` and `Set` will silently lose data. Build your `eq` and `hash` from the same set of fields to keep them in sync.

Next, `Comparable`. This extends `Equatable` with ordering. You only need to implement `lt` (less than); the interface provides default implementations for `le`, `ge`, `gt`, and `compare` based on your `eq` and `lt`:

```pony
  fun lt(that: box->Point): Bool =>
    if x == that.x then
      y < that.y
    else
      x < that.x
    end
```

This gives lexicographic ordering: compare by `x` first, break ties with `y`. The ordering you choose depends on what makes sense for your type. The only requirement is that it's consistent (if `a < b` and `b < c`, then `a < c`).

Finally, `Stringable`. The interface requires a `string` method that returns `String iso^`:

```pony
  fun string(): String iso^ =>
    "(" + x.string() + ", " + y.string() + ")"
```

The `+` operator on `String` returns `String iso^`, which is exactly what `Stringable` requires. You can chain concatenations directly and the result has the right type.

Here's the complete program with all four interfaces:

```pony
use "collections"

class val Point is (Comparable[Point] & Hashable & Stringable)
  let x: I64
  let y: I64

  new val create(x': I64, y': I64) =>
    x = x'
    y = y'

  fun eq(that: box->Point): Bool =>
    (x == that.x) and (y == that.y)

  fun lt(that: box->Point): Bool =>
    if x == that.x then
      y < that.y
    else
      x < that.x
    end

  fun hash(): USize =>
    x.hash() xor (y.hash() << 1)

  fun string(): String iso^ =>
    "(" + x.string() + ", " + y.string() + ")"

actor Main
  new create(env: Env) =>
    let a = Point(3, 4)
    let b = Point(3, 4)
    let c = Point(1, 2)

    // Structural equality
    env.out.print(a.string() + " == " + b.string() + ": " + (a == b).string())
    env.out.print(a.string() + " == " + c.string() + ": " + (a == c).string())

    // Ordering
    env.out.print(c.string() + " < " + a.string() + ": " + (c < a).string())

    // Use as a Map key
    let m = Map[Point, String]
    m(a) = "origin-adjacent"
    m(c) = "near-origin"

    try
      env.out.print("m(" + b.string() + ") = " + m(b)?)
    end
```

The type signature `Comparable[Point] & Hashable & Stringable` is all you need. `Comparable` already extends `Equatable`, so listing `Equatable` separately would be redundant.

## Discussion

Don't confuse "value class" with `class val`. They're different things. `class val` is a Pony keyword combination that makes `val` the default capability for instances of the class. A value class is a design concept: a type whose identity comes from its contents. You can have a `class val` that isn't a value class (it's just an immutable object without structural equality), and you could technically implement value class semantics on a `class ref`. That said, `class val` is the right default for value classes. Immutability means instances can be freely shared between actors, and it prevents a subtle bug: if a value class were mutable, inserting it into a `Map` and then changing a field would break the hash, and the map would silently lose track of the entry.

The `Map` and `Set` types in the `collections` package don't use `Hashable` and `Equatable` directly. They're parameterized by a `HashFunction` that bundles `hash` and `eq` together. The type aliases `Map[K, V]` and `Set[A]` plug in `HashEq`, which delegates to the key's own `hash` and `eq` methods. That's why `Map` requires its key type to be both `Hashable` and `Equatable`: `HashEq` needs both. If you only needed equality without hashing (say, for searching a list), `Equatable` alone would be enough.

Value classes and [Constrained Types](constrained-types.md) solve different problems, but they're complementary. Constrained types ensure data is valid at construction time: a `Username` can only exist if it passed validation. Value classes define what it means for two instances to be "the same": two `Point` objects with identical coordinates are interchangeable. You'll often want both. A validated `Email` type might also need structural equality so you can deduplicate a list of addresses or use them as map keys.
