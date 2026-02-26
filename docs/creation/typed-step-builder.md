---
hide:
  - toc
---

# Typed Step Builder

## Problem

You're building a value that requires several fields, and some of them are mandatory. A common approach is a fluent builder where every method returns the same builder type, letting the caller chain calls in any order.

```pony
class MessageBuilder
  var _to: String = ""
  var _subject: String = ""
  var _body: String = ""

  fun ref to(recipient: String): MessageBuilder =>
    _to = recipient
    this

  fun ref subject(subject': String): MessageBuilder =>
    _subject = subject'
    this

  fun ref body(body': String): MessageBuilder =>
    _body = body'
    this

  fun build(): String =>
    "To: " + _to + "\nSubject: " + _subject + "\n\n" + _body
```

The fields default to empty strings because Pony requires every field to be initialized in the constructor. That means `build()` always has valid data as far as the type system is concerned. Nothing prevents a caller from skipping straight to the end:

```pony
actor Main
  new create(env: Env) =>
    let msg = MessageBuilder.body("How are you?").build()
    env.out.print(msg)
```

This compiles and runs, producing a message with no recipient and no subject. The builder silently accepted incomplete data because every field had a default. The caller forgot `.to()` and the compiler didn't say a word.

## Solution

Instead of a single builder type with all the methods, define a separate interface for each construction phase. Each interface exposes exactly one advancement method whose return type is the next phase's interface. The compiler enforces the build order: you literally can't call the wrong method because it doesn't exist on the type you're holding.

We'll build an email message in three mandatory steps: set the recipient, set the subject, then set the body. Here's the first phase:

```pony
interface MessageRecipient
  fun ref to(recipient: String): MessageSubject
```

A `MessageRecipient` has exactly one method: `to()`. It takes the recipient and returns a `MessageSubject`. There's no way to finish without providing a recipient first.

```pony
interface MessageSubject
  fun ref subject(subject': String): MessageBody
```

Once you've set the recipient, you're holding a `MessageSubject`. The only thing you can do is call `subject()`, which advances you to the final phase.

```pony
interface MessageBody
  fun ref body(body': String): String
```

The last phase collects the body and returns the finished message as a `String`. In a real application this final method would typically return a domain object rather than a string; we're keeping it simple here.

Now the concrete class that ties it all together:

```pony
class _MessageBuilder
  var _to: String = ""
  var _subject: String = ""

  fun ref to(recipient: String): MessageSubject =>
    _to = recipient
    this

  fun ref subject(subject': String): MessageBody =>
    _subject = subject'
    this

  fun ref body(body': String): String =>
    "To: " + _to + "\nSubject: " + _subject + "\n\n" + body'
```

`_MessageBuilder` has all three methods, one from each phase. Notice there's no `is` clause declaring that it implements any of the interfaces. It doesn't need one. Pony interfaces use structural subtyping: any type whose methods match an interface's signatures automatically satisfies that interface. Since `_MessageBuilder` has `to()` returning `MessageSubject`, it satisfies `MessageRecipient`. Since it has `subject()` returning `MessageBody`, it satisfies `MessageSubject`. And since it has `body()` returning `String`, it satisfies `MessageBody`. One class, three interface views.

The caller never sees `_MessageBuilder` directly. A factory primitive provides the entry point:

```pony
primitive Messages
  fun apply(): MessageRecipient =>
    _MessageBuilder
```

`Messages()` returns a `MessageRecipient`, so the caller starts at phase one. Each method advances to the next phase, and the compiler enforces the order.

Here's the complete program:

```pony
interface MessageRecipient
  fun ref to(recipient: String): MessageSubject

interface MessageSubject
  fun ref subject(subject': String): MessageBody

interface MessageBody
  fun ref body(body': String): String

class _MessageBuilder
  var _to: String = ""
  var _subject: String = ""

  fun ref to(recipient: String): MessageSubject =>
    _to = recipient
    this

  fun ref subject(subject': String): MessageBody =>
    _subject = subject'
    this

  fun ref body(body': String): String =>
    "To: " + _to + "\nSubject: " + _subject + "\n\n" + body'

primitive Messages
  fun apply(): MessageRecipient =>
    _MessageBuilder

actor Main
  new create(env: Env) =>
    let message =
      Messages()
        .to("alice@example.com")
        .subject("Hello")
        .body("How are you?")
    env.out.print(message)
```

Try skipping a step and the compiler stops you. Calling `Messages().subject("Hello")` fails because `Messages()` returns a `MessageRecipient`, which only has `to()`. Calling `Messages().to("alice@example.com").body("How are you?")` also fails because `.to()` returns a `MessageSubject`, which only has `subject()`. Out-of-order calls are compile errors, not runtime surprises.

## Discussion

Structural typing is what makes this pattern work without boilerplate. In Pony, an interface is a structural type: any class whose methods match the interface's signatures automatically conforms, with no explicit declaration needed. This is different from traits, which are nominal. A class must write `is SomeTrait` to satisfy a trait, even if it already has all the right methods. Because interfaces are structural, our single `_MessageBuilder` class satisfies all three phase interfaces simultaneously without knowing about any of them at declaration time. Adding a new phase means adding a new interface and a new method on the concrete class.

This pattern shares a key idea with the [State Machine](../behavioral/state-machine.md) pattern: both use different types to represent different phases of a lifecycle. The difference is in what they're solving. State Machine creates separate objects for each state and swaps them at runtime to change behavior. The actor delegates to whichever state object is current, and different states respond to the same message differently. Typed Step Builder uses a single object behind multiple interface views to enforce construction ordering at compile time. State Machine is about runtime behavior dispatch; Typed Step Builder is about compile-time sequencing guarantees.

The pattern extends naturally to repeatable steps within a phase. If one of your phases allows multiple calls before advancing, give it a method that stays in the current phase alongside the advancement method. For example, a `cc()` method on `MessageRecipient` could return `MessageRecipient` while `to()` still advances to `MessageSubject`. The caller can add as many CC recipients as they like, and the compiler still forces them to call `to()` before moving on. The same idea works for optional fields: put them on the phase interface alongside the required advancement method.

Builders can also support reuse. Adding a `reset()` method to each interface that returns `MessageRecipient` gives the caller a way to abandon a partial build and start over from the first phase without allocating a new builder.
