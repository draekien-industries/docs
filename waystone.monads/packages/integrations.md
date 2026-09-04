---
description: >-
  Packages that connect Waystone.Monads to a library you already use. None is
  required, and none changes how Waystone.Monads behaves.
icon: plug
---

# Integrations

Each of these bridges `Waystone.Monads` to a library from somewhere else. They put
their types in that library's namespaces, so you reach them from a `using` you
already have. All ship from the same repository as `Waystone.Monads`, on the same
version number, and all are purely additive.

Packages that add to `Waystone.Monads` itself are under [Add-ons](README.md).

## Pick a package

| If you need to | Install | Page |
| --- | --- | --- |
| Assert on an `Option` or a `Result` in a test | `Waystone.Monads.Shouldly` | [Shouldly](shouldly.md) |
| Configure the library from a container | `Waystone.Monads.Extensions.DependencyInjection` | [Dependency injection](dependency-injection.md) |
| Do that, on `Microsoft.Extensions.Hosting` | `Waystone.Monads.Extensions.Hosting` | [Hosting](hosting.md) |
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

* Two add vocabulary. Remove [Shouldly](shouldly.md) or
  [FluentValidation](fluent-validation.md) and the code that used it stops compiling,
  and nothing else moves.
* Two change only *who writes* your configuration, not what the settings mean. You
  can write the same settings by hand with `MonadOptions.Configure`.
* Two teach a serializer a format it did not know. Your own code does not change at
  all.
