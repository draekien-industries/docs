---
description: Serialize Option and Result with System.Text.Json, in a format a consumer already agreed to.
icon: file-code
---

# System.Text.Json

`Waystone.Monads.SystemTextJson` — converters for `Option` and `Result`.

## What it adds

Converters for `Option<T>` and `Result<TOk, TErr>`, and one call that registers
them:

```csharp
using System.Text.Json;

JsonSerializerOptions options = new();
options.AddMonadConverters();

string json = JsonSerializer.Serialize(model, options);
```

Call it while you are still building the options. `System.Text.Json` freezes a
`JsonSerializerOptions` the first time it serializes with it, and adding a
converter after that throws.

Without this package, an `Option<T>` on a DTO serializes as its own internals,
`{"IsSome":false,"IsNone":true}`, and does not read back.

## When to reach for it

Reach for it when an `Option` or a `Result` is a member of a type you serialize, and
your application already uses `System.Text.Json`. Without it, both types serialize as
whatever their properties happen to expose, which is not a format anything can read
back.

If your application uses Json.NET instead, install
[Newtonsoft.Json](newtonsoft-json.md) — the two write the same bytes, so the choice
is decided by the serializer you already have, not by preference.

## Install it

```
dotnet add package Waystone.Monads.SystemTextJson
```

The package supports `System.Text.Json >= 8.0.5 && < 11.0.0`. Bring your own
version inside that range. Every version in the range ships a `netstandard2.0`
and a `net462` asset, so you can use it on .NET Framework too.

## Where the types live

The package shadows `System.Text.Json`'s own namespaces, so its types sit where
you already look for them.

| Member | Namespace |
| --- | --- |
| `AddMonadConverters` | `System.Text.Json` |
| `OptionJsonConverter<T>`, `OptionJsonConverterFactory` | `System.Text.Json.Serialization` |
| `ResultJsonConverter<TOk, TErr>`, `ResultJsonConverterFactory` | `System.Text.Json.Serialization` |

The converters follow `JsonConverter<T>` down into
`System.Text.Json.Serialization`. The extension method sits beside the
`JsonSerializerOptions` it extends.

The package and assembly are still called `Waystone.Monads.SystemTextJson`. Only
the namespaces shadow.

## `Option<T>` is the value, or null

This is the format Rust's serde uses. A `Some` contributes what the value alone
would have written. A `None` writes `null`.

```csharp
public sealed class Person
{
    public Option<string> Nickname { get; set; } = Option.None<string>();
}
```

```jsonc
{ "Nickname": "Ally" }   // Option.Some("Ally")
{ "Nickname": null }     // Option.None<string>()
```

So swapping a `string` property for an `Option<string>` does not change the JSON
a consumer receives. That is the point of the format.

### A None writes the property, it does not remove it

A converter cannot delete its own property from the object around it. A `None`
writes `"Nickname": null`.

**`JsonIgnoreCondition.WhenWritingNull` does not change that.** It tests the CLR
value for null, and `Option.None<T>()` is an object like any other. It is never
null, so the property is always written.
`[JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)]` fails the same
way.

A type-info modifier does work. It decides per property whether to write it:

```csharp
options.TypeInfoResolver = new DefaultJsonTypeInfoResolver
{
    Modifiers = { SkipNoneProperties },
};

static void SkipNoneProperties(JsonTypeInfo typeInfo)
{
    foreach (JsonPropertyInfo property in typeInfo.Properties)
    {
        property.ShouldSerialize = static (_, value) =>
            value is null
         || value.GetType() is not { IsGenericType: true } type
         || type.GetGenericTypeDefinition() != typeof(None<>);
    }
}
```

The package does not ship that modifier. Whether to omit a property is a
decision about your wire contract, not about `Option<T>`, and most people who
want it want it for some models and not others.

### An absent property does not read back as a None

If the property is missing from the payload, `System.Text.Json` never calls the
converter for that member. The member keeps its CLR default, which for
`Option<T>` is `null` — not `None<T>()`.

Initialise the member, as `Person.Nickname` does above. Otherwise the model holds
a null where it promised an option.

### A nested option collapses

`Option<Option<T>>` does not survive a round trip. `Some(None)` and `None` both
write `null`, and both read back as `None`:

```csharp
Option<Option<int>> before = Option.Some(Option.None<int>());
string json = JsonSerializer.Serialize(before, options);   // "null"
Option<Option<int>> after = JsonSerializer.Deserialize<Option<Option<int>>>(json, options)!;
// after is None<Option<int>>(), not Some(None<int>())
```

The converter accepts this rather than throwing. Throwing on a shape the type
system allows is worse than losing a distinction you should not be relying on.
The [`WM2009` analyzer rule](../analyzers/README.md) already
reports the declaration, which is the better place to catch it.

## `Result<TOk, TErr>` names its case

A result has no idiomatic JSON shape to borrow. Both cases carry ordinary values
of different types, so the case has to be named on the wire.

```jsonc
{ "$type": "ok",  "value": 42 }
{ "$type": "err", "value": { "Code": "validation.failed", "Message": "..." } }
```

Property order does not matter. `{"value":42,"$type":"ok"}` reads the same.

### Why the payload is nested

`$type` is also `System.Text.Json`'s own polymorphism discriminator. If your
`TOk` or `TErr` is a polymorphic base carrying `[JsonDerivedType]`, it writes a
`$type` of its own.

Nesting puts that one *inside* `value`, a level below the result's:

```jsonc
{ "$type": "ok", "value": { "$type": "cat", "Name": "Tom" } }
```

Flattening the payload beside the discriminator would have made the two
siblings. The collision would then surface only for people whose payload happens
to be polymorphic. Nesting rules it out.

### The four names are fixed

`$type`, `value`, `ok` and `err` never change.
`JsonSerializerOptions.PropertyNamingPolicy` does not rename them, so a
camel-casing service and a snake-casing one still exchange the same payload.

### Reading rejects what a result cannot hold

Deserializing throws `JsonException` when:

- the payload is not an object
- `$type` is missing, or is not a string
- `$type` names neither case
- `value` is missing
- `value` reads as null

A result has no null case. Accepting one would push the failure somewhere later
and harder to trace.

A null `value` is not rejected on sight, though. It is read as the case's own
type first. So a payload whose own converter reads `null` still round-trips —
most usefully `Result<Option<T>, TErr>`, where `Ok(None)` writes
`"value": null` and reads back as `Ok(None)`.

## Trimming and NativeAOT

Both factories close their converter reflectively, once per monad type, and the
serializer caches the result. Under NativeAOT that **throws** when a type
argument is a value type:

```
NotSupportedException: 'OptionJsonConverter`1[System.Int32]' is missing native
code or metadata.
```

A generic instantiation over a value type needs its own compiled code. The
compiler cannot see through `MakeGenericType` to know it will be asked for one.
Reference types all share a single compiled converter, so they are unaffected.

This is measured under `PublishAot` on .NET 10, not inferred:

| Registered through | Type argument | Under NativeAOT |
| --- | --- | --- |
| `AddMonadConverters()` | reference type | works |
| `AddMonadConverters()` | value type | throws |
| `options.Converters.Add(new …)` | either | works |

For `Result<TOk, TErr>` it is enough for one of the two arguments to be a value
type.

Register value-type monads explicitly instead. The concrete converters are
public, with public parameterless constructors, precisely so this path exists.
It uses no reflection at all:

```csharp
options.Converters.Add(new OptionJsonConverter<int>());
options.Converters.Add(new ResultJsonConverter<int, string>());
```

A model made of `Option<string>` and `Result<Uri, Error>` needs nothing extra.

`Option<T>` and `Result<TOk, TErr>` members do not get the source-generation fast
path from a `JsonSerializerContext`. A factory-produced converter works from one,
but you get correctness, not the speed.

## It matches the Newtonsoft.Json package byte for byte

[`Waystone.Monads.NewtonsoftJson`](newtonsoft-json.md) writes the same JSON. A
test in the repository serializes with one package, deserializes with the other,
both directions, and asserts the two write identical bytes.

So you can switch serializers, or run both in one system, without a migration on
the wire.

## What it does not do

* It does not remove a property for a `None`. The property is written with a
  `null` value, and an absent property is a read error rather than a `None`.
* It does not give `Option<T>` or `Result<TOk, TErr>` the source-generation fast
  path from a `JsonSerializerContext`. You get correctness, not speed.
* It does not change either type. Remove the package and only serialization
  breaks.
