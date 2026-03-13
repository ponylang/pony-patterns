---
hide:
  - toc
---

# Resource Management Patterns

Some resources live outside Pony's garbage collector. FFI handles, file descriptors, event subscriptions: the runtime can reclaim the Pony object that wraps them, but it won't automatically clean up the underlying resource. That cleanup is your responsibility, and getting it wrong means leaks, double-frees, dangling pointers, or a program that simply refuses to exit.

The patterns in this chapter show how to manage these resources safely. For C library handles wrapped via FFI, there's the sentinel-and-finalizer approach with `dispose()` and `_final()`. For actors that subscribe to asynchronous events (TCP listeners, timers, signal handlers), there's the `DisposableActor` interface and `Custodian` for coordinated shutdown.
