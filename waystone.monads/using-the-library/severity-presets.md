---
description: >-
  Turn the analyzer up in one line, without listing thirty rules in your
  .editorconfig.
---

# Severity presets


{% hint style="warning" %}
**This page describes `7.0.0-beta.x`, a pre-release.** NuGet gives you `6.x` unless you ask for a pre-release:

```
dotnet add package Waystone.Monads --prerelease
```

Or set the version yourself: `<PackageReference Include="Waystone.Monads" Version="7.0.0-beta.*" />`.

The API can still change before `7.0.0` is stable.
{% endhint %}

## Pick a preset in one line

Set `WaystoneMonadsRuleset` in the project that references the package:

```xml
<PropertyGroup>
    <WaystoneMonadsRuleset>recommended</WaystoneMonadsRuleset>
</PropertyGroup>
```

Three values are valid: `recommended`, `strict`, and `none`. The default is `none`,
so nothing changes until you ask.

Get the value wrong and the build fails before the compiler runs, naming the three
valid values. That is deliberate — the name is matched, not turned into a file path,
so a typo cannot reach the compiler as a missing analyzer config.

## What each preset sets

The library ships 29 rules. Here is what each preset does to them.

| Tier | Rules | Shipped default | `recommended` | `strict` |
| --- | --- | --- | --- | --- |
| Misuse | `WM1001`, `WM1002`, `WM1003`, `WM1005`, `WM1006`, `WM1008`, `WM1011` | Warning | **Error** | **Error** |
| Idiom | 20 rules, `WM2001` through `WM2022` | Suggestion | Suggestion | **Warning** |
| Adoption | `WM3001`, `WM3002` | Off | Off | **Warning** |

`recommended` changes seven rules and leaves twenty-two alone. Every one of the seven
reports code that throws at run time, ignores a failure, or means the opposite of what
it reads as. Failing the build on those is a statement the rule messages already make.

`strict` is the posture of a codebase adopting the library wholesale. It turns
everything on.

{% hint style="warning" %}
**`strict` will report a lot on an existing codebase**, and the two adoption rules
will account for most of it by a wide margin. They fire on every nullable return and
every `throw`, converted or not — see
[Migration aids](analyzer-rules.md#migration-aids).

If you want the idiom rules enforced without that noise, take `recommended` and raise
the idiom tier yourself. That is not what `strict` is for.
{% endhint %}

## The test package has the same switch

`Waystone.Monads.Shouldly` reads the same `WaystoneMonadsRuleset` property, so setting
it once covers both.

| Rules | Shipped default | `recommended` | `strict` |
| --- | --- | --- | --- |
| `WMS2001`, `WMS2002` | Suggestion | Suggestion | **Warning** |

`recommended` leaves the assertion rules where they are. They fire on tests that pass,
and a test suite is not where you want a build-breaking opinion about assertion style.

## Overriding one rule

A preset is a floor, not a ceiling. Anything you write wins.

```ini
[*.cs]
dotnet_diagnostic.WM1006.severity = warning
```

That works because the presets ship as global analyzer configs at `global_level = -1`,
which sits below every level you can author — so your own `.globalconfig` wins a
conflict outright rather than producing a tie, and a path-matched `.editorconfig`
section beats any global config regardless of level.

Scoping a preset to part of a solution works the same way as scoping a rule: set the
property in the projects you want it in, and leave it out of the ones you do not. Test
projects are the usual exception.

## Why this is not an .editorconfig fragment you copy

It cannot be one, and the reason is not style.

`WM2020` reports against `ErrorCodes.txt`, which has no syntax tree. Roslyn resolves
`dotnet_diagnostic` severities per tree, so a path-matched section cannot reach that
rule — not even `[*]`. An `.editorconfig` fragment would therefore ship a preset with
one rule silently missing from it. A global analyzer config has no such limit.

## A preset does not flow to your dependents

The preset files ship in the package's `build/` folder rather than
`buildTransitive/`, so a project that depends on yours does not inherit your choice.

That matches where the rules themselves go. A `PackageReference` excludes analyzers
from transitive consumers by default, so a project that only depends on this package
indirectly never gets the analyzer — and a preset that travelled further than the
analyzer would set severities for rules nothing in that project reports.
