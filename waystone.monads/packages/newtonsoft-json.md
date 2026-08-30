---
description: Serialize Option and Result with Newtonsoft.Json, in the same format the System.Text.Json package writes.
---

# Newtonsoft.Json

`Waystone.Monads.NewtonsoftJson` — converters for `Option` and `Result`.

## What it adds

Converters for `Option<T>` and `Result<TOk, TErr>`, and one call that registers
them:

```csharp
using Newtonsoft.Json;

JsonSerializerSettings settings = new JsonSerializerSettings().AddMonadConverters();

string json = JsonConvert.SerializeObject(model, settings);
```

`AddMonadConverters` returns the settings you gave it, so you can chain from it.
It appends both converters to `Converters`. Json.NET takes the first converter
that accepts a type, so a converter you registered for an option or a result
beforehand keeps priority.

Without this package, an `Option<T>` on a DTO serializes as its own internals,
`{"IsSome":false,"IsNone":true}`, and does not read back.

## When to reach for it

Reach for it when an `Option` or a `Result` is a member of a type you serialize, and
your application already uses Json.NET. Without it, both types serialize as whatever
their properties happen to expose, which is not a format anything can read back.

If your application uses `System.Text.Json` instead, install
[System.Text.Json](system-text-json.md) — the two write the same bytes, so the choice
is decided by the serializer you already have, not by preference.

## Install it

```
dotnet add package Waystone.Monads.NewtonsoftJson
```

The package supports `Newtonsoft.Json >= 13.0.1 && < 14.0.0`. Bring your own
version inside that range. Every version in the range ships a `netstandard2.0`,
a `net45` and a `net20` asset, so you can use it on .NET Framework too.

## Where the types live

The package shadows `Newtonsoft.Json`'s own namespace, so its types sit where
you already look for them.

| Member | Namespace |
| --- | --- |
| `AddMonadConverters` | `Newtonsoft.Json` |
| `OptionJsonConverter` | `Newtonsoft.Json` |
| `ResultJsonConverter` | `Newtonsoft.Json` |

`JsonConverter` and `JsonSerializerSettings` both live in the root
`Newtonsoft.Json` namespace, so everything here lands there too.

The package and assembly are still called `Waystone.Monads.NewtonsoftJson`. Only
the namespace shadows.

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

**`NullValueHandling.Ignore` does not change that.** It tests the CLR value for
null, and `Option.None<T>()` is an object like any other. It is never null, so
the property is always written.
`[JsonProperty(NullValueHandling = NullValueHandling.Ignore)]` fails the same
way.

A contract resolver does work. It decides per property whether to write it:

```csharp
settings.ContractResolver = new SkipNoneContractResolver();

public sealed class SkipNoneContractResolver : DefaultContractResolver
{
    protected override JsonProperty CreateProperty(
        MemberInfo member,
        MemberSerialization memberSerialization)
    {
        JsonProperty property = base.CreateProperty(member, memberSerialization);

        property.ShouldSerialize = instance =>
            property.ValueProvider?.GetValue(instance)?.GetType() is not
                { IsGenericType: true } type
         || type.GetGenericTypeDefinition() != typeof(None<>);

        return property;
    }
}
```

A `ShouldSerialize{PropertyName}` method on the model does the same job for one
property.

The package does not ship that resolver. Whether to omit a property is a
decision about your wire contract, not about `Option<T>`, and most people who
want it want it for some models and not others.

### An absent property does not read back as a None

If the property is missing from the payload, Json.NET never calls the converter
for that member. The member keeps whatever the model gave it, which without an
initialiser is `null` — not `None<T>()`.

Initialise the member, as `Person.Nickname` does above. Otherwise the model holds
a null where it promised an option.

### A nested option collapses

`Option<Option<T>>` does not survive a round trip. `Some(None)` and `None` both
write `null`, and both read back as `None`:

```csharp
Option<Option<int>> before = Option.Some(Option.None<int>());
string json = JsonConvert.SerializeObject(before, settings);   // "null"
Option<Option<int>> after = JsonConvert.DeserializeObject<Option<Option<int>>>(json, settings)!;
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

`$type` is a busy name. Json.NET writes one of its own when `TypeNameHandling`
is on, and it is also `System.Text.Json`'s polymorphism discriminator.

Nesting puts any such `$type` *inside* `value`, a level below the result's:

```jsonc
{ "$type": "ok", "value": { "$type": "MyApp.Cat, MyApp", "Name": "Tom" } }
```

Flattening the payload beside the discriminator would have made the two
siblings. The collision would then surface only for people who happen to turn
`TypeNameHandling` on. Nesting rules it out.

### The four names are fixed

`$type`, `value`, `ok` and `err` never change. A
`CamelCasePropertyNamesContractResolver` does not rename them, so a camel-casing
service and a snake-casing one still exchange the same payload.

### Reading rejects what a result cannot hold

Deserializing throws `JsonSerializationException` when:

- the payload is not an object
- `$type` is missing, or is not a string
- `$type` names neither case
- `value` is missing
- `value` deserializes to null

A result has no null case. Accepting one would push the failure somewhere later
and harder to trace.

A null `value` is not rejected on sight, though. It is deserialized as the
case's own type first. So a payload whose own converter reads `null` still
round-trips — most usefully `Result<Option<T>, TErr>`, where `Ok(None)` writes
`"value": null` and reads back as `Ok(None)`.

## Reflection and NativeAOT

Json.NET picks a converter from the **runtime** type of the value it is writing.
For a monad that is always `Some<T>`, `None<T>`, `Ok<TOk, TErr>` or
`Err<TOk, TErr>` — never the option or result itself. Both converters therefore
accept all three shapes.

Each converter closes an internal adapter over the type arguments once per
closed type and caches it. Only the first monad of a given type costs any
reflection. Nothing reflects per call.

{% hint style="warning" %}
**Publishing with `PublishAot`? Use
[`Waystone.Monads.SystemTextJson`](system-text-json.md) instead.**

That first construction is the one that fails under NativeAOT for a value-type
argument. There is no escape hatch here, because Json.NET cannot register a
converter for a single closed generic type. Json.NET has no first-class
NativeAOT support of its own either.
{% endhint %}

## It matches the System.Text.Json package byte for byte

[`Waystone.Monads.SystemTextJson`](system-text-json.md) writes the same JSON. A
test in the repository serializes with one package, deserializes with the other,
both directions, and asserts the two write identical bytes.

So you can switch serializers, or run both in one system, without a migration on
the wire.

## What it does not do

* It does not remove a property for a `None`. The property is written with a
  `null` value, and an absent property is a read error rather than a `None`.
* It does not support NativeAOT. Json.NET cannot register a converter for a single
  closed generic type, so there is no escape hatch — use
  [System.Text.Json](system-text-json.md) if you publish with `PublishAot`.
* It does not change either type. Remove the package and only serialization
  breaks.
