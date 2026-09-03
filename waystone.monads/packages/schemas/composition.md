---
description: >-
  Add a rule, change the type, combine two schemas, and control the message and
  the code a failure carries.
icon: layer-group
---

# Composition

<!-- prerelease:7.1.0 -->
{% hint style="warning" %}
**`Waystone.Monads.Schemas` is in pre-release.** The current version is
`7.1.0-beta.1`, so installing it needs `--prerelease`. Everything on this page is
what 7.1.0 is expected to ship, but the surface can still change before it does.
{% endhint %}

Every schema is built the same way: start at a primitive, and add. This page covers
what you can add.

## Check

`Check` adds a rule. The value survives a failure, so every later rule on the chain
still runs and one parse reports all of them.

<!-- snippet: schema-composition-check -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Composition.cs -->
```csharp
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
`{Predicate}` and `{Code}` are filled in for you.

### Naming the condition in the message

Writing the rule out twice — once as the predicate, once as prose — is how those
two drift apart. `{Predicate}` renders the rule's own source text, so you write
the condition once.

<!-- snippet: schema-composition-check-predicate -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Composition.cs -->
```csharp
// {Predicate} renders the rule's own source text, so the condition is
// written once. A failure here reads: "Expected reward to satisfy
// reward => reward % 10 == 0."
public static readonly Schema<int, int> Reward =
    Schema.Number.Int32.Positive()
          .Check(
               reward => reward % 10 == 0,
               ViolationCode.Mismatched,
               "Expected {Path} to satisfy {Predicate}.");
```
<!-- endSnippet -->

You pass nothing to get this. The compiler captures the text for you.

Where the lambda reads badly in the middle of a sentence, pass your own wording as
a fourth argument.

<!-- snippet: schema-composition-check-predicate-override -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Composition.cs -->
```csharp
// The fourth argument replaces that source text where the lambda reads
// badly mid-sentence: "Expected reward to satisfy a multiple of ten."
public static readonly Schema<int, int> RoundReward =
    Schema.Number.Int32.Positive()
          .Check(
               reward => reward % 10 == 0,
               ViolationCode.Mismatched,
               "Expected {Path} to satisfy {Predicate}.",
               "a multiple of ten");
```
<!-- endSnippet -->

{% hint style="info" %}
`{Expected}` is the one token `Check` cannot fill. It renders a bound, and `Check`
has nowhere for you to put one, so it reaches your caller as those exact
characters. The rules that ship with the package — `AtLeast`, `MaxLength` and the
rest — supply their own bound and do fill it. In a message you write, either
interpolate the bound yourself or reach for `{Predicate}`.
{% endhint %}

## Transform

`Transform` changes the type the schema produces. From the transform onward, the
chain is over the new type.

<!-- snippet: schema-composition-transform -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Composition.cs -->
```csharp
public static readonly Schema<string, QuestTitle> Titled =
    Schema.Text.Trim().NotEmpty().Transform(text => new QuestTitle(text));
```
<!-- endSnippet -->

### A transform that can fail

The second overload returns a `Result`. An `Err` becomes a violation.

<!-- snippet: schema-composition-transform-result -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Composition.cs -->
```csharp
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
// A schema worth naming, so Not has something to invert.
public static readonly Schema<string, string> ReservedPrefixes =
    Schema.Text.StartsWith("guild:");

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
// An email address or a phone number, either being fine.
public static readonly Schema<string, string> Contact = Schema.Any(
    Schema.Text.Matches(LooksLikeAnEmail),
    Schema.Text.Matches(LooksLikeAPhone));
```
<!-- endSnippet -->

When no branch accepts, you get one violation at the `Schema.Any` schema's own path,
with each branch's failures nested beneath it.

## Messages

`WithMessage` replaces the message of every violation the chain produced, not only the
last one.

<!-- snippet: schema-composition-message -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Composition.cs -->
```csharp
// Four rules, one message.
public static readonly Schema<string, string> Slug =
    Schema.Text.Trim()
          .NotEmpty()
          .MaxLength(40)
          .Matches(LowerCaseAndHyphens)
          .WithMessage("{Path} has to be lower case words joined by hyphens.");
```
<!-- endSnippet -->

Reach for it when the rules are an implementation detail and you only need to say what
shape you expected. Skip it when the individual messages are what makes the failure
useful.

`{Expected}` and `{Predicate}` reach your caller as those exact characters here. One
message now stands for four rules, so there is no single bound or predicate left to
name. Say it in the text instead.

## Codes

`WithCode` sets a domain code, so a caller can branch on the failure without matching
text.

<!-- snippet: schema-composition-code -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Composition.cs -->
```csharp
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
// A violation reports "patron", not "patronEmail".
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
// Naming the branches, so a failure says which one was tried.
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

* **Mark the outermost schema and stop.** Everything nested inside it is redacted
  too, including a nested schema that reported before the outer one ran. Marking an
  inner schema as well changes nothing.
* **`{Expected}` and `{Predicate}` are not redacted.** One renders a bound your
  schema's author wrote and the other their rule's source text. Neither is anything
  that arrived from outside.
* **The raw value cannot be read back.** A `Violation` exposes its path, its code and
  its rendered message, and nothing else. There is no way to recover the value the
  redaction exists to withhold.
