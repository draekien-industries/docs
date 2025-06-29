---
icon: option
---

# Options

## What is an Option?

The `Option<T>` type is the functional programming answer to uncertainty. It represents a computation or a value that might exist - `Some<T>` - or might not exist - `None<T>`. Where object-oriented programming languages like C# have historically used `null` to represent absence, functional languages - and increasingly, modern C# developers adopting functional programming patterns - use `Option` to make that absence explicit and composable.

{% hint style="info" %}
`Option<T>` is sometimes called `Maybe<T>` in other languages
{% endhint %}

## The problem with null

{% hint style="danger" %}
`null` is a semantic black hole.
{% endhint %}

It tells you nothing about why a value is missing or whether it was supposed to be missing in the first place. It leaks into every corner of your code, forcing defensive programming and riddling APIs with ambiguity. It's not a value, it's the absence of value, but the compiler won't stop you from de-referencing it.

{% hint style="success" %}
The `Option<T>` type doesn't allow for ambiguity. It doesn't just fail to hold a value - it says so up front.&#x20;
{% endhint %}

## Monadic Behaviour

`Option<T>` is a [monad](monads.md), and being a monad has practical consequences:

* You can `Map` over the value if it exists, leaving `None` untouched
* You can `FlatMap` chained computations that might each return an `Option`
* You can pattern match or inspect the state with confidence - no more `if (x is not null)` littered everywhere

Here's what a basic pipeline might look like when using `Option<T>`

```csharp
Option<string> TryGetEmailDomain(User user) =>
    Option.Some(user)
          .Map(user => user.Email)
          .FlatMap(ParseDomain);
          
Option<string> ParseDomain(string email) =>
    Option.Try(() => email.Split('@')[1]);    
```

If at any point the value is missing, the entire chain short-circuits and propagates `None`. You don't have to write `if` guards, `try/catch` blocks, or null-coalescing fallbacks.

{% hint style="info" %}
This is exactly how `Promise` chains short-circuit on exceptions in JavaScript. `Option` does it for presence instead of failure.
{% endhint %}

## Intentional Absence vs. Failure

Unlike exceptions, which signal something has gone wrong, `Option<T>` is a declaration that a value is optional. It is not an error to get `None` . Instead, it is part of the domain model. Consider the following method:

```csharp
Option<User> TryFindUser(string id);
```

You're not saying "this might blow up" - you're saying "this might not yield anything, and that's expected". The difference matters - in readability, maintainability, and in how you reason about the control flow.

## Composability Over Conditionals

Functional programming is all about composition. You want to be able to write small, simple functions that can be glued together.

{% hint style="success" %}
`Option<T>` is glue - it makes your functions composable even when values are missing
{% endhint %}

With `Option<T>`, you can stop writing code like this:

```csharp
record User(string? Email);

string? DoWork()
{
    User? user = TryGetUser("Pike");

    if (user is { Email: not null })
    {
        var domain = ParseDomain(user.Email);
        if (domain is not null)
        {
            return NormaliseDomain(domain);
        }
    }
    
    return null;
}
```

And start writing code like:

```csharp
record User(Option<string> Email);

Option<string> DoWork() => 
    TryGetUser("Scanlan")
        .FlatMap(u => u.Email)
        .FlatMap(ParseDomain)
        .Flatmap(NormaliseDomain);
```

### When to use Option

You should reach for `Option<T>` when:

* You want to represent intentionally missing data
* You want to chain functions but bail early on absence
* You don't care about the reason for the absence
* You want the caller to intentionally handle the `None` case

Avoid `Option<T>` when:

* The default of a value type does not represent absence, i.e. `0` for `int`&#x20;
* You care about the reason for the absence (reach for [`Result<T, E>`](results.md) instead)

## Summary

`Option<T>` is a core abstraction in functional programming for modeling optional values. It turns invisible, error-prone absence into an explicit, composable structure. It helps you reason without uncertainty, eliminates entire classes of bugs, and leads to cleaner, more predictable code.

Stop checking for `null`. Start using `Option<T>` - and let the type system work with you instead of against you.
