---
hide:
  - toc
---

# Mutable and Sendable

## Problem

You have a central actor that manages a large data structure. Other actors send it updates, and still other actors need to read from it. The data structure has to be both mutable — so updates are cheap — and shareable — so reads can be served to other actors.

This is a common scenario. Think of a Supervisor actor that collects results from a pool of Workers and responds to queries from Requesters. Workers produce results and send them to the Supervisor. Requesters ask for results, and the Supervisor sends them back.

The tension is between mutating data and sharing it. In Pony, mutable data is `ref`, which can't be sent to another actor. Shareable data is `val`, which can be sent freely but can't be mutated. We need both properties, but on different parts of the structure.

Let's look at why the obvious approaches fall short.

If you use a fully mutable structure — say a `Map[USize, Array[I32]]` where both the map and its values are `ref` — updating is cheap, but sharing is expensive. Every time a Requester asks for data, you'd have to [copy](copying.md) the relevant `ref` Array into a new `iso` to send it. If requests are frequent, that's a lot of copying.

If you go fully persistent — a persistent `Map` holding persistent `Lists` — sharing is free because everything is `val`. But every update to the map creates a new map. [Persistent data structures](persistent-data-structures.md) are designed to share structure with their previous versions, so this isn't as bad as a full copy, but for a very large map with frequent updates, the overhead adds up.

You might think `iso` solves the problem. An `iso` reference is both mutable and sendable — it's the one reference capability that allows both. But if you send it to a Requester, the Supervisor no longer has it. The Supervisor can't process any more updates or requests until the Requester sends the data back. You've effectively serialized all access to the data, which defeats the purpose. You could try the [isolated field pattern](isolated-field.md), but that's designed for cases where you're giving data away, not sharing it while keeping it.

What we need is a way to get cheap updates to the structure as a whole while still being able to share individual pieces of it freely.

## Solution

Use a mutable `ref` collection to hold persistent `val` values. The container is cheap to update because it's mutable. The values inside are cheap to share because they're `val`.

```pony
use mut = "collections"
use "collections/persistent"

type Datum is List[I32]

actor Worker
  let _supervisor: Supervisor

  new create(supervisor: Supervisor) =>
    _supervisor = supervisor

  be do_work(id: USize) =>
    // do heavy lifting here, build up a result
    let result = [as I32: 1; 2; 3; 4]
    // convert to a persistent List and send to Supervisor
    _supervisor.update(id, Lists[I32](result))

actor Requester
  let _supervisor: Supervisor

  new create(supervisor: Supervisor) =>
    _supervisor = supervisor

  be request(id: USize) =>
    _supervisor.get(id, this)

  be receive(id: USize, result: Datum) =>
    // use the result here
    None

actor Supervisor
  let _data: mut.Map[USize, Datum]
  let _default: Datum = Lists[I32]([0; 0; 0; 0])

  new create() =>
    _data = mut.Map[USize, Datum]

  be update(id: USize, new_data: Datum) =>
    _data.update(id, new_data)

  be get(id: USize, requester: Requester) =>
    requester.receive(id, _data.get_or_else(id, _default))
```

Let's walk through the key parts.

```pony
actor Supervisor
  let _data: mut.Map[USize, Datum]
```

The Supervisor holds a mutable `Map` — imported as `mut.Map` to avoid a name collision with the persistent collections. Because `_data` is a `ref`, the Supervisor can add, update, and remove entries cheaply with normal mutable map operations.

```pony
type Datum is List[I32]
```

`Datum` is a type alias for `List[I32]` — a persistent list from `collections/persistent`. Persistent lists are `val`, which means they can be shared freely between actors without copying.

```pony
  be update(id: USize, new_data: Datum) =>
    _data.update(id, new_data)
```

When a Worker sends new results, the Supervisor swaps the value in its mutable map. This is cheap — it's updating a pointer in the map, not copying the data.

```pony
  be get(id: USize, requester: Requester) =>
    requester.receive(id, _data.get_or_else(id, _default))
```

When a Requester asks for data, the Supervisor looks up the persistent `List` and sends it directly. No copying needed — the `List` is already `val`, so it's safe to share. The Supervisor keeps its reference to the map and can continue processing other updates and requests immediately.

```pony
  be do_work(id: USize) =>
    // do heavy lifting here, build up a result
    let result = [as I32: 1; 2; 3; 4]
    // convert to a persistent List and send to Supervisor
    _supervisor.update(id, Lists[I32](result))
```

Workers build up their results as mutable arrays, then convert to a persistent `List` using `Lists[I32]` before sending. The conversion happens once, and from then on the data is `val` and can flow freely between actors.

## Discussion

This pattern builds on two existing data sharing patterns and fills a gap between them.

The [copying pattern](copying.md) solves the mutable-and-shareable problem by copying the data each time you send it. That works well when sends are infrequent relative to updates. The [persistent data structures pattern](persistent-data-structures.md) solves it by making everything immutable — sharing is free, but every update creates a new structure. That works well when updates are infrequent relative to sends.

This pattern is for when the container itself changes frequently but individual values need to be shared often. By using a mutable container holding persistent values, you get cheap updates to the map and cheap reads from it. The trade-off is that you pay the cost of converting to a persistent structure once per update, when the Worker creates the `List`.

You might also consider the [isolated field pattern](isolated-field.md) for sending mutable data. The key difference is that an `iso` field requires giving away your reference when you send — the Supervisor would lose access to the data until it's sent back. With this pattern, the values are `val`, so any number of actors can hold references to them simultaneously.
