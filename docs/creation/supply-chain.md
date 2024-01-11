---
hide:
  - toc
---

# Supply Chain

## Problem

The Pony type system is very demanding when it comes to handling errors. The lack of `null` means that you are forced to initialize every variable and explicitly handle every possible source of initialization error. In return, you get freedom from `Null Pointer Exceptions` and their equivalents. However, a naive use of Pony's `None` type when initializing dependencies can lead to poor programmer ergonomics and frustration. This is particularly true when constructing actors. In Pony, an actor's constructor runs asynchronously so, unlike a class, it can't be a partial function.

Take, as an example, a Pony actor that receives messages and writes them to a file within a temporary directory. Our first naive pass might look something like:

```pony
use "files"

actor TempWriter
  let _file: File

  new create(auth: FileAuth, file_name: String) =>
    let dir = FilePath.mkdtemp(auth)?
    let log = FilePath.from(dir, file_name)?
    _file = File(log)

  be record(it: String) =>
    _file.write(it)
```

We've already hit our first problem. Our above code won't compile. Why? Well, it doesn't handle errors. For starters, `FilePath.mkdtemp(auth)?` and `FilePath.from(dir, file_name)?` can both fail. We might not be able to create the directory.  If we were to address that, we would also need to address that our `File` object might not be able to be initialized. In order to deal with our errors, we'll need to make `file` be of type `(File | None)`. This union type states that we can have a file or nothing. An iteration to address this gets us almost all the way to being able to compile but not quite:

```pony
use "files"

actor TempWriter
  var _file: (File | None) = None

  new create(auth: FileAuth, file_name: String) =>
    try
      let dir = FilePath.mkdtemp(auth)?
      let log = FilePath.from(dir, file_name)?
      _file = File(log)
    end

  be record(it: String) =>
    _file.write(it)
```

We're now left with one more compiler error to address.

```text
x.pony:14:10: couldn't find write in None val
    _file.write(it)
         ^
```

One more change address that `file` could be uninitialized:

```pony
use "files"

actor TempWriter
  var _file: (File | None) = None

  new create(auth: FileAuth, file_name: String) =>
    try
      let dir = FilePath.mkdtemp(auth)?
      let log = FilePath.from(dir, file_name)?
      _file = File(log)
    end

  be record(it: String) =>
    match _file
    | let f: File => f.write(it)
    end
```

With this change, in our `record` behavior, we match on `file` and only attempt to write if it is of type `File`. Awesome. We have working code. Except, ugh. There is are actually a couple problems still lurking.

First, while `File` doesn't return an error, it can fail. The `File` constructor docs state:

> Attempt to open for read/write, creating if it doesn't exist, preserving the contents if it does exist.
> Set `errno` according to result.

To correctly use `File`, we have to check the `errno()` method to see if there was a failure. Oof. We can address this hidden issue by using the `CreateFile` primitive that will return a `(File | FileErrNo)`. If we get a `FileErrNo`, we can leave `_file` as `None`.

```pony
use "files"

actor TempWriter
  var _file: (File | None) = None

  new create(auth: FileAuth, file_name: String) =>
    try
      let dir = FilePath.mkdtemp(auth)?
      let log = FilePath.from(dir, file_name)?
      match CreateFile(log)
      | let f: File => _file = f
      end
    end

  be record(it: String) =>
    match _file
    | let f: File => f.write(it)
    end
```

Phew! Are we done? Nope. There's still a couple of problems.

One is programmer ergonomics. If you are using `_file` a lot in this actor, you are going to be constantly matching to make sure you are handling `None` correctly.

Even worse, we are also silently eating failures. If this actor can't start up properly then we might not want to continue running. We aren't going to know. As far as a caller is concerned, we've successfully initialized and we are writing data to the file. What might actually be happening is that every call to `record` results in absolutely nothing. Tracking that down could turn out to be a nightmare.

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

In our previous case, we were relying on our ability to successfully create the temporary directory that we will create our file in. If we initialize the directory and file outside of our actor then we can easily report construction errors and avoid messing with `None` as a possibility inside our `TempWriter` actor:

```pony
use "files"

actor Main
  new create(env: Env) =>
    try
      let dir = FilePath.mkdtemp(FileAuth(env.root))?
      let log = FilePath.from(dir, "free-candy.txt")?
      let file = recover iso
        match CreateFile(log)
        | let f: File => f
        else
          error
        end
      end
      TempWriter(consume file)
    else
      env.err.print("Couldn't create dependencies")
    end

actor TempWriter
  let _file: File

  new create(file: File iso) =>
    _file = consume file

  be record(it: String) =>
    _file.write(it)
```

## Discussion

This pattern is applicable across a wide swath of Pony code. There are many methods that like `File.mkdtemp` can fail to successfully complete. Some examples include network sockets, regular expressions, and anything that involves parsing user input.

In addition to the benefits we've already enumerated previously, by using [dependency injection](https://en.wikipedia.org/wiki/Dependency_injection) to solve our Pony specific problem, we also reap the advantages of DI, in particular, a much more testable actor.

For dependencies that themselves have complicated dependencies, we could combine together with a pattern like [Builder](https://en.wikipedia.org/wiki/Builder_pattern) or [Factory](https://en.wikipedia.org/wiki/Factory_%28object-oriented_programming%29) to abstract away a lot of the gory details.
