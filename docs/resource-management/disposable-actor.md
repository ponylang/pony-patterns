---
hide:
  - toc
---

# Disposable Actor

## Problem

Your Pony program has actors that hold resources: a TCP listener waiting for connections, a timer firing periodically, a process monitor watching a child. You want the program to shut down cleanly when it's done, but it just hangs:

```pony
use "net"

class MyListener is TCPListenNotify
  let _env: Env

  new iso create(env: Env) =>
    _env = env

  fun ref listening(listen: TCPListener ref) =>
    _env.out.print("Listening")

  fun ref not_listening(listen: TCPListener ref) =>
    _env.out.print("Failed to listen")

  fun ref connected(listen: TCPListener ref): TCPConnectionNotify iso^ =>
    object iso is TCPConnectionNotify
      fun ref received(conn: TCPConnection ref, data: Array[U8] iso,
        times: USize): Bool => true
      fun ref connect_failed(conn: TCPConnection ref) => None
    end

actor Main
  new create(env: Env) =>
    TCPListener(TCPListenAuth(env.root),
      recover MyListener(env) end, "localhost", "8989")
    env.out.print("Started")
```

This program prints "Started" and "Listening" and then sits there forever. It will never exit on its own. The Pony runtime shuts down when every actor has finished processing its messages and has no more work to do. But a TCP listener has subscribed to the runtime's ASIO event system, telling it "wake me up when a connection arrives." As far as the runtime is concerned, that actor always has pending work. The same is true for timers, UDP sockets, signal handlers, and anything else that subscribes to asynchronous events.

You need a way to tell these actors to release their resources so the program can exit. And when you have more than a handful of them, you need a way to do it without threading shutdown logic through every corner of your code.

## Solution

Pony's standard library gives you two pieces that solve this together: the `DisposableActor` interface and the `Custodian` actor.

`DisposableActor` lives in `builtin`, so it's available everywhere without a `use` statement. It's about as simple as an interface gets:

```pony
interface tag DisposableActor
  be dispose()
```

One behavior, no arguments, no return value. An actor that implements `dispose()` is making a promise: "call this and I'll clean up after myself." The `tag` capability means you can call `dispose()` on any reference to the actor, regardless of what capability you hold. That's the whole point: shutdown messages need to reach actors from anywhere.

Many standard library actors already implement this. `TCPListener.dispose()` stops listening and closes the socket. `TCPConnection.dispose()` finishes pending writes and closes the connection. `Timers.dispose()` cancels all pending timers and unsubscribes from the event system. You don't need to do anything special to use them with this pattern; they're ready to go.

For your own actors, implementing `dispose()` means deciding what "clean up" looks like. An actor managing a database connection pool might close all connections. An actor coordinating workers might tell each worker to stop. The specifics depend on what your actor owns:

```pony
actor Ticker
  let _timers: Timers

  new create(env: Env) =>
    _timers = Timers
    let t = Timer(object iso is TimerNotify
      let _env: Env = env
      fun ref apply(timer: Timer ref, count: U64): Bool =>
        _env.out.print("tick")
        true
    end, 1_000_000_000, 1_000_000_000)
    _timers(consume t)

  be dispose() =>
    _timers.dispose()
```

`Ticker` owns a `Timers` instance, so its `dispose()` disposes the timers. The chain propagates: `Timers.dispose()` cancels all pending timers and unsubscribes from ASIO events, which lets that actor become idle, which lets the runtime see one fewer actor with pending work.

Now, you could call `dispose()` on each actor yourself. With two or three actors, that's fine. But programs grow. You end up with a listener, several connections, a timer, maybe a process monitor. Threading individual `dispose()` calls through your shutdown path gets tedious and fragile. Miss one and your program hangs.

`Custodian` from the `bureaucracy` package solves this. It's an actor that keeps a set of `DisposableActor` references. When you dispose the custodian, it disposes everything in its set:

```pony
use "bureaucracy"

actor Main
  new create(env: Env) =>
    let custodian = Custodian

    let listener = TCPListener(...)
    let ticker = Ticker(env)

    custodian(listener)
    custodian(ticker)

    // Later, when it's time to shut down:
    custodian.dispose()
```

The `custodian(actor)` syntax works because `Custodian` implements `apply`. You register actors as you create them, and when shutdown time comes, one call to `custodian.dispose()` fans out to everything.

In practice, "shutdown time" is usually a signal. Here's a complete program that starts a TCP listener and a ticker, then shuts down cleanly when it receives SIGTERM:

```pony
use "bureaucracy"
use "net"
use "signals"
use "time"

class MyListener is TCPListenNotify
  let _env: Env

  new iso create(env: Env) =>
    _env = env

  fun ref listening(listen: TCPListener ref) =>
    _env.out.print("Listening on 8989")

  fun ref not_listening(listen: TCPListener ref) =>
    _env.out.print("Failed to listen")

  fun ref connected(listen: TCPListener ref): TCPConnectionNotify iso^ =>
    object iso is TCPConnectionNotify
      fun ref received(conn: TCPConnection ref, data: Array[U8] iso,
        times: USize): Bool => true
      fun ref connect_failed(conn: TCPConnection ref) => None
    end

class TermHandler is SignalNotify
  let _custodian: Custodian

  new iso create(custodian: Custodian) =>
    _custodian = custodian

  fun ref apply(count: U32): Bool =>
    _custodian.dispose()
    true

actor Ticker
  let _timers: Timers

  new create(env: Env) =>
    _timers = Timers
    let t = Timer(object iso is TimerNotify
      let _env: Env = env
      fun ref apply(timer: Timer ref, count: U64): Bool =>
        _env.out.print("tick")
        true
    end, 1_000_000_000, 1_000_000_000)
    _timers(consume t)

  be dispose() =>
    _timers.dispose()

actor Main
  new create(env: Env) =>
    let custodian = Custodian

    let listener = TCPListener(TCPListenAuth(env.root),
      recover MyListener(env) end, "localhost", "8989")
    let ticker = Ticker(env)

    custodian(listener)
    custodian(ticker)

    let signal = SignalHandler(recover TermHandler(custodian) end,
      Sig.term())
    custodian(signal)
```

When the process receives SIGTERM, `TermHandler.apply` fires, which disposes the custodian, which disposes the listener, the ticker, and the signal handler. Each of those actors releases its ASIO resources, and the runtime exits cleanly.

## Discussion

Pony uses structural typing, so actors don't need to explicitly declare `is DisposableActor`. If an actor has a `dispose()` behavior, it satisfies the interface automatically. That's why `TCPListener`, `TCPConnection`, `Timers`, and `ProcessMonitor` all work with `Custodian` even though none of them mention `DisposableActor` in their type declarations. You just pass them in and it works.

`Custodian` itself implements `dispose()`, which means it's a `DisposableActor` too. You can nest custodians. A subsystem might have its own custodian managing its internal actors, and you register that custodian with a top-level one. Disposing the top-level custodian cascades through the tree. This is useful for larger programs where different subsystems have independent lifecycles but you still want a single kill switch at the top.

`Custodian` also has a `remove` behavior for actors that shut down before the program does. If a TCP connection closes on its own, you can remove it from the custodian so it doesn't try to dispose an already-finished actor. Calling `dispose()` on an actor that's already cleaned up is usually harmless (it's good practice to make `dispose()` idempotent), but removing it keeps the custodian's set from growing unboundedly in long-running programs where connections come and go.

The order of disposal is not guaranteed. `Custodian` iterates its internal set, and `SetIs` doesn't promise any particular ordering. If your actors have dependencies (actor A should shut down before actor B), you need to handle that yourself, either by nesting custodians with explicit ordering or by having actor A dispose actor B as part of its own `dispose()` implementation. For most programs this doesn't matter; each actor just releases its own resources independently.

This pattern is the actor-level complement to [FFI Resource Lifecycle](ffi-resource-lifecycle.md), which handles cleanup for C library handles within a single class. FFI Resource Lifecycle uses `dispose()` plus `_final()` as a GC safety net for non-actor objects. Disposable Actor uses `dispose()` to coordinate shutdown across actors that hold ASIO subscriptions. In a real program you'll often use both: an actor implements `dispose()` (so it works with `Custodian`), and inside that actor, a class wraps a C handle with the sentinel-and-finalizer pattern. The two patterns layer naturally because they both speak the same `dispose()` protocol.
