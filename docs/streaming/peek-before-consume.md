---
hide:
  - toc
---

# Peek Before Consume

## Problem

You're building a streaming protocol parser on top of `buffered.Reader`. Data arrives over the network in chunks, and you need to parse structured messages as they come in. The challenge is that when you read from a `Reader`, you consume bytes. If a message is only partially available, you've eaten some bytes, failed to parse the rest, and now the buffer is in a broken state.

Here's a concrete example. Suppose your wire protocol has two message types: strings (tag `0x01`, followed by a big-endian `U16` length, followed by that many bytes) and 32-bit integers (tag `0x02`, followed by 4 bytes big-endian). A straightforward parser might look like this:

```pony
use "buffered"

primitive NaiveParser
  fun apply(buffer: Reader ref): (String | I32) ? =>
    let tag = buffer.u8()?
    match tag
    | 0x01 =>
      let len = buffer.u16_be()?
      String.from_array(buffer.block(len.usize())?)
    | 0x02 =>
      buffer.i32_be()?
    else
      error
    end
```

This works when complete messages are sitting in the buffer. But streaming data doesn't arrive that neatly. Suppose only the tag byte `0x02` has arrived so far and the 4-byte integer payload hasn't come in yet. The parser calls `buffer.u8()?` and successfully consumes the tag. Then it calls `buffer.i32_be()?`, which fails because there aren't 4 bytes available. The caller's `try`/`else` catches the error, but the damage is done: the tag byte is gone from the buffer. When the remaining 4 bytes arrive and the parser tries again, it reads what should be the first payload byte as a tag. The buffer is now permanently misaligned and every subsequent parse produces garbage.

## Solution

The fix is to separate checking from consuming. First, peek at the buffer to determine whether a complete message is available. Peeking reads data at a given offset without advancing the read position. Only after confirming the message is complete do you consume the bytes.

`Reader` provides two families of methods that make this possible. Peek methods like `peek_u8`, `peek_u16_be`, and `peek_i32_be` take an offset parameter and read without consuming. Consume methods like `u8()`, `u16_be()`, `i32_be()`, and `block()` advance the read position. The peek methods let you scan ahead through the buffer, and the consume methods let you actually extract the data once you know it's all there.

Start by defining the result types. A parse attempt has three possible outcomes: a successfully parsed value, an indication that the data is incomplete (we need to wait for more bytes), or an indication that the data is malformed. A union type captures all three:

```pony
use "buffered"

primitive ParseError
  """
  The data in the buffer is malformed.
  """

type ParseResult is (String | I32 | None | ParseError)
```

`None` means "not enough data yet, try again later." `ParseError` means "the data is structurally invalid." Parsed values (`String` or `I32`) represent success.

The completeness check uses only peek methods and `size()`. It scans the buffer from a given offset to determine how many bytes the next message occupies, without consuming anything:

```pony
primitive Parser
  fun _complete_size(buffer: Reader ref, offset: USize)
    : (USize | None | ParseError)
  =>
    try
      let tag = buffer.peek_u8(offset)?
      match tag
      | 0x01 =>
        let len = buffer.peek_u16_be(offset + 1)?
        let total = 1 + 2 + len.usize()
        if buffer.size() >= (offset + total) then
          total
        else
          None
        end
      | 0x02 =>
        let total: USize = 1 + 4
        if buffer.size() >= (offset + total) then
          total
        else
          None
        end
      else
        ParseError
      end
    else
      None
    end
```

Each peek method is partial: it raises an error if the offset is beyond the available data. The outer `try`/`else` catches that and returns `None` (incomplete). Inside, we peek at the tag to determine the message type, then check whether enough bytes are available for the full message. For strings, we also peek at the length field to find out how many payload bytes to expect. If the tag is unrecognized, we return `ParseError`.

The consume pass only runs after the completeness check confirms the data is there. It reads the same fields in the same order, but this time using consume methods that advance the read position:

```pony
  fun _parse(buffer: Reader ref): (String | I32 | ParseError) =>
    try
      let tag = buffer.u8()?
      match tag
      | 0x01 =>
        let len = buffer.u16_be()?
        String.from_array(buffer.block(len.usize())?)
      | 0x02 =>
        buffer.i32_be()?
      else
        ParseError
      end
    else
      // Unreachable: _complete_size confirmed the data is available.
      ParseError
    end
```

The structure mirrors `_complete_size` exactly. The `try`/`else` is required by the compiler because the consume methods are still partial, but the else path should never execute since the completeness check already verified the data.

The orchestrator ties the two passes together:

```pony
  fun apply(buffer: Reader ref): ParseResult =>
    match _complete_size(buffer, 0)
    | let _: USize => _parse(buffer)
    | None => None
    | ParseError => ParseError
    end
```

If the completeness check returns a byte count, the data is ready and we consume it. If it returns `None`, we pass that through to the caller. If it returns `ParseError`, we pass that through too.

Here's the complete program showing incremental data arrival:

```pony
use "buffered"

primitive ParseError
  """
  The data in the buffer is malformed.
  """

type ParseResult is (String | I32 | None | ParseError)

primitive Parser
  fun _complete_size(buffer: Reader ref, offset: USize)
    : (USize | None | ParseError)
  =>
    try
      let tag = buffer.peek_u8(offset)?
      match tag
      | 0x01 =>
        let len = buffer.peek_u16_be(offset + 1)?
        let total = 1 + 2 + len.usize()
        if buffer.size() >= (offset + total) then
          total
        else
          None
        end
      | 0x02 =>
        let total: USize = 1 + 4
        if buffer.size() >= (offset + total) then
          total
        else
          None
        end
      else
        ParseError
      end
    else
      None
    end

  fun _parse(buffer: Reader ref): (String | I32 | ParseError) =>
    try
      let tag = buffer.u8()?
      match tag
      | 0x01 =>
        let len = buffer.u16_be()?
        String.from_array(buffer.block(len.usize())?)
      | 0x02 =>
        buffer.i32_be()?
      else
        ParseError
      end
    else
      // Unreachable: _complete_size confirmed the data is available.
      ParseError
    end

  fun apply(buffer: Reader ref): ParseResult =>
    match _complete_size(buffer, 0)
    | let _: USize => _parse(buffer)
    | None => None
    | ParseError => ParseError
    end

actor Main
  let _env: Env

  new create(env: Env) =>
    _env = env
    let buffer = Reader

    // First chunk: tag 0x02 (I32) arrives, but no payload yet
    buffer.append(recover val [as U8: 0x02] end)
    _print_result(Parser(buffer))

    // Second chunk: the 4-byte payload arrives
    buffer.append(recover val [as U8: 0x00; 0x00; 0x01; 0xA4] end)
    _print_result(Parser(buffer))

  fun _print_result(result: ParseResult) =>
    match result
    | let s: String => _env.out.print("Parsed string: " + s)
    | let n: I32 => _env.out.print("Parsed integer: " + n.string())
    | None => _env.out.print("Incomplete, waiting for more data...")
    | ParseError => _env.out.print("Error: malformed data")
    end
```

The first call to `Parser(buffer)` returns `None` because only the tag byte is in the buffer. No bytes are consumed. When the 4-byte payload arrives in the second chunk, the buffer now contains all 5 bytes (tag + payload), and the second call successfully parses the integer 420.

## Discussion

The `ParseResult` type is the [Error as Union Type](../error-handling/error-as-union-type.md) pattern applied to parsing. A partial function could only tell you "it worked" or "something went wrong." The union type distinguishes three fundamentally different outcomes: a parsed value means the caller can proceed, `None` means it should wait for more data and try again, and `ParseError` means the data is bad and the caller needs to take corrective action (close the connection, skip bytes, log an error). The caller's `match` handles each case, and the compiler verifies that every possibility is covered.

The offset parameter on peek methods is what makes the completeness check work for messages with variable-length fields. For the string type in our protocol, we first peek at the tag at offset 0, then peek at the 2-byte length field at offset 1. Only after reading the length do we know the total size of the message. Without offsets, we'd need to consume the tag and length to learn the total size, which is exactly the problem we're trying to avoid.

One refinement worth knowing about: peek methods on `Reader` are declared as `fun` (which means the receiver capability is `box`), while consume methods are `fun ref`. That means you could declare the completeness check parameter as `Reader box` instead of `Reader ref`. With that signature, the compiler would reject any attempt to call consume methods inside `_complete_size`, because `box` doesn't satisfy the `ref` requirement. The two-pass discipline becomes compiler-enforced rather than just convention. Most parsers don't bother with this refinement since the peek/consume split is already clear from the method names, but it's there if you want the extra safety net.

There's an asymmetry in `Reader`'s API worth calling out: there's no `peek_block` method. You can peek at individual integers (`peek_u8`, `peek_u16_be`, `peek_i32_be`, and so on), but there's no way to peek at an arbitrary byte sequence. For variable-length data like strings, you peek at the length field to find out how many bytes to expect, then check with `size()` that the buffer has enough total bytes. The actual byte extraction happens in the consume pass with `block()`. This is something you'll run into immediately when building a real parser, and it's why the completeness check focuses on byte counts rather than peeking at the payload itself.

This pattern scales naturally to nested and recursive protocols. Consider a protocol where one message type is an array containing other messages, like RESP (the Redis Serialization Protocol). The completeness check for an array would peek at the element count, then call `_complete_size` recursively for each element with an advancing offset. Each recursive call returns the size of that element, and the offset advances by that amount before checking the next one. If any nested element is incomplete, `None` propagates up and the entire buffer is untouched. No partial consumption, no matter how deeply nested the structure.

For real-world examples of this pattern, the [ponylang/redis](https://github.com/ponylang/redis) library's RESP parser is the canonical implementation. RESP has multiple types (simple strings, errors, integers, bulk strings, arrays), with arrays containing arbitrary nested elements. The parser uses exactly this two-pass approach: a completeness check that recurses through nested arrays using peek methods with advancing offsets, followed by a consume pass that extracts the data. The [Pony MessagePack](https://github.com/seantallen-org/msgpack) library applies the same pattern to the MessagePack binary serialization format, where dozens of format families each have their own length encoding schemes.

Everything in this pattern is presented in terms of `buffered.Reader`, but the core idea doesn't depend on it. Any buffering API that lets you inspect data without consuming it supports the same two-pass approach. The important thing is the separation: check completeness non-destructively, then consume only when you know the data is there.
