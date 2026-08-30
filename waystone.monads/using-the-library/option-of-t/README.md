# Option\<T>

{% hint style="info" %}
**Looking for how to use `Option<T>`?** That is the
[Option\<T> guide](../../guides/option.md). This page covers the collection
extensions only.
{% endhint %}

## Collections

Sometimes you may find yourself working with a collection of optionals that contain the same value type, e.g. `List<Option<T>>`. The following methods help you work with collections containing optional types.

### Filter

Does the same as [Filter](../../guides/option.md#filter "mention") but for a collection of options. Every option that does not match the predicate you gave is flipped into a `None`.

```csharp
List<Option<string>> collection = [
    Option.Some("Hello"),
    Option.Some("World"),
    Option.None<string>()
];

IEnumerable<Option<string>> filtered = collection.Filter(x => x == "Hello");
//                          ^? [Option.Some("Hello"), Option.None<string>(), Option.None<string>()]
```

### Map

Does the same as [Map](../../guides/option.md#map "mention") but for a collection of options. The same transformation will be applied to all members of the collection if they are `Some`.

```csharp
List<Option<string>> collection = [
    Option.Some("Hello"),
    Option.Some("World"),
    Option.None<string>()
];

IEnumerable<Option<string>> mapped = collection.Map(x => $"{x}!");
//                          ^? [Option.Some("Hello!"), Option.Some("World!"), Option.None<string>()]
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

It is not how you write a LINQ query over an `Option`. For that (`from`, `select`, `where`, staying inside the `Option` throughout) see [Waystone.Monads.Linq](../../companion-packages/linq.md).

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

Returns the first element of the collection that matches the predicate, or the fallback value you gave if there are no matches.

```csharp
List<Option<string>> collection = [
    Option.Some("Hello"),
    Option.Some("World")
];

string first = collection.FirstOr(x => x.StartsWith("V"), "Victor");
//     ^? "Victor"
```

{% hint style="info" %}
If your fallback value is expensive to generate, consider using `FirstOrElse` to defer execution to when a match is not found.
{% endhint %}

### FirstOrElse

Returns the first element of the collection that matches the predicate, or runs the factory you gave to construct the fallback value if there are no matches.

```csharp
List<Option<string>> collection = [
    Option.Some("Hello"),
    Option.Some("World")
];

string first = collection.FirstOrElse(x => x.StartsWith("V"), () => "Victor");
//     ^? "Victor"
```
