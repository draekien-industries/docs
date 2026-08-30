---
description: Methods that take an Option<T> and give you back an Option.
---

# Transform

Every method here returns an `Option`, so the chain continues. To end it, see
[Consume](consume.md).

## Map

```csharp
Option<TOut> Map<TOut>(Func<T, TOut> map)
```

Applies a transformation to the value if there is one.

```csharp
Option<string> maybeName = Option.Some("Henry Crabgrass");
Option<int> maybeLength = maybeName.Map(name => name.Length);
```

**On a `None`:** the delegate never runs, and you get `None<TOut>`.

## AndThen

```csharp
Option<TOut> AndThen<TOut>(Func<T, Option<TOut>> map)
```

Chains a step that itself returns an `Option`. `Map` would give you
`Option<Option<TOut>>`; `AndThen` keeps it flat.

```csharp
Option<string> maybeDomain = maybeSigil.AndThen(TryExtractDomain);
```

**On a `None`:** short-circuits. Later steps never run.

{% hint style="warning" %}
Called `FlatMap` before 5.4.0. `FlatMap` was `[Obsolete]` through 5.x and 6.0.0
removed it, so a call to it is `CS0117` rather than a warning. `WM2014`, the rule
that reported each call site, retired with it — delete any `.editorconfig` entry
for that id.
{% endhint %}

## Filter

```csharp
Option<T> Filter(Predicate<T> predicate)
```

Keeps the value only if it passes. If it does not, you get `None`.

```csharp
Option<string> maybeName = Option.Some("Thordak");

Option<string> nonEmpty = maybeName.Filter(name => name.Length > 0); // Some("Thordak")
Option<string> blank = maybeName.Filter(name => name.Length == 0);   // None
```

**On a `None`:** the predicate never runs.

## Zip

```csharp
Option<(T, T2)> Zip<T2>(Option<T2> other)
```

Pairs two options into one holding a tuple.

```csharp
Option<string> vex = Option.Some("Vex'ahlia");
Option<string> vax = Option.Some("Vax'ildan");
Option<string> missing = Option.None<string>();

Option<(string, string)> twins = vex.Zip(vax);     // Some(("Vex'ahlia", "Vax'ildan"))
Option<(string, string)> alone = vex.Zip(missing); // None
```

**If either side is `None`:** you get `None`.

## ZipWith

```csharp
Option<TOut> ZipWith<T2, TOut>(Option<T2> other, Func<T, T2, TOut> zip)
```

The same, but you combine the two values instead of getting a tuple.

```csharp
Option<int> fireball = Option.Some(24);
Option<int> sneakAttack = Option.Some(18);

Option<int> total = fireball.ZipWith(sneakAttack, (a, b) => a + b);
//         ^? Some(42)
```

**If either side is `None`:** you get `None`, and the delegate never runs.

`ZipWith` has no state overload and is not getting one — it already hands the
delegate both values.

## Unzip

```csharp
(Option<T1>, Option<T2>) Unzip<T1, T2>(this Option<(T1, T2)> option)
```

Reverses a `Zip`. An extension method, in
`Waystone.Monads.Options.Extensions`.

```csharp
Option<(string, string)> twins = Option.Some(("Vex'ahlia", "Vax'ildan"));
Option<(string, string)> none = Option.None<(string, string)>();

twins.Unzip(); // (Some("Vex'ahlia"), Some("Vax'ildan"))
none.Unzip();  // (None, None)
```

A component that equals its type's default is an ordinary value, so
`Option.Some((0, "x")).Unzip()` gives `(Some(0), Some("x"))`. This threw before
6.0.0.

## And

```csharp
Option<T2> And<T2>(Option<T2> other)
```

Returns the second option, but only if the first was `Some`. It ignores what the
first held, so it answers "did both arrive?" rather than combining them.

```csharp
Option<string> maybeName = Option.Some("Grog");
Option<int> maybeLevel = Option.Some(19);

Option<int> both = maybeName.And(maybeLevel);                // Some(19)
Option<int> neither = Option.None<string>().And(maybeLevel); // None
```

**Evaluated eagerly.** Reach for [`AndThen`](#andthen) when producing the second
one costs something, or when it depends on the first one's value.

## Reduce

```csharp
Option<T> Reduce(Option<T> other, Func<T, T, T> reduce)
```

Merges two options of the same type. When both hold a value your function combines
them; when only one does, you get that one back untouched.

```csharp
Option<int> firstRoll = Option.Some(3);
Option<int> secondRoll = Option.Some(4);

firstRoll.Reduce(secondRoll, (a, b) => a + b);          // Some(7)
firstRoll.Reduce(Option.None<int>(), (a, b) => a + b);  // Some(3)
Option.None<int>().Reduce(secondRoll, (a, b) => a + b); // Some(4)
```

**When both are `None`:** you get `None`, and the delegate never runs.

`Reduce` has no state overload and is not getting one.

## Or

```csharp
Option<T> Or(Option<T> other)
```

The first `Some` of the two.

```csharp
Option<string> result = chosen.Or(absent).Or(fallback);
//             ^? Some("Keyleth")
```

**Evaluated eagerly.** Use [`OrElse`](#orelse) if the fallback costs something.

## OrElse

```csharp
Option<T> OrElse(Func<Option<T>> createElse)
```

The same as `Or`, but the factory runs only when the receiver is `None`.

```csharp
Option<string> result = first
    .OrElse(() => RollForAnother())
    .OrElse(() => SendInTheHireling());
//     ^? Some("The understudy")
```

## Xor

```csharp
Option<T> Xor(Option<T> other)
```

Exclusive or. The value comes back only if exactly one of the two is `Some`.

```csharp
Option<string> result = bardsong
    .Xor(silence)     // Some("Scanlan")
    .Xor(secondBard); // None
```

**When both are `Some`, or both `None`:** you get `None`.
