# Interrogating Actors with Promises

## Problem
Pony gives us an excellent abstraction for actors. We can define fields within those actors to maintain state and rely on the single-threaded nature of inbound message processing to ensure safe access to those fields. A problem arises when one actor wants to access the internal _state_ of another actor.

Let's say that we've built an event-sourced system that is dealing with financial transactions. We have an actor called `AccountAggregate` that is maintaining an _internal_ balance. This balance is determined by receiving events in the form of transactions (withdrawals, deposits, transfers, adjustments, etc). This sequence of events is processed to compute the balance for any given account.

Skipping some of the not-so-relevant details, we might have an actor that looks like this:

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

We can't write a method like this that returns the internal state:
```pony
    fun balance(): U64 =>
      _balance
```
This compiles, so we think we might be onto something. Then, we attempt to invoke this method:
```pony
let bal = savings.balance()
```
The above code _does not_ compile. This is because the receiver (_savings_) is a **tag** (an opaque reference that allows neither read nor write, only _send_) and in order to get something back from the function as written, it needs a box target. Our options are getting more and more limited, it seems.

## Solution
Unlike Akka, we don't have an _ask_ pattern. As mentioned in the [access](./access.md) pattern, we can send a lambda value to the actor which allows for internal state to be captured as a parameter, but there might be a cleaner way to deal with this one problem: _Promises_.

A _promise_ allows to declare that we realize that some value will either be fulfilled or rejected sometime in the future by whatever has been tasked with that promise. Since a `Promise` is an actor, we can send a promise to an actor as a `tag` without breaking any of the safety rules of actors and messaging.

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
This is somewhat useful, but we still want to be able to respond to the value used to fulfill the promise somehow. We can do this with _promise chaining_:

```pony
let p = Promise[U64]
p.next[None](Outputter~output(env))
agg.balance(p)
```
This gets us a little closer to what we want. Now, when the aggregate actor fulfills the promise, the result of that fulfillment will be sent as a parameter to the partially-applied `output` function on the `Outputter` primitive.

What we really want to be able to do is query multiple actors to get the account summary data and then send _all_ of that data (preferably bundled up in a nice array) to a destination actor that can then display and/or process the information. For this we're going to need an intermediary - something that awaits promise fulfillment and adds to a collection when fulfilled. Finally, once this intermediary has received all expected fulfillments, it can then send the results to a destination. You can think of this intermediary as a _promise buffer_:

```pony
interface CollectionReceiver[A]
  be receivecollection(coll: Array[A] val)
  
actor Collector[A: Any #send]  
  var _coll: Array[A] iso
  let _max: USize
  let _target: CollectionReceiver[A] tag
  
  new create(max: USize, target: CollectionReceiver[A] tag) =>    
    _coll = recover Array[A] end
    _max = max
    _target = target
    
  be receive(s: A) =>
    _coll.push(consume s)    
    if _coll.size() == _max then      
      let output = _coll = recover Array[A] end
      _target.receivecollection(consume output)
    end
```
Now that we have an intermediary, we can create multiple promises to send to multiple bank accounts:
```pony
let accounts = ["0001"; "0002"; "0003"; "0004"]
let collector = Collector[AccountSummary](accounts.size(), this)
    
for account in accounts.values() do
  let aggregate = AccountAggregate(account, 6000)
  let p = Promise[AccountSummary]
  p.next[None](recover collector~receive() end)
  aggregate.summarize(p)
end 
```
Our bank account aggregate can be modified to include an account summary with the summarize method:

```pony
be summarize(p: Promise[AccountSummary]) =>    
  p(recover AccountSummary(_balance, _account) end)
```

And we can add the method that the collector is expecting to the calling actor in order to respond to the list of account summaries:

```pony
be receivecollection(coll: Array[AccountSummary] val) =>
  _env.out.print("received account summaries:")
  for summary in coll.values() do
    _env.out.print("Account " + summary.accountnumber() + ": $" + summary.currentbalance().string())
  end
```
## Discussion
TBD
