# Testing Output Only Actors

## Problem

A lot of the Pony code you will need to test involves actors that take input and create output that leaves your system. A good example of this is testing writing to a file. How can you verify that the contents of the file are what you expect? You could write the file as normal and then compare its contents to what you were expecting. In the end, though, that doesn't work for everything. What if you are writing to standard out or over a network? Luckily, there is a general purpose pattern to address this problem.

## Solution

Our solution draws on three primary elements:

* Pony promises
* Stub objects
* Pony's causal messaging

The code below can be used to test that you are outputting data correctly to a file stream. In this particular case, "correctly" means that when we call `print` on `MyImportantClass` with the argument "Hello World!" that we would get "Hello World!" as output from the stream that `MyImportantClass` is using.

```pony
use "ponytest"
use "promises"

class MyImportantClass
  let _stream: OutStream
  new create(s: OutStream) =>
    _stream = s

  fun print(s: String) =>
    _stream.print(s)

actor Main is TestList
  new create(env: Env) => PonyTest(env, this)
  new make() => None

  fun tag tests(test: PonyTest) =>
    test(_TestImportantPrinting)

class iso _TestImportantPrinting is UnitTest
  fun name(): String => "my important printing test"

  fun apply(h: TestHelper) =>
    h.long_test(1_000_000_000)

    let promise = Promise[String]
    promise.next[String](recover this~_fulfill(h) end)

    let stream = _TestStream(promise)
    let important = MyImportantClass(stream)
    important.print("Hello World!")
    stream.written()

  fun tag _fulfill(h: TestHelper, value: String): String =>
    h.assert_eq[String](value, "Hello World!")
    h.complete(true)
    value

  fun timed_out(h: TestHelper) =>
    h.complete(false)

actor _TestStream is OutStream
  let _output: String ref = String
  let _promise: Promise[String]

  new create(promise: Promise[String]) =>
    _promise = promise

  be print(data: ByteSeq) =>
    _collect(data)

  be write(data: ByteSeq) =>
    _collect(data)

  be printv(data: ByteSeqIter) =>
    for bytes in data.values() do
      _collect(bytes)
    end

  be writev(data: ByteSeqIter) =>
    for bytes in data.values() do
      _collect(bytes)
    end

  fun ref _collect(data: ByteSeq) =>
    _output.append(data)

  be written() =>
    let s: String = _output.clone()
    _promise(s)
```

That's a nice chunk of code, let's break it down and focus on the important bits. Here is the core of our test, the `apply` method on our test class:

```pony
  fun apply(h: TestHelper) =>
    h.long_test(1_000_000_000)

    let promise = Promise[String]
    promise.next[String](recover this~_fulfill(h) end)

    let stream = _TestStream(promise)
    let important = MyImportantClass(stream)
    important.print("Hello World!")
    stream.written()

```

Note the use of `TestHelper.long_test(1_000_000_000)`, where we tell the test framework that our test will continue to run until an assertion fails or one of `TestHelper.complete(true)` or `TestHelper.complete(false)` is called. We also provide it with a 1-second timeout after which it will fail the test with a timeout error.

Remember, we are attempting to verify that when we print to `MyImportantClass`, we get the correct output on the stream. In a real world example, our class would probably be doing some sort of formatting and wouldn't just be a pass through of the data. Instead of testing file stream directly, we are testing a stub `_TestStream` that is standing in for a standard library `OutStream` interface. In real code, this would probably be the concrete actor `FileStream` or similar. Our stub implements the OutStream interface and records everything we write to it:

```pony
actor _TestStream is OutStream
  let _output: String ref = String
  let _promise: Promise[String]

  new create(promise: Promise[String]) =>
    _promise = promise

  be print(data: ByteSeq) =>
    _collect(data)

  be write(data: ByteSeq) =>
    _collect(data)

  be printv(data: ByteSeqIter) =>
    for bytes in data.values() do
      _collect(bytes)
    end

  be writev(data: ByteSeqIter) =>
    for bytes in data.values() do
      _collect(bytes)
    end

  fun ref _collect(data: ByteSeq) =>
    _output.append(data)

  be written() =>
    let s: String = _output.clone()
    _promise(s)
```

The most interesting method in `_TestStream` is `be written()`, what's going on in there?

```pony
  be written() =>
    let s: String = _output.clone()
    _promise(s)
```

When invoked, it takes the promise that was supplied upon construction:

```pony
    let stream = _TestStream(promise)
```

And fulfills it with any data we have collected so far:

```pony
    let s: String = _output.clone()
    _promise(s)
```

Our promise was set up to call the `_fulfill` method on our test class when the promise is fulfilled.

```pony
  fun tag _fulfill(h: TestHelper, value: String): String =>
    h.assert_eq[String](value, "Hello World!")
    h.complete(true)
    value
```

What's going on in there? Well, we take our string of output that our stub got and compare it to our expected value (in this case "Hello World!") and then indicate that our test is complete.
We have access to the `TestHelper` we need to run `assert_eq` in `_fulfill` because when we constructed our promise initially, we created the promise as partially applied, supplying the `TestHelper` parameter:

```pony
    promise.next[String](recover this~_fulfill(h) end)
```

It's important to note that the above solution only works because we can rely on Pony's causal messaging. That is, each message sent to an actor from another actor will arrive in order. In our test, we call:

```pony
    important.print("Hello World!")
    stream.written()
```

because important.print calls a method on stream:

```pony
  fun print(s: String) =>
    _stream.print(s)
```

`stream.written()` is guaranteed to happen after the `_stream.print(s)`. Without that guarantee, this test wouldn't work. Without causal messaging, our promise might fire before it ever saw any data and our test could pass sometimes and fail other times.

## Discussion

Well, there you have it. How to test our interactions with an actor whose side-effects are only observable outside out system. As mentioned in the _Problem_ section, this pattern can be applied in many different scenarios so long as it involves testing an actor. And the code above can be used to test any actor that implements `OutStream`. Before we wrap up, let's cover one additional benefit of using a stub to test a `FileStream`.

The `FileStream` constructor takes a `File` object. That's an external dependency. When testing, avoiding external dependencies is good. In the case of a file, that might seem like a relatively benign dependency right up until the day the file system permissions on your test machine change or a directory you depend on isn't there and your tests start failing. The important part of all this is, we've left our process and can't rely on the external party to help our test (nor do we want to).

If you want to see an example of this pattern in the wild, check out the  [tests for the `logger` package](https://github.com/ponylang/ponyc/blob/master/packages/logger/_test.pony) in the [Pony standard library](https://stdlib.ponylang.io/logger--index).
