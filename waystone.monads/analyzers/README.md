---
description: >-
  The Roslyn rules that ship inside Waystone.Monads, what each tier means, and
  how to turn a rule up or off.
icon: shield-check
---

# Analyzers

## What this page is for

`Waystone.Monads` ships a Roslyn analyzer inside the package. Install or upgrade to
7.0.0 and you get these rules. You add no reference and configure nothing.

Every rule has an ID like `WM1002`. The first digit tells you how much it matters:

| Tier | Severity | What they report | Page |
| --- | --- | --- | --- |
| `WM1xxx` | Warning | Code that throws or quietly does the wrong thing at run time | [Runtime bugs](runtime-bugs.md) |
| `WM2xxx` | Suggestion | Working code that reads better another way | [Idioms](idioms.md) |
| `WM3xxx` | Off | Migration aids you turn on while you adopt the library | [Migration aids](migration-aids.md) |
| `WMSxxxx` | Suggestion | Test assertions that read better another way | [Assertion rules](assertion-rules.md) |

Warnings show up in your build. Suggestions show up in your IDE only, so they never
break a build that passes today. The `WM3xxx` rules stay off until you enable them.

The `WMS` rules ship in the `Waystone.Monads.Shouldly` package, which you install in
test projects only. They are not in the core package.

A separate set of diagnostics uses a `WMG` prefix. Those come from the source
generator rather than the analyzer, they are all errors, and they only fire on an
enum you marked with `[ErrorCodeCatalog]`. They are on
[Source generation](../source-generation/README.md)
instead.

{% hint style="info" %}
These pages list the rules as at the version they were written for. For the set
that ships in the version you installed, read
[`Rules.cs`](https://github.com/draekien-industries/waystone-dotnet/blob/main/src/Waystone.Monads.Analyzers/Rules.cs)
in the repository — every descriptor is declared there in one file.
{% endhint %}

{% hint style="warning" %}
Do you build with `TreatWarningsAsErrors`? Then a `WM1xxx` rule that fires breaks
your build after you upgrade. We chose that on purpose. Every one of these rules
marks code that throws or returns the wrong value at run time. Read the rule before
you suppress it.
{% endhint %}

## Every rule

| ID | What it reports | Default |
| --- | --- | --- |
| [`WM1001`](runtime-bugs.md#wm1001) | `Option.Some` given a value that is provably null, which always throws | Warning |
| [`WM1002`](runtime-bugs.md#wm1002) | `null` written where an `Option` or `Result` belongs | Warning |
| [`WM1003`](runtime-bugs.md#wm1003) | `default` on an `Option` or `Result`, which is null rather than the empty case | Warning |
| [`WM1005`](runtime-bugs.md#wm1005) | `Option.Some` given a value the compiler treats as maybe-null | Warning |
| [`WM1006`](runtime-bugs.md#wm1006) | A discarded `Result`, so the failure vanishes | Warning |
| [`WM1008`](runtime-bugs.md#wm1008) | An `Option` or `Result` declared nullable, which adds a third state | Warning |
| [`WM1011`](runtime-bugs.md#wm1011) | An async delegate passed to a synchronous method, so the task is never awaited | Warning |
| [`WM2001`](idioms.md#wm2001) | `Unwrap` and `UnwrapErr`, which throw when there is no value | Suggestion |
| [`WM2002`](idioms.md#wm2002) | `Expect`, which throws when its invariant does not hold | Suggestion |
| [`WM2003`](idioms.md#wm2003) | A `throw` inside a member that returns `Result` | Suggestion |
| [`WM2004`](idioms.md#wm2004) | An `IsSome` check with an `Unwrap` inside it | Suggestion |
| [`WM2005`](idioms.md#wm2005) | `Map` followed by `Flatten`, which is `AndThen` | Suggestion |
| [`WM2006`](idioms.md#wm2006) | A state check combined with an unwrap of the same value | Suggestion |
| [`WM2007`](idioms.md#wm2007) | `UnwrapOr` given the default of the type | Suggestion |
| [`WM2008`](idioms.md#wm2008) | An `Option` or `Result` compared to `null` | Suggestion |
| [`WM2009`](idioms.md#wm2009) | `Option<Option<T>>`, which has three states where two mean anything | Suggestion |
| [`WM2010`](idioms.md#wm2010) | **Retired in 7.0.0.** `Result<T, T>` | Not shipped |
| [`WM2011`](idioms.md#wm2011) | A declaration that names `Some`, `None`, `Ok` or `Err` instead of the base type | Suggestion |
| [`WM2012`](idioms.md#wm2012) | A nullable member sitting alongside members that use `Option` | Suggestion |
| [`WM2013`](idioms.md#wm2013) | A discarded `Option` | Suggestion |
| [`WM2015`](idioms.md#wm2015) | `UnwrapOrDefault` or `MapOrDefault` producing a value type | Suggestion |
| [`WM2016`](idioms.md#wm2016) | An eager argument that is not free to evaluate | Suggestion |
| [`WM2017`](idioms.md#wm2017) | A delegate that captures, where a state overload would avoid the closure | Suggestion |
| [`WM2018`](idioms.md#wm2018) | Two `[ErrorCodeCatalog]` enums that generate the same error code | Suggestion |
| [`WM2019`](idioms.md#wm2019) | A generated error code that `ErrorCodes.txt` does not list | Suggestion |
| [`WM2020`](idioms.md#wm2020) | An `ErrorCodes.txt` entry no catalog generates | Suggestion |
| [`WM2021`](idioms.md#wm2021) | A state check read through a property pattern | Suggestion |
| [`WM2022`](idioms.md#wm2022) | A `Task`-returning method group passed to `AndThenAsync` or `OrElseAsync` | Suggestion |
| [`WM3001`](migration-aids.md#wm3001) | A member that returns a nullable type, where `Option<T>` would fit | Off |
| [`WM3002`](migration-aids.md#wm3002) | A `throw`, where returning `Result<TOk, Error>` would fit | Off |
| [`WMS2001`](assertion-rules.md#wms2001) | An assertion on `IsSome`, `IsOk` or `Unwrap` instead of on the monad | Suggestion |
| [`WMS2002`](assertion-rules.md#wms2002) | An `await` wrapped in parentheses so a synchronous assertion can run | Suggestion |

The id space has gaps. `WM1004`, `WM1007`, `WM1009`, `WM1010` and `WM2014` shipped in
5.x and were removed in 6.0.0. `WM2010` was removed in 7.0.0. A removed id is never
reused.

## Changing a rule

You configure every rule through `.editorconfig`, the standard way. Raise one:

```ini
[*.cs]
dotnet_diagnostic.WM2001.severity = warning
```

Silence one:

```ini
[*.cs]
dotnet_diagnostic.WM1005.severity = none
```

Silence one at a single line:

```csharp
#pragma warning disable WM1001
Option<int> option = Option.Some(0);
#pragma warning restore WM1001
```

Put an `.editorconfig` in a subdirectory to scope a rule to part of your solution.
Test projects are the common case. `WM2001` earns its keep less in a test, where an
unwrap that throws fails the test anyway.

You cannot drop the analyzer and keep the library, because both ship in one package.
`.editorconfig` is how you turn rules off.

To raise a whole tier rather than a rule at a time, see
[Severity presets](severity-presets.md). One MSBuild property covers the set.

The `WMS` rules configure the same way, through `dotnet_diagnostic.WMS2001.severity`
and the like. You can drop those entirely by removing the `Waystone.Monads.Shouldly`
package reference, since the analyzer ships with it — which you cannot do for the `WM`
rules, because they ship inside the library.
