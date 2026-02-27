---
hide:
  - toc
---

# Embed and Delegate

## Problem

You're building a library that manages something with complex, stateful protocol logic. Think connection management: tracking connection state, accumulating incoming data, notifying callers of events. You need actors for concurrency, so you put the logic directly in the actor:

```pony
actor EchoServer
  let _env: Env
  var _connected: Bool = false
  var _bytes_received: USize = 0

  new create(env: Env) =>
    _env = env

  be connected() =>
    _connected = true
    _env.out.print("Connected")

  be received(data: Array[U8] val) =>
    if _connected then
      _bytes_received = _bytes_received + data.size()
      _env.out.print("Echoing " + data.size().string() + " bytes")
      // ... echo data back to client
    end

  be closed() =>
    _env.out.print(
      "Disconnected after " + _bytes_received.string() + " bytes")
    _connected = false
    _bytes_received = 0
```

For a small example like this, inline logic is fine. But real connection handlers run to hundreds or even thousands of lines of buffering, flow control, and error handling. At that scale, two problems emerge. First, the logic can't be reused. If you want a chat server that manages connections the same way but processes messages differently, you have to duplicate all the connection management code into a second actor. Second, the logic can't be tested independently. Actor behaviors are asynchronous, so testing internal state transitions means wrestling with the actor runtime instead of writing straightforward synchronous assertions.

The [Mixin](mixin.md) pattern gets you partway there. You can share stateful logic through trait default methods that access state via abstract getters and setters. But when the logic is large, the ceremony of abstract getters and setters for every piece of state becomes unwieldy. What you really want is to put the logic in its own class.

## Solution

The idea is to separate protocol logic into a `class ref` that lives inside the actor. The actor becomes a thin shell that receives behavior calls and forwards them to the class. The class holds all the state, does all the work, and calls back to the actor when something noteworthy happens. This gives you reuse (multiple actors can embed the same class with different callback implementations) and testability (the class can be instantiated and tested synchronously without the actor runtime).

Getting there takes a few steps because of how Pony handles object construction.

### The `this` wrinkle

The intuitive first attempt is to pull the logic into a class and pass `this` to its constructor so the class has a reference back to the actor:

```pony
class Connection
  var _owner: EchoServer ref

  new create(owner: EchoServer ref) =>
    _owner = owner
    // ... connection management logic

actor EchoServer
  let _conn: Connection

  new create(env: Env) =>
    _conn = Connection(this)  // won't compile
```

This doesn't compile. In Pony, `this` isn't `ref` until all fields are initialized, but `_conn` is one of those fields. We're stuck in a chicken-and-egg loop: we need `this` to create the `Connection`, but `this` isn't ready until the `Connection` is created.

### The `none()` constructor

The way out is a placeholder constructor. Give the class a lightweight `none()` constructor that produces a minimal instance, and use it as the field's default value:

```pony
class Connection
  var _owner: (EchoServer ref | None)

  new none() =>
    _owner = None

  new create(owner: EchoServer ref) =>
    _owner = owner

actor EchoServer
  var _conn: Connection = Connection.none()
  let _env: Env

  new create(env: Env) =>
    _env = env
    _conn = Connection(this)  // this is ref now
```

With `Connection.none()` as the default, all fields have values before the constructor body runs. Now `this` is `ref`, and we can pass it to `Connection.create` to build the real instance. The field has to be `var` instead of `let` because we're reassigning it in the constructor body.

The `_owner` field becomes a union type `(EchoServer ref | None)` because the placeholder instance has no actor to reference. In practice, the `none()` instance is never used for real processing. It exists only to satisfy field initialization.

### The delegation trait

The actor still manually forwards each behavior to the class. If multiple actor types want to use the same protocol logic, each one has to write the same forwarding code. A trait with default behavior implementations handles that:

```pony
trait ConnectionActor
  fun ref _connection(): Connection

  be connected() =>
    _connection().connected()

  be received(data: Array[U8] val) =>
    _connection().received(data)

  be closed() =>
    _connection().closed()
```

Any actor that implements `ConnectionActor` just needs to provide `_connection()` returning its stored `Connection` instance, and it gets all the behaviors for free. This is the [Mixin](mixin.md) pattern applied to the forwarding layer: the trait provides default behavior implementations, and the actor provides the state they operate on through an abstract accessor.

### The callback direction

The delegation trait handles messages flowing into the class, but the class also needs to talk back to the actor. When the class detects something noteworthy (a state change, a threshold crossed), it needs to notify the actor so the actor can take application-specific action. Right now the class stores a concrete `EchoServer ref`, which ties it to a single actor type.

A callback trait fixes both problems. It defines the notifications the class can send, and any actor can implement it:

```pony
trait ConnectionNotify
  fun ref on_connected()
  fun ref on_received(data: Array[U8] val)
  fun ref on_closed(bytes_received: USize)
```

The class stores a `ConnectionNotify ref` instead of a concrete actor type. The concrete actor implements `ConnectionNotify` to handle the callbacks. By having `ConnectionActor` extend `ConnectionNotify`, a single `is ConnectionActor` on the actor declaration pulls in both the inbound behaviors and the outbound callback obligations.

Here's how the pieces fit together. A caller sends a behavior to `EchoServer`, the `ConnectionActor` trait forwards it to the `Connection` class, and `Connection` calls back to the actor through `ConnectionNotify`:

``` mermaid
sequenceDiagram
  autonumber
  Caller->>EchoServer: connected()
  EchoServer->>Connection: connected()
  Connection->>EchoServer: on_connected()
```

### Complete program

Here's everything together. `Connection` holds the protocol state and logic, `ConnectionNotify` defines how the class talks back to the actor, `ConnectionActor` provides the behavior forwarding, and `EchoServer` ties them together:

```pony
trait ConnectionNotify
  fun ref on_connected()
  fun ref on_received(data: Array[U8] val)
  fun ref on_closed(bytes_received: USize)

class Connection
  var _connected: Bool = false
  var _bytes_received: USize = 0
  var _notify: (ConnectionNotify ref | None)

  new none() =>
    _notify = None

  new create(notify: ConnectionNotify ref) =>
    _notify = notify

  fun ref connected() =>
    _connected = true
    match _notify
    | let n: ConnectionNotify ref => n.on_connected()
    end

  fun ref received(data: Array[U8] val) =>
    if _connected then
      _bytes_received = _bytes_received + data.size()
      match _notify
      | let n: ConnectionNotify ref => n.on_received(data)
      end
    end

  fun ref closed() =>
    let bytes = _bytes_received
    _connected = false
    _bytes_received = 0
    match _notify
    | let n: ConnectionNotify ref => n.on_closed(bytes)
    end

trait ConnectionActor is ConnectionNotify
  fun ref _connection(): Connection

  be connected() =>
    _connection().connected()

  be received(data: Array[U8] val) =>
    _connection().received(data)

  be closed() =>
    _connection().closed()

actor EchoServer is ConnectionActor
  var _conn: Connection = Connection.none()
  let _env: Env

  new create(env: Env) =>
    _env = env
    _conn = Connection(this)

  fun ref _connection(): Connection =>
    _conn

  fun ref on_connected() =>
    _env.out.print("Client connected")

  fun ref on_received(data: Array[U8] val) =>
    _env.out.print("Echoing " + data.size().string() + " bytes")

  fun ref on_closed(bytes_received: USize) =>
    _env.out.print(
      "Disconnected after " + bytes_received.string() + " bytes")

actor Main
  new create(env: Env) =>
    let server = EchoServer(env)
    server.connected()
    server.received(recover [as U8: 1; 2; 3; 4; 5] end)
    server.received(recover [as U8: 6; 7; 8] end)
    server.closed()
```

Look at how little `EchoServer` actually does. It stores the connection, provides access to it, and implements three callback methods. All the protocol logic (tracking state, counting bytes, deciding when to notify) lives in `Connection`. A `ChatServer` or `ProxyServer` would look almost identical: same `Connection` class, same `ConnectionActor` trait, different callback implementations.

## Discussion

This pattern sits between the [Notifier](notifier.md) and [Mixin](mixin.md) patterns in weight. The Notifier pattern is lighter: a callback object is passed into a class or actor, and the framework calls its methods at key points. But a notifier can only be called by its owner; it can't receive messages from other actors. The Mixin pattern shares stateful logic through trait default methods, which works well for moderate amounts of state but becomes cumbersome when the logic grows large. Embed and Delegate takes the Mixin idea to its logical conclusion: instead of spreading logic across trait default methods with abstract getters and setters for every field, consolidate everything into a dedicated class. The delegation trait is still a mixin in spirit, but it's a thin one that just forwards calls to the class where the real work happens.

Despite the pattern's name, Pony's `embed` keyword can't be used here. An `embed` field is inlined into the containing object and can't be reassigned. The `none()` trick requires a `var` field because we assign the placeholder first and then the real instance in the constructor body. The "embed" in the name refers to the conceptual idea (the class lives inside the actor and is never exposed to the outside), not the Pony keyword.

You might have noticed the `match _notify` in every callback within `Connection`. That's the cost of the `none()` constructor: the notify field is a union type because the placeholder instance has no actor to reference. In practice, the `none()` instance is never used for real processing, so the `None` branch never fires at runtime. The match is a type-system ceremony, not a meaningful code path.

The real payoff shows up in production libraries. [Lori](https://github.com/ponylang/lori), the Pony networking library, uses exactly this pattern. Its `TCPConnection` class runs to roughly 1300 lines of connection management logic: socket setup, read/write buffering, backpressure, error handling. The `TCPConnectionActor` trait is about 40 lines of behavior forwarding. A concrete echo server actor built on top of lori is around 17 lines. All the complexity lives in the class; the actor just plugs in its application-specific callbacks. [Stallion](https://github.com/ponylang/stallion), a Pony HTTP server, layers on top of lori: its `HTTPServer` wraps lori's TCP connection class, demonstrating that the pattern composes across library boundaries.

Because the logic lives in a `class ref` rather than an actor, it can be instantiated and tested synchronously. You don't need the actor runtime to verify that your protocol handler correctly tracks state transitions or counts bytes. Create the class, call its methods, check the results. Different actors can embed the same class and provide different callback implementations, giving you reuse without duplication. That combination of testability and reuse is what makes this pattern foundational for anyone building I/O libraries in Pony.
