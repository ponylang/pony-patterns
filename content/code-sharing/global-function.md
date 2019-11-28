---
title: "Global Function"
section: "Code Sharing"
menu:
  toc:
    parent: "code-sharing"
    weight: 40
---
## Problem

Your design calls for a global function, but Pony doesn't have them.

## Solution

Use a `primitive`.

```pony
primitive Doubler
  fun apply(U64: num): U64 =>
    num * 2
```

## Discussion

Primitives serve a couple of different purposes in Pony. Here, we are using a primitive like a global function. Any place in our codebase, we can use `Doubler` like so:

```pony
Doubler(U64(1))
```

Regular Pony package rules still apply, so the package that `Doubler` is a part of needs to be imported, but otherwise, you can use a primitive with an `apply` method much as you would global functions in a language like `C`.

Further, you can use a `primitive` to namespace several "globally accessible functions." Here's an example from the Pony standard library:

```pony
primitive Nanos
  """
  Collection of utility functions for converting various durations of time
  to nanoseconds, for passing to other functions in the time package.
  """
  fun from_seconds(t: U64): U64 =>
    t * 1_000_000_000

  fun from_millis(t: U64): U64 =>
    t * 1_000_000

  fun from_micros(t: U64): U64 =>
    t * 1_000

  fun from_seconds_f(t: F64): U64 =>
    (t * 1_000_000_000).trunc().u64()

  fun from_millis_f(t: F64): U64 =>
    (t * 1_000_000).trunc().u64()

  fun from_micros_f(t: F64): U64 =>
    (t * 1_000).trunc().u64()

  fun from_wall_clock(wall: (I64, I64)): U64 =>
    ((wall._1 * 1000000000) + wall._2).u64()
```

In general, if you are looking to do anything that might be classified as a static function or a utility function in another language, you probably want to use a Primitive in Pony.
