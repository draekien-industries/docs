---
description: >-
  Parse lists and dictionaries, read the path a violation was found at, and cap
  what a hostile payload can cost you.
icon: list-tree
---

# Structures

A list or a dictionary is parsed with a schema per part. The item schema can be any
schema — including one you wrote.

## Lists

<!-- snippet: schema-structures-list -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Structures.cs -->
```csharp
// At least one objective, at most ten, and each one trimmed and bounded.
public static readonly Schema<IReadOnlyList<string>, IReadOnlyList<string>>
    Objectives = Schema.List(Schema.Text.Trim().NotEmpty().MaxLength(120))
                       .MinCount(1)
                       .MaxCount(10);
```
<!-- endSnippet -->

Every item is parsed, so a bad item at index 3 does not hide a bad one at index 7.
Both are reported.

`MaxCount` is the exception. It counts *before* parsing anything, which is what makes
it a guard on untrusted input rather than a report afterwards. An eleventh objective
is rejected on its own, with nothing said about the ten.

The item schema is just a schema, so nothing stops it being one of yours.

<!-- snippet: schema-structures-list-of-objects -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Structures.cs -->
```csharp
public static readonly Schema<IReadOnlyList<LeaderDto>, IReadOnlyList<Leader>>
    Leaders = Schema.List(LeaderSchema.Instance).MinCount(1);
```
<!-- endSnippet -->

## Dictionaries

<!-- snippet: schema-structures-dictionary -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Structures.cs -->
```csharp
// A schema for the keys and a schema for the values.
public static readonly
    Schema<IReadOnlyDictionary<string, int>, IReadOnlyDictionary<string, int>>
    Bounties = Schema.Dictionary(
                          Schema.Text.Trim().Matches(BountyName),
                          Schema.Number.Int32.Positive())
                     .MaxCount(50);
```
<!-- endSnippet -->

Keys are parsed too, so a malformed key is a violation rather than a silent entry
nobody ever looks up. `MaxCount` counts first here as well — a fifty-first entry is
rejected before any key is read.

## Paths

A violation inside a structure carries where it was found. The path reads `[1]` for a
list position and `leader.email` through a nested schema; in a field set both are
prefixed by the field, giving `objectives[1]`.

<!-- snippet: schema-structures-indexed-paths -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Structures.cs -->
```csharp
// A violation inside a structure carries where it was found, so the path
// reads "[1]" for a list and "leader.email" through a nested schema. In a
// field set both are prefixed by the field: "objectives[1]".
SchemaViolation violation =
    Objectives.Parse(["Rescue the cleric", "  "]).UnwrapErr();

IReadOnlyList<string> paths =
    violation.Violations.Select(failure => failure.Path.ToString())
             .ToList();
```
<!-- endSnippet -->

### Reading a path in code

The rendered path is written for a human. Branch on the segments instead.

<!-- snippet: schema-structures-path-segments -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Structures.cs -->
```csharp
// The rendered path is written for a human. Branch on the segments
// instead: a list position, a dictionary key and a failed Schema.Any
// branch all render inside brackets, and only the segment says which
// one you are looking at.
SchemaViolation violation =
    Objectives.Parse(["Rescue the cleric", "  "]).UnwrapErr();

PathSegment last = violation.Violations[0].Path.Segments[^1];

string located = last.Kind switch
{
    PathSegmentKind.Index => $"entry {last.Text}",
    PathSegmentKind.Key => $"key {last.Text}",
    PathSegmentKind.Branch => $"alternative {last.Text}",
    _ => last.Text,
};
```
<!-- endSnippet -->

A list position, a dictionary key and a failed `Schema.Any` branch all render inside
brackets. Only the segment's `Kind` tells you which one you are looking at.

## The report is capped

One list or dictionary reports at most 64 problems and then stops.

<!-- snippet: schema-structures-too-many-problems -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Structures.cs -->
```csharp
// One list or dictionary reports at most 64 problems and then stops, so a
// hostile payload cannot make the report as expensive as it likes. When it
// stops, it says so with a truncated violation rather than trailing off.
string[] blanks = Enumerable.Repeat("  ", 500).ToArray();

SchemaViolation violation =
    Schema.List(Schema.Text.NotEmpty()).Parse(blanks).UnwrapErr();

bool thereAreMore = violation.Violations.Any(
    failure =>
        failure.Code == ViolationCodeCatalog.Codes.Truncated);
```
<!-- endSnippet -->

That cap is a guard, not a limitation of the design. Without it, a payload of ten
thousand blank strings costs ten thousand rendered messages to reject — and whoever
sent it chose the number.

When the cap is reached, the report says so with a `Truncated` violation rather than
trailing off. Check for that code before you tell a caller the list they sent has
exactly 64 problems.
