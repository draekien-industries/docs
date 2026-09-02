---
description: >-
  Turn several fields into one object, gate a parse without producing a value,
  and write a rule that spans two fields.
icon: table-columns
---

# Field sets

A field set is how several schemas become one object. You list the fields; the
generator writes the ladder that assembles them.

Derive from `SchemaConfig<TIn, TOut>` and mark the class `partial`. That is the whole
trigger — there is no attribute.

## The four kinds of field

### Required

<!-- snippet: schema-field-sets-required -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/FieldSets.cs -->
```csharp
// Absent, null, or failing the schema is a violation at "name". The value
// reaches the constructor as a plain string.
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
// Overrides the message for the value being absent. The schema's own rules
// keep the messages they came with.
Field<string> title =
    Schema.Required(subject.Title, Guild.Title, "Every party needs {Path}.");
```
<!-- endSnippet -->

### Optional

<!-- snippet: schema-field-sets-optional -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/FieldSets.cs -->
```csharp
// Absent is accepted. The value arrives as Option<int>, so null never
// reaches a rule and never reaches the constructed object.
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
// Yields Checked rather than a value, so it goes to Refine and takes no
// slot in the Into lambda.
Field<Checked> legacy =
    Schema.Forbidden(subject.LegacyId, "Do not send {Path}.");
```
<!-- endSnippet -->

Reject a value at a path where you permit none. It yields `Checked` rather than a
value, so it goes to `Refine` and takes no slot in the `Into` lambda.

### Extend

<!-- snippet: schema-field-sets-extend -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/FieldSets.cs -->
```csharp
Field<Checked> chronology = Schema.Extend(subject, Chronology);
```
<!-- endSnippet -->

Runs a schema over the whole subject. Also yields `Checked`.

## A rule that spans two fields

`Schema.For<T>()` over the subject is the right home for a rule about more than one
field. Its violations land at the subject's own path rather than under a field name,
which is honest — the failure belongs to neither field alone.

<!-- snippet: schema-field-sets-cross-field -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Schemas.Sample/FieldSets.cs -->
```csharp
// A schema over the whole subject. Its violations land at the subject's own
// path rather than under a field name, which is what makes it the right home
// for a rule that spans two fields.
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

Three things, into your `partial` class:

1. `Instance` — a shared, ready-to-use schema, so you never new one up.
2. `Schema` — a nested type carrying the `Fields` overload at exactly your field
   count.
3. `FieldSet<…>` — the struct that `Fields` returns, with `Refine`, `Into` and
   `Checked` on it.

That nested `Schema` is why `Schema.Fields` is arity-checked. It also means the name
`Schema` inside your class resolves to the generated type first, and the runtime
`Schema` members reach you through it by inheritance.

Three consequences follow, and each has a diagnostic:

* Your class and every type containing it must be `partial` —
  [`WMSC0001`](../../source-generation/diagnostics.md#wmsc0001).
* It needs an accessible parameterless constructor —
  [`WMSC0002`](../../source-generation/diagnostics.md#wmsc0002).
* You cannot declare a member named `Instance`, `Schema` or `FieldSet` yourself —
  [`WMSC0003`](../../source-generation/diagnostics.md#wmsc0003).

## Paths come from your source

A field's path is read from the expression you passed, using
`CallerArgumentExpression`. `subject.Title` gives `title`, which is the case the whole
design is built around.

Anything else keeps its punctuation. A method call, an indexer or a null-forgiving
operator gives a path that reaches your logs and your API responses looking like
source code. [`WMSC0008`](../../source-generation/diagnostics.md#wmsc0008) warns when
that happens; add `.Named("...")` to fix it.
