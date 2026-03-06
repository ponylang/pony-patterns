---
hide:
  - toc
---

# Recover for Isolated Return

## Problem

You're writing a function that builds up some data and needs to return it as `iso^`. Maybe you're constructing a response that another actor will consume, or you're implementing an interface that requires an `iso^` return type. You write the obvious code:

```pony
fun make_greeting(name: String val): String iso^ =>
  let greeting = String
  greeting.append("Hello, ")
  greeting.append(name)
  greeting.append("!")
  greeting
```

The compiler rejects this. `greeting` is a `String ref` because that's what `String` gives you by default: a mutable reference. But the return type demands `iso^`, and the compiler can't let a `ref` become `iso`. There might be other references to that object in scope, and `iso` means "I am the only reference." The compiler has no way to verify that promise starting from a `ref`.

## Solution

Wrap the construction in a `recover` block. Inside the block, you create objects and work with them as `ref` just like normal. When the block ends, the compiler lifts the result to `iso^`.

```pony
fun make_greeting(name: String val): String iso^ =>
  recover
    let greeting = String
    greeting.append("Hello, ")
    greeting.append(name)
    greeting.append("!")
    greeting
  end
```

The compiler enforces a rule inside recover blocks: you can only access variables from outside the block if their capability is sendable (`val`, `tag`, or `iso`). No `ref` references from the enclosing scope can leak in. That means when the block finishes, the compiler knows for certain that nothing else aliases the result. It's safe to call it `iso`.

In this example, `name` is `String val`, which is sendable, so it crosses into the block without issue. Fields that are `val` work the same way. A `String val` field on your actor can be read directly inside a recover block because the field itself is sendable.

Where things get more involved is when you need data from a non-sendable field. If your actor has a `ref` field (like a mutable class instance), you can't access it inside a recover block, even if the piece of data you want from it is `val`:

```pony
class Settings
  let prefix: String val = "Hello"

actor Greeter
  let _settings: Settings  // Settings ref — not sendable

  fun _make_message(name: String val): String iso^ =>
    recover
      let msg = String
      msg.append(_settings.prefix)  // Error: _settings is ref
      msg
    end
```

The compiler sees `_settings` is `ref` and blocks the access, even though `_settings.prefix` is `val`. The fix is to extract the value you need before entering the recover block:

```pony
  fun _make_message(name: String val): String iso^ =>
    let prefix = _settings.prefix  // String val, extracted from the ref
    recover
      let msg = String
      msg.append(prefix)  // OK: prefix is val
      msg.append(", ")
      msg.append(name)
      msg.append("!")
      msg
    end
```

`_settings.prefix` evaluates to `String val` outside the recover block, and binding it to a local gives you a `val` reference that the block accepts. Any time you have a non-sendable object holding `val` data you need inside a recover block, extracting the values first is the way through.

Here's a complete program that builds an `iso` string inside one actor and sends it to another:

```pony
actor Main
  new create(env: Env) =>
    let printer = Printer(env.out)
    let greeter = Greeter(printer, "Hello")
    greeter.greet("Alice")
    greeter.greet("Bob")

actor Greeter
  let _printer: Printer
  let _prefix: String val

  new create(printer: Printer, prefix: String val) =>
    _printer = printer
    _prefix = prefix

  be greet(name: String val) =>
    let message = _make_message(name)
    _printer.print_it(consume message)

  fun _make_message(name: String val): String iso^ =>
    recover
      let msg = String
      msg.append(_prefix)
      msg.append(", ")
      msg.append(name)
      msg.append("!")
      msg
    end

actor Printer
  let _out: OutStream

  new create(out: OutStream) =>
    _out = out

  be print_it(message: String iso) =>
    _out.print(consume message)
```

`Greeter._make_message` builds the greeting string mutably inside a recover block, then the `greet` behavior consumes the result and sends it to the `Printer` actor. Since `_prefix` is `String val`, it can be used directly inside the recover block without extracting it first.

## Discussion

A bare `recover` block lifts the result to whatever the surrounding context requires: `iso^` or `val`. When the return type is `iso^`, you get `iso^`. When you're assigning to a `val` binding, you get `val`. You can also be explicit by writing `recover val` or `recover iso` to spell out what you want. Both forms enforce the same constraint on what can enter the block; the only difference is what comes out.

This pattern is the foundation beneath several of the [Data Sharing](../data-sharing/index.md) patterns. The [Copying](../data-sharing/copying.md) pattern uses it directly: `let copy: Array[U8] iso = recover Array[U8] end` creates the empty `iso` array that gets populated with copied data. The [Isolated Field](../data-sharing/isolated-field.md) pattern uses it to reinitialize the field after a destructive read: `_data = recover Array[U8] end`. In both cases, the recover block is doing the same job: producing an `iso` value that the compiler can trust is truly isolated.

The standard library leans on this idiom heavily. `File.read` returns `Array[U8] iso^` by creating an array inside a recover block and filling it through FFI calls. `Directory.entries` returns `Array[String] iso^` by building the directory listing in a recover block. If you find yourself writing a function that returns `iso^`, this is almost certainly the technique you'll reach for.
