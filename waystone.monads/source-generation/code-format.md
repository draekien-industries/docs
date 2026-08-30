---
description: >-
  Shape the generated strings with {enum} and {member}, an optional casing, and
  one default for the whole assembly.
icon: pen-ruler
---

# Code format language

## Choosing the code format

`OrderErrorCode.NotFound` is the default, not the only option. Set `Format` on the
attribute and the generated codes follow it:

```csharp
[ErrorCodeCatalog(Format = "order.{member:kebab}")]
public enum OrderErrorCode
{
    NotFound,
    AlreadyShipped,
}
```

```csharp
OrderErrorCodeCatalog.Names.NotFound       // "order.not-found"
OrderErrorCodeCatalog.Names.AlreadyShipped // "order.already-shipped"
```

Everything the format does happens at build time, so what you get out is still a
`const string`.

## The format language

Two placeholders, and everything else is literal text:

| Placeholder | Substitutes |
| --- | --- |
| `{enum}` | The enum's name, `OrderErrorCode` |
| `{member}` | The member's name, `NotFound` |

Either one takes an optional casing after a colon — `{member:kebab}`:

| Casing | `NotFound` | `HTTPNotFound` | `Error404` |
| --- | --- | --- | --- |
| *(none)* | `NotFound` | `HTTPNotFound` | `Error404` |
| `kebab` | `not-found` | `http-not-found` | `error-404` |
| `snake` | `not_found` | `http_not_found` | `error_404` |
| `lower` | `notfound` | `httpnotfound` | `error404` |
| `upper` | `NOTFOUND` | `HTTPNOTFOUND` | `ERROR404` |

`kebab` and `snake` split the identifier into words; `lower` and `upper` only change
case. A word boundary falls where a lowercase or a digit meets an uppercase, before the
last uppercase of a run that runs into a lowercase, and between letters and digits. An
underscore already in the name counts as a boundary rather than doubling up, so
`Already_Shipped` gives `already-shipped`.

Write a literal brace by doubling it: `{{` and `}}`.

The default is `{enum}.{member}`, which is what an enum that sets nothing gets.

## One format for the whole assembly

Put the attribute on the assembly and every attributed enum in the project uses it:

```csharp
[assembly: ErrorCodeFormat("{enum:kebab}/{member:kebab}")]
```

An enum's own `Format` wins over the assembly's, so the assembly attribute sets the
house style and an individual enum departs from it where it needs to.

## The format is your contract, not your factory

The generated strings come from the format and nothing else. In particular:

{% hint style="warning" %}
**A generated member never consults your `ErrorCodeFactory`.** The generator runs at
build time and cannot run a factory you install at run time. A factory installed through
`MonadOptions.UseErrorCodeFactory` still shapes the codes it derives from an exception;
it has no say over an attributed enum.

That holds for every generated member, including the fallback for an undeclared value.
If you were using a custom factory in 6.x to shape the codes an enum produces, say the
same thing with `Format` instead — you get the same strings as constants, and one
answer rather than two.
{% endhint %}

Shaping enum codes is the part of `ErrorCodeFactory` the format replaces, and 7.0.0
removes that part. `ErrorCodeFactory.FromEnum` and `ErrorCode.FromEnum` reported
`CS0618` from 6.2.0 and are gone now — the override produces `CS0115`. See
[deprecations.md](../upgrading-and-deprecations/deprecations.md "mention"). `FromException` is not affected.

