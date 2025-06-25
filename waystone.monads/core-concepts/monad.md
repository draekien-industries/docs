---
description: Learn about monads and why they are important to use in your code
icon: diamond-half-stroke
---

# Monad

## What is a monad?

Monads sound intimidating - because people often describe them in terms of abstract math. But in practice, they're just a way to chain operations while carrying some context.

In C#, we usually deal with values directly. Monads, on the other hand, wrap values and give you a consistent way to work with them, even when things get messy.

{% hint style="success" %}
Monads provide a structured way to represent chained computations, especially ones that might fail, return nothing, or produce side effects
{% endhint %}

Monads are used to encode optional values and fallible computations without resorting to `null` or `try/catch`. The library implements two monads:

* `Option<T>` for "maybe there's a value"
* `Result<T, E>` for "this might succeed or fail"

## A practical definition

A monad is a type that wraps a value and provides consistent semantics for composing operations on it.&#x20;

{% hint style="success" %}
A monad must:

1. let you wrap a value, e.g. `Some`, `Ok`&#x20;
2. let you chain operations, e.g. `Map`, `AndThen`&#x20;
3. preserve context, e.g. whether it's missing or failed
{% endhint %}

## Why use monads?

Because C# encourages code like this:

```csharp
Request? PrepareRequest(decimal input); // can return null or throw
Response DoWork(Request request);       // can throw

Response SaveData(decmial? input)
{
    if (input is null) {
        return Fallback();
    }
    
    try
    {
        Request? request = PrepareRequest(input.Value);
        
        return request is not null
            ? DoWork(request)
            : Fallback();
    } 
    catch (FailedToPrepareRequestException ex)
    {
        _logger.LogWarning(ex, "Failed to parse request");
        throw;
    }
}

var response = SaveData(10); // can be response or throw an exception

string output = response?.Message ?? "Something went wrong"; // errors potentially lost
```

And monads let you write this:

```csharp
Option<Request> PrepareRequest(decimal input);   // never throws, may be some or none
Result<Response, Error> DoWork(Request request); // may be ok or error

Result<Response, Error> SaveData(Option<decimal> input) => 
    input.FlatMap(PrepareRequest) // Option<Request>
         .Map(DoWork)             // Option<Result<Response, Error>>
         .Transpose()             // Result<Option<Response>, Error>
         .InspectErr(error => _logger.LogWarning(error))    // Triggered on error
         .Map(response => response.UnwrapOrElse(Fallback)); // Result<Response, Error>
         
string output = SaveData(Option.Some(10.0m)).Match(
     ok: response => response.Message,
     err: error => error.Message  // always the first error encountered in the pipeline
);
```

{% hint style="success" %}
Monads enable you to build a pipeline of transformations that can short-circuit cleanly. No conditionals, no defensive null-checking, no local variables spread everywhere.
{% endhint %}

## Example: The railway tracks

Imagine your program as a train moving along a railway track. Each step of your logic is a station. You want the train to move from station to station - loading data, transforming it, performing checks, etc, but things can go wrong.

{% hint style="danger" %}
* The passenger might be missing
* A validation might fail
* A file might not be found, or an input might be malformed
{% endhint %}

### Without monads (derailment everywhere)

In regular C# code, errors or missing values derail the train. You get thrown into:

* null checks everywhere
* try/catch blocks scattered across your code
* unpredictable paths&#x20;
  * some functions might return `null`,&#x20;
  * some throw exceptions,&#x20;
  * some return data

It becomes hard to know what to expect, and even harder to safely chain operations together.

### With monads (safe, predictable tracks)

Monads like `Option<T>` and `Result<T,E>` put your logic on two parallel railway tracks:

* :train2: success track - your train keeps moving smoothly
* :construction: failure or none track - your train gets rerouted safely to a dead-end, without crashing

The train never jumps tracks randomly, it stays on one track or the other, and every station (function) is built to handle both cases.

```csharp
Option<string> email = GetUserById(id)
    .Map(u => u.Profile)
    .Map(p => p.Email);
```

If the user doesn't exist, or the profile is missing, the train never crashes. It just stops safely at `None`. You can later decide what to do (show an error message, fall back to defaults, etc).

```csharp
string displayEmail = email.Match(
    some => some,
    none => "[Not Provided]"
);
```

## TL;DR

Monads give you a standard pattern for chaining operations with context - like failure or absence - without losing your mind in `if` statements or exception handling.

In `Waystone.Monads`:

* Use `Option<T>` when you might have no value
* Use `Result<T, E>` when something might go wrong
* Use `FlatMap` , `Map`, `AndThen` and more to compose operations
* Use `Try` to safely enter the monadic world from unsafe operations
