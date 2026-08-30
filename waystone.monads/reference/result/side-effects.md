---
description: Run something against either side without changing it.
---

# Side effects

## Inspect

```csharp
Result<TOk, TErr> Inspect(Action<TOk> action)
```

Runs an action against the success value and hands the result back unchanged, so
the chain continues.

```csharp
Result<string, string> nameResult = Result.Ok<string, string>("Percival");
nameResult.Inspect(name => Console.WriteLine(name.Length));
```

**On an `Err`:** the action never runs.

{% hint style="info" %}
Reach for [`Map`](transform.md#map) instead if you want to *change* the value.
{% endhint %}

## InspectErr

```csharp
Result<TOk, TErr> InspectErr(Action<TErr> action)
```

The counterpart. Runs against the error, and again hands the result back
unchanged. Logging a failure is what it is for.

```csharp
Result<string, string> username = FindCharacter("Percy")
    .InspectErr(err => Console.WriteLine($"Find character failed: {err.Message}"))
    .Map(character => character.Username)
    .MapErr(err => err.Message);
```

**On an `Ok`:** the action never runs.

{% hint style="info" %}
Reach for [`MapErr`](transform.md#maperr) if you want to *change* the error.
{% endhint %}

## Using both together

Each runs only on its own branch, so a chain that carries both logs exactly once
either way:

```csharp
Result<Quest, Error> quest = LoadQuest(id)
    .Inspect(q => logger.LogInformation("Loaded quest {Id}", q.Id))
    .InspectErr(e => logger.LogWarning("Load failed: {Code} {Message}", e.Code, e.Message));
```

## Why not just ToString it?

**`ToString()` never shows the wrapped value.** You get the state and nothing
else:

```csharp
Result.Ok<int, Error>(1).ToString()  // "Ok { IsOk = True, IsErr = False }"
Result.Err<int, Error>(e).ToString() // "Err { IsOk = False, IsErr = True }"
```

`Result<TOk, TErr>` is a record and both sides keep their value in a private
property, so the compiler-generated `ToString()` has nothing to print.
Interpolating a result into a log message tells you which branch you are on, never
what it holds. That is what the inspect pair is for.
