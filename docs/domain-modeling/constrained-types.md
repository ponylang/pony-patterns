---
hide:
  - toc
---

# Constrained Types

## Problem

Your system only allows usernames between 6 and 12 characters, containing only lowercase ASCII letters. You need to enforce that constraint, and you'd like to enforce it in the type system so that invalid usernames can't flow through the program unchecked. If a username is just a `String`, then every function that touches it might need to validate that it's actually a valid username. Otherwise, bugs creep in.

```pony
actor Main
  new create(env: Env) =>
    try
      let username = env.args(1)?
      if _is_valid_username(username) then
        do_something_with_username(username)
      end
    end

  fun _is_valid_username(name: String): Bool =>
    if (name.size() < 6) or (name.size() > 12) then
      return false
    end
    for c in name.values() do
      if (c < 97) or (c > 122) then
        return false
      end
    end
    true

  fun do_something_with_username(username: String) =>
    // username is just a String — nothing stops a caller
    // from passing an unvalidated value here
    None
```

Here `do_something_with_username` accepts any `String` at all. The compiler can't tell whether validation happened before the call. Each call site must either duplicate the validation check or trust that some earlier caller already validated, and when that trust is misplaced, invalid data silently flows through.

## Solution

Use the `constrained_types` standard library package. Instead of passing a plain `String` that might or might not have been validated, you define a `Username` type whose instances can only be created by going through validation. Functions that accept `Username` instead of `String` get a compile-time guarantee: callers can't skip validation because there's no other way to obtain an instance. Validation happens once at the boundary, and the rest of the code can trust what it receives.

The first step is to encode your constraints as a `Validator`. A validator is a primitive that implements `Validator[T]`. Its `apply` method examines a value and returns either `ValidationSuccess` or a `ValidationFailure` containing error messages.

```pony
use "constrained_types"

primitive UsernameValidator is Validator[String]
  fun apply(string: String): ValidationResult =>
    recover val
      let errors: Array[String] = Array[String]()

      if not _valid_length(string) then
        errors.push("Username must be between 6 and 12 characters")
      end

      if not _all_lower_case_ascii(string) then
        errors.push("Username can only contain lower case ASCII characters")
      end

      if errors.size() == 0 then
        ValidationSuccess
      else
        let failure = ValidationFailure
        for e in errors.values() do
          failure(e)
        end
        failure
      end
    end

  fun _valid_length(string: String): Bool =>
    (string.size() >= 6) and (string.size() <= 12)

  fun _all_lower_case_ascii(string: String): Bool =>
    for c in string.values() do
      if (c < 97) or (c > 122) then
        return false
      end
    end
    true
```

`UsernameValidator` is a primitive that implements `Validator[String]`. All of the validation logic lives in its `apply` method, which takes a `String` and returns a `ValidationResult`, which is a type alias for `(ValidationSuccess | ValidationFailure)`.

The `apply` method runs each constraint check (`_valid_length` and `_all_lower_case_ascii`) and collects error messages for any that fail. If there are no errors, it returns `ValidationSuccess`. If there are errors, it creates a `ValidationFailure` and adds each error message to it by calling `failure(e)` (which is `ValidationFailure`'s `apply` method). The `ValidationFailure` is then returned, carrying all the reasons validation failed.

The entire body of `apply` is wrapped in a `recover val` block because the `constrained_types` package requires all values to be `val`. The `ValidationFailure` is created as `ref` inside the recover block so error messages can be added to it, and then it is recovered to `val` when returned.

Next, create type aliases that tie the base type to the validator. `Constrained` and `MakeConstrained` are both provided by the `constrained_types` package. You don't write them yourself; you just parameterize them with your base type and your validator.

`Constrained[String, UsernameValidator]` wraps a `String` that has been validated by `UsernameValidator`. Its constructor is private, so there is no way to create one directly; you must go through `MakeConstrained`. `MakeConstrained[String, UsernameValidator]` is a primitive whose `apply` method takes a `String`, runs it through `UsernameValidator`, and returns either a `Constrained[String, UsernameValidator]` on success or a `ValidationFailure` on failure.

The type aliases give these parameterized types readable names:

```pony
type Username is Constrained[String, UsernameValidator]
type MakeUsername is MakeConstrained[String, UsernameValidator]
```

Now functions can accept `Username` instead of `String`. The compiler enforces this: callers must go through `MakeUsername` because there's no other way to produce the type.

```pony
  fun print_username(username: Username) =>
    _env.out.print(username() + " is a valid username!")
```

The call `username()` unwraps the validated `String` from the `Constrained` wrapper.

At system boundaries, where unvalidated input enters the program, use `MakeUsername` and pattern match on the result:

```pony
    match MakeUsername(arg1)
    | let u: Username =>
      print_username(u)
    | let e: ValidationFailure =>
      print_errors(e)
    end
```

Putting it all together, here's a complete program that takes a potential username as a command line argument, validates it, and prints the result:

```pony
use "constrained_types"

type Username is Constrained[String, UsernameValidator]
type MakeUsername is MakeConstrained[String, UsernameValidator]

primitive UsernameValidator is Validator[String]
  fun apply(string: String): ValidationResult =>
    recover val
      let errors: Array[String] = Array[String]()

      if not _valid_length(string) then
        errors.push("Username must be between 6 and 12 characters")
      end

      if not _all_lower_case_ascii(string) then
        errors.push("Username can only contain lower case ASCII characters")
      end

      if errors.size() == 0 then
        ValidationSuccess
      else
        let failure = ValidationFailure
        for e in errors.values() do
          failure(e)
        end
        failure
      end
    end

  fun _valid_length(string: String): Bool =>
    (string.size() >= 6) and (string.size() <= 12)

  fun _all_lower_case_ascii(string: String): Bool =>
    for c in string.values() do
      if (c < 97) or (c > 122) then
        return false
      end
    end
    true

actor Main
  let _env: Env

  new create(env: Env) =>
    _env = env

    try
      let arg1 = env.args(1)?
      match MakeUsername(arg1)
      | let u: Username =>
        print_username(u)
      | let e: ValidationFailure =>
        print_errors(e)
      end
    end

  fun print_username(username: Username) =>
    _env.out.print(username() + " is a valid username!")

  fun print_errors(errors: ValidationFailure) =>
    _env.err.print("Unable to create username")
    for s in errors.errors().values() do
      _env.err.print("\t- " + s)
    end
```

## Discussion

The guarantee that constrained types provide depends on immutability. Only `val` entities can be used with the `constrained_types` package. If the wrapped value were mutable, it could be changed after validation in a way that violates the constraints, defeating the entire purpose. The `Constrained` wrapper requires `val`, which guarantees the value is immutable and the constraints hold for the lifetime of the object.

This immutability requirement extends to validators themselves. Validators must be `val` and provide a zero-argument constructor that returns a `val` instance. In practice, always use a `primitive` because validators are stateless, so there is no advantage to using a `class`. The `Validator` interface is:

```pony
interface val Validator[T]
  new val create()
  fun apply(i: T): ValidationResult
```

One limitation to be aware of is that constrained types aren't composable through the type system. You can't use a `Username` where a "lowercase string" is expected, even though `Username` has been validated to contain only lowercase characters. Pony's type system can't express subset relationships between constrained types. Each `Constrained[T, V]` is a distinct type regardless of whether one validator's constraints are a superset of another's.

If you've seen the [Static Constructor](../creation/static-constructor.md) pattern, constrained types are a standardized version of the same idea. A static constructor is a one-off primitive whose `apply` returns either the constructed object or an error. The `constrained_types` package provides reusable infrastructure for this: a standard `Validator` interface and a `Constrained` wrapper that makes the validation guarantee visible in the type. The return type of `MakeConstrained` is `(Constrained[T, F] | ValidationFailure)`, which is the [Error as Union Type](../error-handling/error-as-union-type.md) pattern, where success and failure are distinct types in a union, and the caller pattern matches on the result.

The theoretical foundation for this approach is often called "Parse, Don't Validate": instead of checking whether data is valid and proceeding with the original untyped value, you parse it into a type that carries the proof of validity. The `Constrained` wrapper is that proof.
