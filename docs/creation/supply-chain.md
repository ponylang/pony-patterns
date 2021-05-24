---
hide:
  - toc
---

# Supply Chain

## Problem

The Pony type system is very demanding when it comes to handling errors. The lack of `null` means that you are forced to initialize every variable and explicitly handle every possible source of initialization error. In return, you get freedom from `Null Pointer Exceptions` and their equivalents. However, a naive use of Pony's `None` type when initializing dependencies can lead to poor programmer ergononmics and frustration. This is particularly true when constructing actors. In Pony, an actor's constructor runs asynchronously so, unlike a class, it can't be a partial function.

Take, as an example, a Pony actor that receives messages and writes them to a file. Our first naive pass might look something like:

```pony
use "files"

actor Writer
  let _file: File

  new create(env: Env, file_name: String) =>
    let auth = env.root as AmbientAuth
    _file = File(FilePath(auth, file_name)?)

  be record(it: String) =>
    _file.write(it)
```

We've already hit our first problem. Our above code won't compile. Why? Well, it doesn't handle errors. For starters, `env.root as AmbientAuth` can fail. We might not be able to get the capability to acquire ambient authority. [Capabilities](https://en.wikipedia.org/wiki/Capability-based_addressing) as implemented in Pony allow you to restrict what subsystems in your program can do. Whereas in a non-capabilities language like Java, any class can open a file or a socket, in Pony you have to be given the capability to do that. This means that you might lack that capability and it could fail. If we were to address that, we would also need to address that our `File` object might not be able to be initialized. In order to deal with our errors, we'll need to make `file` be of type `(File | None)`. This union type states that we can have a file or nothing. An iteration to address this gets us almost all the way to being able to compile but not quite:

```pony
use "files"

actor Writer
  var _file: (File | None) = None

  new create(env: Env, file_name: String) =>
    try
      let auth = env.root as AmbientAuth
      _file = File(FilePath(auth, file_name)?)
    end

  be record(it: String) =>
    _file.write(it)
```

We're now left with one more compiler error to address.

```text
x.pony:13:9: couldn't find write in None val
    _file.write(it)
         ^
```

One more change address that `file` could be uninitialized:

```pony
use "files"

actor Writer
  var _file: (File | None) = None

  new create(env: Env, file_name: String) =>
    try
      let auth = env.root as AmbientAuth
      _file = File(FilePath(auth, file_name)?)
    end

  be record(it: String) =>
    match _file
    | let f: File => f.write(it)
    end
```

In the final above example, in our `record` behavior, we match on `file` and only attempt to write if it is of type `File`. Awesome. We have working code. Except, ugh. There are actually several problems lurking.

One is programmer ergonomics. If you are using `file` a lot in this actor, you are going to be constantly matching to make sure you are handling `None` correctly.

We are also silently eating failures. If this actor can't start up properly then we might not want to continue running. We aren't going to know that of course. As far as a caller is concerned, we've successfully initialized and we are writing data to the file. What might actually be happening is that every call to `record` results in absolutely nothing. Tracking that down could turn out to be a nightmare.

Lastly, even if we wanted to communicate failure back, this is an actor. Everything is asynchronous and there's no straightforward way to say something like:

```pony
if (my_actor.is_initialized()) then
  my_actor.record(it)
else
  error
end
```

And even if there was, we want to fail on initialization, not lazily at some unknown time in the future.

## Solution

All of the problems that we enumerated above come from attempting to create objects whose creation can fail in the constructor of our actor. Rather than delay errors until we are in our actor's constructor, a much better approach is to supply our dependencies fully initialized.

In our previous case, we were relying on two things:

* Ambient Authority
* Existence of our File

If we initialize these outside of our actor then we can easily report construction errors and avoid messing with `None` as a possibility inside our `Writer` actor:

```pony
use "files"

actor Main
  new create(env: Env) =>
    try
      let auth = env.root as AmbientAuth
      let file = recover File(FilePath(auth, "free-candy.txt")?) end
      let writer = Writer(consume file)
    else
      env.err.print("Couldn't initialize dependencies")
    end

actor Writer
  let _file: File

  new create(file: File iso) =>
    _file = consume file

  be record(it: String) =>
    _file.write(it)
```

## Discussion

This pattern is applicable across a wide swath of Pony code. There are many objects that like `File` can fail to initialize. Some examples include network sockets, regular expressions, and anything that involves parsing user input.

In addition to the benefits we've already enumerated previously, by using [dependency injection](https://en.wikipedia.org/wiki/Dependency_injection) to solve our Pony specific problem, we also reap the advantages of DI, in particular, a much more testable actor.

For dependencies that themselves have complicated dependencies, we could combine together with a pattern like [Builder](https://en.wikipedia.org/wiki/Builder_pattern) or [Factory](https://en.wikipedia.org/wiki/Factory_%28object-oriented_programming%29) to abstract away a lot of the gory details.
