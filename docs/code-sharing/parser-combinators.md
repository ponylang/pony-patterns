---
hide:
  - toc
---

# Parser Combinators

## Problem

You need to recognize structured text. Maybe it's a configuration format, a query language, or just a number with an optional sign and decimal point. Given a string, you want to answer one question: is this a valid number or not? `"42"` yes. `"-7"` yes. `"3.14"` yes. `"abc"` no. `""` no. Number or not number.

You could reach for a regular expression, but regexes get unreadable fast and can't handle recursive structures like nested parentheses. Another common approach is to write a parser: a function that walks through the input character by character, checks whether it matches some expected pattern, and tracks how far it got. Parsers have a reputation for being scary, but the concept is simple. You can build small parsers for each piece of the grammar (match a literal string, match a character in a range) and then combine them with functions like "try A then B" or "try A, and if that fails, try B instead."

The rules for our number are simple enough to state in one line:

```text
number = '-'? (digit19 digits | digit) ('.' digits)?
```

An optional minus sign, followed by either a multi-digit integer (starting with 1-9) or a single digit, optionally followed by a decimal point and more digits. Clear enough on paper. But try to express that with parsing functions:

```pony
let number = sequence(
  opt(literal("-")),
  sequence(
    choice(
      sequence(range('1', '9'), many1(range('0', '9'))),
      range('0', '9')),
    opt(sequence(literal("."), many1(range('0', '9'))))))
```

It works, but the structure of the code doesn't resemble the structure of the grammar. You read it inside-out, matching parentheses carefully to figure out what's nested inside what. Changing the grammar means restructuring the nesting, which means getting the parentheses right again. As grammars grow, this gets worse.

The idea of combining small parsers into bigger ones is sound. The problem is that the combining functions bury the grammar's structure under layers of nesting. Parser combinators clean this up.

## Solution

A parser combinator is a function that takes one or more parsers as input and returns a new parser. "Match A then B" is a combinator called _sequence_. "Match A or else try B" is _choice_. "Match A zero or one times" is _optional_. You build complex grammars by snapping these pieces together, the same way you'd describe the grammar in English: "a minus sign, optionally, followed by some digits."

The combining functions from the Problem section are already combinators. They just don't read well. The fix is operator overloading. If sequence is `*` and choice is `/`, the grammar reads like a specification instead of a puzzle.

### The parser trait

The core is a trait that defines two things: how to parse, and how to combine parsers using operators.

```pony
trait box Parser
  fun parse(input: String val, offset: USize): (Bool, USize)

  fun mul(that: Parser): Parser =>
    Sequence(this, that)

  fun div(that: Parser): Parser =>
    Choice(this, that)

  fun opt(): Parser =>
    Optional(this)

  fun many1(): Parser =>
    Many(this)
```

Every parser takes a string and an offset, and returns whether it matched and where parsing ended up. Pony maps `*` to `mul` and `/` to `div`, so any two parsers can be sequenced with `*` or alternated with `/`. The `opt()` and `many1()` methods handle repetition. Each combinator method returns a new `Parser`, which means the result can immediately be combined again. That's what makes the chaining work.

The trait is `box`, meaning combinators only need read access to the parsers they contain. Any `ref`, `val`, `trn`, or `box` parser can be stored as `box`; the combinator doesn't need to write to its children, just call `parse` on them. The combinator methods capture `this` and store it inside a new combinator object, building a tree of parsers as you compose them.

### Leaf parsers

Leaf parsers are the bottom of the tree. They look directly at the input text rather than delegating to other parsers. Everything else is built on top of them.

#### Literal

`Literal` matches an exact string at the current position:

```pony
class Literal is Parser
  let _text: String val

  new create(text: String val) =>
    _text = text

  fun parse(input: String val, offset: USize): (Bool, USize) =>
    var i: USize = 0
    try
      while i < _text.size() do
        if input(offset + i)? != _text(i)? then
          return (false, offset)
        end
        i = i + 1
      end
      (true, offset + _text.size())
    else
      (false, offset)
    end
```

It walks through the input byte by byte, comparing each one against the stored text. The `?` on the array indexing marks the call as partial: indexing can fail if the offset is past the end of the input, so the compiler requires us to handle that. The `try` block wraps the whole loop; if any index is out of bounds, the `else` branch catches the error and returns a non-match. When every byte matches, it returns `true` and advances the position past the matched text.

#### CharRange

`CharRange` matches a single character that falls within a range:

```pony
class CharRange is Parser
  let _lo: U8
  let _hi: U8

  new create(lo: U8, hi: U8) =>
    _lo = lo
    _hi = hi

  fun parse(input: String val, offset: USize): (Bool, USize) =>
    try
      let c = input(offset)?
      if (c >= _lo) and (c <= _hi) then
        (true, offset + 1)
      else
        (false, offset)
      end
    else
      (false, offset)
    end
```

Same structure as `Literal` but simpler. It reads one byte from the input, checks whether it falls between `_lo` and `_hi` inclusive, and advances by one position on a match. The `try`/`else` handles the same out-of-bounds case: if there's no character at the current offset, that's a non-match.

### Combinators

Leaf parsers look at the input directly. Combinators don't. A combinator is a parser that wraps one or more other parsers and coordinates how they run. It implements the same `Parser` trait, so from the outside it looks just like a leaf. But inside, it delegates to its children and decides what to do based on whether they succeed or fail. That's the whole trick: because both leaves and combinators are `Parser`, you can nest them freely.

#### Sequence

"Sequence" means "match A then B." It takes two parsers and runs them one after the other. If the first one matches, the second one picks up where the first left off. If either fails, the whole thing fails:

```pony
class Sequence is Parser
  let _left: Parser
  let _right: Parser

  new create(left: Parser, right: Parser) =>
    _left = left
    _right = right

  fun parse(input: String val, offset: USize): (Bool, USize) =>
    (let ok, let pos) = _left.parse(input, offset)
    if ok then
      _right.parse(input, pos)
    else
      (false, offset)
    end
```

Notice that `_right` receives `pos`, the position where `_left` finished. That's how parsers chain: each one picks up where the previous one stopped.

#### Choice

"Choice" means "match A or, failing that, try B." It takes two parsers, tries the first, and falls back to the second only if the first fails:

```pony
class Choice is Parser
  let _left: Parser
  let _right: Parser

  new create(left: Parser, right: Parser) =>
    _left = left
    _right = right

  fun parse(input: String val, offset: USize): (Bool, USize) =>
    (let ok, let pos) = _left.parse(input, offset)
    if ok then
      (true, pos)
    else
      _right.parse(input, offset)
    end
```

When the left parser fails, `Choice` passes the original `offset` to the right parser, not `pos`. The first parser's failure didn't consume any input, so the second one starts from the same place.

#### Optional

`Optional` wraps a single parser and always succeeds. If the inner parser matches, it advances. If not, it stays put and reports success anyway:

```pony
class Optional is Parser
  let _inner: Parser

  new create(inner: Parser) =>
    _inner = inner

  fun parse(input: String val, offset: USize): (Bool, USize) =>
    (let ok, let pos) = _inner.parse(input, offset)
    if ok then (true, pos) else (true, offset) end
```

Both branches return `true`. The only difference is the position: either advanced past the match or unchanged.

#### Many

`Many` requires at least one match, then keeps running the inner parser until it fails. It's the "one or more" combinator:

```pony
class Many is Parser
  let _inner: Parser

  new create(inner: Parser) =>
    _inner = inner

  fun parse(input: String val, offset: USize): (Bool, USize) =>
    var result = _inner.parse(input, offset)
    if not result._1 then return (false, offset) end
    var pos = result._2
    var running = true
    while running do
      result = _inner.parse(input, pos)
      if result._1 then
        pos = result._2
      else
        running = false
      end
    end
    (true, pos)
```

The first call to the inner parser is the required one. If it fails, `Many` fails. After that, the `while` loop greedily consumes as many matches as it can, advancing `pos` each time. When the inner parser finally fails, the loop stops and `Many` returns however far it got.

### Putting it together

First, short type aliases keep the grammar concise:

```pony
type L is Literal
type R is CharRange
```

Then we build up the grammar from small pieces. Each `let` binding creates a parser that we can use in later definitions:

```pony
let digit = R('0', '9')
let digit19 = R('1', '9')
let digits = digit.many1()
```

`digit` matches any character from `'0'` to `'9'`. `digit19` matches `'1'` through `'9'` (no leading zeros). `digits` is `digit.many1()`, which means "one or more digits." Remember, `many1()` is defined on the `Parser` trait, so calling it on `digit` wraps it in a `Many` combinator and returns a new parser.

Now the full number grammar:

```pony
let number =
  L("-").opt() * ((digit19 * digits) / digit) * (L(".") * digits).opt()
```

Reading left to right: `L("-").opt()` creates a `Literal` parser for `"-"` and wraps it in `Optional`, so the minus sign is allowed but not required. The `*` after it is sequence (the `mul` method from the trait), so whatever comes next must follow the optional minus.

Inside the parentheses, `(digit19 * digits) / digit` is a choice (the `div` method). It first tries a multi-digit integer: a digit from 1-9 followed by one or more digits. If that fails, it falls back to a single digit. The choice is ordered this way because `digit` alone would match the first character of `"42"` and stop, leaving the `"2"` unconsumed.

Finally, `(L(".") * digits).opt()` handles the optional decimal part: a literal `"."` followed by one or more digits, all wrapped in `Optional`.

Compare that to the nested function calls in the Problem section. The structure of the code now mirrors the structure of the grammar.

### Complete program

```pony
trait box Parser
  fun parse(input: String val, offset: USize): (Bool, USize)

  fun mul(that: Parser): Parser =>
    Sequence(this, that)

  fun div(that: Parser): Parser =>
    Choice(this, that)

  fun opt(): Parser =>
    Optional(this)

  fun many1(): Parser =>
    Many(this)

class Literal is Parser
  let _text: String val

  new create(text: String val) =>
    _text = text

  fun parse(input: String val, offset: USize): (Bool, USize) =>
    var i: USize = 0
    try
      while i < _text.size() do
        if input(offset + i)? != _text(i)? then
          return (false, offset)
        end
        i = i + 1
      end
      (true, offset + _text.size())
    else
      (false, offset)
    end

class CharRange is Parser
  let _lo: U8
  let _hi: U8

  new create(lo: U8, hi: U8) =>
    _lo = lo
    _hi = hi

  fun parse(input: String val, offset: USize): (Bool, USize) =>
    try
      let c = input(offset)?
      if (c >= _lo) and (c <= _hi) then
        (true, offset + 1)
      else
        (false, offset)
      end
    else
      (false, offset)
    end

class Sequence is Parser
  let _left: Parser
  let _right: Parser

  new create(left: Parser, right: Parser) =>
    _left = left
    _right = right

  fun parse(input: String val, offset: USize): (Bool, USize) =>
    (let ok, let pos) = _left.parse(input, offset)
    if ok then
      _right.parse(input, pos)
    else
      (false, offset)
    end

class Choice is Parser
  let _left: Parser
  let _right: Parser

  new create(left: Parser, right: Parser) =>
    _left = left
    _right = right

  fun parse(input: String val, offset: USize): (Bool, USize) =>
    (let ok, let pos) = _left.parse(input, offset)
    if ok then
      (true, pos)
    else
      _right.parse(input, offset)
    end

class Optional is Parser
  let _inner: Parser

  new create(inner: Parser) =>
    _inner = inner

  fun parse(input: String val, offset: USize): (Bool, USize) =>
    (let ok, let pos) = _inner.parse(input, offset)
    if ok then (true, pos) else (true, offset) end

class Many is Parser
  let _inner: Parser

  new create(inner: Parser) =>
    _inner = inner

  fun parse(input: String val, offset: USize): (Bool, USize) =>
    var result = _inner.parse(input, offset)
    if not result._1 then return (false, offset) end
    var pos = result._2
    var running = true
    while running do
      result = _inner.parse(input, pos)
      if result._1 then
        pos = result._2
      else
        running = false
      end
    end
    (true, pos)

type L is Literal
type R is CharRange

actor Main
  new create(env: Env) =>
    let digit = R('0', '9')
    let digit19 = R('1', '9')
    let digits = digit.many1()
    let number =
      L("-").opt() * ((digit19 * digits) / digit) * (L(".") * digits).opt()

    let tests = ["42"; "-7"; "3.14"; "abc"; ""]
    for input in tests.values() do
      (let ok, let pos) = number.parse(input, 0)
      if ok and (pos == input.size()) then
        env.out.print("\"" + input + "\" => match")
      else
        env.out.print("\"" + input + "\" => no match")
      end
    end
```

The output:

```text
"42" => match
"-7" => match
"3.14" => match
"abc" => no match
"" => no match
```

## Discussion

### Where combinators fit

The Problem section mentioned regexes and hand-written parsers. There's a third alternative worth knowing about: parser generators like yacc or ANTLR. You write the grammar in a special notation, then run a tool that generates parsing code. That's powerful but introduces a build step, a separate language to learn, and generated code that's hard to debug.

Parser combinators avoid all of that. The grammar lives right next to the code that uses it and is written in the same language. No external tools, no generated code, no separate grammar files. The tradeoff is performance: parser combinators are generally slower than generated parsers for large grammars. But for the kinds of grammars most applications deal with (config formats, DSLs, structured input), the difference rarely matters, and being able to read your grammar at a glance is worth a lot.

### Why Pony

This pattern works in any language with operator overloading, but Pony makes it particularly clean. Pony maps arithmetic operators directly to method names: `*` calls `mul`, `/` calls `div`. Because these are regular methods defined on a trait, any type that implements `Parser` gets the operators for free. There's no special syntax, no macros, no metaprogramming. You define methods on a trait and the operators just work.

The `box` capability on the trait matters too. Combinators store their children as `Parser box`, which means they only need read access. You can build the entire parser tree out of `ref` objects, `val` objects, or a mix, and everything composes without capability conflicts. The type system stays out of the way while still enforcing the rules.

### What real parsers add

Our example only recognizes whether text matches a grammar. It returns `true` or `false` and a position. A real parser needs to do more than that.

Most parsers build an abstract syntax tree (AST) as they match. Instead of just knowing "this is a valid number," you get a data structure that says "here's a number with a negative sign, integer part 3, and fractional part 14." That's the structure your program actually works with.

Real parsers also report errors with location information. When parsing fails, "expected a digit at line 3, column 12" is a lot more helpful than `false`. They handle whitespace so the grammar doesn't have to mention spaces everywhere. And they support recursive grammars, where a rule can refer to itself. Think matching nested parentheses, or an expression language where `1 + (2 * 3)` is valid. Our example can't express that because a parser would need to reference itself before it's been created.

Pony's [`peg`](https://github.com/ponylang/peg) library is a full parser combinator implementation that handles all of these: ASTs, error reporting, whitespace, and recursive grammars. [`changelog-tool`](https://github.com/ponylang/changelog-tool), which validates and manipulates CHANGELOG files, is a good example of `peg` in production with a real grammar.

### Beyond parsing

The technique isn't limited to parsing. Any domain where you compose small operations into larger ones can benefit from the same approach: define a trait with the composition operators, implement the leaves and combinators, and let operator overloading turn the construction code into something that reads like a specification. Query builders, validation pipelines, and workflow definitions are all natural fits. The key ingredient is a closed set of composition operators that Pony's operator overloading can express.

If the compositional idea resonates, the [Object Algebra](object-algebra.md) pattern takes it in a different direction. Where parser combinators compose operations (sequencing, choice, repetition) over a fixed interpretation, object algebras compose interpretations over a fixed set of operations. Both patterns build complex behavior from small, reusable pieces; they just vary different axes.

### Learn more

- [Monadic Parser Combinators](https://www.cs.nott.ac.uk/~pszgmh/monparsing.pdf) by Graham Hutton and Erik Meijer. The foundational paper on building parsers from composable functions. It uses Haskell, but the ideas translate to any language.
- [Parsing Expression Grammars](https://bford.info/pub/lang/peg.pdf) by Bryan Ford. PEGs are the formal grammar theory behind most combinator libraries, including Pony's `peg`. The key insight is ordered choice: try alternatives in order and commit to the first one that matches.
- The Pony [`peg`](https://github.com/ponylang/peg) library. A real parser combinator library for Pony that builds on the same ideas shown here.
