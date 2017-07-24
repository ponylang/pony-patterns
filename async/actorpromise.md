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


## Discussion
TBD
