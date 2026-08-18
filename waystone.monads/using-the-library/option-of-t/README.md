# Option\<T>

## Control Flow

### IsSomeAnd

Use `IsSomeAnd` when you want to check if the `Option` is a `Some` and that the value contained inside the `Some` matches a predicate.

```csharp
Option<string> maybeName = Option.Some("Raven");
maybeName.IsSomeAnd(name => name.Length > 0); // true
```

### IsNoneOr

Use `IsNoneOr` when you want to check if the `Option` is a `None` or the value contained inside the `Some` matches a predicate.

```csharp
Option<string> maybeName = Option.Some("Queen");
maybeName.IsNoneOr(name => name.Length > 0); // true
maybeName.IsNoneOr(name => string.IsNullOrWhiteSpace(name)); // false
```

## Transform

{% hint style="info" %}
Refer to the [#transform](../core-functionality.md#transform "mention") section on the [core-functionality.md](../core-functionality.md "mention") page to learn about the other transform methods available for an `Option<T>`
{% endhint %}

### AndThen

Use `AndThen` to compose monadic pipelines where each operation might fail and return an `Option<T>` itself. It prevents nested `Option<Option<T>>` results and keeps the pipeline flat and clean.

```csharp
Option<string> TryExtractDomain(string email);

Option<string> maybeEmail = GetEmail(userId);
Option<string> maybeDomain = maybeEmail.AndThen(TryExtractDomain);
```

Without `AndThen`, you'd need to `Map` and then flatten manually, or deal with nested options.

{% hint style="info" %}
`AndThen` short-circuits: if one of the options in the chain is `None`, the subsequent functions are skipped.
{% endhint %}

{% hint style="warning" %}
This method was called `FlatMap` before 5.4.0. `FlatMap` still works and still forwards here, but it is `[Obsolete]` and goes away in v6.0.0. `WM2014` reports each call site and its quick fix does the rename. See [deprecations.md](../deprecations.md "mention").
{% endhint %}

`Result<TOk, TErr>` has spelled this `AndThen` all along, so the two monads now agree.

### Filter

Use `Filter` to retain only the values that pass a predicate. If the value doesn't match, the result becomes a `None`.

```csharp
Option<string> maybeName = Option.Some("Thordak");

Option<string> nonEmpty = maybeName.Filter(name => name.Length > 0); // Some("Thordak")
Option<string> blank = maybeName.Filter(name => name.Length == 0);   //  None
```

{% hint style="info" %}
This is the clean alternative to `if`-guards. It keeps the monadic flow and makes intent obvious.
{% endhint %}

### Zip

Use `Zip` to combine two options into a single option containing both values as a tuple.

```csharp
Option<string> a = Option.Some("a");
Option<string> b = Option.Some("b");
Option<string> c = Option.None<string>();

Option<(string, string)> ab = a.Zip(b); // Some(("a", "b"))
Option<(string, string)> ac = a.Zip(c); // None
```

{% hint style="info" %}
If either `Option` being zipped is a `None`, then a `None` will be returned.
{% endhint %}

### ZipWith

Use `ZipWith` to combine two options into a single option using a custom zipping function.

```csharp
Option<int> a = Option.Some(1);
Option<int> b = Option.Some(2);
Option<int> result = a.ZipWith(b, (x, y) => x + y);
//          ^? Some(3)
```

{% hint style="info" %}
If either `Option` being zipped is a `None`, then a `None` will be returned.
{% endhint %}

### Unzip

Reverses a `Zip`, splitting the tuple into two options.

```csharp
Option<(string, string)> some = Option.Some(("a", "b"));
Option<(string, string)> none = Option.None<(string, string)>();

(Option<string>, Option<string>) unzippedSome = some.Unzip(); // (Some("a"), Some("b"))
(Option<string>, Option<string>) unzippedNone = none.Unzip(); // (None, None)
```

## Logical Operators

Sometimes you want to combine two `Option` values using logical operators without leaving the monadic model.

### And

Use `And` when you want the second `Option` only if the first one holds a value. It ignores what the first one holds and returns the second, so it answers "did both arrive?" rather than combining them.

```csharp
Option<string> maybeName = Option.Some("Grog");
Option<int> maybeLevel = Option.Some(19);

Option<int> both = maybeName.And(maybeLevel);   // Some(19)
Option<int> neither = Option.None<string>().And(maybeLevel);   // None
```

{% hint style="info" %}
`And` takes an `Option` you already have. Reach for [#andthen](./#andthen "mention") when producing the second one costs something, or when it depends on the first one's value.
{% endhint %}

### Reduce

Use `Reduce` to merge two `Option`s of the same type into one. When both hold a value your function combines them. When only one does, you get that one back untouched, and when neither does you get `None`.

```csharp
Option<int> first = Option.Some(3);
Option<int> second = Option.Some(4);

first.Reduce(second, (a, b) => a + b);                 // Some(7)
first.Reduce(Option.None<int>(), (a, b) => a + b);     // Some(3)
Option.None<int>().Reduce(second, (a, b) => a + b);    // Some(4)
```

{% hint style="info" %}
Your reduce function runs only when both options are `Some`, so it never has to handle an absent side.
{% endhint %}

### Or

`Or` returns the first `Some` value encountered in the chain.

```csharp
Option<string> first = Option.Some("John");
Option<string> second = Option.None<string>();
Option<string> fallback = Option.Some("Default");

Option<string> result = first.Or(second).Or(fallback); // Some("John")
```

{% hint style="info" %}
Use `Or` when you want a prioritized fallback chain
{% endhint %}

### OrElse

Like `Or`, but lazily evaluated. The fallback is only invoked if the preceding is `None`.

```csharp
Option<string> first = Option.None<string>();

Option<string> CreateSecond() => Option.None<string>();
Option<string> CreateFallback() => Option.Some("Default");

Option<string> result = first
    .OrElse(() => CreateSecond())
    .OrElse(() => CreateFallback()); // Some("Default")
```

### Xor

`Xor` is an exclusive-or. It returns the first `Some` value encountered in the chain if exactly one of the options is `Some`.

```csharp
Option<string> first = Option.Some("Art");
Option<string> second = Option.None<string>();
Option<string> third = Option.Some("Dad");

Option<string> result = first
    .Xor(second) // Some("Art")
    .Xor(third); // None
```

{% hint style="info" %}
`Xor` is niche, but useful when you're checking mutually exclusive conditions
{% endhint %}
