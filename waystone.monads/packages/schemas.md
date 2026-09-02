---
description: >-
  Parse untrusted input into a type you could not have built without passing, and
  get every failure back at once.
icon: shield-check
---

# Schemas

`Waystone.Monads.Schemas` — a parser that hands back a `Result`.

{% hint style="info" %}
**New to this? Start with the [Schemas guide](../guides/schemas.md).** It builds one
schema end to end and explains why you would want to. This page is the reference.
{% endhint %}

## What it adds

A schema takes one type in and gives another type out. The type it gives out is one
your caller could not have constructed by hand, so holding it *is* the proof that the
input passed.

Reach for it at the edge — a request body, a message off a queue, a row from a file.
Skip it inside your domain, where the types already say what is true.

Comparing it against [FluentValidation](fluent-validation.md)? That one checks the
object you built. This one builds it.

## Write the checks once

A schema is a value. Declare it, name it, and reuse it.

<!-- snippet: schemas-a-reusable-check -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Schemas.cs -->
```csharp
public static class Guild
{
    public static readonly Schema<string, string> Title =
        Schema.Text.Trim().LengthBetween(3, 80);

    public static readonly Schema<string, string> Email =
        Schema.Text.Trim().Email();

    public static readonly Schema<decimal, decimal> Reward =
        Schema.Number.Decimal.Between(1m, 10_000m);
}
```
<!-- endSnippet -->

## Put them together

Two types are involved. The input is whatever arrived — every field nullable, nothing
checked. The output is the type you actually wanted.

<!-- snippet: schemas-the-two-types -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Cast.cs -->
```csharp
public sealed record QuestDto(
    string? Title,
    string? PatronEmail,
    decimal? GoldReward,
    int? PartySize,
    QuestRank? Rank,
    string? Nickname);

/// <summary>The thing a parse produces.</summary>
public sealed class Quest
{
    internal Quest(
        string title,
        string patronEmail,
        decimal goldReward,
        Option<int> partySize)
    {
        Title = title;
        PatronEmail = patronEmail;
        GoldReward = goldReward;
        PartySize = partySize;
    }

    public string Title { get; }

    public string PatronEmail { get; }

    public decimal GoldReward { get; }

    public Option<int> PartySize { get; }
}
```
<!-- endSnippet -->

`Quest` has no public constructor, so the schema is the only way to get one. That is
the part doing the work.

`QuestDto` carries two fields this schema ignores, which is what a real payload looks
like — you parse what you need and leave the rest.

Now derive from `SchemaConfig<TIn, TOut>`, mark the class `partial`, and list the
fields. The generator writes the rest.

<!-- snippet: schemas-a-field-set -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Schemas.cs -->
```csharp
public partial class QuestSchema : SchemaConfig<QuestDto, Quest>
{
    protected override Result<Quest, SchemaViolation> Configure(QuestDto subject) =>
        Schema.Fields(
                   Schema.Required(subject.Title, Guild.Title),

                   // The path a caller is shown is "patron", not the property
                   // name the compiler read off the argument.
                   Schema.Required(subject.PatronEmail, Guild.Email)
                         .Named("patron"),
                   Schema.Required(subject.GoldReward, Guild.Reward),
                   Schema.Optional(subject.PartySize, Schema.Number.Int32.Positive()))
              .Into(
                   (title, patron, reward, party) =>
                       new Quest(title, patron, reward, party));
}
```
<!-- endSnippet -->

Two things in that snippet are worth a second look.

* `Schema.Fields` is generated into your class. It takes exactly the number of fields
  you passed, so `Into` is checked at compile time rather than at run time.
* `Schema.Optional` yields `Option<int>`, not `int?`. A missing value never reaches a
  rule and never reaches the constructor.

## Parse something

The generator also writes a shared `Instance`, so there is nothing to new up.

<!-- snippet: schemas-parse -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Schemas.cs -->
```csharp
Result<Quest, SchemaViolation> result =
    QuestSchema.Instance.Parse(posting);
```
<!-- endSnippet -->

## Read the failures

An `Err` carries a `SchemaViolation`. It holds every individual `Violation`, each with
the path it was found at and a message written for a human.

<!-- snippet: schemas-read-the-failures -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Schemas.cs -->
```csharp
return QuestSchema.Instance.Parse(posting)
                  .Match(
                       quest => $"Accepted {quest.Title}.",
                       violation => string.Join(
                           "; ",
                           violation.Violations.Select(
                               failure =>
                                   $"{failure.Path}: {failure.Message}")));
```
<!-- endSnippet -->

`ToDictionary` gives the shape most APIs return — one entry per path, holding that
path's messages.

<!-- snippet: schemas-failures-as-a-dictionary -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Schemas.cs -->
```csharp
return QuestSchema.Instance.Parse(posting)
                  .Match(
                       _ => new Dictionary<string, string[]>(),
                       violation => violation.ToDictionary());
```
<!-- endSnippet -->

## One parse reports everything

A schema does not stop at the first problem. Three bad fields give three violations,
so your caller fixes their payload once instead of three times.

<!-- snippet: schemas-every-failure-at-once -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Schemas.cs -->
```csharp
// An empty title, no patron, and a reward of zero.
SchemaViolation violation =
    QuestSchema.Instance
               .Parse(new QuestDto("", null, 0m, null, null, null))
               .UnwrapErr();

// Three, not one. A parse reports every field it could not accept.
int failures = violation.Violations.Count;
```
<!-- endSnippet -->

There is one exception, and it is deliberate. A failed
[`Transform`](schemas/composition.md#transform) produces no value, so the rules after
it on *that* chain cannot run. Its siblings are unaffected and still report.

## Install it

```
dotnet add package Waystone.Monads.Schemas
```

The generator ships inside that package. There is nothing else to install and nothing
to wire up.

The generator's own diagnostics use the `WMSC` prefix and are listed on
[Generator diagnostics](../source-generation/diagnostics.md#wmsc-schemas).

## Where to go next

| Page | Covers |
| --- | --- |
| [Primitives](schemas/primitives.md) | `Schema.Text`, `Schema.Number`, dates, enums, and the rules on each |
| [Composition](schemas/composition.md) | `Check`, `Transform`, `Not`, `When`, `All`, `Any`, messages and codes |
| [Structures](schemas/structures.md) | Lists, dictionaries, and the paths a violation carries |
| [Field sets](schemas/field-sets.md) | `Required`, `Optional`, `Forbidden`, `Extend`, `Refine` |
| [Asynchrony](schemas/asynchrony.md) | `CheckAsync`, `ParseAsync`, and where an async rule may not go |
