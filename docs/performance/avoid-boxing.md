---
hide:
  - toc
---

# Avoid Boxing with Parameterization

## Problem

You need a function that works with multiple numeric types. A natural first approach is to accept `Any val` and match on each possible type:

```pony
primitive Formatter
  fun int(value: Any val): String =>
    match value
    | let v: I8 => v.string()
    | let v: I16 => v.string()
    | let v: I32 => v.string()
    | let v: I64 => v.string()
    | let v: I128 => v.string()
    | let v: ILong => v.string()
    | let v: ISize => v.string()
    | let v: U8 => v.string()
    | let v: U16 => v.string()
    | let v: U32 => v.string()
    | let v: U64 => v.string()
    | let v: U128 => v.string()
    | let v: ULong => v.string()
    | let v: USize => v.string()
    else
      ""
    end
```

This compiles and works, but it has two problems. The obvious one is that the code is verbose and fragile: every new numeric type needs another match arm, and it's easy to miss one. The less obvious one is performance. Every call to `int` boxes the argument, wrapping the primitive value in a heap-allocated object even though it would fit in a machine register. That's a heap allocation and eventual garbage collection for every call. Call `int` in a hot loop and you'll destroy your performance.

## Solution

Use a type parameter to let the compiler know the concrete type at each call site:

```pony
primitive Formatter
  fun int[A: (Int & Integer[A])](value: A): String =>
    value.string()
```

The constraint `(Int & Integer[A])` tells the compiler that `A` is some integer type that implements `Integer`. At each call site, the compiler knows the exact type (`U32`, `I64`, or whatever the caller passes) and generates code that works directly with that type. No boxing, no matching, and a compile error if someone passes a non-integer.

## Discussion

Primitive values like `U32` and `Bool` are small enough to live in a machine register. But when Pony needs to pass one where any type is expected (an `Any val` parameter, or a union like `(U32 | U64)`), it wraps the value in a heap-allocated object. This wrapping is called boxing. The runtime allocates memory, copies the value in, and later the garbage collector has to reclaim that memory. For a single call, the cost is negligible. In a hot loop processing thousands of values, it adds up fast.

You might think narrowing the parameter from `Any val` to a specific union like `(U32 | U64)` would help. It doesn't. The runtime still needs a tagged representation to distinguish the variants, so both types get boxed at the call site. The only way to avoid boxing is to let the compiler know the single concrete type, which is what type parameters provide.

The example above parameterizes a single function, but the pattern applies equally to classes and actors. Consider a collector actor that accumulates values:

```pony
// Boxing version: every value sent to this actor gets boxed
actor Collector
  let _data: Array[Any val] = Array[Any val]

  be collect(value: Any val) =>
    _data.push(value)
```

Each message sent to `collect` will box its argument if it's a primitive. Parameterizing the actor eliminates the boxing:

```pony
// No boxing: the compiler knows the concrete type
actor Collector[A: Any val]
  let _data: Array[A] = Array[A]

  be collect(value: A) =>
    _data.push(value)
```

Now `Collector[U64]` stores unboxed `U64` values, and `Collector[String]` stores `String` references. Each instantiation is specialized to its type. The trade-off is that a single collector instance can only hold one type, but in practice that's usually what you want. Code that genuinely needs mixed types can still use `Any val`, paying the boxing cost only where heterogeneity is actually needed.

The standard library uses this pattern in several places. `Format.int` is the most direct example, with the same `[A: (Int & Integer[A])]` constraint shown in the Solution above. `String.read_int` uses `[A: ((Signed | Unsigned) & Integer[A] val)]` to parse an integer from a string into whatever concrete type the caller requests. The `math` package's `GreatestCommonDivisor` and `LeastCommonMultiple` both parameterize their `apply` methods over integer types. Whenever you find yourself reaching for `Any val` or a match across numeric types, check whether a type parameter can do the job instead.
