---
hide:
  - toc
---

# Object Algebra

## Problem

There's a real problem inherent in most data modeling we tend to do with our programming languages. It's so common that Philip Wadler gave it a name, "[the Expression Problem](https://homepages.inf.ed.ac.uk/wadler/papers/expression/expression.txt)". A full discussion is out of scope for this pattern — refer to the linked original formulation for the details. Here's the short version:

### Expression Problem in Object-Oriented Languages

Object-oriented data abstraction makes it easy to add new data variants via the class mechanism, but adding new operations to existing variants is difficult. It requires changing the interface for all classes, and the source code might not even be available to us.

### Expression Problem in Functional Languages

Functional data abstraction makes it easy to extend our data types with new operations, but adding new data types is difficult. When we define our operations as functions, adding a new function is trivial. However, those functions typically have to know about all the data types they operate on, and if we want to add a new data type, we have to modify their source code — which again might not be available to us.

### Solutions to the Expression Problem

There are a variety of solutions to the Expression Problem in various languages, all with various levels of success. One such solution is "[Object Algebras](/assets/extensibility-for-the-masses.pdf)". This pattern shows how you can use Object Algebras in Pony to solve the Expression Problem. The website "[Solutions to the Expression Problem Using Object Algebras](https://i.cs.hku.hk/~bruno/oa/)" has an excellent overview of the Expression Problem and its constraints. In short, for any data type we should be able to add new operations and new data types of the same family while:

* Maintaining strong static type safety
* Not having to modify or duplicate existing code
* Having compile time (not runtime) safety checks
* Being able to combine independently developed extensions

## Solution

The [Solutions to the Expression Problem Using Object Algebras](https://i.cs.hku.hk/~bruno/oa/) site presents solutions in a variety of languages. What follows is a Pony version modeled on the [Java version](https://i.cs.hku.hk/~bruno/oa/OA_J.zip).

The problem presented is the need to create a simple expression interpreter that starts with two operations, `lit` and `add`, which we need to be able to evaluate. We then need to extend our data types by adding a new operation `sub` and a new interpretation that pretty-prints the expression.

```pony
interface ExpAlg[E]
  fun lit(x: I32): E
  fun add(e1: E, e2: E): E

interface Eval
  fun eval(): I32

interface EvalExpAlg
  fun lit(x: I32): Eval val =>
    recover
      object
        let x: I32 = x
        fun eval(): I32 => x
      end
    end

  fun add(e1: Eval val, e2: Eval val): Eval val =>
    recover
      object
        let e1: Eval val = e1
        let e2: Eval val = e2
        fun eval(): I32 => e1.eval() + e2.eval()
      end
    end

interface SubExpAlg[E] is ExpAlg[E]
  fun sub(e1: E, e2: E): E

interface EvalSubExpAlg
  fun sub(e1: Eval val, e2: Eval val): Eval val =>
    recover
      object
        let e1: Eval val = e1
        let e2: Eval val = e2
        fun eval(): I32 => e1.eval() - e2.eval()
      end
    end

interface PrintExpAlg
  fun lit(x: I32): String =>
    x.string()

  fun add(e1: String, e2: String): String =>
    e1 + " + " + e2

  fun sub(e1: String, e2: String): String =>
    e1 + " - " + e2

actor Main
  new create(env: Env) =>
    let eval_alg = object is (EvalSubExpAlg & EvalExpAlg) end
    let print_alg = object is PrintExpAlg end

    env.out.print(exp1[Eval val](eval_alg).eval().string())
    env.out.print(exp1[String](print_alg))
    env.out.print(exp2[String](print_alg))

  fun exp1[E: Any #read](alg: ExpAlg[E] box): E =>
    alg.add(alg.lit(3), alg.lit(4))

  fun exp2[E: Any #read](alg: SubExpAlg[E] box): E =>
    alg.sub(exp1[E](alg), alg.lit(4))
```

The output of this program is:

```text
7
3 + 4
3 + 4 - 4
```

Let's walk through how this works. The core idea is that instead of representing expressions as a data structure like an AST, we define them through a factory interface that's generic in its result type.

```pony
interface ExpAlg[E]
  fun lit(x: I32): E
  fun add(e1: E, e2: E): E
```

`ExpAlg[E]` is our object algebra interface. It describes how to construct expressions, but the result type `E` is abstract. We don't know what an expression "is" — we only know how to build one from a literal and how to combine two of them with addition.

```pony
interface EvalExpAlg
  fun lit(x: I32): Eval val =>
    recover
      object
        let x: I32 = x
        fun eval(): I32 => x
      end
    end

  fun add(e1: Eval val, e2: Eval val): Eval val =>
    recover
      object
        let e1: Eval val = e1
        let e2: Eval val = e2
        fun eval(): I32 => e1.eval() + e2.eval()
      end
    end
```

`EvalExpAlg` provides a concrete interpretation where `E` becomes `Eval val`. Each factory method returns an anonymous object that knows how to evaluate itself. Pony's interfaces with default implementations are doing the heavy lifting here — we define the algebra as an interface with defaults, then create an instance with `object is EvalExpAlg end`.

Now suppose we need to add subtraction. In a traditional OOP approach, we'd have to modify our expression class hierarchy. With Object Algebras, we just extend the algebra interface:

```pony
interface SubExpAlg[E] is ExpAlg[E]
  fun sub(e1: E, e2: E): E
```

`SubExpAlg` extends `ExpAlg` with a new `sub` operation. No existing code is modified. We provide its evaluation the same way:

```pony
interface EvalSubExpAlg
  fun sub(e1: Eval val, e2: Eval val): Eval val =>
    ...
```

And when we need an algebra that can do both, we compose them:

```pony
let eval_alg = object is (EvalSubExpAlg & EvalExpAlg) end
```

Pony's structural typing makes this natural — we intersect the two interfaces and get an object that can evaluate addition and subtraction.

Adding a completely new interpretation — pretty printing — is equally straightforward:

```pony
interface PrintExpAlg
  fun lit(x: I32): String =>
    x.string()

  fun add(e1: String, e2: String): String =>
    e1 + " + " + e2

  fun sub(e1: String, e2: String): String =>
    e1 + " - " + e2
```

Here `E` becomes `String`. Same expression structure, different interpretation. Again, no existing code is modified.

Expressions themselves are built using generic functions that take the algebra as a parameter:

```pony
  fun exp1[E: Any #read](alg: ExpAlg[E] box): E =>
    alg.add(alg.lit(3), alg.lit(4))
```

The same expression `exp1` can be evaluated or printed depending on which algebra you pass in. That's the payoff — expressions are written once and interpreted many ways.

## Discussion

Object Algebras work particularly well in Pony because of two language features: interfaces with default method implementations and structural typing.

Interfaces with defaults let us define each algebra as an interface rather than a class. This means we can compose algebras with intersection types like `(EvalSubExpAlg & EvalExpAlg)` without needing class hierarchies. Structural typing means we don't need an explicit relationship between `EvalExpAlg` and `ExpAlg[Eval val]` — if it satisfies the interface, it works.

The pattern satisfies all four constraints of the Expression Problem:

* **New data variants** like `sub` are added by extending the algebra interface. Existing algebras are untouched.
* **New interpretations** like pretty printing are added by creating a new algebra with a different result type. Existing interpretations are untouched.
* **Type safety** is maintained — the generic type parameter `E` ensures that all parts of an expression agree on their result type.
* **Separate compilation** — each algebra can be compiled independently.

If you're building something that needs to be extended in both dimensions — new data variants and new operations over them — this pattern is worth considering. Interpreters, compilers, serialization formats, and document processors are all natural fits.

The [Solutions to the Expression Problem Using Object Algebras](https://i.cs.hku.hk/~bruno/oa/) site presents solutions in several languages beyond Java. We modeled our Pony version on the Java solution because it's the closest to what most Object Oriented programmers would be familiar with, but the other solutions could also be implemented in Pony.

For a deeper treatment, see the original "[Extensibility for the Masses](https://www.cs.utexas.edu/~wcook/Drafts/2012/ecoop2012.pdf)" paper by Bruno Oliveira and William Cook.

The [Parser Combinators](parser-combinators.md) pattern applies a similar compositional idea to a different domain. Where object algebras compose interpretations over a fixed set of operations, parser combinators compose operations (sequencing, choice, repetition) over a fixed interpretation. Both build complex behavior from small, reusable pieces; they just vary different axes.
