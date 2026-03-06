---
hide:
  - toc
---

# Supervisor and Worker

## Problem

You need to farm out work to one or more actors and know when they're all done. Maybe you're processing a batch of files, running computations in parallel, or initializing a set of subsystems before proceeding. In a language with threads, you'd spawn them and join. In Pony, actors are asynchronous. You send a message and move on. There's no way to block and wait for a result.

If you just need to query existing actors for their current state, the [Interrogating Actors with Promises](actorpromise.md) pattern is a better fit. This pattern is for when you're creating actors specifically to perform work on your behalf, and you need to coordinate their lifecycle: dispatching tasks, tracking who's still working, and reacting when everyone finishes.

A first instinct might be to have the supervisor just fire off work and hope for the best:

```pony
actor Main
  new create(env: Env) =>
    for i in Range(0, 10) do
      Worker(env.out).work(i)
    end
    // ...now what? When are they done?
```

The workers will do their thing, but the supervisor has no idea when they finish. It can't print a summary, trigger a next phase, or shut down cleanly.

## Solution

The core idea is simple: workers report back to the supervisor when they're done. The supervisor keeps track of outstanding workers and acts when the last one checks in.

Let's start with the simplest case. One supervisor, one worker:

```pony
actor Worker
  let _supervisor: Supervisor

  new create(supervisor: Supervisor) =>
    _supervisor = supervisor

  be work() =>
    // do some stuff
    _supervisor.done(this)

actor Supervisor
  let _out: OutStream
  let _worker: Worker

  new create(out: OutStream) =>
    _out = out
    _worker = Worker(this)

  be run() =>
    _worker.work()

  be done(worker: Worker) =>
    _out.print("Worker finished")
```

The supervisor creates a worker, passing `this` so the worker has a way to call back. When the worker finishes, it calls `_supervisor.done(this)`, sending itself as a parameter so the supervisor knows who reported in.

That handles one worker, but the real value shows up when you have many. The supervisor needs to track which workers are still outstanding. Pony's `SetIs` (from the `collections` package) is perfect for this because it uses identity comparison, exactly what we want when tracking specific actor instances:

```pony
use "collections"

actor Worker
  let _supervisor: Supervisor

  new create(supervisor: Supervisor) =>
    _supervisor = supervisor

  be work(input: String val) =>
    // process input somehow
    _supervisor.done(this)

actor Supervisor
  let _out: OutStream
  let _pending: SetIs[Worker]

  new create(out: OutStream) =>
    _out = out
    _pending = SetIs[Worker]

  be run() =>
    let inputs = ["alpha"; "bravo"; "charlie"]
    for input in inputs.values() do
      let worker = Worker(this)
      _pending.set(worker)
      worker.work(input)
    end

  be done(worker: Worker) =>
    _pending.unset(worker)
    if _pending.size() == 0 then
      _out.print("All workers finished")
    end
```

The supervisor creates a worker for each input, adds it to the pending set, and sends it to work. As workers report back, they're removed from the set. When the set empties, all work is done.

Usually you want workers to send results back, not just signal completion. The `done` behavior can carry a result alongside the worker identity:

```pony
use "collections"

actor Worker
  let _supervisor: Supervisor

  new create(supervisor: Supervisor) =>
    _supervisor = supervisor

  be work(input: String val) =>
    let result = recover val input.upper() end
    _supervisor.done(this, result)

actor Supervisor
  let _out: OutStream
  let _pending: SetIs[Worker]
  let _results: Array[String val]

  new create(out: OutStream) =>
    _out = out
    _pending = SetIs[Worker]
    _results = Array[String val]

  be run() =>
    let inputs = ["alpha"; "bravo"; "charlie"]
    for input in inputs.values() do
      let worker = Worker(this)
      _pending.set(worker)
      worker.work(input)
    end

  be done(worker: Worker, result: String val) =>
    _pending.unset(worker)
    _results.push(result)
    if _pending.size() == 0 then
      _out.print("All workers finished. Results:")
      for r in _results.values() do
        _out.print("  " + r)
      end
    end
```

Here's a complete runnable program that ties it all together:

```pony
use "collections"

actor Worker
  let _supervisor: Supervisor

  new create(supervisor: Supervisor) =>
    _supervisor = supervisor

  be work(input: String val) =>
    // simulate some processing
    let result = recover val
      input.size().string() + " chars in '" + input + "'"
    end
    _supervisor.done(this, result)

actor Supervisor
  let _out: OutStream
  let _pending: SetIs[Worker]
  let _results: Array[String val]

  new create(out: OutStream) =>
    _out = out
    _pending = SetIs[Worker]
    _results = Array[String val]

  be run() =>
    let inputs = ["alpha"; "bravo"; "charlie"; "delta"; "echo"]
    for input in inputs.values() do
      let worker = Worker(this)
      _pending.set(worker)
      worker.work(input)
    end

  be done(worker: Worker, result: String val) =>
    _pending.unset(worker)
    _results.push(result)
    if _pending.size() == 0 then
      _out.print("All " + _results.size().string() + " workers finished:")
      for r in _results.values() do
        _out.print("  " + r)
      end
    end

actor Main
  new create(env: Env) =>
    Supervisor(env.out).run()
```

## Discussion

The supervisor passes `this` to each worker at construction time, and the worker passes `this` back when it reports done. That `this` is what makes the tracking work. `SetIs` uses Pony's `is` operator for identity comparison rather than structural equality, so the supervisor is checking whether the exact actor instance that reported in is one it's waiting for. This gives you a natural guard against stray messages from workers you didn't create or have already finished.

You can extend this pattern with early termination. If the supervisor decides to cancel remaining work (maybe one worker found the answer, or a timeout fired), it can send a `cancel` behavior to all pending workers. The workers check a flag and skip their remaining processing. The supervisor should still expect `done` callbacks from cancelled workers, since the cancel message might arrive after the worker has already started or finished its work. A clean approach is to have the worker call `done` regardless and let the supervisor ignore results from workers it's already removed from the pending set.

For straightforward cases, this pattern is all you need. When things get more complex (dynamic work generation, result aggregation with early exit, scheduling-aware worker pools), take a look at the [fork_join](https://github.com/ponylang/fork_join) library. It formalizes the supervisor/worker idea into a `Generator`/`Worker`/`Collector` pipeline: a `Generator` produces work items on demand, workers process them in parallel, and a `Collector` aggregates results with the option to terminate early. The library handles worker lifecycle, scheduling, and backpressure so you can focus on the processing logic.

The [Batch and Yield](batch-and-yield.md) pattern is a natural companion. If your workers are processing large datasets inside a single behavior, they should break the work into batches and yield between them so they don't monopolize a scheduler thread. The two patterns compose well: the supervisor/worker pattern distributes work across actors, and batch-and-yield ensures each actor plays fair with the scheduler while doing its share.
