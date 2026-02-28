---
hide:
  - toc
---

# Resource Management Patterns

Some resources live outside Pony's garbage collector. FFI handles, file descriptors, event subscriptions: the runtime can reclaim the Pony object that wraps them, but it won't automatically clean up the underlying resource. That cleanup is your responsibility, and getting it wrong means leaks, double-frees, or dangling pointers.

The patterns in this chapter show how to manage these resources safely, using Pony's `dispose()` convention for explicit cleanup and `_final()` as a safety net during garbage collection.
