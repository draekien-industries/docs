---
description: Keep exceptions exceptional
icon: hand-wave
cover: https://gitbookio.github.io/onboarding-template-images/header.png
coverY: 0
layout:
  cover:
    visible: true
    size: full
  title:
    visible: true
  description:
    visible: true
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
---

# Welcome

{% hint style="warning" %}
**These docs describe `7.0.0-beta.x`, a pre-release.** NuGet gives you `6.x` unless you ask for a pre-release:

```
dotnet add package Waystone.Monads --prerelease
```

Or set the version yourself: `<PackageReference Include="Waystone.Monads" Version="7.0.0-beta.*" />`.

The API can still change before `7.0.0` is stable.
{% endhint %}

Waystone.Monads is a lightweight, idiomatic C# library that gives you two fundamental functional types: `Option<T>` and `Result<T, E>`. It is inspired by Rust's `std::option` and `std::result` types and brings them into C#.

## Why this library exists

Most C# codebases default to `null` and exceptions for absence and failure. That's fine, until it isn't.

{% hint style="warning" %}
`null` and exceptions result in guard clauses everywhere, unpredictable runtime crashes, and unclear API intent.
{% endhint %}

Waystone.Monads replaces that with explicit types that make the intent clear at the type level:

* `Option<T>` means a value might be there.
* `Result<T, E>` means a computation might fail.

## Who should use this

You should use this library if:

* You want to remove `null` and exceptions from business logic
* You want your exceptions to be exceptional
* You prefer expressive, explicit code over defensive boilerplate
* You appreciate functional patterns but still want to write C#

{% hint style="success" %}
If you've ever used `Option` and `Result` in Rust or F#, you'll feel right at home. If you haven't, you'll pick it up quickly - and wonder how you ever lived without it.
{% endhint %}

## What else ships

The `Waystone.Monads` package carries a Roslyn analyzer and a source generator. You
install nothing extra and configure nothing to get them. See
[Analyzer rules](using-the-library/analyzer-rules.md) and
[Generated error codes](using-the-library/generated-error-codes.md).

Five optional packages sit beside the library. None is required, and none changes how
`Waystone.Monads` behaves — see [Companion Packages](companion-packages/README.md) for
what each one adds.

## Links

* [Source on GitHub](https://github.com/draekien-industries/waystone-dotnet) — the `Waystone.Monads` package lives in `src/Waystone.Monads`
* [Waystone.Monads on NuGet](https://www.nuget.org/packages/Waystone.Monads)
* [Waystone.Monads.Extensions.Logging on NuGet](https://www.nuget.org/packages/Waystone.Monads.Extensions.Logging) — sends the exceptions the library catches to your `ILogger`
* [Report an issue](https://github.com/draekien-industries/waystone-dotnet/issues) — for the library or for these docs
