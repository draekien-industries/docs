---
description: >-
  Mark an enum with [ErrorCodeCatalog] and get its error codes as compile-time
  constants, generated at build time.
icon: gears
---

# Source generation

`Waystone.Monads` ships a source generator inside the package. It ships from 6.2.0.
You add no reference and configure nothing. It stays silent until you mark an enum
with `[ErrorCodeCatalog]`.

## Which page do you want

| You want to | Read |
| --- | --- |
| Know why this exists and turn it on | This page |
| Know exactly what it emits | [Error code catalogs](error-code-catalogs.md) |
| Change the shape of the generated strings | [Code format language](code-format.md) |
| Review your codes, or fail a build when they drift | [Reviewing generated codes](reviewing-codes.md) |
| Understand a `WMG` build error | [Generator diagnostics](diagnostics.md) |

Hand-written `ErrorCode` values are a different thing, and they are on
[Errors](../guides/errors.md). This group covers the generated ones only.

## Why it exists

Up to 6.x, `ErrorCode.FromEnum(InputErrors.Missing)` worked out the string
`"InputErrors.Missing"` at run time, by reflecting over the enum. That meant the string
your callers saw was not written anywhere you could point at. You could not use it in a
`switch`, you could not put it in an attribute, and you could not find every place that
read it.

Mark the enum with `[ErrorCodeCatalog]` and `Waystone.Monads` generates those strings
as constants when you build.

{% hint style="warning" %}
`ErrorCode.FromEnum` was obsolete from 6.2.0 and 7.0.0 removes it, so this group
describes the only supported way to get an error code from an enum. See
[deprecations.md](../upgrading-and-deprecations/deprecations.md "mention") for the migration.
{% endhint %}

## Marking an enum

```csharp
using Waystone.Monads.Results.Errors;

namespace Ordering;

[ErrorCodeCatalog]
public enum OrderErrorCode
{
    NotFound,
    AlreadyShipped,
}
```

That gives you a new class next to the enum, `OrderErrorCodeCatalog`, in the same
namespace and with the same accessibility as the enum.

The name is the enum's own name with `Catalog` on the end, and nothing is taken off it.
`OrderFailure` gives you `OrderFailureCatalog`; `OrderErrorCode` gives you
`OrderErrorCodeCatalog`. If that reads twice, rename the enum — two enums whose names
differ always get two catalogs whose names differ.

Next: [what the catalog contains](error-code-catalogs.md).
