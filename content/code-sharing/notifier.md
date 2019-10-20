---
title: "Notifier"
section: "Code Sharing"
menu:
  toc:
    parent: "code-sharing"
    weight: 10
---
## Problem

You need to specialize an actor so that at certain times during it's lifecycle, you can take an approriate action. For example, all TCP connections are fundamentally the same but, what they do at certain points like:

- when opened
- sending data
- receiving data
- when closing

will need to change based on "the type" of TCP connection in question.

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

And our corresponding actor:

```pony
use "collections"

class Timer
  """
  The `Timer` class represents a timer that fires after an expiration
  time, and then fires at an interval. When a `Timer` fires, it calls
  the `apply` method of the `TimerNotify` object that was passed to it
  when it was created.

  The following example waits 5 seconds and then fires every 2
  seconds, and when it fires the `TimerNotify` object prints how many
  times it has been called:

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
