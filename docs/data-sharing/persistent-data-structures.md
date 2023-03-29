---
hide:
  - toc
---

# Persistent Data Structures

## Problem

You need to send mutable data from one actor to another while keeping a copy of it in your original actor.

## Solution

```pony
use "collections/persistent"

actor Collector
  """
  Receives characters via it's `collect` behavior and stores them.
  Every 10 characters we receive results in the entire array being sent all to
  the receiver.
  """
  let _receiver: Receiver
  var _data: Vec[U8] = Vec[U8]

  new create(receiver: Receiver) =>
    _receiver = receiver

  be collect(char: U8) =>
    _data = _data.push(char)

  be send(to: Receiver) =>
    to.receive(_data)

actor Receiver
  """
  Receives an array of characters from a collector
  """
  be receive(data: Vec[U8]) =>
    // do something with `data`
    None
```

## Discussion

That's a pretty simple looking solution, especially when you compare it to the [copying pattern](copying.md), which is another way to solve this problem. So what's going on here?

The key is that our `_data` vector isn't mutable. In fact, `var _data: Vec[U8] = Vec[U8]` is creating a `val`. So, we aren't dealing with mutable data; we are creating a new immutable vector each time we "mutate" it. That's why we need to assign to `_data` in the `collect` method:

```pony
  be collect(char: U8) =>
    _data = _data.push(char)
```

And it's also why we defined `_data` using a `var` rather than a `let` binding. Each time we update, we are creating a new vector and binding our `_data` variable to the new vector.

Because `_data` is a `val`, it's already safe to share between actors. We don't need to do anything special with it when we want to send it to another actor:

```pony
  be send(to: Receiver) =>
    to.receive(_data)
```

Persistent data structures go hand in hand with the [copying pattern](copying.md) as a means of sharing mutable data between actors. Each method involves copying data. Which pattern should you pick? Whichever one will minimize the number of copies.

Persistent data structures involve copying on each update. The copying pattern involves a copy each time we send the data to another actor. As a general rule of thumb, figure out which you will do more: update or send. If you are updating more, use the copying pattern. If you are sending more, use persistent data structures.

In the end, that's just a rule of thumb. Your best bet is to benchmark and pick the method that gives you the best performance for your use case.

If you aren't familiar with persistent data structures, we suggest you pick up a copy of the book [Purely Functional Data Structures](https://www.thriftbooks.com/w/purely-functional-data-structures_chris-okasaki/648821/item/4430756). Another option is to download a copy of [the thesis](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.64.3080&rep=rep1&type=pdf) upon which the book is based. The Pony [standard library](https://stdlib.ponylang.io/collections-persistent--index/) contains a few, but you may need to design your own.

If you are interested in learning more about the persistent [`Vec`](https://stdlib.ponylang.io/collections-persistent-Vec/) and [`HashMap`](https://stdlib.ponylang.io/collections-persistent-HashMap/) data structures from the Pony standard library, you can check out:

- [Understanding Clojure's Persistent Vectors Pt 1](https://hypirion.com/musings/understanding-persistent-vector-pt-1)
- [Persistent Vector Performance Summarised](https://hypirion.com/musings/persistent-vector-performance-summarised)
