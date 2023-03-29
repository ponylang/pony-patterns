---
hide:
  - toc
---

# Mixin

## Problem

You have some common logic that you want to share between classes (or actors). You could use the ["inheritance pattern"](inheritance.md), however, if your logic is stateful and needs to access fields, it won't work. What you probably want is something akin to "mixins". Pony doesn't directly support mixins, however, you can emulate them fairly easily.

## Solution

Creating a "mixin like" thing in Pony is pretty straightforward. If you checked out the [inheritance pattern](inheritance.md), then you've already seen most of the mixin pattern in action.

Let's start by recapping the inheritance pattern and then add in our mixin twist.

Pony interfaces and traits can provide default implementations for their defined methods. You can use default implementations to provide the "behavior inheritance" aspect of a mixin. All that we need to get a full-on mixin is the ability to manipulate our enclosing objects state.

Ideally we want something like

```pony
trait MixinLike
  fun double_field(): U64 =>
    _a_field * 2
```

The above code won't compile. `_a_field` isn't defined on `MixinLike` so it won't compile. And, traits and interfaces can't have fields so we can't define it. What we can do is this...

```pony
trait MixinLike
  fun _a_field(): U64

  fun double_field(): U64 =>
    _a_field() * 2
```

Did you catch what we did there? We defined a new, unimplemented function `_a_field` that returns a `U64`. Any actor or class that implements `MixinLike` is required to implement `_a_field`, something like this:

```pony
class Foo is MixinLike
  var _counter: U64

  new create() =>
    _counter = 1

  fun _a_field(): U64 =>
    _counter
```

See what is going on? `Foo` has to define *something* that can supply the value that `_a_field` returns which is in turn used by `double_field`. We can expand this further to allow both reading and writing of values "within" our mixin.
Let's take a look at what that looks like with some naming that is a bit more realistic:

```pony
trait CounterIncrementer
  fun _current_counter_value(): U64
  fun ref _set_counter_value(v: U64)

  fun _increment_counter() =>
    let v = _current_counter_value() + 1
    _set_counter_value(v)

class Counter is CounterIncrementer
  var _counter: U64

  new create() =>
    _counter = 1

  fun _current_counter_value(): U64 =>
    _counter

  fun ref _set_counter_value(v: U64) =>
    _counter = v
```

## Discussion

Our `Counter` example is a little contrived, but it demonstrates everything we need to implement a mixin like usage pattern in Pony. There's more ceremony than you have in languages that support mixins natively, but unlike many of them you also have more control over how "foreign" code access the internal state of your objects.

For a more advanced usage of the mixin pattern where it is used to "turn the [notifier pattern](notifier.md) inside out", check out the code for [Lori networking library](https://github.com/seantallen/lori).
