---
hide:
  - toc
---

# FFI Resource Lifecycle

## Problem

You're wrapping a C library via [Pony's C-FFI](https://tutorial.ponylang.io/c-ffi/). The library gives you a pointer handle that you're responsible for freeing when you're done with it. A straightforward wrapper might look like this:

```pony
use @img_load[Pointer[None]](path: Pointer[U8] tag)
use @img_width[U32](handle: Pointer[None])
use @img_height[U32](handle: Pointer[None])
use @img_free[None](handle: Pointer[None])

class Image
  let _handle: Pointer[None]

  new create(path: String) =>
    _handle = @img_load(path.cstring())

  fun width(): U32 =>
    @img_width(_handle)

  fun height(): U32 =>
    @img_height(_handle)

  fun ref close() =>
    @img_free(_handle)
```

This works until it doesn't. Three things can go wrong:

1. **Resource leak.** If the caller forgets to call `close()`, the C library never frees its internal memory. Pony's garbage collector will eventually reclaim the `Image` object, but it has no idea about the C-side allocation.

2. **Double-free.** If the caller calls `close()` twice (maybe from two different code paths that both try to clean up), the second call passes the same pointer to `img_free`. Depending on the C library, this could corrupt memory or crash.

3. **Dangling pointer.** If the caller calls `close()` and then `width()`, you're passing a freed pointer to `img_width`. The C library might return garbage, crash, or worse.

The root cause is the same in all three cases: nothing in the code tracks whether the handle is still valid.

Pony is a memory-safe language, and Pony users expect memory safety (and concurrency safety) to be guaranteed for any program that compiles. An FFI-wrapping Pony package is responsible for carefully upholding these guarantees, and any package that fails to do so will be distrusted and unused in the Pony community.

## Solution

The fix is to track the handle's validity inside the wrapper itself using a sentinel value. A sentinel is a known-invalid value that marks the resource as "already released." Combined with Pony's `dispose()` convention for explicit cleanup and `_final()` as a garbage-collection safety net, this gives you three guarantees: no resource leaks (the finalizer catches forgotten cleanups), no double-frees (the sentinel prevents freeing twice), and no dangling pointer access (every method checks the sentinel before touching the handle).

The first change is small but essential. The handle becomes `var` instead of `let`:

```pony
class Image
  var _handle: Pointer[None]

  new create(path: String) =>
    _handle = @img_load(path.cstring())
```

That one-character change makes the rest of the pattern possible. After freeing the resource, we can set the handle to a null pointer to signal that it's been released. For pointer handles, null is the natural sentinel: it's a value the C library would never return for a valid resource, and `Pointer` has a built-in `is_null()` method.

With the sentinel in place, `dispose()` can check whether the resource has already been freed before doing anything:

```pony
  fun ref dispose() =>
    if not _handle.is_null() then
      @img_free(_handle)
      _handle = Pointer[None]
    end
```

The null check prevents double-frees. After freeing, we set the handle to `Pointer[None]` (a null pointer) so any subsequent call to `dispose()` is a no-op. The name `dispose` follows Pony's convention for explicit resource cleanup.

Next, `_final()` acts as a safety net. Pony calls it during garbage collection, so even if the caller forgets to call `dispose()`, the C resource still gets freed:

```pony
  fun _final() =>
    if not _handle.is_null() then
      @img_free(_handle)
    end
```

It looks almost identical to `dispose()`, with one difference: it doesn't set the sentinel afterward. The finalizer runs exactly once during garbage collection, so there's no re-entry to guard against and no reason to update state that nobody will read again.

If `dispose()` was already called, the sentinel is null and `_final()` skips the free. If `dispose()` was never called, `_final()` frees the resource. Either way, the C library sees exactly one call to `img_free`.

Finally, every method that touches the handle needs the same guard. After `dispose()`, the handle is a null pointer, and passing it to any C function is undefined behavior:

```pony
  fun width(): U32 =>
    if _handle.is_null() then return 0 end
    @img_width(_handle)

  fun height(): U32 =>
    if _handle.is_null() then return 0 end
    @img_height(_handle)
```

When the resource has been disposed, these methods return a safe default instead of calling into the C library with an invalid pointer.

Here's the complete wrapper with all the pieces together:

```pony
use @img_load[Pointer[None]](path: Pointer[U8] tag)
use @img_width[U32](handle: Pointer[None])
use @img_height[U32](handle: Pointer[None])
use @img_free[None](handle: Pointer[None])

class Image
  var _handle: Pointer[None]

  new create(path: String) =>
    _handle = @img_load(path.cstring())

  fun width(): U32 =>
    if _handle.is_null() then return 0 end
    @img_width(_handle)

  fun height(): U32 =>
    if _handle.is_null() then return 0 end
    @img_height(_handle)

  fun ref dispose() =>
    if not _handle.is_null() then
      @img_free(_handle)
      _handle = Pointer[None]
    end

  fun _final() =>
    if not _handle.is_null() then
      @img_free(_handle)
    end

actor Main
  new create(env: Env) =>
    let img = Image("photo.png")
    env.out.print(
      "Size: " + img.width().string() + "x" + img.height().string())
    img.dispose()
```

## Discussion

Why have both `dispose()` and `_final()`? Pony's garbage collector runs on its own schedule. If your code creates an `Image`, uses it, and lets it go out of scope, the GC will eventually collect the `Image` object and run `_final()`. But "eventually" might mean the C library holds onto a large allocation for much longer than necessary. `dispose()` gives callers a way to free the resource immediately when they know they're done with it. Think of `_final()` as a safety net: it catches leaks from code paths that forgot to call `dispose()`, but it shouldn't be your primary cleanup mechanism.

You might notice that `dispose()` sets the handle to `Pointer[None]` after freeing, but `_final()` doesn't bother. That's intentional. `dispose()` can be called from any code path at any time, so it needs the sentinel to prevent double-frees if someone calls it again. The finalizer runs exactly once during garbage collection, so there's no re-entry to guard against. If `dispose()` already ran, the sentinel is null and `_final()` simply skips the free.

Guarding isn't just for cleanup methods. After `dispose()`, the handle points to freed memory, and passing it to any C function is undefined behavior. Every method that touches the handle needs to check the sentinel first. Without those guards, a caller who disposes an image and then accidentally calls `width()` would pass a freed pointer to the C library. The guard turns that into a safe no-op that returns a default value. The choice of default depends on the method. For dimensions, 0 is reasonable. For methods where no default makes sense, you could return a union type that includes an error (see the [Error as Union Type](../error-handling/error-as-union-type.md) pattern).

The sentinel value depends on the kind of handle you're wrapping. For pointer handles (`Pointer[None]`), a null pointer is the natural choice: it's a value the C library would never return for a valid resource, and `Pointer` has a built-in `is_null()` check. For integer handles like file descriptors, -1 is the conventional invalid value. The check looks different (`fd != -1` instead of `not _handle.is_null()`) but the structure is identical: a known-invalid value that means "this resource has been released."

This pattern shows up throughout the standard library. `File` manages an OS file descriptor, guarding reads and writes and freeing the descriptor in both `dispose()` and `_final()`. `Directory` does the same for directory handles. Each wraps a different kind of resource, but they both follow the same shape: track the resource in a mutable field, check before use, clean up explicitly with `dispose()`, and backstop with `_final()`.

For the related case of one-time library initialization and teardown (not per-instance resources), see the [FFI Global Initializer](../creation/ffi-global-initializer.md) pattern.
