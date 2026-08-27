# Option\<T>

## Printing and logging

**`ToString()` never shows the wrapped value.** You get the state and nothing else:

```csharp
Option.Some("Liam").ToString()   // "Some { IsSome = True, IsNone = False }"
Option.None<string>().ToString() // "None { IsSome = False, IsNone = True }"
```

`Option<T>` is a record, and `Some<T>` keeps its value in a private property, so
the compiler-generated `ToString()` has nothing to print. Interpolating an option
into a log message therefore tells you whether a value was there, never what it
was.

To log the value when it exists, use
[#inspect](../core-functionality.md#inspect "mention"). It runs your action only on
a `Some`, and returns the option unchanged so you can keep chaining:

```csharp
Option<User> user = FindUser(id)
    .Inspect(u => logger.LogInformation("Found user {Name}", u.Name));
```

Nothing runs on a `None`. If you need to log both branches, use
[#match](../core-functionality.md#match "mention") instead.

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
This method was called `FlatMap` before 5.4.0. `FlatMap` still works and still forwards here, but it is `[Obsolete]` and goes away in v6.0.0. `WM2014` reports each call site and its quick fix does the rename. See [deprecations.md](../../upgrading-and-deprecations/deprecations.md "mention").
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

A component that equals the default of its type is an ordinary value here, so
`Option.Some((0, "x")).Unzip()` gives you `(Some(0), Some("x"))`. This threw
before 6.0.0.

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

## Collections

Sometimes you may find yourself working with a collection of optionals that contain the same value type, e.g. `List<Option<T>>`. The following methods are provided to assist you when working with collections containing optional types.

### Filter

Provides the same functionality as [#filter](./#filter "mention") but for a collection of options. All options that do not match the provided predicate are flipped into a `None`.

```csharp
List<Option<string>> collection = [
    Option.Some("Hello"),
    Option.Some("World"),
    Option.None<string>()
];

List<Option<string>> filtered = collection.Filter(x => x == "Hello");
//                   ^? [Option.Some("Hello"), Option.None<string>(), Option.None<string>()]
```

### Map

Provides the same functionality as [#map](../core-functionality.md#map "mention") but for a collection of options. The same transformation will be applied to all members of the collection if they are `Some`.

```csharp
List<Option<string>> collection = [
    Option.Some("Hello"),
    Option.Some("World"),
    Option.None<string>()
];

List<Option<string>> mapped = collection.Map(x => $"{x}!");
//                   ^? [Option.Some("Hello!"), Option.Some("World!"), Option.None<string>()]
```

### Flatten

Use `Flatten` to drop the `None`s and keep the values. You get an `IEnumerable<T>` of everything that was present, in order.

```csharp
List<Option<string>> collection = [
    Option.Some("Hello"),
    Option.None<string>(),
    Option.Some("World")
];

IEnumerable<string> values = collection.Flatten();
//                  ^? ["Hello", "World"]
```

{% hint style="info" %}
`Flatten` is lazy and walks the source once, so it composes with the rest of LINQ as you would expect. Nothing runs until you enumerate the result.
{% endhint %}

This is the sequence version. The `Flatten` that collapses a single nested `Option<Option<T>>` is a different method, covered under [#flatten](../core-functionality.md#flatten "mention"). No receiver is both, so the two never compete.

### Collect

Use `Collect` when every value has to be present. You get a `Some` holding all of them, or a single `None` if any element is missing.

```csharp
List<Option<string>> collection = [
    Option.Some("Hello"),
    Option.Some("World")
];

Option<IReadOnlyList<string>> all = collection.Collect();
//                            ^? Some(["Hello", "World"])
```

One `None` anywhere fails the whole call:

```csharp
List<Option<string>> withAGap = [
    Option.Some("Hello"),
    Option.None<string>(),
    Option.Some("World")
];

Option<IReadOnlyList<string>> all = withAGap.Collect();
//                            ^? None
```

This is the opposite of `Flatten`. `Flatten` drops what is missing and carries on; `Collect` treats one missing value as a failure of the whole batch.

`Collect` stops at the first `None`. It never looks at the rest of the source, so anything that would have produced the later elements does not run.

{% hint style="info" %}
`Collect` is eager, and it builds a list as it goes. Do not call it on an unbounded sequence.
{% endhint %}

An empty sequence gives you `Some` of an empty list, not `None`. There is nothing missing in it.

The result tells you only that something was absent. It does not tell you which element. Use `Partition` on a sequence of `Result` when you need to know what failed.

### CollectAsync

Use `CollectAsync` for the same job over an `IAsyncEnumerable<Option<T>>`.

```csharp
Option<IReadOnlyList<string>> all = await stream.CollectAsync(cancellationToken);
```

It stops pulling from the stream at the first `None`, so the work behind the later elements never happens. That is the reason to use it instead of reading the whole stream into a list and calling `Collect`.

### AsEnumerable

Use `AsEnumerable` on a single `Option` to treat it as a sequence of nothing or one. `Flatten` above is built out of it, and it is the way out of the monad into `System.Linq`.

It is not how you write a LINQ query over an `Option`. For that — `from`, `select`, `where`, staying inside the `Option` throughout — see [Waystone.Monads.Linq](../../companion-packages/linq.md).

```csharp
Option<string> maybeName = Option.Some("Pike");

IEnumerable<string> sequence = maybeName.AsEnumerable();
//                  ^? ["Pike"], and [] for a None
```

### FirstOrNone

Returns the first element of the collection that matches the predicate, or a `None` if there are no matches.

```csharp
List<Option<string>> collection = [
    Option.Some("Hello"),
    Option.Some("World")
];

Option<string> first = collection.FirstOrNone(x => x.StartsWith("H"));
//             ^? Option.Some("Hello")
```

### FirstOr

Returns the first element of the collection that matches the predicate, or the provided fallback value if there are no matches.

```csharp
List<Option<string>> collection = [
    Option.Some("Hello"),
    Option.Some("World")
];

Option<string> first = collection.FirstOr("Victor", x => x.StartsWith("V"));
//             ^? Option.Some("Victor")
```

{% hint style="info" %}
If your fallback value is expensive to generate, consider using `FirstOrElse` to defer execution to when a match is not found.
{% endhint %}

### FirstOrElse

Returns the first element of the collection that matches the predicate, or executes the provided factory to construct the fallback value if there are no matches.

```csharp
List<Option<string>> collection = [
    Option.Some("Hello"),
    Option.Some("World")
];

Option<string> first = collection.FirstOrElse(() => "Victor", x => x.StartsWith("V"));
//             ^? Option.Some("Victor")
```
