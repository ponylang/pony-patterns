---
title: "Inheritance"
section: "Code Sharing"
menu:
  toc:
    parent: "code-sharing"
    weight: 20
---
## Problem

You want to share code between classes (or actors), but Pony doesn't have inheritance. What's an enterprising programmer to do? Pony is object-oriented and object-orientation means inheritance, right?

Pony believes in "composition over inheritance" so, inheritance isn't available. However, this doesn't mean that you can't share implementation across differing types. Enter [default implementations on traits and interfaces](https://tutorial.ponylang.io/types/traits-and-interfaces.html).

## Solution

Pony interfaces and traits can provide default implementations for their defined methods. For example, here's the definition of `TimerNotify` from the standard library:

```pony
interface TimerNotify
  """
  Notifications for timer.
  """
  fun ref apply(timer: Timer, count: U64): Bool =>
    """
    Called with the the number of times the timer has fired since this was last
    called. Usually, the value of `count` will be 1. If it is not 1, it means
    that the timer isn't firing on schedule.

    For example, if your timer is set to fire every 10 milliseconds, and
    `count` is 2, that means it has been between 20-29 milliseconds since the
    last time your timer fired. Non 1 values for a timer are rare and indicate
    a system under heavy load.

    Return true to reschedule the timer (if it has an interval), or
    false to cancel the timer (even if it has an interval).
    """
    true

  fun ref cancel(timer: Timer) =>
    """
    Called if the timer is cancelled. This is also called if the notifier
    returns false from its `apply` method.
    """
    None
```

Note that each of the methods defines a default implementation. In the case of `apply`, that default implementation is to return `true`.

```pony
  fun ref apply(timer: Timer, count: U64): Bool =>
    true
```

Not the most exciting of logic, we grant you that, however, it can be very useful. By defining standard interfaces for shared functionality, you can share default implementations across implementers. It's not quite inheritance, but it can still take you a long way.

## Discussion

There are a couple of important points to note about using default implementations as a means of code sharing.

### All default implementations must be stateless

Traits and interfaces can't have fields, so, your implementation will be limited to processing incoming data and returning a value.

### The user must opt-in to using default implementations

Default implementations are limited to nominally typed objects. What this means is that, any time a class implements a `trait` all default implementations will be picked up. If you are using an `interface` instead of a `trait`, then the default type won't be picked up unless a class is specifically declared as implementing the interface.

For example:

```pony
interface CandyMachine
  fun do_you_want_free_candy(): Bool =>
    "Who doesn't want free candy? Not me! Gimme, Gimme, Gimme!"
    true

class ChocolateMachine is CandyMachine

class CookieMachine
  fun do_you_want_free_candy(): Bool =>
    false
```

In our above example, `ChocolateMachine` declares that it is a `CandyMachine` by using `is CandyMachine`. By nominally typing itself as a `CandyMachine`, it picks up the default implementation of `do_you_want_free_candy`. `CookieMachine`, on the other hand, is not nominally typed as a `CandyMachine` and therefore doesn't pick up the default implementation.

Default implementations can be great way to share implementations across classes. However, the limitations that requires them to be stateless can at times be very constricting. If you need "stateful default implementations", check out the [Mixin pattern](/code-sharing/mixin.html).



