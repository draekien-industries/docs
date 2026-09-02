---
description: >-
  Packages that sit beside the library. None is required, and none changes how
  Waystone.Monads behaves.
icon: boxes-stacked
---

# Overview

These all ship from the same repository as `Waystone.Monads`, on the same version
number, and all are purely additive. You install one because you want what it adds,
not because an upgrade made you.

## Pick a package

| If you need to | Install | Page |
| --- | --- | --- |
| Assert on an `Option` or a `Result` in a test | `Waystone.Monads.Shouldly` | [Shouldly](shouldly.md) |
| Write `from … select` over an `Option` or a `Result` | `Waystone.Monads.Linq` | [LINQ](linq.md) |
| Configure the library from a container | `Waystone.Monads.Extensions.DependencyInjection` | [Dependency injection](dependency-injection.md) |
| Do that, on `Microsoft.Extensions.Hosting` | `Waystone.Monads.Extensions.Hosting` | [Hosting](hosting.md) |
| See the exceptions the library swallows | `Waystone.Monads.Extensions.Logging` | [Logging](logging.md) |
| Parse untrusted input into a domain type | `Waystone.Monads.Schemas` <!-- prerelease:7.1.0 -->(pre-release) | [Schemas](schemas.md) |
| Get a `Result` back from a validator | `Waystone.Monads.FluentValidation` | [FluentValidation](fluent-validation.md) |
| Serialize either type with `System.Text.Json` | `Waystone.Monads.SystemTextJson` | [System.Text.Json](system-text-json.md) |
| Serialize either type with Json.NET | `Waystone.Monads.NewtonsoftJson` | [Newtonsoft.Json](newtonsoft-json.md) |

## Two of these are a pair

The two JSON packages write the same format on purpose. Pick the serializer your
application already uses — a payload one of them writes is a payload the other reads,
and a test in the repository asserts that in both directions.

One exception: if you publish with `PublishAot`, use
[System.Text.Json](system-text-json.md). Json.NET has no NativeAOT story of its own.

## Two more are a pair

[Hosting](hosting.md) depends on [Dependency injection](dependency-injection.md), so
installing Hosting gives you both. On a host, install Hosting. Everywhere else —
a console application, a test, a container you built by hand — install the
dependency injection package alone.

## None of them changes the library

No package here changes the behaviour of anything in `Waystone.Monads`.

* Four add vocabulary. Remove one and the code that used it stops compiling, and
  nothing else moves.
* Three change only *who writes* your configuration, not what the settings mean.
  You can write the same settings by hand with `MonadOptions.Configure`.
* Two teach a serializer a format it did not know. Your own code does not change at
  all.

[Observability](../guides/observability.md) covers the signals that need no package
at all.
