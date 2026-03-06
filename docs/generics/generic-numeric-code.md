---
hide:
  - toc
---

# Generic Numeric Code

## Problem

You want to write a class that works with any unsigned integer type. A block cipher component, say, that stores a value and provides parity checking and XOR operations. Your first instinct might be to use `Unsigned` as the type constraint:

```pony
class Block[T: Unsigned]
  var data: T

  new create(data': T) =>
    data = data'

  fun parity(): Bool =>
    (data.popcount() % 2) == 1

  fun ref apply_xor(o: Block[T]) =>
    data = o.data xor data
```

This doesn't compile. You'll get errors on the `%`, `==`, and `xor` operations, plus the numeric literals `2` and `1`. The constraint looks right, but `Unsigned` alone isn't enough to make generic numeric code work in Pony.

## Solution

Two things need to change. First, the type constraint needs to be `(Unsigned & UnsignedInteger[T])` instead of just `Unsigned`. Second, numeric literals need to be wrapped with `T.from[U8](n)`.

Let's start with the constraint. `Unsigned` in Pony is a union type containing all the built-in unsigned integers: `U8`, `U16`, `U32`, `U64`, `U128`, `ULong`, and `USize`. When you use only this union as a constraint, the compiler has to consider that `T` could be instantiated with a union like `(U8 | U16)`. That's a problem because you can't XOR a `U8` with a `U16`. Binary operations need both operands to be the same type.

Adding `UnsignedInteger[T]` to the constraint fixes this. `UnsignedInteger` is a trait that guarantees its type parameter can do binary operations, arithmetic, and comparisons with another value of the same type. By constraining `T` to be both `Unsigned` and `UnsignedInteger[T]`, you're telling the compiler: `T` must be one of the concrete unsigned integer types, and it must support operations with other values of exactly type `T`.

```pony
class Block[T: (Unsigned & UnsignedInteger[T])]
```

The second fix is numeric literals. Pony can't create a number literal for a generic type. When the compiler sees `2`, it needs to know the concrete type right then. In non-generic code that's inferred from context, but inside a generic function the compiler only knows `T`, not whether it's `U8` or `U64`.

The workaround is `T.from[U8](n)`. This tells the compiler the literal `n` is a `U8`, then converts it to whatever `T` turns out to be:

```pony
  fun parity(): Bool =>
    (data.popcount() % T.from[U8](2)) == T.from[U8](1)
```

Since we're dealing with constant values, LLVM trivially optimizes away the conversion. There's no runtime cost.

Putting it all together:

```pony
class Block[T: (Unsigned & UnsignedInteger[T])]
  var data: T

  new create(data': T) =>
    data = data'

  fun parity(): Bool =>
    (data.popcount() % T.from[U8](2)) == T.from[U8](1)

  fun ref apply_xor(o: Block[T]) =>
    data = o.data xor data

actor Main
  new create(env: Env) =>
    let a = Block[U32](0b1011)
    let b = Block[U32](0b1100)
    env.out.print("a parity (odd number of 1s): " + a.parity().string())
    env.out.print("b parity (even number of 1s): " + b.parity().string())
```

You can instantiate `Block[U8]`, `Block[U32]`, `Block[U128]`, or any other concrete unsigned integer type, and the compiler generates specialized code for each.

## Discussion

The same pattern works for signed integers and for all integers. For signed types, use `(Signed & Integer[T])`. For code that works with any integer regardless of signedness, use `(Int & Integer[T])`. You might expect `SignedInteger[T]` to mirror `UnsignedInteger[T]`, but `SignedInteger` takes two type parameters (the signed type and its unsigned counterpart), making it awkward to use in constraints. `Integer[T]` is simpler and is what the standard library uses. The standard library follows this convention throughout. `Format.int` uses `[A: (Int & Integer[A])]`, and `String.read_int` uses `[A: ((Signed | Unsigned) & Integer[A] val)]`. Whenever you see a generic function constrained to a numeric type in the standard library, you'll find this intersection pattern.

The reason `Unsigned` alone doesn't work comes down to how Pony's type system handles union types as constraints. A constraint like `[T: Unsigned]` means `T` can be any subtype of `Unsigned`. Since `Unsigned` is a union, `(U8 | U16)` is a valid subtype of it, so `Block[(U8 | U16)]` would be a legal instantiation. With a union as `T`, binary operations between two values of type `T` could end up mixing a `U8` and a `U16`, which Pony doesn't allow. The `UnsignedInteger[T]` trait constrains `T` to a single concrete type that supports operations with itself, ruling out union instantiations.

The `T.from[U8](n)` pattern for literals deserves a closer look. `U8` is the natural choice for the source type because it's the smallest unsigned integer, and numeric literals used in generic code tend to be small constants (0, 1, 2, bitmasks). You could use any integer type as the source, but `U8` keeps the intent clear: this is a small constant being widened to whatever `T` is. The `from` method is part of the `Integer` trait, so it's available on any type that satisfies the constraint.

If you're reaching for generic numeric code because of performance concerns around boxing, the [Avoid Boxing with Parameterization](../performance/avoid-boxing.md) pattern covers that angle. Boxing happens when a primitive value gets passed where any type is expected (`Any val` or a union parameter), forcing a heap allocation. Using type parameters avoids boxing because the compiler knows the concrete type at each call site. The constraint patterns described here and in that pattern are the same; the difference is the motivation. Here the focus is on making generic numeric code compile correctly. There the focus is on eliminating unnecessary heap allocations.
