---
description: Methods for working with a sequence of Option<T>.
icon: list
---

# Collections

A `List<Option<T>>` — the results of looking something up once per item — comes up
often enough to have its own methods.

Every method here is an extension method, in
`Waystone.Monads.Options.Extensions`. Add the `using`.

## Filter

```csharp
IEnumerable<Option<T>> Filter<T>(this IEnumerable<Option<T>> source, Predicate<T> predicate)
```

The sequence version of [`Filter`](transform.md#filter). Every option that fails
the predicate is flipped to `None`. Nothing is dropped.

```csharp
List<Option<string>> collection = [
    Option.Some("Hello"),
    Option.Some("World"),
    Option.None<string>()
];

IEnumerable<Option<string>> filtered = collection.Filter(x => x == "Hello");
//                          ^? [Some("Hello"), None, None]
```

## Map

```csharp
IEnumerable<Option<TOut>> Map<T, TOut>(this IEnumerable<Option<T>> source, Func<T, TOut> map)
```

The sequence version of [`Map`](transform.md#map). The transformation runs on
every `Some`; the `None`s pass through untouched.

```csharp
IEnumerable<Option<string>> mapped = collection.Map(x => $"{x}!");
//                          ^? [Some("Hello!"), Some("World!"), None]
```

## Flatten

```csharp
IEnumerable<T> Flatten<T>(this IEnumerable<Option<T>> source)
```

Drops the `None`s and keeps the values, in order.

```csharp
List<Option<string>> collection = [
    Option.Some("Hello"),
    Option.None<string>(),
    Option.Some("World")
];

IEnumerable<string> values = collection.Flatten();
//                  ^? ["Hello", "World"]
```

**Lazy.** It walks the source once and composes with the rest of LINQ as you would
expect. Nothing runs until you enumerate the result.

{% hint style="info" %}
This is the sequence version. The `Flatten` that collapses a single nested
`Option<Option<T>>` is a different method — see
[Nesting](nesting.md#flatten). No receiver is both, so the two never compete.
{% endhint %}

## Collect

```csharp
Option<IReadOnlyList<T>> Collect<T>(this IEnumerable<Option<T>> source)
```

For when every value has to be present. You get a `Some` holding all of them, or a
single `None` if any is missing.

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

This is the opposite of `Flatten`. `Flatten` drops what is missing and carries on;
`Collect` treats one missing value as a failure of the whole batch.

**It stops at the first `None`.** It never looks at the rest of the source, so
anything that would have produced the later elements does not run.

**On an empty sequence:** you get `Some` of an empty list, not `None`. There is
nothing missing in it.

**The result does not tell you *which* element was absent.** Use `Partition` on a
sequence of `Result` when you need to know what failed.

{% hint style="info" %}
`Collect` is eager, and builds a list as it goes. Do not call it on an unbounded
sequence.
{% endhint %}

## CollectAsync

```csharp
ValueTask<Option<IReadOnlyList<T>>> CollectAsync<T>(
    this IAsyncEnumerable<Option<T>> source,
    CancellationToken cancellationToken = default)
```

The same job over an `IAsyncEnumerable`.

```csharp
Option<IReadOnlyList<string>> all = await stream.CollectAsync(cancellationToken);
```

It stops pulling from the stream at the first `None`, so the work behind the later
elements never happens. That is the reason to use it rather than reading the whole
stream into a list and calling `Collect`.

Returned `Task` up to 6.7.0. Returns `ValueTask` from 7.0.0.

## AsEnumerable

```csharp
IEnumerable<T> AsEnumerable<T>(this Option<T> option)
```

Treats a single option as a sequence of nothing or one. `Flatten` above is built
out of it, and it is the way out of the monad into `System.Linq`.

```csharp
Option<string> maybeName = Option.Some("Pike");

IEnumerable<string> sequence = maybeName.AsEnumerable();
//                  ^? ["Pike"], and [] for a None
```

It is **not** how you write a LINQ query over an `Option`. For that — `from`,
`select`, `where`, staying inside the `Option` throughout — see
[Waystone.Monads.Linq](../../packages/linq.md).

## FirstOrNone

```csharp
Option<T> FirstOrNone<T>(this IEnumerable<Option<T>> source, Predicate<T> predicate)
```

The first element matching the predicate, or `None`.

```csharp
List<Option<string>> collection = [
    Option.Some("Hello"),
    Option.Some("World")
];

Option<string> first = collection.FirstOrNone(x => x.StartsWith("H"));
//             ^? Some("Hello")
```

## FirstOr

```csharp
T FirstOr<T>(this IEnumerable<Option<T>> source, Predicate<T> predicate, T fallback)
```

The first match, or the fallback you supplied.

```csharp
string first = collection.FirstOr(x => x.StartsWith("V"), "Victor");
//     ^? "Victor"
```

**Evaluated eagerly.** Use [`FirstOrElse`](#firstorelse) if the fallback costs
something to build.

## FirstOrElse

```csharp
T FirstOrElse<T>(this IEnumerable<Option<T>> source, Predicate<T> predicate, Func<T> createFallback)
```

The same, building the fallback only when there is no match.

```csharp
string first = collection.FirstOrElse(x => x.StartsWith("V"), () => "Victor");
//     ^? "Victor"
```
