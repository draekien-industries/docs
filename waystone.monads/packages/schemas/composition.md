---
description: >-
  Add a rule, change the type, combine two schemas, and control the message and
  the code a failure carries.
icon: layer-group
---

# Composition

Every schema is built the same way: start at a primitive, and add. This page covers
what you can add.

## Check

`Check` adds a rule. The value survives a failure, so every later rule on the chain
still runs and one parse reports all of them.

<!-- snippet: schema-composition-check -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Composition.cs -->
```csharp
// A refinement. The value survives a failure, so every later rule on the
// chain still runs and one parse reports all of them.
public static readonly Schema<string, string> Title =
    Schema.Text.Trim()
          .NotEmpty()
          .Check(
               title => !title.Contains("dragon", StringComparison.OrdinalIgnoreCase),
               ViolationCode.NotAllowed,
               "{Path} may not name a dragon, got {Received}.");
```
<!-- endSnippet -->

A rule takes three things: the predicate, a code a caller can branch on, and a
message a human reads. The message is a template — `{Path}`, `{Received}`,
`{Expected}` and `{Code}` are filled in for you.

## Transform

`Transform` changes the type the schema produces. From the transform onward, the
chain is over the new type.

<!-- snippet: schema-composition-transform -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Composition.cs -->
```csharp
// Changes the type the schema produces. From here on the chain is over
// QuestTitle, not string.
public static readonly Schema<string, QuestTitle> Titled =
    Schema.Text.Trim().NotEmpty().Transform(text => new QuestTitle(text));
```
<!-- endSnippet -->

### A transform that can fail

The second overload returns a `Result`. An `Err` becomes a violation.

<!-- snippet: schema-composition-transform-result -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Composition.cs -->
```csharp
// A transform that can fail produces no value, so its own chain stops there.
// Its siblings in a field set carry on and still report.
public static readonly Schema<string, QuestRank> Rank =
    Schema.Text.Trim()
          .Transform(
               text => Enum.TryParse(text, true, out QuestRank rank)
                   ? Result.Ok<QuestRank, Error>(rank)
                   : Result.Err<QuestRank, Error>(
                       ViolationCodeCatalog.Errors.Malformed(
                           $"'{text}' is not a rank.")));
```
<!-- endSnippet -->

**This is the one seam in the "report everything" promise.** A refinement fails and
the value survives, so the rest of that chain runs. A transform fails and there is no
value to carry, so its chain stops there. Its siblings in a field set are unaffected
and still report.

**A conversion that returns `null` is a violation, not an exception.** If the function
you passed to the non-`Result` overload returns `null`, the parse reports a
`Malformed` violation at that path and carries on gathering. It does not throw. Reach
for the `Result` overload anyway when a conversion can fail — it lets you say *why*.

## Not

`Not` inverts a schema you already have.

<!-- snippet: schema-composition-not -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Composition.cs -->
```csharp
// Reach for Not when the thing being rejected is already a schema worth
// naming. For a one-off condition, Check with the negated predicate reads
// better and costs less.
public static readonly Schema<string, string> ReservedPrefixes =
    Schema.Text.StartsWith("guild:");

// Negation has no message of its own to borrow, so one is required.
public static readonly Schema<string, string> PublicTitle =
    Schema.Text.Trim()
          .NotEmpty()
          .Not(ReservedPrefixes, "{Path} may not use a reserved prefix.");
```
<!-- endSnippet -->

Negation has no message of its own to borrow, so one is required.

Reach for `Not` when the thing being rejected is already a schema worth naming. For a
one-off condition, `Check` with the negated predicate reads better and costs less.

## When and unless

Both take the whole value, so they read as a condition on the subject rather than on
one rule.

<!-- snippet: schema-composition-when-unless -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Composition.cs -->
```csharp
// The rules run only when the predicate holds. Both take the whole value, so
// they read as a condition on the subject rather than on one field.
public static readonly Schema<string, string> SigilOfALongName =
    Schema.Text.MinLength(8).When(text => text.StartsWith("guild:"));

public static readonly Schema<string, string> NoShoutingUnlessUrgent =
    Schema.Text.Matches(NoCapitals).Unless(text => text.EndsWith("!"));
```
<!-- endSnippet -->

The rules run only when the predicate holds. `Unless` is the same thing with the
predicate inverted.

## All

Every branch runs, and every failure is reported.

<!-- snippet: schema-composition-all -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Composition.cs -->
```csharp
// Every branch runs, and every failure is reported.
public static readonly Schema<string, string> Passphrase = Schema.All(
    Schema.Text.MinLength(12),
    Schema.Text.Matches(HasADigit),
    Schema.Text.Matches(HasASymbol));
```
<!-- endSnippet -->

A passphrase that is too short, has no digit and has no symbol comes back with three
violations, not one.

## Any

The first branch that accepts wins.

<!-- snippet: schema-composition-any -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Composition.cs -->
```csharp
// The first branch that accepts wins. When none does, one violation is
// reported at the union's own path carrying the branch failures beneath it,
// rather than a flat list a reader cannot attribute.
public static readonly Schema<string, string> Contact = Schema.Any(
    Schema.Text.Matches(LooksLikeAnEmail),
    Schema.Text.Matches(LooksLikeAPhone));
```
<!-- endSnippet -->

When no branch accepts, one violation is reported at the union's own path, carrying
the branch failures beneath it. It is not flattened — a flat list of every branch's
complaints is something a reader cannot attribute to anything.

## Messages

`WithMessage` replaces the message of every violation the chain produced, not only the
last one.

<!-- snippet: schema-composition-message -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Composition.cs -->
```csharp
// Replaces the message of every violation the chain produced, not only the
// last one. Reach for it when the rules are an implementation detail and the
// reader only needs to know what shape was expected.
public static readonly Schema<string, string> Slug =
    Schema.Text.Trim()
          .NotEmpty()
          .MaxLength(40)
          .Matches(LowerCaseAndHyphens)
          .WithMessage("{Path} has to be lower case words joined by hyphens.");
```
<!-- endSnippet -->

Reach for it when the rules are an implementation detail and the reader only needs to
know what shape was expected. Skip it when the individual messages are what makes the
failure actionable.

## Codes

`WithCode` sets a domain code, so a caller can branch on the failure without matching
text.

<!-- snippet: schema-composition-code -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Composition.cs -->
```csharp
// A domain code, so a caller can branch on the failure without matching text.
public static readonly Schema<string, string> ReservedTitle =
    Schema.Text.Not(ReservedPrefixes, "Reserved prefix.")
          .WithCode(new ErrorCode("quest.title_reserved"));
```
<!-- endSnippet -->

The built-in `ViolationCode` values cover the generic cases — `Incomplete`,
`Malformed`, `NotAllowed`, `OutOfRange`, `Mismatched`, `Duplicate`, `Conflicting` and
`Truncated`. Reach for one of those when a domain code would only restate the check.

## Names

A violation's path is derived from the expression you passed, which is usually the
property name and is occasionally not what you want a caller to see.

<!-- snippet: schema-composition-named -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Composition.cs -->
```csharp
// Set the path on the field, not on the schema. A schema is shared, so a
// name baked into one renames every field of its shape and nothing
// reports it; a field is built per parse and cannot leak.
return Schema.Required(subject.PatronEmail, Schema.Text.Email())
             .Named("patron");
```
<!-- endSnippet -->

**Set the name on the field, not on the schema.** A schema is shared, so a name baked
into one renames every field of its shape and nothing reports it. A field is built per
parse and cannot leak.

`Schema.Named` is the other half, for a schema that is not reached through a field —
a branch of `Schema.Any`, or one handed straight to `Parse`.

<!-- snippet: schema-composition-named-on-a-schema -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Composition.cs -->
```csharp
// Schema.Named is the other half, for a schema that is not reached through a
// field: a branch of Schema.Any, or one handed straight to Parse.
public static readonly Schema<string, string> Contactable = Schema.Any(
    Schema.Text.Email().Named("email"),
    Schema.Text.Matches(LooksLikeAPhone).Named("phone"));
```
<!-- endSnippet -->

## Sensitive values

`{Received}` renders the value that was rejected. That is what makes most messages
useful, and it is exactly wrong for a password, a token or a tax file number — those
would land in your logs and in your API response.

`Sensitive()` opts that path out.

<!-- snippet: schema-composition-sensitive -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Composition.cs -->
```csharp
// {Received} renders *** for this schema and everything beneath it, so the
// rejected value stays out of logs and out of the response. Opt-in, because
// seeing what was rejected is what makes most messages useful.
public static readonly Schema<string, string> Secret =
    Schema.Text.NotEmpty().MinLength(12).Sensitive();
```
<!-- endSnippet -->

`{Received}` then renders `***` for this schema and everything beneath it.

Three things about it are worth knowing.

* **Marking the outermost schema is enough.** A message is rendered when it is first
  read, not when the violation is created, so a nested schema that already reported
  still renders redacted once the outer schema is known to be sensitive. Marking an
  inner one as well changes nothing.
* **`{Expected}` is not redacted.** It renders a bound your schema's author wrote, not
  anything that arrived from outside.
* **The raw value cannot be read back.** A `Violation` exposes its path, its code and
  its rendered message, and nothing else. There is no way to recover the value the
  redaction exists to withhold.
