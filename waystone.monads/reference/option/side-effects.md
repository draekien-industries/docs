---
description: Run something against the value without changing it.
icon: bolt
---

# Side effects

## Inspect

```csharp
Option<T> Inspect(Action<T> action)
```

Runs an action against the value when the option is `Some`, and hands the option
back unchanged so the chain continues. Logging is the usual reason.

```csharp
Option<string> maybeName = Option.Some("Geladon");
maybeName.Inspect(name => Console.WriteLine(name.Length));
```

**On a `None`:** the action never runs, and the `None` comes back.

{% hint style="info" %}
Reach for [`Map`](transform.md#map) instead if you want to *change* the value.
{% endhint %}

There is no `InspectNone`. Use [`Match`](consume.md#match) when both branches need
to do something.

## Why not just ToString it?

**`ToString()` never shows the wrapped value.** You get the state and nothing
else:

```csharp
Option.Some("Vex'ahlia").ToString() // "Some { IsSome = True, IsNone = False }"
Option.None<string>().ToString()    // "None { IsSome = False, IsNone = True }"
```

`Option<T>` is a record and `Some<T>` keeps its value in a private property, so
the compiler-generated `ToString()` has nothing to print. Interpolating an option
into a log message tells you whether a value was there, never what it was. That is
what `Inspect` is for.
