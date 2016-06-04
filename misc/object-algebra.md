# Solving the Expression Problem with Object Algebra

## Problem

There's a real problem inherint in most data modelling we tend to do with our programming language. Its a problem that is so commin that Phillip Wadler gave it a name, "[the Expression Problem](http://homepages.inf.ed.ac.uk/wadler/papers/expression/expression.txt)". A full discussion of the Expression Problem is out of the scope of this pattern, please refer to the linked original formulation of the issue for a more in depth discussion. A summarized version of the Expression Problem is this:

### Expression Problem in Object-Oriented languages

The kinds of data abstraction found in object-oriented languages allows for easily adding new data variants via the class mechanism but adding new operations to existing variants is difficult because it requires changing the interface for all classes and the source code might be unavailable to us.

### Expression Problem in Functional languages

The kinds of data abstraction in functional languages allows for extending our data types with new operations but adding new adding new types is difficult. When we define our operations as functions, it is trivial to add a new function, however, those functions typically have to know about all the data types to be operated on and if you want to add a new data type, you have to modify the source code, which might be unavailable to us, of said functions.

### Expresion problem solutions

There are a variety of solutions to the Expression Problem in various languages, all with various levels of success. One such solution is "[Object Algebras](http://ropas.snu.ac.kr/~bruno/papers/ecoop2012.pdf)". 

This pattern shows how you can use Object Algrebras in Pony to solve the Expression Problem. The website "[Solutions to the Expression Problem Using Object Algebras](http://i.cs.hku.hk/~bruno/oa/)" has an excellent overview of the Expression Problem and its various constraints. In short these are that for any data type we should be able to add new operations and new data types of the same family while...

* Maintainting strong static type safety
* Not having to modify or duplicate existing code
* Compile time (not runtime) safety checks
* Ability to combine independently developed extensions

## Solution

The [Solutions to the Expression Problem Using Object Algebras](http://i.cs.hku.hk/~bruno/oa/) presents a number of solutions to the same problem in a variety of languages. What follows is a Pony solution modelled on the [Java version](http://i.cs.hku.hk/~bruno/oa/OA_J.zip) therein.

The problem presented is the need to create a simple interpreter that starts with two operations `lit` and `add` which we need to be able to `eval`. We then latter need to extend our data types by adding new operations `sub` and `print`.

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

interface PPrint
  fun print(): String

interface PrintExpAlg2
  fun lit(x: I32): String =>
    x.string()

  fun add(e1: String, e2: String): String =>
    e1 + " + " +  e2

  fun sub(e1: String, e2: String): String =>
    e1 + " - " +  e2

actor Main
  new create(env: Env) =>
    let ea = object is EvalExpAlg end
    let esa = object is (EvalSubExpAlg & EvalExpAlg) end
    let pa2 = object is PrintExpAlg2 end

    let ev = exp1[Eval val](esa)

    env.out.print("PRINTING\n" + exp1[String](pa2) + "\n" + exp2[String](pa2))

  fun exp1[E: Any #read](alg: ExpAlg[E] box): E =>
    alg.add(alg.lit(3), alg.lit(4))

  fun exp2[E: Any #read](alg: SubExpAlg[E] box): E =>
    alg.sub(exp1[E](alg), alg.lit(4))
```

## Discussion
