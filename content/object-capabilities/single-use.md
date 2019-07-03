---
title: "Single Use Object Capabilities"
section: "Object Capabilities"
menu:
  toc:
    parent: "object-capabilities"
    weight: 10
---
## Problem

As shown in the [tutorial page about object capabilities](https://tutorial.ponylang.io/object-capabilities/object-capabilities.html), we can limit the actions of other objects or actors with object capabilities. This is used in the standard library of Pony for network connections and file access, for example. But it can also be used for other systems created in Pony.

For example, let's say we want to implement a capability-restricted service, that returns one unique number every time it is called. We will use a `CustomAuth` primitive as a token to restrict its access, created when provided another token, `AmbientAuth`:

```pony
use "promises"

primitive CustomAuth
  new create(auth: AmbientAuth) => None

actor RestrictedService
  var current_count: USize = 1

  be apply(auth: CustomAuth, promise: Promise[USize]) =>
    promise(current_count = current_count + 1)
```

Our `Main` actor receives the `AmbientAuth` token on creation from `env.root`, which means only itself or something it provided with that capability can receive an unforgeable `CustomAuth` token. We can then hand out that token to other actors or objects that need to call our `RestrictedService`.

However, let's suppose that we want these other actors to only call this restricted service _once_. As it currently stands, nothing prevents them from calling `RestrictedService.apply` several times, thus requesting a new number. The current object capability example doesn't allow us to limit how many times the token can be used by anyone.

You might think that tracking every caller of the service with a `HashMap` of identities could solve this problem, but keep in mind that not only is this cumbersome, but it incurs in runtime costs related to hash map lookups. There's also nothing stopping the caller from lying about their identity, since the only way to get the identity of a caller is to have the caller pass a reference to themselves as an argument. They could easily construct and use a new "throwaway" identity object to pass in as the supposed identity of the caller. This circumvention could be mitigated by using a direct actor callback for the response path instead of a promise - the identity can't be faked if you need to use it as the "return address" of the message.

The actual problem lies in how to make our tokens more restrictive, so that they cannot be used more than once. That's where single use object capabilities come in.

## Solution

There is a way to create a single use object capability, and it actually derives from Pony's own reference capabilities system. We showed that primitives (global references with type `val`) can be used as a token, but even an `iso` object could be used, too:

```pony
use "promises"

class SingleUseAuth
  new iso create(auth: AmbientAuth) => None

actor RestrictedService
  var current_count: USize = 1

  be apply(auth: SingleUseAuth iso, promise: Promise[USize]) =>
    promise(current_count = current_count + 1)
```

Now, we can provide our actors and objects with a controlled limited access must consume their `SingleUseAuth` tokens in order to use them, making them single-use object capabilities received from an authorized source. This guarantees us that it cannot call our service more than once, since the token must be expended in order to use it:

```pony
use "promises"

actor AuthorizedActor
  let service: RestrictedService
  var number: (USize | None) = None

  new create(service': RestrictedService) =>
    service = service'

  be request_new_number(auth: SingleUseAuth iso) =>
    let promise = Promise[USize] .> next[None](
      {(number: USize)(self: AuthorizedActor = this) =>
        self._update_number(number) 
      })
    service(consume auth, promise)

  be _update_number(number': USize) =>
    number = number'
```

Finally, `Main` can create individual tokens and provide them to our `AuthorizedActor`s as it sees fit. Here, this is done in a loop:

```pony
use "collections"

actor Main
  new create(env: Env) =>
    let service = RestrictedService
    for i in Range(0, 10) do
      try
        let foo = AuthorizedActor(service)
        let auth = SingleUseAuth(env.root as AmbientAuth)
        foo.request_new_number(consume auth)
      end
    end
```

Putting it all together:

```pony
use "collections"
use "promises"

class SingleUseAuth
  new iso create(auth: AmbientAuth) => None

actor RestrictedService
  var current_count: USize = 1

  be apply(auth: SingleUseAuth iso, promise: Promise[USize]) =>
    promise(current_count = current_count + 1)

actor AuthorizedActor
  let service: RestrictedService
  var number: (USize | None) = None

  new create(service': RestrictedService) =>
    service = service'

  be request_new_number(auth: SingleUseAuth iso) =>
    let promise = Promise[USize] .> next[None](
      {(number: USize)(self: AuthorizedActor = this) =>
        self._update_number(number) 
      })
    service(consume auth, promise)

  be _update_number(number': USize) =>
    number = number'

actor Main
  new create(env: Env) =>
    let service = RestrictedService
    for i in Range(0, 10) do
      try
        let foo = AuthorizedActor(service)
        let auth = SingleUseAuth(env.root as AmbientAuth)
        foo.request_new_number(consume auth)
      end
    end
```

Now, our service will only create numbers as much as we authorize our actors and objects to!

## Discussion

The biggest runtime impact with the use of object capabilities with object tokens instead of primitive tokens is that the former will have runtime costs, since every new token will require a memory allocation for each creation, while the latter simply reuses the primitive reference for every token. However, alternative solutions for a single-use token incur in a greater runtime penalty than the proposed pattern, as well as in an added complexity for handling permissions to a service or ambient authority with a different mechanism.

The concept presented in this pattern could be extended for any number of tokens we want to give to our actors. For example, if you ever needed something like a credit flow control protocol where you didn't trust the clients of the service to behave -- you could dole out multiple unforgeable tickets to limit their use of the service, based on how many clients exist, or how often they request a token, or any other criteria you need, without worrying about the internal workings of these clients.

All in all, the object capabilities system can be used as a way to have better control over our programs' accesses, as well as give clients the freedom to handle these tokens as they see fit. Using a single-use token can make this access more restrictive, without actually limiting the API of our libraries.
