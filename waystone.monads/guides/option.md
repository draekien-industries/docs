---
description: Model a value that might not be there, and work with it without null checks.
icon: option
---

# Option\<T>

`Option<T>` says a value might not be there. It has exactly two shapes:

* `Some<T>` — there is a value.
* `None<T>` — there is not.

That is the whole type. What makes it worth using is that you cannot read the
value without acknowledging the second case, so the compiler catches what a
`null` check would have let through.

{% hint style="info" %}
Other languages call this `Maybe<T>`. Same idea.
{% endhint %}

## Why not just use null?

`null` tells you nothing. It does not say why the value is missing, or whether it
was ever meant to be there. It spreads guard clauses through your code, and the
compiler will still happily let you dereference it.

`Option<T>` says the absence out loud, in the signature, where you already look.

There is a second difference that matters more than it sounds. `None` is not an
error. When you write:

```csharp
Option<Character> FindCharacter(string name);
```

you are not saying "this might blow up". You are saying "this might not find
anything, and that is a normal outcome". If you need to know *why* it failed,
reach for [`Result<T, E>`](result.md) instead.

## Create one

```csharp
Option<string> some = Option.Some("Keyleth");
Option<string> none = Option.None<string>();
Option<string> fromNullable = Option.FromNullable(sigil);
Option<string> fromTry = Option.Try(() => sigil!.Split('@')[1]);
```

* `Option.Some` and `Option.None<T>` are the two you will write most.
* `Option.FromNullable` takes something that might already be `null` — usually at
  the edge of your code, where you cannot control the shape.
* `Option.Try` runs a function that might throw and gives you `None` if it does.

## Transform it

You rarely want to look inside an `Option`. You want to keep working, and let the
`None` case take care of itself.

### Map

`Map` changes the value if there is one, and does nothing if there is not.

```csharp
Option<int> nameLength = FindCharacter(name)
    .Map(character => character.Name.Length);
```

### AndThen

Use `AndThen` when the next step *also* returns an `Option`. `Map` would give you
an `Option<Option<T>>`; `AndThen` keeps it flat.

```csharp
Option<string> domain = FindPatron(name).AndThen(TryExtractDomain);
```

{% hint style="info" %}
`AndThen` short-circuits. If anything in the chain is `None`, the later functions
never run.
{% endhint %}

{% hint style="warning" %}
This was called `FlatMap` before 5.4.0. It was `[Obsolete]` through 5.x and 6.0.0
removed it, so a call to it is `CS0117` rather than a warning. `WM2014`, the rule
that reported each call site, retired with it — delete any `.editorconfig` entry
for that id. See [deprecations.md](../upgrading-and-deprecations/deprecations.md "mention").
{% endhint %}

### Filter

`Filter` keeps a value only if it passes your predicate. If it does not, you get
`None`.

```csharp
Option<string> maybeName = Option.Some("Thordak");

Option<string> nonEmpty = maybeName.Filter(name => name.Length > 0); // Some("Thordak")
Option<string> blank = maybeName.Filter(name => name.Length == 0);   // None
```

This is the replacement for an `if`-guard in the middle of a pipeline.

### Chain them

Put those three together and the whole thing reads top to bottom, with no
branching at all:

```csharp
Option<string> patron = FindCharacter(name)
    .Filter(character => character.Name.Length > 0)
    .AndThen(character => character.Patron)
    .Map(patron => patron.ToUpperInvariant());
```

Compare that with the version you would otherwise write:

```csharp
string? DoWork(string name)
{
    Character? character = FindCharacter(name);

    if (character is { Name.Length: > 0, Patron: not null })
    {
        return character.Patron.ToUpperInvariant();
    }

    return null;
}
```

Same behaviour. One of them tells you what it is doing.

## Get the value back out

Every chain ends somewhere. These are the ways out.

### Match

`Match` is the honest one. You supply both branches and get a plain value.

```csharp
string patron = maybePatron.Match(
    patron => patron,
    () => "[No patron]");
```

### Unwrap with a fallback

```csharp
string orFallback = maybePatron.UnwrapOr("[No patron]");
string orComputed = maybePatron.UnwrapOrElse(() => "[No patron]");
```

Use `UnwrapOr` when the fallback is already sitting there. Use `UnwrapOrElse`
when producing it costs something — the function only runs on a `None`.

{% hint style="danger" %}
There is also a bare `Unwrap()`. It throws on a `None`. It exists for the cases
where absence really is a bug, and the analyzer will tell you when you have
reached for it out of habit.
{% endhint %}

### Pattern matching

Since 7.0.0, `Option<T>` deconstructs, so C# pattern matching works on it
directly:

```csharp
if (maybePatron is Some<string>(var patron))
{
    logger.LogInformation("Sworn to {Patron}", patron);
}
```

And exhaustively, in a `switch`:

```csharp
string display = maybePatron switch
{
    Some<string>(var patron) => patron,
    None<string> => "[No patron]",
    _ => throw new UnreachableException(),
};
```

{% hint style="info" %}
The discard arm is there because the compiler cannot see that `Some` and `None`
are the only two cases. It never runs.
{% endhint %}

## Check without unwrapping

Sometimes you only want a `bool`.

```csharp
Option<string> maybePatron = Option.Some("The Raven Queen");

maybePatron.IsSomeAnd(patron => patron.Length > 0);                 // true
maybePatron.IsNoneOr(patron => patron.Length > 0);                  // true
maybePatron.IsNoneOr(patron => string.IsNullOrWhiteSpace(patron));  // false
```

* `IsSomeAnd` — there is a value **and** it passes the predicate.
* `IsNoneOr` — there is no value, **or** the one there passes.

## Combine two options

### Zip, ZipWith and Unzip

`Zip` pairs two options into one, and gives `None` if either side is missing.

```csharp
Option<string> vex = Option.Some("Vex'ahlia");
Option<string> vax = Option.Some("Vax'ildan");
Option<string> missing = Option.None<string>();

Option<(string, string)> twins = vex.Zip(vax);     // Some(("Vex'ahlia", "Vax'ildan"))
Option<(string, string)> alone = vex.Zip(missing); // None
```

`ZipWith` does the same but combines the two values yourself instead of making a
tuple:

```csharp
Option<int> fireball = Option.Some(24);
Option<int> sneakAttack = Option.Some(18);

Option<int> total = fireball.ZipWith(sneakAttack, (a, b) => a + b);
//         ^? Some(42)
```

`Unzip` reverses a `Zip`:

```csharp
(Option<string>, Option<string>) unzipped = twins.Unzip();
//                              ^? (Some("Vex'ahlia"), Some("Vax'ildan"))
```

A component that happens to equal its type's default is an ordinary value here,
so `Option.Some((0, "x")).Unzip()` gives `(Some(0), Some("x"))`. This threw
before 6.0.0.

### Fallback chains

`Or` takes the first `Some` it finds. `OrElse` is the same, but the fallback is
only built if it is needed.

```csharp
Option<string> result = chosen.Or(absent).Or(fallback);
//             ^? Some("Keyleth")

Option<string> lazy = first
    .OrElse(() => RollForAnother())
    .OrElse(() => SendInTheHireling());
//     ^? Some("The understudy")
```

### The rest

Three more exist, and each is occasionally exactly what you want.

| Method | What it does |
| --- | --- |
| `And` | Returns the second option, but only if the first was `Some`. Answers "did both arrive?" |
| `Reduce` | Merges two options of the same type. Your function runs only when both are `Some`; otherwise the one that exists comes back untouched. |
| `Xor` | Returns the value only if exactly one of the two is `Some`. |

```csharp
Option<int> both = maybeName.And(maybeLevel);                    // Some(19)
Option<int> merged = firstRoll.Reduce(secondRoll, (a, b) => a + b); // Some(7)
Option<string> exclusive = bardsong.Xor(silence);                // Some("Scanlan")
```

## Work with a collection of them

A `List<Option<T>>` has its own set of helpers — `Collect`, `Flatten`, `Map`,
`Filter`, `FirstOrNone` and friends. They live in
`Waystone.Monads.Options.Extensions`, and are covered on the
[Option\<T> reference](../using-the-library/option-of-t/README.md#collections).

To step out of a single `Option` and into `System.Linq`, use `AsEnumerable`. It
gives you a sequence of nothing or one:

```csharp
Option<string> maybeName = Option.Some("Pike");

IEnumerable<string> sequence = maybeName.AsEnumerable();
//                  ^? ["Pike"], and [] for a None
```

For real LINQ query syntax over an `Option` — `from`, `where`, `select`, staying
inside the monad the whole way — see
[Waystone.Monads.Linq](../companion-packages/linq.md).

## Printing and logging

**`ToString()` never shows the wrapped value.** You get the state and nothing
else:

```csharp
Option.Some("Vex'ahlia").ToString() // "Some { IsSome = True, IsNone = False }"
Option.None<string>().ToString()    // "None { IsSome = False, IsNone = True }"
```

`Option<T>` is a record and `Some<T>` keeps its value in a private property, so
the compiler-generated `ToString()` has nothing to print. Interpolating an option
into a log message tells you whether a value was there, never what it was.

To log the value when it exists, use `Inspect`. It runs your action only on a
`Some`, and hands the option back so the chain continues:

```csharp
Option<Character> character = FindCharacter(name)
    .Inspect(c => logger.LogInformation("Found character {Name}", c.Name));
```

Nothing runs on a `None`. To log both branches, use `Match`.

## When to reach for it

Use `Option<T>` when:

* The value is intentionally optional, not missing by accident.
* You want a chain that bails out early on absence.
* You do not care *why* it is absent.
* You want the caller to have to deal with the empty case.

Reach for something else when:

* The default of a value type already means absence — `0` for a count, say.
* You care about the reason. That is
  [`Result<T, E>`](result.md).

## Where to go next

* [Result\<T, E>](result.md) — the same idea, for failure.
* [Async](async.md) — keeping a chain intact across an `await`.
* [Option\<T> reference](../using-the-library/option-of-t/README.md) — every overload, when you need one this page did not show.
