---
hide:
  - toc
---

# Visitor

## Problem

If you've spent time in the object-oriented world, you've probably run into the Visitor pattern. The Gang of Four catalogued it in 1994, and it's been a workhorse ever since: separate the algorithm from the data structure it operates on, so you can add new operations without touching the structure. In Java or C++, that means double dispatch, accept methods on every node, and a fair bit of boilerplate. Pony strips most of that away, but the core idea is just as useful.

Here's when you need it. You have a data structure and code that walks it to produce output. Consider a simple document made of literal text and variable references:

```pony
use "collections"

primitive _Literal
primitive _Variable
type _Segment is ((_Literal, String) | (_Variable, String))

class val Document
  let _segments: Array[_Segment] val

  new val create(segments: Array[_Segment] val) =>
    _segments = segments

  fun render(values: Map[String, String] box): String =>
    var result: String = ""
    for segment in _segments.values() do
      match segment
      | (_Literal, let text: String) =>
        result = result + text
      | (_Variable, let name: String) =>
        try result = result + values(name)? end
      end
    end
    result
```

This works fine when all you need is a string. Then you need the literal text and the dynamic values as separate arrays, maybe for parameterized database queries or to hand off to a frontend that applies its own formatting. So you write `render_split`:

```pony
  fun render_split(
    values: Map[String, String] box
  ): (Array[String] ref, Array[String] ref) =>
    let statics = Array[String]
    let dynamics = Array[String]
    for segment in _segments.values() do
      match segment
      | (_Literal, let text: String) =>
        statics.push(text)
      | (_Variable, let name: String) =>
        try dynamics.push(values(name)?) end
      end
    end
    (statics, dynamics)
```

Look at the two methods side by side. The walking logic is identical: the same loop, the same match, the same segment types. The only thing that differs is what happens inside each branch. Two methods is tolerable. Add HTML escaping, Markdown output, and a debug dumper, and you're maintaining five copies of the same walk. Fix a bug in the traversal and you get to fix it five times.

## Solution

Pull the "what to do with each segment" part out into an interface. The walker calls the interface; different implementations decide what happens with each piece.

```pony
interface ref DocumentSink
  fun ref literal(text: String)
  fun ref dynamic_value(value: String)
```

The interface has one method per segment type. It's `ref` because implementations will almost certainly accumulate state: building a string, collecting arrays, writing to an output stream.

The walker becomes a single method that pushes segments to whatever sink it receives:

```pony
  fun walk(sink: DocumentSink ref, values: Map[String, String] box) =>
    for segment in _segments.values() do
      match segment
      | (_Literal, let text: String) =>
        sink.literal(text)
      | (_Variable, let name: String) =>
        try
          sink.dynamic_value(values(name)?)
        else
          sink.dynamic_value("")
        end
      end
    end
```

One method. One place where the structure is traversed. New output strategies are new classes, not new methods on `Document`.

String rendering becomes a sink that concatenates everything:

```pony
class ref StringSink is DocumentSink
  var _result: String ref = String

  fun ref literal(text: String) =>
    _result.append(text)

  fun ref dynamic_value(value: String) =>
    _result.append(value)

  fun ref result(): String =>
    _result.clone()
```

Split rendering becomes a sink that collects the two segment types into separate arrays:

```pony
class ref SplitSink is DocumentSink
  let _statics: Array[String] = Array[String]
  let _dynamics: Array[String] = Array[String]

  fun ref literal(text: String) =>
    _statics.push(text)

  fun ref dynamic_value(value: String) =>
    _dynamics.push(value)

  fun statics(): this->Array[String] =>
    _statics

  fun dynamics(): this->Array[String] =>
    _dynamics
```

An HTML-escaping sink would override `dynamic_value` to escape its input while passing literals through unchanged. A streaming sink would write each piece to a file as it arrives. Each output strategy is its own class, and adding one never means touching the walker.

Here's the complete program:

```pony
use "collections"

primitive _Literal
primitive _Variable
type _Segment is ((_Literal, String) | (_Variable, String))

interface ref DocumentSink
  fun ref literal(text: String)
  fun ref dynamic_value(value: String)

class val Document
  let _segments: Array[_Segment] val

  new val create(segments: Array[_Segment] val) =>
    _segments = segments

  fun walk(sink: DocumentSink ref, values: Map[String, String] box) =>
    for segment in _segments.values() do
      match segment
      | (_Literal, let text: String) =>
        sink.literal(text)
      | (_Variable, let name: String) =>
        try
          sink.dynamic_value(values(name)?)
        else
          sink.dynamic_value("")
        end
      end
    end

class ref StringSink is DocumentSink
  var _result: String ref = String

  fun ref literal(text: String) =>
    _result.append(text)

  fun ref dynamic_value(value: String) =>
    _result.append(value)

  fun ref result(): String =>
    _result.clone()

class ref SplitSink is DocumentSink
  let _statics: Array[String] = Array[String]
  let _dynamics: Array[String] = Array[String]

  fun ref literal(text: String) =>
    _statics.push(text)

  fun ref dynamic_value(value: String) =>
    _dynamics.push(value)

  fun statics(): this->Array[String] =>
    _statics

  fun dynamics(): this->Array[String] =>
    _dynamics

actor Main
  new create(env: Env) =>
    let doc = Document(
      recover val
        let s = Array[_Segment]
        s.push((_Literal, "Hello "))
        s.push((_Variable, "name"))
        s.push((_Literal, ", you have "))
        s.push((_Variable, "count"))
        s.push((_Literal, " messages."))
        s
      end)

    let values = Map[String, String]
    values("name") = "Alice"
    values("count") = "3"

    // String rendering
    let string_sink: StringSink ref = StringSink
    doc.walk(string_sink, values)
    env.out.print("String: " + string_sink.result())

    // Split rendering
    let split_sink: SplitSink ref = SplitSink
    doc.walk(split_sink, values)
    env.out.print("Statics:")
    for s in split_sink.statics().values() do
      env.out.print("  \"" + s + "\"")
    end
    env.out.print("Dynamics:")
    for d in split_sink.dynamics().values() do
      env.out.print("  \"" + d + "\"")
    end
```

## Discussion

In the classic Gang of Four formulation, the Visitor pattern requires double dispatch. The data structure's nodes each have an `accept(visitor)` method that calls `visitor.visit(this)`, and the visitor interface has a `visit` overload for every concrete node type. This is how Java and C++ work around single dispatch: you need two virtual calls to select the right method for both the node type and the visitor type.

Pony doesn't need any of that. The walker identifies element types with `match` on a union type, so there are no accept methods and no class hierarchy on the data structure side. The sink interface uses structural subtyping: any class with `literal(String)` and `dynamic_value(String)` methods satisfies `DocumentSink` whether or not it declares `is DocumentSink`. The declaration on the sink classes is documentation and a compile-time check, not a requirement. You could remove it and the walker would still accept them. What remains of the GoF Visitor is its essence: the interface, the walk, and the separation of concerns, without the dispatch gymnastics.

At a mechanical level, this looks a lot like the [Notifier](../code-sharing/notifier.md) pattern: both define an interface with callback methods, and a driver calls those methods at the right moments. The difference is purpose. A notifier specializes an actor's behavior at predefined lifecycle points. The actor decides when to call the notifier, and there's typically one notifier per actor instance. A visitor separates traversal from processing. The walker drives through a data structure, and different visitors define different operations over the same walk. Notifier answers "what should this actor do when X happens?" Visitor answers "what should we do with each piece of this structure?"

A visitor interface can carry guarantees beyond what the method signatures express. The [ponylang/templates](https://github.com/ponylang/templates) library uses this pattern for template rendering, and its `TemplateSink` interface guarantees strict alternation between `literal` and `dynamic_value` calls. Rendering always starts and ends with `literal`, with exactly N+1 literal calls for N dynamic values. Implementations can rely on this: a sink that builds separate arrays knows the statics array will always have one more element than the dynamics array. Pony's type system can't encode call-sequence constraints like this, so the guarantee lives in the walker's implementation and the interface's documentation. When your own visitor carries ordering properties, document them on the interface.

This pattern pays off when multiple consumers need to process the same data structure differently. One walk method, many output strategies. If you only ever need one output format, the extra interface is ceremony for its own sake. But once a second consumer shows up, the Visitor earns its keep by giving you a clean extension point that doesn't require touching the walker.
