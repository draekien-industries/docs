---
description: >-
  Two packages that sit beside the library. Neither is required, and neither
  changes how Waystone.Monads behaves.
---

# Overview


{% hint style="warning" %}
**This page describes `7.0.0-beta.x`, a pre-release.** NuGet gives you `6.x` unless you ask for a pre-release:

```
dotnet add package Waystone.Monads --prerelease
```

Or set the version yourself: `<PackageReference Include="Waystone.Monads" Version="7.0.0-beta.*" />`.

The API can still change before `7.0.0` is stable.
{% endhint %}

Both of these ship from the same repository as `Waystone.Monads`, on the same version
number, and both are purely additive. You install one because you want what it adds,
not because an upgrade made you.

| Package | Install it in | What it adds |
| --- | --- | --- |
| [Waystone.Monads.Shouldly](shouldly.md) | Test projects | Shouldly assertions that take an `Option` or a `Result` directly, so a failure names the state it found |
| [Waystone.Monads.Linq](linq.md) | Anywhere | `Select`, `SelectMany` and `Where`, so C# query syntax works over `Option` and `Result` |

Neither package changes the behaviour of anything in `Waystone.Monads`. Remove either
one and the code that used it stops compiling; nothing else moves.

There is a third package, `Waystone.Monads.Extensions.Logging`, which is not on this
list because it configures the library rather than extending it. It is on
[Observability](../using-the-library/observability.md).
