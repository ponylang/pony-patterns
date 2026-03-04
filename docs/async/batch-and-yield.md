---
hide:
  - toc
---

# Batch and Yield

## Problem

You have an actor that needs to process a large collection. The straightforward approach is to loop through everything inside a single behavior:

```pony
actor Processor
  let _out: OutStream

  new create(out: OutStream) =>
    _out = out

  be process(data: Array[U64] val) =>
    for value in data.values() do
      // imagine expensive per-item work here
      _out.print(value.string())
    end
```

This works, but it has a hidden cost. In Pony, a behavior runs to completion before the scheduler will give the actor's thread to someone else. While `Processor` is grinding through a million items, it's tying up a scheduler thread. Other scheduler threads can steal work, so the system doesn't freeze, but it becomes a fairness issue. One actor is monopolizing a scheduler thread. The situation is worse if you're running with only one scheduler thread — no other work will happen until the behavior finishes.

The problem gets worse as the data grows. A batch of a hundred items is fine. A batch of a million items monopolizes a scheduler thread for the entire duration of the loop. If enough actors do this at the same time, the runtime's ability to schedule work fairly degrades significantly.

## Solution

The fix is to break the work into chunks and give the scheduler a chance to breathe between them. Instead of processing everything in one shot, handle a batch of items, then send yourself a message to continue where you left off. That self-message goes to the back of the actor's queue, creating a point where the scheduler can run other actors.

Here's a first attempt at the batching behavior:

```pony
use "collections"

actor Processor
  let _out: OutStream
  let _batch_size: USize

  new create(out: OutStream, batch_size: USize = 100) =>
    _out = out
    _batch_size = batch_size

  be process(data: Array[U64] val, from: USize = 0) =>
    let until = from + _batch_size
    try
      for i in Range(from, until.min(data.size())) do
        _out.print(data(i)?.string())
      end
    end

    if until < data.size() then
      process(data, until)
    end

actor Main
  new create(env: Env) =>
    let data = recover val
      let arr = Array[U64](1000)
      for i in Range[U64](0, 1000) do
        arr.push(i)
      end
      arr
    end

    Processor(env.out).process(data)
```

The behavior takes an offset (`from`) that defaults to zero so the initial call looks clean. It processes up to `_batch_size` items starting at that offset, then checks whether there's more work to do. If so, it calls `process` on itself with the new offset. That call doesn't execute immediately. It becomes a message in this actor's queue, behind any messages that arrived while the current batch was running.

This works, but there's a problem. The `process` behavior is public, and its `from` parameter is an internal detail of the batching mechanism. Any actor with a reference to `Processor` can call `process(data, 500)`, skipping the first 500 items or passing an arbitrary offset.

The fix is to split the work into a public behavior for the entry point, a private behavior for the batch-and-yield loop, and a shared function that does the actual processing:

```pony
use "collections"

actor Processor
  let _out: OutStream
  let _batch_size: USize

  new create(out: OutStream, batch_size: USize = 100) =>
    _out = out
    _batch_size = batch_size

  be process(data: Array[U64] val) =>
    _process_batch(data, 0)

  be _process_again(data: Array[U64] val, from: USize) =>
    _process_batch(data, from)

  fun ref _process_batch(data: Array[U64] val, from: USize) =>
    let until = from + _batch_size
    try
      for i in Range(from, until.min(data.size())) do
        _out.print(data(i)?.string())
      end
    end

    if until < data.size() then
      _process_again(data, until)
    end

actor Main
  new create(env: Env) =>
    let data = recover val
      let arr = Array[U64](1000)
      for i in Range[U64](0, 1000) do
        arr.push(i)
      end
      arr
    end

    Processor(env.out).process(data)
```

Now `process` is the only public entry point. The `_process_again` behavior and `_process_batch` function are private — external actors can't call them. The public `process` behavior and the private `_process_again` behavior both delegate to `_process_batch`, which contains the batching logic and the self-message to continue.

## Discussion

The reason this works comes down to how Pony schedules actors. Each scheduler thread picks an actor and runs behaviors from its queue up to a batch size that's set when the runtime is compiled. The scheduler won't interrupt a behavior mid-execution. By splitting work across multiple behavior calls, you're cooperating with the scheduler, giving it natural points to interleave other actors' work.

Batch size is a tradeoff. Smaller batches mean the actor yields more often, which improves fairness for other actors but adds overhead from the repeated message sends and behavior dispatches. Larger batches are more efficient and more cache-friendly, but hold the scheduler thread longer. There's no single right answer. A batch of 100 items is a reasonable starting point, but the right size depends on how expensive each item is to process. Cheap items (incrementing a counter) can use larger batches; expensive items (serializing complex structures, doing I/O) should use smaller ones.

One thing to be aware of: between batches, other messages sent to this actor will be processed. If another actor sends a second `process` call while the first is still working through its chunks, the two will interleave. Depending on your application, that might be perfectly fine or it might be a problem. If you need multi-step processing to be atomic (no interleaving), the [Access](access.md) pattern addresses exactly that concern. Batch and Yield and Access solve opposite problems: Access prevents interleaving when you don't want it; Batch and Yield deliberately introduces interleaving so the scheduler can share time fairly.

You'll see this same structure in Pony's networking layer. `TCPConnection` delivers incoming data to a `received` method that returns a `Bool`. Returning `true` tells the enclosing behavior to continue delivering data. Returning `false` causes the behavior to exit — it's literally Batch and Yield. The next read will come from a new behavior call, giving the scheduler a chance to run other actors in between.

The [Waiting](waiting.md) pattern tackles a related scheduling concern from the opposite direction. Waiting is about how to delay work when you can't block (using timers to schedule future actions). Batch and Yield is about yielding during work that's happening now. Both are about cooperating with Pony's non-blocking scheduler, just from different angles.
