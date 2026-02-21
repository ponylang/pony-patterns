---
hide:
  - toc
---

# Error as Union Type

## Problem

Pony's built-in `error` mechanism is untyped — a partial function either succeeds or raises `error`, and the caller's `else` block has no way to know *what* went wrong. When a function can fail for multiple distinct reasons, partial functions force you into workarounds: setting error state on the object before raising, or collapsing all failures into a single `error` and losing the distinction.

Consider a function that sends data over a connection. Sending can fail because the connection isn't established yet, or because the socket is under backpressure and can't accept writes. With a partial function, the caller can't tell these apart:

```pony
class Connection
  var _connected: Bool = false
  var _writeable: Bool = false

  fun ref send(data: Array[U8] val): USize ? =>
    if not _connected then error end
    if not _writeable then error end
    // actual send logic
    data.size()
```

The caller's `try`/`else` just sees `error`:

```pony
try
  let sent = conn.send(data)?
  env.out.print("Sent " + sent.string() + " bytes")
else
  // Not connected? Backpressure? We can't tell.
  env.out.print("Send failed")
end
```

Was the connection not established? Was the socket full? The caller has no way to find out without inspecting out-of-band state on the object.

## Solution

Define a primitive for each distinct error condition, group them into a union type alias, and return the union from the function. Callers pattern match on the result to handle each case.

```pony
primitive SendErrorNotConnected
  """
  The connection is not yet established or has already been closed.
  """

primitive SendErrorNotWriteable
  """
  The socket is not writeable — a previous send is still pending or
  the send buffer is full. Wait for the connection to become writeable
  before retrying.
  """

type SendError is (SendErrorNotConnected | SendErrorNotWriteable)

actor Connection
  var _connected: Bool = false
  var _writeable: Bool = false

  be connect() =>
    _connected = true
    _writeable = true

  be send(data: Array[U8] val, out: OutStream) =>
    match _do_send(data)
    | let sent: USize => out.print("Sent " + sent.string() + " bytes")
    | SendErrorNotConnected => out.print("Error: not connected")
    | SendErrorNotWriteable => out.print("Error: backpressure active")
    end

  fun ref _do_send(data: Array[U8] val): (USize | SendError) =>
    if not _connected then
      return SendErrorNotConnected
    end

    if not _writeable then
      return SendErrorNotWriteable
    end

    // actual send logic would go here
    data.size()

actor Main
  new create(env: Env) =>
    let conn = Connection
    conn.connect()
    conn.send("hello".array(), env.out)
```

Each error condition is a named primitive with a docstring explaining when it occurs. The `SendError` type alias groups them into a single type for use in return signatures. The function returns `(USize | SendError)` — either the number of bytes sent or a specific error. The caller's `match` handles each case, and the compiler verifies that every variant is covered.

## Discussion

### Why primitives

Primitives are singleton values that exist for the lifetime of the program. They're never allocated, never garbage collected, and carry no data — they're just globally unique labels. That makes them ideal for error conditions that don't need to carry information beyond their identity.

The key advantage over partial functions is that the error vocabulary is visible in the type. When you match on a `(USize | SendError)`, the compiler knows every possible variant. If you later add a third error condition to `SendError`, any `match` whose result is used in a typed context — assigned to a variable, returned from a function — will fail to compile unless it handles the new variant. With partial functions, adding a new failure mode is invisible to callers — their `else` blocks silently absorb it.

### Data-carrying errors

When an error needs to carry information — an exit code, a signal number, an offset into a buffer — use a class instead of a primitive. The pattern works the same way; the only difference is that the `match` arm binds a variable to access the data.

The standard library's `process` package uses this for process exit status:

```pony
class val Exited
  let exit_code: I32
  new val create(code: I32) => exit_code = code

class val Signaled
  let signal: U32
  new val create(sig: U32) => signal = sig

type ProcessExitStatus is (Exited | Signaled)
```

A process either exited normally (with a code) or was killed by a signal (with a signal number). Callers match on the result and extract the relevant data:

```pony
match status
| let e: Exited => env.out.print("exit code: " + e.exit_code.string())
| let s: Signaled => env.out.print("signal: " + s.signal.string())
end
```

The type alias and pattern matching work identically to the primitive case. Choose primitives when the error is just a label; choose classes when it needs to carry context.

### Related patterns and real-world usage

The [Static Constructor](../creation/static-constructor.md) pattern applies union-type returns to object construction — the factory function returns either the constructed object or an error describing why construction failed.

This pattern appears throughout the Pony ecosystem. The [ponylang/lori](https://github.com/ponylang/lori) networking library defines `SendError` as a union of `SendErrorNotConnected` and `SendErrorNotWriteable`. The [ponylang/postgres](https://github.com/ponylang/postgres) driver uses `ClientQueryError` to distinguish query failures. The standard library's `files` package uses `FileErrNo` to represent OS-level file errors.
