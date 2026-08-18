# Collections

Sometimes you may find yourself working with a collection of optionals that contain the same value type, e.g. `List<Option<T>>`. The following methods are provided to assist you when working with collections containing optional types.

## Filter

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

## Map

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

## Flatten

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

## AsEnumerable

Use `AsEnumerable` on a single `Option` to treat it as a sequence of nothing or one. `Flatten` above is built out of it, and it is what lets an `Option` join a LINQ query.

```csharp
Option<string> maybeName = Option.Some("Pike");

IEnumerable<string> sequence = maybeName.AsEnumerable();
//                  ^? ["Pike"], and [] for a None
```

## FirstOrNone

Returns the first element of the collection that matches the predicate, or a `None` if there are no matches.

```csharp
List<Option<string>> collection = [
    Option.Some("Hello"),
    Option.Some("World")
];

Option<string> first = collection.FirstOrNone(x => x.StartsWith("H"));
//             ^? Option.Some("Hello")
```

## FirstOr

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

## FirstOrElse

Returns the first element of the collection that matches the predicate, or executes the provided factory to construct the fallback value if there are no matches.

```csharp
List<Option<string>> collection = [
    Option.Some("Hello"),
    Option.Some("World")
];

Option<string> first = collection.FirstOrElse(() => "Victor", x => x.StartsWith("V"));
//             ^? Option.Some("Victor")
```
