---
hide:
  - toc
---

# Interrogating Actors with Promises

## Problem

Pony gives us an excellent abstraction for actors. We can define fields within those actors to maintain state and rely on the single-threaded nature of inbound message processing to ensure safe access to those fields. A problem arises when one actor wants to access the internal _state_ of another actor.

Let's say that you want to collect values obtained from multiple actors without having to create a giant state machine. To illustrate this problem, we'll use an actor called `AccountAggregate` that is maintaining an _internal_ balance.

This actor might look something like this:

```pony
actor AccountAggregate
  let _account: String
  var _balance: U64

  new create(account: String, starting_balance: U64) =>
    _account = account
    _balance = starting_balance

  be handle_tx_event(tx: TransactionEvent val) =>
    // imagine lots of complex processing here
    _balance = _balance + tx.amount()
```

In our sample problem, the system might be holding onto hundreds of instances of the `AccountAggregate` actor, each with its own balance. What if we want to make a quick tour through all of these actors and ask them for their balances for display on a dashboard of some kind? We can't access the individual fields of the actor.

We can try to write a method like this that returns the internal state:

```pony
    fun balance(): U64 =>
      _balance
```

Adding this method compiles. But what happens if we attempt to use this method?

```pony
let bal = savings.balance()
```

This line of code doesn't compile. This is because the receiver (_savings_) is a **tag** (an opaque reference that allows neither read nor write, only _send_). Our options are getting more and more limited, it seems.

## Solution

Unlike some other languages with native actor patterns, we don't have primitives to ask for values or await responses from actors in Pony. As mentioned in the [access](./access.md) pattern, we can send a lambda value to the actor which allows for internal state to be captured as a parameter, but there might be a cleaner way to deal with this one problem: _Promises_.

A _promise_ lets us declare that we realize that some value will either be fulfilled or rejected sometime in the future by whatever has been tasked with that promise. Since a `Promise` is an actor, we can send a promise to an actor as a `tag` without breaking any of the safety rules of actors and messaging.

In the simplest case, we can have the `AccountAggregate` actor fulfill the promise inside a behavior:

```pony
 be balance(p: Promise[U64]) =>
    p(_balance)
```

We can then send the promise to the aggregate with the following code:

```pony
let p = Promise[U64]
agg.balance(p)
```

This is somewhat useful, but the value of the promise is lost. We still want to be able to respond to the value used to fulfill the promise somehow. We can do this with _promise chaining_:

```pony
let p = Promise[U64]
p.next[None](Outputter~output(env))
agg.balance(p)
```

This gets us a little closer to what we want. Now, when the aggregate actor fulfills the promise, the result of that fulfillment will be sent as a parameter to the partially-applied `output` function on the `Outputter` primitive.

Getting better, but not good enough. What we _really_ want to be able to do is query multiple actors to get the account summary data and then send _all_ of that data (preferably bundled up in a nice array) to a destination actor that can then display and/or process the information. For this we're going to need an intermediary - something that awaits promise fulfillment and adds to a collection when fulfilled. Once this intermediary has received every expected fulfillment, it can then fulfill a single promise of the collection. This intermediary promise can be created using the `Promises.join` function.

Now we can create multiple promises to send to multiple bank accounts:

```pony
let accounts = ["0001"; "0002"; "0003"; "0004"]

let create_summary_promise =
  {(account: String): Promise[AccountSummary] =>
    let aggregate = AccountAggregate(account, 6000)
    // just to illustrate mutable balance
    aggregate.handle_tx_event(recover TransactionEvent(351) end)
    aggregate.handle_tx_event(recover TransactionEvent(224) end)

    let p = Promise[AccountSummary]
    aggregate.summarize(p)
    p
  } iso

Promises[AccountSummary].join(
  Iter[String](accounts.values())
    .map[Promise[AccountSummary]](consume create_summary_promise))
  .next[None](recover this~receive_collection() end)
```

Our bank account aggregate can be modified to include an account summary with the **summarize** method:

```pony
be summarize(p: Promise[AccountSummary]) =>
  p(recover AccountSummary(_balance, _account) end)
```

Finally, we add the behavior to our Main actor that will respond to the list of account summaries:

```pony
be receive_collection(coll: Array[AccountSummary] val) =>
  _env.out.print("received account summaries:")
  for summary in coll.values() do
    _env.out.print("Account " + summary.accountnumber() + ": $" +
      summary.currentbalance().string())
  end
```

Putting it all together, we can now write code like the following that creates multiple actors and queries their internal state in a completely asynchronous fashion:

```pony
use "itertools"
use "promises"

class val TransactionEvent
  let _amount : U64

  new create(amount: U64) =>
    _amount = amount

  fun transaction_amount() : U64 =>
    _amount

class val AccountSummary
  let _balance : U64
  let _account : String

  new create(balance: U64, account: String) =>
    _balance = balance
    _account = account

  fun currentbalance() : U64 =>
    _balance

  fun accountnumber() : String =>
    _account

actor AccountAggregate
  let _account: String
  var _balance: U64

  new create(account: String, starting_balance: U64) =>
    _account = account
    _balance = starting_balance

  be handle_tx_event(tx: TransactionEvent val) =>
    // imagine lots of complex processing here
    _balance = _balance + tx.transaction_amount()

  be summarize(p: Promise[AccountSummary]) =>
    p(recover AccountSummary(_balance, _account) end)

actor Main
  let _env: Env

  new create(env: Env) =>
    _env = env

    let accounts = ["0001"; "0002"; "0003"; "0004"]

    let create_summary_promise =
      {(account: String): Promise[AccountSummary] =>
        let aggregate = AccountAggregate(account, 6000)
        // just to illustrate mutable balance
        aggregate.handle_tx_event(recover TransactionEvent(351) end)
        aggregate.handle_tx_event(recover TransactionEvent(224) end)

        let p = Promise[AccountSummary]
        aggregate.summarize(p)
        p
      } iso

    Promises[AccountSummary].join(
      Iter[String](accounts.values())
        .map[Promise[AccountSummary]](consume create_summary_promise))
      .next[None](recover this~receive_collection() end)

  be receive_collection(coll: Array[AccountSummary] val) =>
    _env.out.print("received account summaries:")
    for summary in coll.values() do
      _env.out.print("Account " + summary.accountnumber() + ": $" +
        summary.currentbalance().string())
    end
```

## Discussion

Actor systems have been around for quite some time now, but most developers don't default to modeling their problems as actor patterns. Most of us want to solve this problem with synchronous code that looks like this:

```pony
for acct in _accounts.values() do
  _summaries.push(acct.summarize())
end
```

The problem with this is that as our real-world problems get more complex, simple loops like this are just not powerful enough. In bigger, more complex models, there is often a _cost_ to asking an actor for its internal state. It might not be a precalculated field. Instead, invoking `summarize` might make calls to external systems, databases, or microservices.

Naively running through the summarization method in a for loop could cause a consumer to wait an indeterminate amount of time. By sending out a flood of promises, we can let each of the actors fulfill the promise on their own time and we'll get the results back far sooner than if we'd done the requests synchronously. This also gives us an added degree of reliability - by sending out these promises, we can also set a timeout in the collector so that we can build in things like a "circuit breaker" and return data indicating that we couldn't summarize all of the accounts.

In conclusion, Pony's actor system is incredibly powerful and some of that power comes from its deliberate restrictions. Learning how to embrace the actor model in combination with promises can provide an elegant solution to complex problems.
