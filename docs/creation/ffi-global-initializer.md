---
hide:
  - toc
---

# FFI Global Initializer

## Problem

You are working with a library via [Pony's C-FFI](https://tutorial.ponylang.io/c-ffi/) and need to initialize the library before you can use it. And you'd like to do this initialization once and only once.

## Solution

Pony's primitives can serve a variety of design purposes. Here we will use a primitive to initialize our imaginary C library.

```pony
use @magic_global_initialization[None]()

primitive LibraryInitializer
  fun _init() =>
    @magic_global_initialization()
```

## Discussion

Only a single instance of a Pony `primitive` will exist in our binary. User defined primitives are singletons. We can combine this with the fact that primitives have an `_init` function that is called when the primitive is created.

Wrapping our "should only be done once" initialization code in a primitive's `_init` method is a good way to ensure that the initialization code is only run once. It is important to note that this approach doesn't protect you from someone mistakenly calling the initializer via C-FFI directly again,

If we need to teardown the library, we can add a `_final` method to `LibraryInitializer` which will be executed on program shutdown.

```pony
use @magic_global_initialization[None]()
use @magic_global_shutdown[None]()

primitive LibraryInitializer
  fun _init() =>
    @magic_global_initialization()

  fun _final() =>
    @magic_global_shutdown()
```
