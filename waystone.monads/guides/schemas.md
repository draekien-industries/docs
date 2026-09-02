---
description: >-
  Stop validating an object you already built. Parse the input instead, and let
  the type system carry the proof.
icon: shield-check
---

# Schemas

<!-- prerelease:7.1.0 -->
{% hint style="warning" %}
**`Waystone.Monads.Schemas` is in pre-release.** The current version is
`7.1.0-beta.1`, so installing it needs `--prerelease`. Everything on this page is
what 7.1.0 is expected to ship, but the surface can still change before it does.
{% endhint %}

There is a bug that almost every codebase has a version of. It looks like this.

<!-- snippet: schemas-guide-the-usual-way -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Guide.cs -->
```csharp
public static Registration? Register(
    RegistrationDto dto,
    List<string> problems)
{
    if (string.IsNullOrWhiteSpace(dto.Email))
    {
        problems.Add("Email is required.");
    }

    if (dto.DisplayName is null)
    {
        problems.Add("Display name is required.");
    }

    if (dto.AcceptedTerms != true)
    {
        problems.Add("You have to accept the terms.");
    }

    if (problems.Count > 0)
    {
        return null;
    }

    return new Registration(
        dto.Email!,
        dto.DisplayName!,
        dto.Age is null ? Option.None<int>() : Option.Some(dto.Age.Value));
}
```
<!-- endSnippet -->

Nothing in there is wrong. It is how most of us write it.

But look at the null-forgiving operators in that final `return`. They are the tell. The
compiler has no idea those checks ran, so nothing stops that line moving above them,
and nothing stops it being written against a field nobody checked. The checks and the
construction are two separate things that happen to be next to each other.

## Parse, don't validate

A schema closes that gap by doing both at once.

You do not hand it an object and ask whether it is valid. You hand it the raw input,
and it hands you back the object — or it hands you back every reason it could not
build one.

So holding the object *is* the proof. There is no separate step to forget.

## Three steps

### 1. Make the type unbuildable

Give your domain type a constructor nobody outside can call.

<!-- snippet: schemas-guide-the-type -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Guide.cs -->
```csharp
public sealed class Registration
{
    internal Registration(string email, string displayName, Option<int> age)
    {
        Email = email;
        DisplayName = displayName;
        Age = age;
    }

    public string Email { get; }

    public string DisplayName { get; }

    public Option<int> Age { get; }
}
```
<!-- endSnippet -->

This is the step that does the work. Once the constructor is out of reach, the only
way to hold a `Registration` is to have gone through the schema — and now the compiler
enforces that, not your code review.

Notice `Option<int>` rather than `int?`. A value that may be absent says so in its
type, so there is no null to forget about downstream.

### 2. Write the checks once

A schema is a value. Declare it, name it, and reuse it everywhere that shape of input
turns up.

<!-- snippet: schemas-guide-the-checks -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Guide.cs -->
```csharp
public static class Registrations
{
    public static readonly Schema<string, string> Email =
        Schema.Text.Trim().Email();

    public static readonly Schema<string, string> DisplayName =
        Schema.Text.Trim().LengthBetween(2, 40);

    public static readonly Schema<int, int> Age =
        Schema.Number.Int32.Between(13, 130);

    // A rule over the whole subject rather than over one field.
    public static readonly Schema<RegistrationDto, RegistrationDto> Terms =
        Schema.For<RegistrationDto>()
              .Check(
                   subject => subject.AcceptedTerms == true,
                   ViolationCode.NotAllowed,
                   "You have to accept the terms.");
}
```
<!-- endSnippet -->

`Registrations.Email` is now the *only* definition of what an email address is in your
application. When it changes, it changes in one place.

### 3. Describe the object

Derive from `SchemaConfig<TIn, TOut>`, mark the class `partial`, and list the fields.

<!-- snippet: schemas-guide-the-schema -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Guide.cs -->
```csharp
public partial class RegistrationSchema
    : SchemaConfig<RegistrationDto, Registration>
{
    protected override Result<Registration, SchemaViolation> Configure(
        RegistrationDto subject) =>
        Schema.Fields(
                   Schema.Required(subject.Email, Registrations.Email),
                   Schema.Required(subject.DisplayName, Registrations.DisplayName),
                   Schema.Optional(subject.Age, Registrations.Age))
              .Refine(Schema.Extend(subject, Registrations.Terms))
              .Into(
                   (email, name, age) => new Registration(email, name, age));
}
```
<!-- endSnippet -->

Three things are happening there.

* **`Schema.Fields` is written for you**, at exactly the number of fields you passed.
  So the `Into` lambda is checked when you compile, not when someone posts a
  registration.
* **`Schema.Optional` gives you `Option<int>`.** An absent age never reaches a rule and
  never reaches the constructor.
* **`Refine` is for rules that gate without contributing.** Accepting the terms has to
  be true, but there is nowhere on `Registration` to put it, so it goes here instead
  of into the lambda.

## Use it

<!-- snippet: schemas-guide-at-the-edge -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/Guide.cs -->
```csharp
return RegistrationSchema.Instance
                         .Parse(body)
                         .Match<IResponse>(
                              registration => new Created(registration),
                              violation => new BadRequest(
                                  violation.ToDictionary()));
```
<!-- endSnippet -->

One call, two outcomes. Either you are holding a `Registration`, or you are holding
every reason you are not.

## What you get back

A failure is a `SchemaViolation`. It carries a list of individual violations, and each
one knows the path it was found at and has a message written for a human.

Two properties of that are worth knowing up front.

**Every failure comes back at once.** A schema does not stop at the first problem. A
payload with a bad email, a short display name and no terms gives you three violations,
so whoever sent it fixes their request once instead of three times.

**Paths nest.** A violation inside a list reads `roles[1]`. One inside a nested schema
reads `address.postcode`. `ToDictionary()` turns the whole set into the
path-to-messages shape most APIs already return.

## Where this belongs

**At the edge.** A request body, a message off a queue, a row from a file — anywhere
input arrives from somewhere you do not control and has to become a domain type.

**Not inside your domain.** Past the edge, the types already say what is true. That is
the whole point of getting them right at the boundary.

## Compared to a validator

If you already write FluentValidation validators and you only want a `Result` back
from them, [FluentValidation](../packages/fluent-validation.md) is a much smaller
change. It checks the object you built.

Reach for a schema when *building the object* is where your bugs come from. That is a
different problem, and it is the one this solves.

## Read on

[Schemas](../packages/schemas.md) is the reference for the package — every primitive,
every rule, lists and dictionaries, and the parts that only come up once you are
past the first parse.
