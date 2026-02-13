---
hide:
  - toc
---

# State Machine

## Problem

Your actor has distinct lifecycle phases, and the set of valid operations changes between them. A common first approach is to track the phase with a flag and check it in every behavior.

Consider an actor that accepts data while open and should stop accepting writes after being closed. Our first pass might look like:

```pony
actor Writer
  var _out: (OutStream | None)
  var _open: Bool = true

  new create(out: OutStream) =>
    _out = out

  be write(data: String) =>
    if _open then
      match _out
      | let o: OutStream => o.print(data)
      end
    end

  be close() =>
    _open = false
    _out = None
```

This works, but there are a couple of problems lurking.

The `_out` field is declared as `(OutStream | None)` so we can release it on close. That means every behavior that touches `_out` has to match on it, even though we know it's always a valid `OutStream` when the writer is open. The compiler doesn't know that — as far as it's concerned, `_out` could be `None` at any time.

Every behavior also needs the `if _open` guard. With two behaviors that's manageable. With five or ten, the same check is scattered everywhere. The flag and the optional type are related — `_open` being `true` implies `_out` is an `OutStream` — but nothing in the type system connects them. Forget a guard in one behavior and you've got a bug that the compiler won't catch.

## Solution

Rather than track the phase with a flag, represent each phase as its own type. Define an interface for the operations the actor supports, and have each phase implement the behavior that's valid for it.

```pony
interface _WriterState
  fun write(w: Writer ref, data: String)
  fun close(w: Writer ref)

class _OpenWriter is _WriterState
  let _out: OutStream

  new create(out: OutStream) =>
    _out = out

  fun write(w: Writer ref, data: String) =>
    _out.print(data)

  fun close(w: Writer ref) =>
    w.state = _ClosedWriter

class _ClosedWriter is _WriterState
  fun write(w: Writer ref, data: String) =>
    None

  fun close(w: Writer ref) =>
    None

actor Writer
  var state: _WriterState

  new create(out: OutStream) =>
    state = _OpenWriter(out)

  be write(data: String) =>
    state.write(this, data)

  be close() =>
    state.close(this)

actor Main
  new create(env: Env) =>
    let w = Writer(env.out)
    w.write("hello")
    w.write("world")
    w.close()
    w.write("this will be silently dropped")
```

Let's walk through what changed.

```pony
interface _WriterState
  fun write(w: Writer ref, data: String)
  fun close(w: Writer ref)
```

We define an interface with every operation the actor supports. Each method takes the actor as a `ref` so that the state object can modify the actor's fields — including replacing itself with a different state.

```pony
class _OpenWriter is _WriterState
  let _out: OutStream
```

The open state holds `let _out: OutStream`. No `None`, no matching. If we're in this state, we have a valid output stream. That's what it means to be open.

```pony
class _ClosedWriter is _WriterState
  fun write(w: Writer ref, data: String) =>
    None
```

The closed state holds nothing. It doesn't need an output stream — it has nothing to write to. Writes are silently dropped.

```pony
  fun close(w: Writer ref) =>
    w.state = _ClosedWriter
```

State transitions happen by assigning a new state object to `w.state`. When `_OpenWriter.close` fires, it replaces itself with a `_ClosedWriter`. The output stream goes out of scope along with the old state — no need to manually set anything to `None`.

```pony
actor Writer
  var state: _WriterState

  be write(data: String) =>
    state.write(this, data)
```

The actor itself becomes a thin shell. Each behavior delegates to the current state, passing `this` so the state can trigger transitions. No flags, no matching on optional types.

## Discussion

This pattern is driven by two ideas.

Each phase is a distinct type carrying exactly the data it needs. `_OpenWriter` holds the output stream; `_ClosedWriter` doesn't. There's no `(OutStream | None)` — the data that exists in a phase is defined by the type that represents that phase.

Invalid operations get isolated implementations. When you `write` to a closed writer, you hit `_ClosedWriter.write` — a no-op. You could just as easily have it log a warning, notify the caller, or panic. The point is that each operation's behavior in each phase lives in its own method on its own type, not tangled up with flag checks.

With just two states, this might feel like a lot of ceremony. The payoff comes when states multiply. Each new state is a new class implementing the interface — you don't touch existing states or add branches to existing methods. When the number of states grows, you can introduce traits with default implementations for groups of operations, so new states only need to override the methods that are meaningful for them.

This pattern also nests naturally. A state can hold its own sub-state machine for finer-grained lifecycle management within a single phase.

For a real-world example of this pattern at scale, see the [ponylang/postgres](https://github.com/ponylang/postgres) driver. Its `Session` actor uses a `_SessionState` interface with six concrete states spanning the full connection lifecycle, from unopened through SSL negotiation, authentication, and query processing. The logged-in state contains a nested `_QueryState` machine tracking query lifecycle. A trait hierarchy provides default implementations for invalid transitions, so each state class only needs to implement what's actually valid for it.
