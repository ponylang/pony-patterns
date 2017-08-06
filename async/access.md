# Accessing an Actor with Arbitrary Transactions

## Problem

With asynchronous APIs on actors, we have the problem of not being able to combine multiple asynchronous calls to form an atomic operation, as messages to an actor may be interleaved. This means that any operation that isn't exposed as a single behaviour on the actor cannot be done in an atomic way, which puts a pressure on our API to provide "ancillary" behaviours that are just combinations and compositions of the "fundamental" behaviours. We'd prefer to solve this problem in a way that alleviated that pressure without creating a larger API surface to maintain, and without having to do the constant guesswork of imagining which combinations and compositions will be needed.

Let's say we want to create a simple set of named registers which can each hold an integer value, and that we want to be able to share read/write access to these registers among multiple actors in our application. So, we declare an actor type named `SharedRegisters` which holds a map of string register names to integer values and provides access to this register data via `read` and `write` behaviours.

```pony
use collections = "collections"

actor SharedRegisters
  let _data: collections.Map[String, I64] = _data.create()

  be write(name: String, value: I64) =>
    """
    Write to the named register, setting its value to the given value.
    """
    _data(name) = value

  be read(name: String, fn: {(I64)} val) =>
    """
    Read the value of the named register and pass it to the given function.
    A register which has never been written to will have a value of zero.
    """
    fn(try _data(name)? else 0 end)
```

We can then write a basic program that writes to a few registers and reads from them.

```pony
actor Main
  new create(env: Env) =>
    let reg = SharedRegisters
    let out = env.out

    reg.write("x", 99)
    reg.write("y", 100)

    reg.read("x", {(value: I64)(out) =>
      out.print("The value of x is " + value.string())
    } val)

    reg.read("y", {(value: I64)(out) =>
      out.print("The value of y is " + value.string())
    } val)
```

Because of Pony's causal messaging, we can expect this program to perform those operations in the written order, printing the following:

```
The value of x is 99
The value of y is 100
```

Now, let's say we want to do some higher-level operations on these registers, and we want to do them from multiple independent actors. To demonstrate a basic example, we'll declare a `Mathematician` actor with an `increment` behaviour that does a read operation followed by a write operation.

```pony
actor Mathematician
  let _reg: SharedRegisters
  let _out: StdStream
  new create(reg: SharedRegisters, out: StdStream) =>
    _reg = reg
    _out = out

  be increment(name: String) =>
    """
    Read the value of the named register, then write the incremented value.
    """
    let reg = _reg
    let out = _out

    reg.read(name, {(value: I64)(reg, out, name) =>
      let new_value = value + 1
      reg.write(name, new_value)
      out.print("Incremented " + name + " to " + new_value.string())
    } val)
```

However, there's a problem with this in the context of concurrent access to the same `SharedRegisters`. What if some other write operation changed the value of the register between the read operation and write operation that make up our `increment`? We can try to experiment with this situation by creating many `Mathematician`s and having them concurrently `increment` the same register.

```pony
use collections = "collections"

actor Main
  new create(env: Env) =>
    let reg = SharedRegisters
    let out = env.out

    reg.write("x", 99)

    for i in collections.Range(0, 10) do
      Mathematician(reg, out).increment("x")
    end
```

Sure enough, when we run this program, we see that our `increment` will only sometimes increase the observed value, with the results varying based on how the concurrent operations happen to line up in any given execution.

```
Incremented x to 100
Incremented x to 101
Incremented x to 101
Incremented x to 102
Incremented x to 103
Incremented x to 103
Incremented x to 104
Incremented x to 104
Incremented x to 104
Incremented x to 104
```

In Pony, an actor's behaviour acts like a single atomic transaction over the actor's internal state - it cannot be interrupted with other behaviours. However, when a conceptual operation spans multiple behaviours (like a `read` followed by a `write`, we can get into trouble because those transactions can be interleaved with others.

We could try defining `increment` as a behaviour of the `SharedRegisters` actor, such that it would be one atomic transaction. However, we'd soon find ourselves wanting to do other multi-operation transactions, and we might find our `SharedRegisters` actor to be getting a bit bloated if we added all of them as behaviours.

Furthermore, if the `SharedRegisters` were part of a library package maintained separately from the applications using it, the maintainer would be hard-pressed to imagine all of the multi-operation transactions that might be useful to the application developers. A more general solution that doesn't require the library developer to worry about this kind of guesswork would be desirable.

## Solution

Essentially, we want to provide a way for the caller to define a custom transaction, then execute it atomically within a single behaviour of the actor. This part is simple enough - the caller can pass a lambda, and the actor can execute it directly, just as we did before with the `read` behaviour when we passed the value to the caller's function.

However, unlike in the `read` behaviour, we also need the transaction lambda passed to the `access` behaviour to have exclusive synchronous access to perform arbitrary combinations of operations on the actor's state. In other words, we need the lambda to see the actor *as the actor sees itself*, instead of how it is seen from the outside.

Luckily, Pony has exactly the concepts we need to implement this idea. An actor is always seen from the outside as a `tag` (an opaque reference that you can only send messages to), but an actor is seen from the inside as a `ref` (by default, behaviours have read and write access to the actor's state). Because the transaction lambda will be executed inside the actor, we can pass the non-sendable `ref` reference to the actor itself as the argument, giving the transaction exclusive synchronous read/write access through that reference.

Let's take a look at a reworked implementation of `SharedRegisters` that includes an `access` behaviour as well as some synchronous versions of the `read` and `write` behaviours for use in the custom transactions.

```pony
actor SharedRegisters
  let _data: collections.Map[String, I64] = _data.create()

  be access(fn: {(SharedRegisters ref)} val) =>
    fn(this)

  be write(name: String, value: I64) =>
    write_now(name, value)

  be read(name: String, fn: {(I64)} val) =>
    fn(read_now(name))

  fun ref write_now(name: String, value: I64) =>
    _data(name) = value

  fun ref read_now(name: String): I64 =>
    try _data(name)? else 0 end
```

Note that the safety of access to the synchronous methods is guaranteed and protected by the Pony type system - any non-exclusive use of the synchronous methods is prevented, simply by keeping the `ref` from leaking outside of the actor. Note also that in this paradigm, the `read` and `write` behaviours are just asynchronous wrappers for their synchronous counterparts, `read_now` and `write_now`.

Now let's take a look at a revised implementation of `Mathematician` that uses the `access` behaviour to combine these synchronous operations into one atomic `increment` transaction:

```pony
actor Mathematician
  let _reg: SharedRegisters
  let _out: StdStream
  new create(reg: SharedRegisters, out: StdStream) =>
    _reg = reg
    _out = out

  be increment(name: String) =>
    """
    Read the value of the named register, then write the incremented value.
    """
    let reg = _reg
    let out = _out

    reg.access({(reg: SharedRegisters ref)(out, name) =>
      let new_value = reg.read_now(name) + 1
      reg.write_now(name, new_value)
      out.print("Incremented " + name + " to " + new_value.string())
    } val)
```

Sure enough, our example program using 10 concurrent `Mathematician`s to increment the same register will now give the expected output, with each `increment` transaction guaranteed to be atomic.

```
Incremented x to 100
Incremented x to 101
Incremented x to 102
Incremented x to 103
Incremented x to 104
Incremented x to 105
Incremented x to 106
Incremented x to 107
Incremented x to 108
Incremented x to 109
```

## Discussion

Using the "access pattern" in Pony, we can create actors that provide not only asynchronous APIs but also synchronous APIs, which can be used together in transactions that are defined and passed in by the caller. This dramatically enhances the realm of possible interactions with a service actor, whose synchronous API can provide the fundamental building blocks from which the caller can build arbitrarily complex atomic transactions.

The accessed actor can control the scope of what is possible in a transaction by controlling the reference capability of the reference that is passed to the transaction lambda. For example, we could choose to pass `box` instead of a `ref` if we wanted to provide read-only access to the transaction.

Even though we said that the transaction lambda is seeing the actor "as it sees itself", the accessed actor can still hide implementation details in the same way as any other class - by making those fields and methods private. The private fields and methods will not be accessible from within the transaction, so the implementation details can remain protected and hidden from the API surface.
