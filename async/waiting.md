# Waiting

## Problem

Here's the problem: you're writing an application that needs to execute an action every few seconds. In a language with blocking operations, I could just call sleep and be done with it. It might not be the most elegant solution but it would work. In Pony, no such obvious solution exists.  One of Pony's key features is there are not many blocking operations (example exception: the files package calls fread). It's a bit of a riddle: how do you wait when you can't wait?

## Solution

You want to use the [Time package](https://stdlib.ponylang.io/time--index). In particular, the [Timer](https://stdlib.ponylang.io/time-Timer/) and [Timers](https://stdlib.ponylang.io/time-Timers/) types.

A timer allows you to execute code at set intervals. Let's walk through using a timer. Below is a simple application that prints out a number to the console every 5 seconds until someone terminates the program:

```pony
use "time"

actor Main
  new create(env: Env) =>
    let timers = Timers
    let timer = Timer(NumberGenerator(env), 0, 5_000_000_000)
    timers(consume timer)

class NumberGenerator is TimerNotify
  let _env: Env
  var _counter: U64

  new iso create(env: Env) =>
    _counter = 0
    _env = env

  fun ref _next(): String =>
    _counter = _counter + 1
    _counter.string()

  fun ref apply(timer: Timer, count: U64): Bool =>
    _env.out.print(_next())
    true
```

Zooming in on the key bits, we first set up our timers, create one and add it to our set of timers:

```pony
    let timers = Timers
    let timer = Timer(NumberGenerator(env), 0, 5_000_000_000)
    timers(consume timer)
```

The Timer constructor takes 3 arguments, the class to notify, how long until our timer expires and how often to fire. In our example code, an instance of NumberGenerator will be called every 5 billion nanoseconds i.e. every 5 seconds until the program is killed.

Here's our method in NumberGenerator that gets executed:

```pony
  fun ref apply(timer: Timer, count: U64): Bool =>
    _env.out.print(_next())
    true
```

If we were to compile and run our application, we'd end up with some output
like:

```bash
$ ./timer
1
2
3
4
5
6
```

## Discussion

It's not the most exciting output in the world but, it's a pattern that can be adapted to many different scenarios. `Timer` can be put to use for rate limiting outgoing network connections, creating buffers that flush at a set interval, implementing timeouts and variety of other time based _blocking_ operations.

---

This pattern is based on a [blog post](http://www.monkeysnatchbanana.com/2016/01/16/pony-patterns-waiting/) previously published by Sean T. Allen.
