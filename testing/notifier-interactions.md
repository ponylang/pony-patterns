# Testing Notifier Interactions

## Problem

Event driven code is very common in Pony. Many classes take a "notifier" class
that has callbacks that get triggered when certain events happen. The network
code such as `UDPNotify` and `TCPNotify` are examples of this. As you write your
own Pony code, the notifier pattern is one you'll end up using quite a bit.
Testing that your code is correctly interacting with notifiers is
straightforward; however, how you go about doing that isn't immediately obvious.
Imagine for a moment that you have the following actor:

```
actor Receiver
  let _notify: Notified

  new create(notify: Notified iso) =>
    _notify = consume notify

  be receive(msg: String) =>
    _notify.received(this, msg)
```

It's really simple. When it receives some string, it will pass its own identity
and that string along to the object it is supposed to notify. What is the object
it will notify? Anything that conforms to the interface

```
interface Notified
  fun ref received(rec: Receiver ref, msg: String)
```

In our contrived, simplified example, we want to know that if we call `receive` 
on the `Receiver` actor, then the string we call it with will end up being 
passed to our notifier where we can process it. So, how do we go about that?

## Solution

```pony
use "ponytest"

actor Main is TestList
  new create(env: Env) => PonyTest(env, this)
  new make() => None

  fun tag tests(test: PonyTest) =>
    test(_TestNotifier)

class iso _TestNotifier is UnitTest
  fun name(): String => "test notifier"

  fun apply(h: TestHelper) =>
    let r = Receiver(recover TestNotifier(h, "Hi") end)

    r.receive("Hi")
    h.long_test(2_000_000_000)

  fun timed_out(h: TestHelper) =>
    h.complete(false)

class TestNotifier is Notified
  let _h: TestHelper
  let _expected: String

  new iso create(h: TestHelper, expected: String) =>
    _h = h
    _expected = expected

  fun ref received(rec: Receiver ref, msg: String) =>
   _h.assert_eq[String](_expected, msg)
   _h.complete(true)

interface Notified
  fun ref received(rec: Receiver ref, msg: String)

actor Receiver
  let _notify: Notified

  new create(notify: Notified iso) =>
    _notify = consume notify

  be receive(msg: String) =>
    _notify.received(this, msg)
```

## Discussion

Notifiers work by using structual typing. We define an interface that the given
notifier has to implement and then create concrete implementations. In our above
solution, you can see this with:

```
interface Notified
  fun ref received(rec: Receiver ref, msg: String)
```

and 

```
class TestNotifier is Notified
  let _h: TestHelper
  let _e: String

  new iso create(h: TestHelper, e: String) =>
    _h = h
    _e = e

  fun ref received(rec: Receiver ref, msg: String) =>
   _h.assert_eq[String](_e, msg)
   _h.complete(true)
```

Our test is verifying that our Receiver correctly uses the notifier and that 
when we call `receive` on a `Receiver`, we pass the correct data to the 
notifier's `received` method:

```
  fun ref received(rec: Receiver ref, msg: String) =>
   _h.assert_eq[String](_e, msg)
   _h.complete(true)
```
