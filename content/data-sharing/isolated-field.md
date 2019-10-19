---
title: "Isolated field"
section: "Data Sharing"
menu:
  toc:
    parent: "data-sharing"
    weight: 30
---

## Problem

You have a mutable data structure that you are building up over time in an actor and eventually need to send it to another actor. You could use the [copying pattern](/data-sharing/copying.html). However, the copying pattern is not without issue.

The problem with the copying pattern is that you are... copying. Copying large data structures isn't cheap. Even with small data structures, copying will result in many allocations and [avoiding allocations](https://www.ponylang.io/reference/pony-performance-cheatsheet/#avoid-allocations) is one of the critical pieces of advice in the [Pony Performance Cheatsheet](https://www.ponylang.io/reference/pony-performance-cheatsheet/).

If your use case meets one critical criterion, you can avoid copying. That criteria? You can "give away" the mutable data rather than having hold on to a reference so you can continue to update your copy later.

If that sounds like your problem, then welcome to your solution: "the isolated field."

## Solution

Let's take a look at an example of the isolated field pattern. Pay particular attention to the `_data` field on the `Collector` actor. `_data` is the variable that we want to share between actors.

```pony
use "collections"

actor Collector
  """
  Receives characters via it's `collect` behavior and stores them.
  Once our collector receives 10 characters, it sends all 10 to the receiver.
  """
  let _receiver: Receiver
  var _data: Array[U8] iso

  new create(receiver: Receiver) =>
    _receiver = receiver
    _data = recover Array[U8] end

  be collect(char: U8) =>
    _data.push(char)

    if _data.size() == 10 then
      let to_send = _data = recover Array[U8] end
      _receiver.receive(consume to_send)
    end

actor Receiver
  """
  Receives an array of characters from a collector and prints them as a string
  to standard out.
  """
  let _out: OutStream

  new create(out: OutStream) =>
    _out = out

  be receive(data: Array[U8] iso) =>
    let s = String.from_array(consume data)
    _out.print(s)
```

## Discussion

The isolated field pattern combines a couple of features:

- The ability to [rebind a variable](https://tutorial.ponylang.io/expressions/variables.html#var-vs-let) by declaring it a `var`
- [Destructive read](https://tutorial.ponylang.io/reference-capabilities/consume-and-destructive-read.html)

Let's zoom in on the one key line from our example:

```pony
let to_send = _data = recover Array[U8] end
```

What's going on with that? If you are new to Pony, that might be a very confusing bit of code. What you are looking at is a destructive read. What happens with the expression is evaluated? Well...

- `_data` is rebound to a new empty `Array[U8] iso`
- the previous value of `_data` is assigned to `to_send`

Did you follow that? When the expression is done, we are left with 2 `Array[U8] iso's in scope:

- The local variable `to_send` which is an isolated array of 10 characters
- Our actor field `_data` that has been reinitialized to an empty array.

We are now free to take our mutable data that we've been collecting and send it along to the waiting `Receiver` actor:

```pony
_receiver.receive(consume to_send)
```
