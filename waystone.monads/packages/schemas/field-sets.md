---
description: >-
  Turn several fields into one object, gate a parse without producing a value,
  and write a rule that spans two fields.
icon: table-columns
---

# Field sets

<!-- prerelease:7.1.0 -->
{% hint style="warning" %}
**`Waystone.Monads.Schemas` is in pre-release.** The current version is
`7.1.0-beta.1`, so installing it needs `--prerelease`. Everything on this page is
what 7.1.0 is expected to ship, but the surface can still change before it does.
{% endhint %}

A field set is how several schemas become one object. You list the fields; the
generator writes the code that assembles them.

Derive from `SchemaConfig<TIn, TOut>` and mark the class `partial`. There is no
attribute. Three rules apply, and each one is the reason a first attempt does not
compile:

* The class, and every type containing it, has to be `partial` —
  [`WMSC0001`](../../source-generation/diagnostics.md#wmsc0001).
* It needs a constructor you can call with no arguments —
  [`WMSC0002`](../../source-generation/diagnostics.md#wmsc0002).
* Do not declare a member called `Instance`, `Schema` or `FieldSet` —
  [`WMSC0003`](../../source-generation/diagnostics.md#wmsc0003).

## The four kinds of field

### Required

<!-- snippet: schema-field-sets-required -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/FieldSets.cs -->
```csharp
Field<string> name =
    Schema.Required(subject.Name, Schema.Text.Trim().NotEmpty());
```
<!-- endSnippet -->

Absent, `null`, or failing the schema is a violation at that field's path. The value
reaches your constructor as a plain `string`, never a `string?`.

An optional third argument overrides the message for the value being *absent*. The
schema's own rules keep the messages they came with.

<!-- snippet: schema-field-sets-required-message -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/FieldSets.cs -->
```csharp
Field<string> title =
    Schema.Required(subject.Title, Guild.Title, "Every party needs {Path}.");
```
<!-- endSnippet -->

### Optional

<!-- snippet: schema-field-sets-optional -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/FieldSets.cs -->
```csharp
Field<Option<int>> size =
    Schema.Optional(subject.Size, Schema.Number.Int32.AtLeast(1).AtMost(6));
```
<!-- endSnippet -->

Absent is accepted. The value arrives as `Option<int>`, so `null` never reaches a rule
and never reaches the constructed object.

A value that *is* present still has to pass the schema. Optional means "may be
absent", not "may be wrong".

### Forbidden

<!-- snippet: schema-field-sets-forbidden -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/FieldSets.cs -->
```csharp
Field<Checked> legacy =
    Schema.Forbidden(subject.LegacyId, "Do not send {Path}.");
```
<!-- endSnippet -->

Reject a value at a path where you allow none.

This is the first of the two fields that yield **`Checked`**. `Checked` means "this
rule passed, and it has nothing to hand you" — the field gates the parse without
contributing to the object. Fields like that go to `Refine` and take no slot in the
`Into` lambda.

### Extend

<!-- snippet: schema-field-sets-extend -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/FieldSets.cs -->
```csharp
Field<Checked> chronology = Schema.Extend(subject, Chronology);
```
<!-- endSnippet -->

Runs a schema over the whole subject. The other `Checked` field, so it goes to
`Refine` too.

## A rule that spans two fields

`Schema.For<T>()` over the subject is the right home for a rule about more than one
field. Its violations land at the subject's own path rather than under a field name,
which is honest — the failure belongs to neither field alone.

<!-- snippet: schema-field-sets-cross-field -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/FieldSets.cs -->
```csharp
// A rule about two fields at once, so it takes the whole subject.
public static readonly Schema<PartyDto, PartyDto> Chronology =
    Schema.For<PartyDto>()
          .Check(
               party => party.Disbanded is null
                     || party.Formed is null
                     || party.Disbanded > party.Formed,
               ViolationCode.Conflicting,
               "A party cannot disband before it forms.");
```
<!-- endSnippet -->

Hand it to the field set with `Schema.Extend`.

## Putting it together

<!-- snippet: schema-field-sets-the-whole-thing -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/FieldSets.cs -->
```csharp
public partial class PartySchema : SchemaConfig<PartyDto, Party>
{
    protected override Result<Party, SchemaViolation> Configure(PartyDto subject) =>
        Schema.Fields(
                   Schema.Required(subject.Name, Schema.Text.Trim().NotEmpty()),

                   // A nested schema is just a schema. Its violations arrive under
                   // "leader", so a reader is told which one failed.
                   Schema.Required(subject.Leader, LeaderSchema.Instance),
                   Schema.Optional(subject.Size, Schema.Number.Int32.AtLeast(1)))

              // Refine takes fields that gate the parse without producing a value,
              // so the Into lambda keeps one parameter per field above and no
              // discards.
              .Refine(
                   Schema.Forbidden(subject.LegacyId, "Do not send {Path}."),
                   Schema.Extend(subject, FieldSetsPage.Chronology))
              .Into((name, leader, size) => new Party(name, leader, size));
}
```
<!-- endSnippet -->

`Refine` takes the fields that gate the parse without producing a value, so the `Into`
lambda keeps one parameter per field in `Schema.Fields` — in order, and with no
discards.

**Pass only value-free fields to `Refine`.** It takes the non-generic `Field` base,
which drops the value side, so it will accept a field that parses something and then
throw that something away. [`WMSC0005`](../../source-generation/diagnostics.md#wmsc0005)
warns when you do.

A nested schema is just a schema. Its violations arrive under the field's name, so a
reader is told which one failed.

## Gating without building

Some schemas exist only to say yes. Finish those with `Checked()` instead of `Into`.

<!-- snippet: schema-field-sets-checked -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/FieldSets.cs -->
```csharp
// A schema that only gates finishes with Checked. There is nothing to construct,
// so there is no lambda and nothing to name.
public partial class ConsentSchema : SchemaConfig<ConsentDto, Checked>
{
    protected override Result<Checked, SchemaViolation> Configure(
        ConsentDto subject) =>
        Schema.Fields(
                   Schema.Required(subject.Terms, Schema.Text.NotEmpty()),
                   Schema.Required(subject.Privacy, Schema.Text.NotEmpty()))
              .Checked();
}
```
<!-- endSnippet -->

There is nothing to construct, so there is no lambda and nothing to name.

## What the generator writes

A shared `Instance`, and the `Fields`, `Refine`, `Into` and `Checked` members you
call.

`Fields` is written at exactly your field count. That is why a wrong-sized `Into`
lambda is a compile error rather than a surprise in production —
[`WMSC0004`](../../source-generation/diagnostics.md#wmsc0004).

## Paths come from your source

A field's path is read from the expression you passed, using
`CallerArgumentExpression`. `subject.Title` gives `title`, which is the case the whole
design is built around.

Anything else keeps its punctuation. A method call, an indexer or a null-forgiving
operator gives a path that reaches your logs and your API responses looking like
source code. [`WMSC0008`](../../source-generation/diagnostics.md#wmsc0008) warns when
that happens; add `.Named("...")` to fix it.
