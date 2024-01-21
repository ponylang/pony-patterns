---
hide:
  - toc
---

# Static Constructor

## Problem

You want to construct an object or return a meaningful error message. Unfortunately, in Pony there's no way to do that. Pony constructors always return an initialized instance of their class unless, the constructor is partial in which case nothing is returned as we jump to the nearest error handler.

```pony
class Foo
  // Always returns a foo
  new create() => None

  // Sometimes returns a foo
  new perhaps(a: Bool) ? =>
    if not a then
      error
    end
```

What you would like to do instead is:

```pony
class Error
  let msg: String

  new create(m: String) =>
    msg = m

class Foo
  // return a Foo or Error message
  new create(a: Bool): (Foo | Error) =>
    if not a then
      Error("Can't build Foo that way")
    else
      this
    end
```

## Solution

Use a `primitive`.

```pony
class Error
  let msg: String

  new create(m: String) =>
    msg = m

class Foo
  new create() => None

primitive FooConstructor
  fun apply(a: Bool): (Foo | Error) =>
    if not a then
      Error("Can't build a Foo that way")
    else
      Foo
    end
```

## Discussion

Static constructor is the [Global Function](../code-sharing/global-function.md) pattern applied to object construction. As we discussed in Global Function, Pony's primitives are a great way to group together stateless "like functions". If you are looking to do anything that might be classified as a static function or a utility function in another language, you probably want to use a primitive in Pony.

If you have an background in [ML](https://en.wikipedia.org/wiki/ML_(programming_language)) type languages, you can think of primitives as similar to modules in [OCaml](https://ocaml.org/) and [F#](https://fsharp.org/).

Finally, here's our static constructor in action:

```pony
actor Main
  new create(env: Env) =>
    match FooConstructor(true)
    | let f: Foo =>
      // ToDo: do something with Foo
      None
    | let e: Error =>
      env.err.print(e.msg)
```
