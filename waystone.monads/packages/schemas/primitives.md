---
description: >-
  The schemas you start a chain from — text, numbers, identifiers, dates,
  booleans, enums — and the rules that hang off each.
icon: shapes
---

# Primitives

Every chain starts at a primitive. The primitive fixes the type; the rules after it
narrow what that type is allowed to hold.

## Text

`Schema.Text` accepts a `string`. Start with `Trim()` wherever the value came off a
form — a trailing space is not something you want to reject over, and it is not
something you want to store either.

<!-- snippet: schema-primitives-text -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Primitives.cs -->
```csharp
public static readonly Schema<string, string> Sigil =
    Schema.Text.Trim().LengthBetween(3, 24);

// A shape with a fixed width says so, rather than bounding both ends at the
// same number.
public static readonly Schema<string, string> CountryCode =
    Schema.Text.Trim().Length(2);

// A closed set of spellings. Schema.Enum is the better home when the domain
// already models the set as an enumeration.
public static readonly Schema<string, string> Difficulty =
    Schema.Text.Trim()
          .OneOf(
               global::System.StringComparison.OrdinalIgnoreCase,
               "easy",
               "standard",
               "deadly");
```
<!-- endSnippet -->

Length rules read as what they are. `Length(2)` is a fixed width; `LengthBetween`,
`MinLength` and `MaxLength` are bounds; `NotEmpty` is the one you will reach for
most.

### Patterns

`Matches` takes a `Regex`, not a pattern string. That is on purpose: it puts the
choice of a match timeout in front of you rather than behind you.

<!-- snippet: schema-primitives-text-pattern -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Primitives.cs -->
```csharp
// Matches takes a Regex rather than a pattern string, which is what puts the
// choice of a match timeout in front of you. [GeneratedRegex] compiles the
// expression at build time instead of at start-up.
[GeneratedRegex("^[a-z-]+$", RegexOptions.None, matchTimeoutMilliseconds: 1000)]
private static partial Regex RunePattern { get; }

public static readonly Schema<string, string> Rune =
    Schema.Text.Matches(RunePattern);

// Build it by hand where the expression is not known at compile time. Give it
// a timeout: the pattern is yours, the value is not, and an expression with no
// ceiling runs against a crafted input for as long as that input takes.
public static readonly Schema<string, string> Incantation =
    Schema.Text.Matches(
        new Regex(
            @"^\p{L}[\p{L}\s]*$",
            RegexOptions.CultureInvariant,
            TimeSpan.FromSeconds(1)));
```
<!-- endSnippet -->

{% hint style="warning" %}
**Always give the expression a timeout.** The pattern is yours, but the value is
not. An expression with no ceiling runs against a crafted input for as long as that
input takes to defeat it.
{% endhint %}

### Shapes with names

Some shapes are common enough to have their own rule, and each one is checked by a
scan rather than by an expression — so there is no pattern to get subtly wrong.

<!-- snippet: schema-primitives-text-shapes -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Primitives.cs -->
```csharp
// Checked by a scan rather than an expression, so there is no pattern to get
// subtly wrong and no matching timeout to trip.
public static readonly Schema<string, string> PatronEmail =
    Schema.Text.Trim().Email();

// Restrict the scheme whenever the value will be followed or rendered. An
// absolute URL also includes javascript: and data:.
public static readonly Schema<string, string> Portrait =
    Schema.Text.Trim().Url("https");

// Literals, not expressions. A dot or a bracket here means itself.
public static readonly Schema<string, string> Tagged =
    Schema.Text.StartsWith("quest:").EndsWith(".md");
```
<!-- endSnippet -->

{% hint style="warning" %}
**`Url()` with no scheme accepts more than you think.** An absolute URL includes
`javascript:`, `data:` and `file:`. Restrict the scheme whenever the value will be
followed or rendered — which is nearly always. Passing an empty scheme list accepts
nothing at all.
{% endhint %}

`StartsWith` and `EndsWith` take literals, not expressions. A dot or a bracket in one
means itself.

## Numbers

Four number schemas, one per type: `Int32`, `Int64`, `Decimal` and `Double`.

<!-- snippet: schema-primitives-numbers -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Primitives.cs -->
```csharp
// One rule, so a party of twelve is one failure rather than two.
public static readonly Schema<int, int> PartySize =
    Schema.Number.Int32.Between(1, 6);

public static readonly Schema<long, long> ExperienceAwarded =
    Schema.Number.Int64.Positive();

// Exclusive at both ends, which is what GreaterThan and LessThan mean.
public static readonly Schema<decimal, decimal> GoldReward =
    Schema.Number.Decimal.GreaterThan(0m).LessThan(10_000m);

public static readonly Schema<double, double> SpellRangeMetres =
    Schema.Number.Double.Positive();
```
<!-- endSnippet -->

Inclusivity is in the name, and it matters.

| Rule | Bound |
| --- | --- |
| `GreaterThan`, `LessThan` | Excluded |
| `AtLeast`, `AtMost` | Included |
| `Between` | Both included |
| `Positive`, `Negative` | Excludes zero |

Prefer one rule over two where one says the same thing. `Between(1, 6)` reports a
party of twelve as one failure; `AtLeast(1).AtMost(6)` reports the same thing and is
just longer to read.

## Identifiers

<!-- snippet: schema-primitives-identifiers -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Primitives.cs -->
```csharp
public static readonly Schema<Guid, Guid> QuestId = Schema.Id.NotEmpty();
```
<!-- endSnippet -->

`Schema.Id` accepts a `Guid`. `NotEmpty()` rejects `Guid.Empty`, which is what an
uninitialised field deserializes to and is almost never a value you meant to receive.

## Dates and times

Two schemas, and picking between them is picking what the value means.

<!-- snippet: schema-primitives-temporal -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Primitives.cs -->
```csharp
// A moment, so a time zone is part of the value.
public static readonly Schema<DateTimeOffset, DateTimeOffset> Deadline =
    Schema.Timestamp.After(DateTimeOffset.UnixEpoch);

// A day, so a time of day would be noise. Not available on netstandard2.0.
public static readonly Schema<DateOnly, DateOnly> Founded =
    Schema.Date.OnOrAfter(new DateOnly(1066, 10, 14));

// Inclusivity is in the name. Before and After exclude the bound; OnOrBefore
// and OnOrAfter include it, which is what a closing date means.
public static readonly Schema<DateOnly, DateOnly> ClosesOn =
    Schema.Date.OnOrBefore(new DateOnly(2026, 12, 31));
```
<!-- endSnippet -->

* `Schema.Timestamp` is a `DateTimeOffset` — a moment, so a time zone is part of the
  value.
* `Schema.Date` is a `DateOnly` — a day, so a time of day would be noise.

`Schema.Date` is not available on `netstandard2.0`, because `DateOnly` is not.

Inclusivity is in the name here too. `Before` and `After` exclude the bound;
`OnOrBefore` and `OnOrAfter` include it, which is what a closing date means.

## Booleans

<!-- snippet: schema-primitives-booleans -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Primitives.cs -->
```csharp
public static readonly Schema<bool, bool> AcceptedTerms =
    Schema.Bool.IsTrue();

// The rarer half, and worth a second look. A flag that has to be clear often
// reads better as the opposite flag that has to be set.
public static readonly Schema<bool, bool> NotSuspended =
    Schema.Bool.IsFalse();
```
<!-- endSnippet -->

`IsFalse` is worth a second look when you write it. A flag that has to be clear
usually reads better as the opposite flag that has to be set.

## Enums

<!-- snippet: schema-primitives-enums -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Primitives.cs -->
```csharp
// Rejects a value outside the declared members, which a cast can produce.
public static readonly Schema<QuestRank, QuestRank> Rank =
    Schema.Enum<QuestRank>();
```
<!-- endSnippet -->

`Schema.Enum<T>()` rejects a value outside the declared members. That is not a
theoretical case — a cast produces one, and so does a deserializer handed a number.

## Anything else

`Schema.For<T>()` is the identity schema. It accepts any value of that type and gives
`Check` and `Transform` somewhere to hang off.

<!-- snippet: schema-primitives-any-type -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Primitives.cs -->
```csharp
// For<T> is the identity schema: it accepts anything of that type and gives
// Check and Transform somewhere to hang off. Every primitive above is one.
public static readonly Schema<TimeSpan, TimeSpan> Duration =
    Schema.For<TimeSpan>()
          .Check(
               span => span > TimeSpan.Zero,
               ViolationCode.OutOfRange,
               "{Path} has to be longer than nothing, got {Received}.");
```
<!-- endSnippet -->

Every primitive above is a `Schema.For<T>()` with rules already attached. Reach for
the bare form when your type has none — or when you want a rule over a whole subject,
which is how [cross-field rules](field-sets.md#a-rule-that-spans-two-fields) work.
