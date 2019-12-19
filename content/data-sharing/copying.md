---
title: "Copying"
section: "Data Sharing"
menu:
  toc:
    parent: "data-sharing"
    weight: 10
---

## Problem

You need to send mutable data from one actor to another while keeping a copy of it in your original actor.

## Solution

```pony
use "collections"

actor Collector
  """
  Receives characters via it's `collect` behavior and stores them.
  Every 10 characters we receive results in the entire array being sent all to
  the receiver.
  """
  let _receiver: Receiver
  let _data: Array[U8] = Array[U8]

  new create(receiver: Receiver) =>
    _receiver = receiver

  be collect(char: U8) =>
    _data.push(char)

    if (_data.size() % 10) == 0 then
      let copy: Array[U8] iso = recover Array[U8] end

      for v in _data.values() do
        copy.push(v)
      end

      _receiver.receive(consume copy)
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
The critical section from our example is:

```pony
      let copy: Array[U8] iso = recover Array[U8] end

      for v in _data.values() do
        copy.push(v)
      end

      _receiver.receive(consume copy)
```

Let's walk through what we are doing. First, we create a new `iso` array that we will populate with the values from `_data` and then to our receiver. That our array is an `iso` is crucial. Because `copy` is isolated, the Pony compiler will make sure that we only ever have a single reference to it and can safely share the isolated copy.

```pony
      let copy: Array[U8] iso = recover Array[U8] end
```

Next, we  copy all the value from our mutable `ref` array into our `iso` array:

```pony
      for v in _data.values() do
        copy.push(v)
      end
```

Finally, we `consume` our reference to the `iso` `copy` array and send it to our receiver:

```pony
      _receiver.receive(consume copy)
```

## Discussion

The copying pattern goes hand in hand with the use of [persistent data structures to share data](persistent-data-structures.html) between actors. Each method involves copying data. Which pattern should you pick? Whichever one will minimize the number of copies.

Persistent data structures involve copying on each update. The copying pattern involves a copy each time we send the data to another actor. As a general rule of thumb, you should figure out which you will do more: update or send. If you are updating more, use the copying pattern. If you are sending more, use persistent data structures.

In the end, that's just a rule of thumb. Your best bet is to benchmark and pick the method that gives you the best performance for your use case. In our simple example, we are going to update our `_data` array 10 times for every one time we copy it to send. In this case, we are pretty sure that our rule of thumb would stand up to benchmarking.
