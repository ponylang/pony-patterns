---
hide:
  - toc
---

# Notifier

## Problem

You need to specialize an actor so that at certain times during its life-cycle, you can take an appropriate action. For example, all TCP connections are fundamentally the same but, what they do at certain points like:

- when opened
- sending data
- receiving data
- when closing

will need to change based on "the type" of TCP connection in question.

Actor specialization arises often when you want to create reusable Pony code. There's a pattern in the standard library that is one of the more common ways to address. Let's have a look at the "Notifier pattern".

Note that our example is based on the `Timers` actor from the standard library. `Timers` is an actor but it has an intermediate class that it uses as part of its design; that class `Timer` is what we will be looking at. It demonstrates the notifier pattern quite well. It's just important to remember that everything that happens in the `Timer` class is reusable logic that calls into logic to specialize a `Timers` actor.

## Solution

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

And our corresponding object to specialise:

```pony
use "collections"

class Timer
  """
  The `Timer` class represents a timer that fires after an expiration
  time, and then fires at an interval. When a `Timer` fires, it calls
  the `apply` method of the `TimerNotify` object that was passed to it
  when it was created.
  """

  var _expiration: U64
  var _interval: U64
  let _notify: TimerNotify
  embed _node: ListNode[Timer]

  new iso create(
    notify: TimerNotify iso,
    expiration: U64,
    interval: U64 = 0)
  =>
    """
    Create a new timer. The expiration time should be a nanosecond count
    until the first expiration. The interval should also be in nanoseconds.
    """
    _expiration = expiration + Time.nanos()
    _interval = interval
    _notify = consume notify
    _node = ListNode[Timer]
    try _node()? = this end

  fun ref _cancel() =>
    """
    Remove the timer from any list.
    """
    _node.remove()
    _notify.cancel(this)

  fun ref _fire(current: U64): Bool =>
    """
    A timer is fired if its expiration time is in the past. The notifier is
    called with a count based on the elapsed time since expiration and the
    timer interval. The expiration time is set to the next expiration. Returns
    true if the timer should be rescheduled, false otherwise.
    """
    let elapsed = current - _expiration

    if elapsed < (1 << 63) then
      let count = (elapsed / _interval) + 1
      _expiration = _expiration + (count * _interval)

      if not _notify(this, count) then
        _notify.cancel(this)
        return false
      end
    end

    (_interval > 0) or ((_expiration - current) < (1 << 63))
```

## Discussion

It's important to note that the notifier pattern is fundamentally one that involves callbacks. And that in particular, it is about specializing actors.

As seen in the `Timer` example from the standard library, all "reusable" logic is part of an actor `Timers`. The points where a user would want to specialize what happens, involve calling methods on the supplied "notify" object.

For example, from our example above:

```pony
      if not _notify(this, count) then
```

Each time the `Timer` fires, we call the `apply` method of the `_notify` object allowing situation-specific code to be executed. In the case of the `TimerNotify` API, a boolean is returned. If the return value is `false`, the timer isn't rearmed and our other callback is called:

```pony
        _notify.cancel(this)
```

What does this look like to a user? Here's an example:

```pony
use "time"

actor Main
  new create(env: Env) =>
    let timers = Timers
    let timer = Timer(Notify(env), 5_000_000_000, 2_000_000_000)
    timers(consume timer)

class Notify is TimerNotify
  let _env: Env
  var _counter: U32 = 0

  new iso create(env: Env) =>
    _env = env

  fun ref apply(timer: Timer, count: U64): Bool =>
    _env.out.print(_counter.string())
    _counter = _counter + 1
    true
```

There's a couple of key points that our example highlights about our implementation that weren't clear before.

First, a notifier should always be an `iso`. Why? Because the notify object will probably need to keep and update state. If we were to supply the same notify to multiple actors, the state would become shared and be unsafe across actors. In the `Timer` implementation you will see this in create taking a `TimerNotify iso`

```pony
  new iso create(
    notify: TimerNotify iso,
    expiration: U64,
    interval: U64 = 0)
```

And in our concrete `Notify` implementation where we default the reference type when creating a new `Notify` to `iso`:

```pony
  new iso create(env: Env) =>
    _env = env
```

The notifier pattern is incredibly powerful. It makes it very easy for programmers to easily plug-in to existing functionality and get up and running. In the majority of specialization cases, the notifier pattern is probably what you want. However, there is one drawback that you need to be aware of for more advanced cases.

Notifiers cannot receive messages. They can send messages to actors they hold references to, but they have no way to receive arbitrary messages. Access to the notifier is completely controlled by the encompassing actor. As limitations go, this usually isn't a problem. However, it can be problematic for some more advanced use-cases. If you find yourself needing to send messages to a notifier, we suggest you take a look at [the Mixin pattern](mixin.md).
