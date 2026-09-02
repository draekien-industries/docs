---
description: >-
  Rules that have to ask something — a database, a service — and the one place
  they may not go.
icon: clock-rotate-left
---

# Asynchrony

<!-- prerelease:7.1.0 -->
{% hint style="warning" %}
**`Waystone.Monads.Schemas` is in pre-release.** The current version is
`7.1.0-beta.1`, so installing it needs `--prerelease`. Everything on this page is
what 7.1.0 is expected to ship, but the surface can still change before it does.
{% endhint %}

Most rules answer from the value alone. Some have to ask something else: is this title
already taken, does this account exist. `CheckAsync` is for those.

## CheckAsync

<!-- snippet: schema-async-check -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Asynchrony.cs -->
```csharp
// The cheap rules first, then the round trip.
public static Schema<string, string> UniqueTitle(IQuestBoard board) =>
    Schema.Text.Trim()
          .NotEmpty()
          .MaxLength(80)
          .CheckAsync(
               board.TitleIsFree,
               ViolationCode.Duplicate,
               "{Path} is already on the board, got {Received}.");
```
<!-- endSnippet -->

The rule is handed the value and the parse's own cancellation token. It runs only when
everything before it accepted, so it never sees a value the chain could not produce —
your round trip is not spent on input that was already going to fail.

Otherwise it is `Check`. Same three arguments, same behaviour on failure: the value
survives and the rest of the chain still runs.

## ParseAsync

<!-- snippet: schema-async-parse -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Asynchrony.cs -->
```csharp
Result<string, SchemaViolation> result =
    await UniqueTitle(board).ParseAsync(title, cancellationToken);
```
<!-- endSnippet -->

`ParseAsync` returns a `ValueTask<Result<TOut, SchemaViolation>>`. Pass a cancellation
token; the rule you wrote is given the same one.

A schema with no asynchronous rules can still be parsed with `ParseAsync`. It just
completes without ever yielding.

## An async rule cannot live in a field set

This is the rule to remember on this page.

`SchemaConfig.Configure` returns a value rather than a task, so a field set only ever
runs the synchronous path — even when the caller used `ParseAsync`. An asynchronous
rule reached from there throws `InvalidOperationException`. Nothing in the type system
says so, because `CheckAsync` returns the same schema type a synchronous rule does.

[`WMSC0006`](../../source-generation/diagnostics.md#wmsc0006) reports it at build
time, which is where you would rather find out.

### Compose it around the outside instead

<!-- snippet: schema-async-outside-a-field-set -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Asynchrony.cs -->
```csharp
// SchemaConfig.Configure returns a value rather than a task, so a field set
// only ever runs the synchronous path. An asynchronous rule reached from there
// throws, and WMSC0006 reports it at build time rather than in production.
//
// Compose the rule around the generated schema instead. The field set stays
// synchronous and the round trip happens once, after every cheap rule has
// already had its say.
public static ValueTask<Result<Quest, SchemaViolation>> ParseAgainstTheBoard(
    QuestDto posting,
    IQuestBoard board,
    CancellationToken cancellationToken) =>
    QuestSchema.Instance
               .CheckAsync(
                    (quest, token) => board.TitleIsFree(quest.Title, token),
                    ViolationCode.Duplicate,
                    "That quest is already on the board.")
               .ParseAsync(posting, cancellationToken);
```
<!-- endSnippet -->

The field set stays synchronous. The round trip happens once, after every cheap rule
has already had its say — so a payload with three malformed fields never reaches your
database at all.
