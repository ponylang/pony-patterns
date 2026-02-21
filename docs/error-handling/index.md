---
hide:
  - toc
---

# Error Handling Patterns

Pony's built-in error mechanism is deliberately simple: a partial function either succeeds or raises `error`, and the caller's `else` block handles the failure. There's no exception hierarchy, no error message, no way to distinguish one failure from another.

That simplicity is a feature — it keeps the runtime lean and the semantics clear. But when a function can fail for multiple distinct reasons, the caller often needs to know *which* reason. The patterns in this chapter use Pony's type system to make error conditions explicit, typed, and compiler-checked.
