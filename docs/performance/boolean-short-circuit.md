---
hide:
  - toc
---

# Boolean Short-Circuit

## Problem

Your application logs messages, and constructing them involves string operations — concatenation, conversion, formatting. Most of these messages won't be logged because the configured log level filters them out. But if your logging API takes the message as an argument, the caller builds the string and sends a message to the output stream regardless of whether the level check passes.

```pony
// Hypothetical API where log takes a level and a message
logger.log(Warn, name + ": " + reason)
```

Even when the logger is configured to only show errors, this code allocates memory, constructs a new string from `name`, `":"`, and `reason`, and sends it to the output stream — all for a message that gets thrown away. In a hot loop or a high-throughput system, the cost adds up.

You could guard every call with an `if`:

```pony
if logger(Warn) then
  logger.log(name + ": " + reason)
end
```

This works, but it's verbose and easy to forget. What you want is an idiom that's as concise as a single call but avoids evaluating the message expression — and sending the resulting message — when the level check fails.

## Solution

Design the API so both the condition check and the action return `Bool`, then chain them with `and`. Pony's `and` operator short-circuits: if the left side is `false`, the right side is never evaluated.

```pony
use "logger"

actor Main
  new create(env: Env) =>
    // Create a logger that only logs Warn and above
    let logger = StringLogger(Warn, env.out)

    // These two are below Warn — the right side of `and` is never
    // evaluated, so no strings are built and no messages are sent
    logger(Fine) and logger.log("fine: " + expensive())
    logger(Info) and logger.log("info: " + expensive())

    // These two are at or above Warn — the full expression is
    // evaluated and the message is sent to the output stream
    logger(Warn) and logger.log("warn: something happened")
    logger(Error) and logger.log("error: something went wrong")

  fun expensive(): String =>
    // imagine something costly here
    "result"
```

This is the pattern used by the [ponylang/logger](https://github.com/ponylang/logger) library. The log levels form a hierarchy — `Fine`, `Info`, `Warn`, `Error` — from most to least verbose. A logger configured at `Warn` will only log messages at `Warn` or `Error`.

Let's walk through how the API design makes this work.

```pony
fun apply(level: LogLevel): Bool =>
  level() >= _level()
```

`Logger` has an `apply` method that takes a `LogLevel` and returns `Bool` — `true` if the requested level is at or above the logger's configured level. Because it's `apply`, calling `logger(Warn)` invokes it directly.

```pony
fun log(value: A, loc: SourceLoc = __loc): Bool =>
  _out.print(_formatter(_f(consume value), loc))
  true
```

The `log` method does the actual work: it formats the value and sends it to the output stream via `_out.print(...)`. It returns `true` so it can participate in the `and` chain.

When you write:

```pony
logger(Warn) and logger.log(name + ": " + reason)
```

two things can happen:

- If `logger(Warn)` returns `false`, Pony's `and` short-circuits. The expression `name + ": " + reason` is never evaluated — no strings are allocated, no concatenation happens. The call to `logger.log(...)` never executes, so `_out.print(...)` never sends a message to the output stream. The cost is a single integer comparison.

- If `logger(Warn)` returns `true`, the right side evaluates normally: the string is built, formatted, and printed.

## Discussion

The two costs this pattern avoids are worth understanding separately.

The first is **expression evaluation**. Arguments to a method are evaluated before the method is called. In `logger.log(name + ": " + reason)`, the string concatenation happens at the call site, producing a new `String` with its own memory allocation. The [Limiting String Allocations](limiting-string-allocations.md) pattern shows how to make string construction cheaper when it does happen, but the boolean short-circuit pattern avoids the construction entirely.

The second is **message sends**. Inside `log`, the call to `_out.print(...)` sends an asynchronous message to the output stream actor. Even if the message content is cheap to produce, the message send itself has overhead — it allocates a message in the actor's queue. When the short-circuit skips the `log` call, this message send never happens.

The pattern generalizes beyond logging. Any API with a condition-then-act shape can use it: make the condition method return `Bool`, make the action method return `Bool`, and let callers chain them with `and`. The trade-off is that callers need to learn the idiom — `logger(Warn) and logger.log(msg)` is less obvious than `logger.log(Warn, msg)` on first encounter. But once learned, it reads naturally and the performance benefit is automatic.
