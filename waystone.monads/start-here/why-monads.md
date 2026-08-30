---
description: Why return an Option or a Result instead of using null and exceptions.
icon: diamond-half-stroke
---

# Why monads

You already have ways to say "no value" and "it failed". C# gives you `null` and
it gives you exceptions. So why add a library?

Because neither one shows up where you read it: the method signature.

## What goes wrong today

Nothing here is exotic. It is the ordinary shape of C# code.

* A method returns `null` when a business rule says there is nothing to return.
* A method throws when a business rule says the work cannot continue.
* Callers wrap the call in `try`/`catch` just to log and re-throw.
* Guard clauses pile up at the top of every method.
* Branching logic spreads across `if`, `else` and `switch` until the happy path
  is hard to find.

The cost is that a signature tells you almost nothing. `Ritual PrepareRitual(decimal)`
looks total. It might return `null`. It might throw. You find out in production.

## What a monad actually is

The word sounds academic. In practice it is small.

A monad is a type that wraps a value and gives you one consistent way to keep
working with it — including when there is no value, or when something failed.

{% hint style="success" %}
A monad has to do three things:

1. Let you wrap a value — `Some`, `Ok`.
2. Let you chain work onto it — `Map`, `AndThen`.
3. Carry the context along — whether it is missing, or failed.
{% endhint %}

This library ships two of them:

* `Option<T>` — there might be a value.
* `Result<T, E>` — this might have failed.

That is the whole idea. The rest is what you can do with it.

## The same code, both ways

Here is a spell being cast. Components go in, an effect comes out, and several
things can go wrong along the way.

Written the usual way:

```csharp
Ritual? PrepareRitual(decimal components); // can return null or throw
SpellEffect Cast(Ritual ritual);           // can still return null or throw

SpellEffect CastSpell(decimal? components)
{
    if (components is null)
    {
        return Cantrip();
    }

    try
    {
        Ritual? ritual = PrepareRitual(components.Value);

        return ritual is not null
            ? Cast(ritual)
            : Cantrip();
    }
    catch (FailedToPrepareRitualException ex) // an exception for a valid outcome
    {
        _logger.LogWarning(ex, "Failed to prepare the ritual");
        return Cantrip();
    }
    catch (Exception ex)
    {
        _logger.LogWarning(ex, "Failed to cast the spell");
        throw;
    }
}

SpellEffect effect = CastSpell(10); // an effect, or a thrown exception

string message = effect?.Message ?? "Something went wrong"; // the error is gone
```

Count what is actually happening. Two of those branches are business rules
wearing an exception costume. The caller still cannot tell success from failure,
and the reason for the failure was thrown away on the last line.

Now the same thing with monads:

```csharp
Option<Ritual> PrepareRitual(decimal components);       // never throws, Some or None
Result<SpellEffect, Error> Cast(Ritual ritual);         // Ok or Err

Result<SpellEffect, Error> CastSpell(Option<decimal> components) =>
    components.AndThen(PrepareRitual) // if components are Some, prepare the ritual
              .Map(Cast)              // if the ritual is Some, cast it
              .Transpose()            // turn the Option<Result> into a Result<Option>
              .InspectErr(error => _logger.LogWarning(error))  // if Err, log it
              .Map(effect => effect.UnwrapOrElse(Cantrip));    // if Ok, take the effect

string message = CastSpell(Option.Some(10.0m)).Match(
    onOk: effect => effect.Message,
    onErr: error => error.Message); // the first error the pipeline hit
```

{% hint style="info" %}
`_logger` is whatever logger you already use. It is scenery in both samples —
nothing in this library requires one.
{% endhint %}

No conditionals. No defensive `null` checks. No local variables scattered
between the steps. Every failure that can happen is named in a signature, and
the reason survives all the way to the caller.

## The picture: two railway tracks

Think of your program as a train.

Each step of your logic is a station. The train loads data, transforms it, runs
a check, and moves on. Things go wrong: a record is missing, a validation fails,
a file is not there.

**Without monads, a problem derails the train.** An exception unwinds the stack
past every station you cared about. A `null` slips through and derails you three
stations later, somewhere unrelated. Some methods return `null`, some throw, some
just work — so you cannot chain them without guessing.

**With monads, there are two tracks.** A success track, where the train keeps
moving. A failure track, where it is quietly diverted to a siding.

* :train2: **Success track** — the next step runs.
* :construction: **Failure or none track** — the next step is skipped, and the
  train arrives carrying the reason.

The train never jumps between tracks by accident. It stays on one or the other,
and every station is built to handle both.

```csharp
Option<string> patron = FindCharacter(name)
    .Map(character => character.Spellbook)
    .AndThen(spellbook => spellbook.Patron);
```

If the character does not exist, or the spellbook has no patron, nothing crashes.
The train stops safely at `None`, and you decide what that means later:

```csharp
string display = patron.Match(
    some => some,
    () => "[No patron]");
```

## Where to go next

* [Quickstart](quickstart.md) — install the package and run both types yourself.
* [Option\<T>](../guides/option.md) — absence, in depth.
* [Result\<T, E>](../guides/result.md) — failure, in depth.
