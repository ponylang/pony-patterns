# Pony Patterns

Welcome to Pony Patterns, a cookbook of recipes for getting things done in Pony. If you've ever found yourself staring at a Pony program wondering "how do I actually do this?", you're in the right place. Each pattern addresses a concrete problem you'll run into and walks you through a solution.

Pony is different from most languages you've used. No blocking. No data races. Reference capabilities instead of locks. These differences are what make Pony great, but they also mean that the tricks you learned in other languages don't always translate. This site is here to bridge that gap.

## What's here

Patterns are organized by the kind of problem they solve.

**[Asynchronous Patterns](async/index.md)** deal with the fact that Pony has zero blocking operations. Need to wait for something? Want to query an actor's state? Need to know when a bunch of workers are done? These patterns show you how.

**[Behavioral Patterns](behavioral/index.md)** cover how to structure the runtime behavior of your actors and objects. State machines, dispatch logic, that sort of thing.

**[Code Sharing Patterns](code-sharing/index.md)** are about reuse. Pony doesn't have inheritance, so if you're coming from a language that does, you'll want to look here. Composition, delegation, mixins, notifiers: there are plenty of ways to share code without an `extends` keyword.

**[Creation Patterns](creation/index.md)** address object construction. Builders, static constructors, supply chains, and how to deal with FFI initialization that needs to happen exactly once.

**[Data Sharing Patterns](data-sharing/index.md)** tackle one of Pony's most common stumbling blocks: getting data from one actor to another. Reference capabilities make this safe, but you need to know the patterns. Copying, isolated fields, persistent data structures: pick the one that fits.

**[Domain Modeling Patterns](domain-modeling/index.md)** help you encode your problem domain into Pony's type system so that invalid states are unrepresentable.

**[Error Handling Patterns](error-handling/index.md)** go beyond Pony's built-in `error` keyword. Union types give you richer error handling where the compiler helps you cover every case.

**[Object Capabilities Patterns](object-capabilities/index.md)** show how to use Pony's object capability system to control what parts of your program can do. Authority hierarchies, single-use capabilities, and the discipline that makes capability security practical.

**[Performance Patterns](performance/index.md)** are for when you need to squeeze out more speed. Avoiding boxing, short-circuiting, preallocating, and keeping string allocations under control.

**[Resource Management Patterns](resource-management/index.md)** cover how to manage external resources, particularly when working with C libraries through Pony's FFI.

**[Streaming Patterns](streaming/index.md)** address processing data as it arrives rather than all at once.

**[Testing Patterns](testing/index.md)** help you test Pony code, especially the tricky parts. Actors are asynchronous, so testing them requires some specific techniques.

## Getting help

If you're stuck on a Pony problem (pattern-related or otherwise), the [Pony Zulip community](https://ponylang.zulipchat.com/) is the place to go. The `#beginner help` channel is there for exactly that purpose. No question is too basic. The community is volunteer-based, so be patient, but folks are genuinely welcoming.

## Contributing

We'd love more patterns. If you've solved a problem in Pony and think others would benefit from seeing how, consider contributing it.

The best place to start is by reading a few existing patterns to get a feel for the format and tone. Each pattern explains a problem, walks through the solution, and includes working code. Keep that structure and you're most of the way there.

If you want to discuss an idea before writing it up, drop into the `#contribute to Pony` channel on [Zulip](https://ponylang.zulipchat.com/). Or if you'd rather just dive in, open a pull request on the [GitHub repo](https://github.com/ponylang/pony-patterns). Either way works.
