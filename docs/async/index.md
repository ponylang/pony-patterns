---
hide:
  - toc
---

# Asynchronous Patterns

Patterns for dealing with the asynchronous nature of Pony. Unlike most languages, Pony has *zero* blocking operations. This can make simple everyday problems difficult until you know how to deal with them. If your problem has you wanting blocking operations, this is the chapter for you.

[Accessing an Actor with Arbitrary Transactions](access.md) lets callers define custom multi-step transactions that execute atomically inside an actor. Instead of bloating the actor's API with every possible combination of operations, you pass in a lambda that gets synchronous access to the actor's state.

[Batch and Yield](batch-and-yield.md) prevents an actor from monopolizing a scheduler thread when processing large collections. By breaking work into batches and sending yourself a message between each batch, you create natural yield points where the scheduler can run other actors.

[Interrogating Actors with Promises](actorpromise.md) shows how to query an actor's internal state when you can't just call a method on a `tag` reference. Promises give you a way to request a value and respond to it asynchronously, and `Promises.join` lets you collect results from many actors at once.

[Supervisor and Worker](supervisor-worker.md) is for when you need to farm out work to one or more actors and know when they're all done. The supervisor creates workers, hands them tasks, and tracks who's still outstanding. Workers report back when they finish, optionally carrying results. When the last worker checks in, the supervisor knows all work is complete.

[Waiting](waiting.md) addresses the question of how to delay work when there's no `sleep`. Pony's `Timer` and `Timers` types let you schedule actions at fixed intervals, covering use cases from rate limiting to periodic flushing to timeouts.
