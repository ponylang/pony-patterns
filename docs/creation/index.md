---
hide:
  - toc
---

# Creation Patterns

Pony constructors always return an initialized instance of their type. There's no `null`, no uninitialized state, and actor constructors can't be partial. Reference capabilities add another dimension: constructing a value with the right capability for its intended use sometimes requires specific techniques. These constraints push you toward patterns that might be unfamiliar if you're coming from languages where constructors can fail freely or return null.

[FFI Global Initializer](ffi-global-initializer.md) uses a primitive's `_init` method to run C library initialization exactly once, taking advantage of the fact that primitives are singletons.

[Recover for Isolated Return](recover-iso.md) shows how to build up mutable data inside a `recover` block and return it as `iso^`. This is the standard technique for writing functions that construct sendable values, and you'll find it throughout the standard library in places like `File.read` and `Directory.entries`.

[Static Constructor](static-constructor.md) wraps object construction in a primitive's `apply` method so it can return either the constructed object or a meaningful error, something Pony constructors can't do on their own.

[Supply Chain](supply-chain.md) solves the problem of actor constructors that depend on things that can fail. Rather than juggling `(File | None)` unions inside the actor, you initialize dependencies before constructing the actor and pass them in fully built.

[Typed Step Builder](typed-step-builder.md) enforces construction order at compile time. Each build step returns a different interface type, so the compiler prevents calling steps out of order or skipping required fields.
