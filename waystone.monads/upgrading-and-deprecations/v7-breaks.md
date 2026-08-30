---
description: >-
  Every break in 7.0.0, the compiler diagnostic it produces, and whether a code
  fix handles it.
---

# Every v7 break

{% hint style="warning" %}
**This page describes `7.0.0-beta.x`, a pre-release.** NuGet gives you `6.x` unless you ask for a pre-release:

```
dotnet add package Waystone.Monads --prerelease
```

Or set the version yourself: `<PackageReference Include="Waystone.Monads" Version="7.0.0-beta.*" />`.

The API can still change before `7.0.0` is stable.
{% endhint %}

This is the reference list. For the upgrade itself, with an agent prompt and the order
to work in, go to [6.x to 7.0.0](v6-to-v7.md) or [5.x to 7.0.0](v5-to-v7.md).

## Start with the silent ones

Three changes leave your code compiling and change what it does. Nothing in your build
output will mention them, so they are the only part of this upgrade you have to go
looking for.

| What changed | What happens now | Where to look |
| --- | --- | --- |
| A delegate passed to `Map`, `MapAsync`, `Reduce` or `ReduceAsync` on an `Option` returns null | Throws `ArgumentNullException`, naming the delegate parameter — `map` or `reduce`. In 6.x the null was carried into the `Some`. | Any projection returning a nullable reference, a `FirstOrDefault`, a dictionary lookup, or an explicit `return null` |
| A factory passed to `AndThen` or `AndThenAsync` returns a null `Option` or `Result` | Throws `ArgumentNullException`, naming `optionFactory` or `resultFactory` | Factories that can return a null monad, usually from a field or a cache |
| A `MonadOptionsScope` is disposed when it is not the innermost open scope | Nothing is restored, and the library writes a `ScopeDisposedOutOfOrder` diagnostic event. In 6.x it restored the wrong options silently. | Any scope disposed by hand rather than with `using`, or held in a field |

The first two throw where 6.x carried a null onward. That is the intended fix — the null
was going to surface later as a `NullReferenceException` from code that had every right
to assume a `Some` held a value. To map a null onto a `None`, use `AndThen` with
`Option.FromNullable`.

The third has its own section on [Configuration](../guides/configuration.md#what-happens-when-you-dispose-out-of-order),
and the event is on [Observability](../guides/observability.md#watching-for-a-scope-disposed-out-of-order).

## The loud ones

These break the build. The compiler finds them for you; the table tells you what the
diagnostic actually means, because several of them name something other than the real
problem.

| What changed | Old → new | Diagnostic | Code fix |
| --- | --- | --- | --- |
| Parameter names across `Option`, `Result` and their case types | `MapOr(default: x, …)` → `MapOr(defaultValue: x, …)`, and the same for `else` → `valueFactory`, `createDefault` → `defaultFactory`, `createOther` → `optionFactory` / `resultFactory` | `CS1739` | **Yes** |
| Implicit conversions to `Option` and `Result` removed | `Option<int> o = 5;` → `Option.Some(5)` | `CS0029`, `CS1503` | **Yes** |
| Per-family extension classes collapsed into one class per monad | `AndThenExtensions.AndThenAsync(o, f)` → `o.AndThenAsync(f)` | `CS0103`, or `CS0234` on a `using static` | No |
| `MonadOptions` authoring moved to `MonadOptionsBuilder` | `Use…` methods are on the builder | `CS1061`, `CS1503`, `CS0029`, `CS0103` | No |
| `MonadOptions.UseExceptionLogger` removed | `UseLogger`, `UseLoggerFactory` or `UseLoggerFactoryFrom` from `Waystone.Monads.Extensions.Logging` | `CS1061` | No |
| `ErrorCode.FromEnum` removed | `[ErrorCodeCatalog]` and the generated `ToErrorCode()` | `CS0117` | No |
| `Error.FromEnum` removed | `[ErrorCodeCatalog]` and the generated `{Enum}Catalog.Errors.{Member}(message)` | `CS0117` | No |
| `ErrorCodeFactory.FromEnum` virtual removed | `[ErrorCodeCatalog]`; enum codes are settled at compile time now | `CS0115` | No |
| `Result.Err<TOk>(Enum, string)` overload removed | `Result.Err(code.ToError(message))` | `CS1501` | No |
| An async chaining step's delegate returns `Task` | Return `ValueTask`, or wrap it in an async lambda | `CS0411`, plus [`WM2022`](../analyzers/idioms.md#wm2022) | **Yes**, the wrap |
| `TryAsync` and `CollectAsync` return `ValueTask` | `Task<Option<T>> t = Option.TryAsync(f);` → add `.AsTask()`, or keep the `await` | `CS0029`, or `CS1503` passing it to `Task.WhenAll` | No |

The four `FromEnum`-family removals and `UseExceptionLogger` were all obsolete in 6.x
with a message naming the replacement. Nothing in 7.0.0 is removed without that warning
release, with one exception below.

### Where a code fix exists, run it first

`Waystone.Monads` ships fixes for the three rows marked **Yes**. They handle the bulk of
a real upgrade, and running them before you edit anything by hand keeps the diff small.

The two keyed to `CS1739`, `CS0029` and `CS1503` attach to the **compiler diagnostic**,
not to an analyzer rule — so `dotnet format analyzers` cannot apply them. They appear on
the error in your IDE, where **Fix all occurrences in Project** applies them in a batch.
The `WM2022` fix is an ordinary analyzer fix and works either way. The
[upgrade pages](v6-to-v7.md) say this again in context.

{% hint style="info" %}
**`.AsTask()` comes up twice.** v6 made every async *extension* return `ValueTask`;
v7 does the same to `TryAsync` and `CollectAsync`, the two static members v6 left
alone. If you are coming from 5.x you hit both. The v6 half is on
[v5.x to v6.x](v5-to-v6.md#loud-change-async-extensions-all-return-valuetask), and the
[5.x to 7.0.0 page](v5-to-v7.md) tells you where in the order to do it.
{% endhint %}

## The one removal with no warning release

**The implicit conversions to `Option` and `Result` were removed without being
obsoleted first**, which is not how this repository normally treats public API.

The reason is mechanical rather than a decision to move fast: an obsoleted implicit
conversion still takes part in overload resolution. Marking it `[Obsolete]` would have
produced a warning and left the conversion working — so the silent wrong-branch
behaviour the removal exists to prevent would have carried on for another major version.
There was no ordering that gave both a warning and a fix.

The extension-class collapse has the same shape for a different reason: two static
classes declaring the same extension member for the same receiver is `CS0121`, so the
old and new spellings could not coexist for a version.

## Diagnostics that mask other diagnostics

Three things fire at the declaration phase, and a declaration error stops the compiler
reporting body errors in the same project. So your first green build is not the whole
job.

**`CS0234` on a `using static`.** A missing type name blocks overload resolution in
every file that has the `using`, so parameter renames and conversion errors in those
files go unreported until you fix the qualifier.

**`CS0115` on an `ErrorCodeFactory.FromEnum` override.** A codebase with both an
override and call sites sees only the override error, fixes it, and then discovers the
call sites on the next build.

**`CS0246` on a signature naming `ValidationErr`.** Only if you install
Waystone.Monads.FluentValidation, covered below. A field, property or return type
spelled `Result<T, ValidationErr>` is a declaration error, so the bodies that call
`Validate` stay quiet until you change the signature.

Build, fix, and build again. Twice is not paranoia here.

## Waystone.Monads.FluentValidation

This companion package was rewritten in 7.0.0. If you do not install it, skip this
section. If you do, it is a clean break rather than a deprecation, so nothing warned you
in 6.x.

`ValidationErr` wrapped a failed `ValidationResult` and converted to an `Error` when you
asked. That meant `Result<TValue, ValidationErr>` could not join a chain without a
`MapErr` at the seam, and `ToError()` read your configuration at the moment you called
it — so the same failure produced a different code depending on where you converted it.

`ValidationError` replaces it and **is** an `Error`.

| What changed | Old → new | Diagnostic |
| --- | --- | --- |
| `ValidationErr` removed | `ValidationError`, which derives from `Error` | `CS0246` |
| `Validate` and `ValidateAsync` err with `Error` | `Result<T, ValidationErr>` → `Result<T, Error>` | `CS0029` |
| `ValidateAsync` returns `ValueTask` | Add `.AsTask()`, or keep the `await` | `CS0029`, `CS1503` |
| `ValidationErr.ToError()` removed | Nothing to call — you already have an `Error` | `CS1061` |
| `ValidationErr.Create()` removed | `Validate` or `ValidateAsync` | `CS0117` |
| `AsValidationResult()` and `RuleSetsExecuted` removed | Use `Failures` for the failure list | `CS1061` |
| `UseFallbackValidationErrorMessage` removed | Nothing — the case it covered cannot happen now | `CS1061` |
| Namespaces shadow FluentValidation's own | `Waystone.Monads.FluentValidation.Results.Extensions` → `FluentValidation.Extensions` | `CS0246`, `CS0234` |

There is no code fix for any of these.

### The namespaces shadow FluentValidation's own

The types moved out of `Waystone.Monads.FluentValidation.*` and into
`FluentValidation.*`, so they sit beside `IValidator` and `ValidationFailure` in the
namespace your validator file already imports.

| Old namespace | New namespace |
| --- | --- |
| `Waystone.Monads.FluentValidation.Results` | `FluentValidation` |
| `Waystone.Monads.FluentValidation.Results.Extensions` | `FluentValidation.Extensions` |
| `Waystone.Monads.FluentValidation.Configs` | `FluentValidation.Configs` |

The package and assembly keep the name `Waystone.Monads.FluentValidation`, so the
`PackageReference` does not change. Delete the old `using` directives; a file that
already has `using FluentValidation;` needs nothing back for `ValidationError`.

No shim ships in the old namespace. A forwarding type there would be a distinct type at
run time, so `error is ValidationError` would compile against it and silently stop
matching — a quiet wrong answer in place of a build error.

### Why the removals had no warning release

`ValidationErr` could not be obsoleted alongside its replacement. The new `Validate`
differs from the old one only in its return type, and C# cannot overload on that. The
two spellings could only coexist in separate namespaces, which would give `CS0121` to
anyone importing both.

### Why `UseFallbackValidationErrorMessage` is gone rather than renamed

It set the message used when a validation failure carried none of its own. A
`ValidationError` now has an `internal` constructor that only the failure branch
reaches, so it always carries at least one failure. There is no empty case left for a
fallback to cover, and `Error` already substitutes the core fallback for a blank
message.

### Recovering the failure detail

Where you used to hold a `ValidationErr`, you now pattern match:

```csharp
if (error is ValidationError validationError)
{
    return ValidationProblem(validationError.ToDictionary());
}
```

`Failures` carries the `ValidationFailure` list, and `ToDictionary()` groups the
messages by property, exactly as before. Full detail is on
[Waystone.Monads.FluentValidation](../companion-packages/fluentvalidation.md).

### The supported FluentValidation range widened

The package now declares `FluentValidation >= 11.1.0 && < 13.0.0`, where 6.x pinned a
single version. 11.1.0 is the first release carrying
`ValidationResult.ToDictionary()` as an instance method.

## Rule ids that no longer exist

`WM2010` is retired in 7.0.0. It reported a `Result<T, T>` whose two implicit
conversions were ambiguous, and there are no implicit conversions left for it to report
on.

Retired ids are never reused, so a stale `.editorconfig` entry or `#pragma` naming one
does nothing at all — it does not error, and it does not warn. The full list of retired
ids is on [Deprecations](deprecations.md#rule-ids-that-are-gaps).
