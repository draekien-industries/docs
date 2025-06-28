---
description: Learn about monads and why they are important to use in your code
icon: diamond-half-stroke
---

# Monads

## What is a monad?

Monads sound intimidating, but in practice, they're just a way to chain operations while carrying some context.

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

When writing C# code without monads, we resort to:

* Returning `null` when there is a valid business rule for the absence of a value
* Throwing `Exceptions` for every instance of an error, even if the error is a valid business rule
* Wrapping methods inside `try/catch` blocks just to log the `Exception` and then re-throw it
* Writing `null` guard clauses everywhere
* Writing `if/else` blocks or `switch` blocks in order to handle branching logic

Take for example the below code:

```csharp
Request? PrepareRequest(decimal input); // can return null or throw an exception
Response DoWork(Request request);       // can still return null or throw

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
    catch (FailedToPrepareRequestException ex) // An exception for a valid use case
    {
        _logger.LogWarning(ex, "Failed to prepare request");
        return Fallback();
    }
    catch (Exception ex)
    {
        _logger.LogWarning(ex, "Failed to save data");
        throw;
    }
}

Response response = SaveData(10); // can be response or throw an exception

string output = response?.Message ?? "Something went wrong"; // errors potentially lost
```

Lets write the same code again but this time using monads:

{% code fullWidth="false" %}
```csharp
Option<Request> PrepareRequest(decimal input);   // never throws, may be some or none
Result<Response, Error> DoWork(Request request); // may be ok or error

Result<Response, Error> SaveData(Option<decimal> input) => 
    input.FlatMap(PrepareRequest) // If input is Some, PrepareRequest
         .Map(DoWork)             // If PrepareRequest is Some, DoWork
         .Transpose()             // Turn the Option<Result> into a Result<Option>
         .InspectErr(error => _logger.LogWarning(error))    // If Err, log a warning
         .Map(response => response.UnwrapOrElse(Fallback)); // If Ok, Get the Response if the Option is Some, or the Fallback
         
string output = SaveData(Option.Some(10.0m)).Match(
     onOk: response => response.Message,
     onErr: error => error.Message  // always the first error encountered in the pipeline
);
```
{% endcode %}

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
* unpredictable paths:
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
