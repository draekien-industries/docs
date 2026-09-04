---
icon: gear
---

# Configuration

This library has a few configurable behaviours. You set them through
`MonadOptions.Configure`, once, at start-up. Configure each option below _once_ in your
application's lifetime.

**If your application has a dependency injection container, configure the library
there instead.** Install
[Waystone.Monads.Extensions.Hosting](../packages/hosting.md) on a host, or
[Waystone.Monads.Extensions.DependencyInjection](../packages/dependency-injection.md)
without one, and the container writes these settings for you. You get a delegate that
can resolve services out of the container, optional binding from `IConfiguration`, and
a diagnostic event when configuration was registered but never installed.

Reach for `MonadOptions.Configure` when there is no container — a library, a test, or
a small console application. Reach for a [scope](#scoped-configuration) when one region
of code needs different settings. Every `Use…` method below is the same one on all
three routes.

<!-- snippet: configuration-the-usual-call -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Guides/Configuration.cs -->
```csharp
MonadOptions.Configure(options => options
    .UseFallbackErrorCode("Unknown")
    .UseFallbackErrorMessage("Something went wrong."));
```
<!-- endSnippet -->

## How configuration works

You do not hold options. You describe them, and the library publishes the result.

`Configure` hands your callback a **`MonadOptionsBuilder`**. Every `Use…` method is on
the builder, and each returns the builder so you can chain. When your callback returns,
the library builds an immutable `MonadOptions` from it and swaps that in as the options
the whole process reads.

Three consequences, all of which matter in practice.

**There is nothing to read.** `MonadOptions` exposes no public property, accessor or
instance member — only the two statics `Configure` and `BeginScope`. Nothing in your
code can inspect the current configuration, and nothing needs to.

**A reader never sees a half-configured state.** The swap is atomic. Code running on
another thread reads either the options from before your `Configure` call or the ones
from after it, never a mixture. Before 7.0.0, configuration mutated a shared object in
place, so a concurrent reader could see one setting applied and the next not.

{% hint style="danger" %}
**Do not keep the builder.** It is authoring state, not the published options. Calls you
make on it after your callback has returned are discarded — no exception, no warning,
nothing takes effect.

<!-- snippet: configuration-do-not-keep-the-builder -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Guides/Configuration.cs -->
```csharp
// Wrong. The second call does nothing.
MonadOptionsBuilder? stashed = null;
MonadOptions.Configure(options => stashed = options.UseFallbackErrorCode("A"));
stashed!.UseFallbackErrorMessage("B");
```
<!-- endSnippet -->

Put every `Use…` call inside the callback.
{% endhint %}

### If you are upgrading from 6.x

The common call shape is unchanged, because the lambda parameter's type is inferred:

<!-- snippet: configuration-call-shape-unchanged -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Guides/Configuration.cs -->
```csharp
MonadOptions.Configure(options => options.UseFallbackErrorCode("Unknown"));
```
<!-- endSnippet -->

That compiled against 6.x and it compiles against 7.0.0. You do not have to touch call
sites that already build.

Three things do change:

* An **explicitly typed** lambda parameter breaks. `(MonadOptions options) =>` becomes
  `(MonadOptionsBuilder options) =>`, or drop the annotation and let it infer.
* A **field, parameter, local or property typed `MonadOptions`** has no replacement.
  Configuration is reachable only inside a `Configure` or `BeginScope` callback now, so
  move the `Use…` calls into one rather than passing options around.
* In the satellite packages, **`MonadOptionsExtensions` is now
  `MonadOptionsBuilderExtensions`**. A `using static` or a qualified static call naming
  the old class needs updating. A normal extension call on the callback's parameter, the
  usual shape, needs nothing.

The [6.x to 7.0.0 upgrade page](../upgrading/v7/from-v6.md) covers this
with the compiler diagnostics you will see.

## Logging

This library catches exceptions in several places and turns them into
non-throwing types. To see those exceptions, install
`Waystone.Monads.Extensions.Logging` and hand the library your logger once:

```csharp
MonadOptions.Configure(options => options.UseLoggerFactoryFrom(app.Services));
```

Use `UseLoggerFactory(factory)` if you have no service provider, or
`UseLogger(logger)` if you already hold a logger.

Each entry carries the exception plus the call site that caught it — the member
name, the source text of the delegate you passed, and the line number.

The three methods above ship in that package, and
[Logging](../packages/logging.md) covers them properly — the levels, the
category, and what lands in each entry.

To count these exceptions instead, install nothing at all. Read
[Observability](observability.md) for the signals that need no package.

{% hint style="warning" %}
**`UseExceptionLogger` is gone.** It was obsolete from 6.7.0 and 7.0.0 removes it. It
took a delegate you wrote yourself and held only one, so a second integration silently
replaced the first. Install the package and call one of the three methods above
instead. See
[Deprecations](../upgrading/deprecations.md#seeing-handled-exceptions-through-a-hand-written-delegate).
{% endhint %}

{% hint style="info" %}
**The library also writes to the console whenever a debugger is attached**, whether or
not you configure a logger. It prints the exception, the call site and the argument
expression, then reports it through the signals above as well. This is a debugging aid,
so it costs nothing in a normal run — but do not read a console message as proof that
your logger ran.
{% endhint %}

## Cancellation

`Option.Try`, `Option.TryAsync`, `Result.Try` and `Result.TryAsync` catch the
exceptions your factory throws and turn them into a `None` or an `Err`. From
6.0.0 they make one exception to that: an `OperationCanceledException` propagates
to your caller instead.

Cancelling is you telling the work to stop. It is not the work failing, and
turning it into a `None` leaves the caller unable to tell "cancelled" from
"genuinely absent".

`TaskCanceledException` inherits from `OperationCanceledException`, so it
propagates too.

If you need the pre-6.0.0 behaviour, opt back in:

<!-- snippet: configuration-cancellation-as-failure -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Guides/Configuration.cs -->
```csharp
MonadOptions.Configure(options => options.UseCancellationAsFailure());
```
<!-- endSnippet -->

A cancellation is then caught, counted and logged like any other handled
exception, and becomes a `None` or an `Err` as it did before.

`UseCancellationAsFailure` takes an optional `bool`, which defaults to `true`, so the
call above turns the behaviour on. Pass `false` to put it back:

<!-- snippet: configuration-cancellation-as-failure-off -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Guides/Configuration.cs -->
```csharp
MonadOptions.Configure(options => options.UseCancellationAsFailure(false));
```
<!-- endSnippet -->

You need that only when something earlier already turned it on and you want to undo it
— a container registration from a library, or a `Configure` call elsewhere in start-up.
A builder inherits the settings in effect when it was handed to you, so passing `false`
is the only way to reverse the decision.

{% hint style="info" %}
We recommend leaving this off. It exists so that upgrading to 6.0.0 does not force
you to rewrite every call site at once. Scope it with
[`MonadOptions.BeginScope`](#scoped-configuration) if only part of your code needs
it.
{% endhint %}

## Error Code Generation

There are a few factory methods included in the library for generating `ErrorCode` instances from `Enum` and from `Exception` instances. To customise how these error codes are generated, create a class inheriting from `ErrorCodeFactory` and override the methods you wish to customise. Then create an instance and pass it into the `MonadOptions` instance via the `UseErrorCodeFactory`.

<!-- snippet: configuration-error-code-factory -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Guides/Configuration.cs -->
```csharp
internal sealed class ShoutingErrorCodeFactory : ErrorCodeFactory
{
    public override ErrorCode FromException(Exception exception) =>
        new(exception.GetType().Name.ToUpperInvariant());
}
```
<!-- endSnippet -->

Register your instance once, at start-up:

<!-- snippet: configuration-use-error-code-factory -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Guides/Configuration.cs -->
```csharp
MonadOptions.Configure(
    options => options.UseErrorCodeFactory(new ShoutingErrorCodeFactory()));
```
<!-- endSnippet -->

{% hint style="warning" %}
**`FromEnum` is no longer one of the methods to override.** It is obsolete from 6.2.0
and removed in 7.0.0, because a factory runs too late for the compiler, the analyzers
or the error code registry to see what it returns. Shape enum codes with
`[ErrorCodeCatalog(Format = "…")]` instead — see
[Code format language](../source-generation/code-format.md) — and keep the factory
for `FromException`, which is unaffected.
{% endhint %}

## Error Code and Message Fallbacks

There may be exception circumstances which cause the `string` used to create the `ErrorCode` or the message of the `Error` classes to be null or white-space. In these situations, a set of fallbacks are used. These fallbacks can be configured.

<!-- snippet: configuration-fallbacks -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Guides/Configuration.cs -->
```csharp
MonadOptions.Configure(options => options
    .UseFallbackErrorCode("unknown")                     // default: Unspecified
    .UseFallbackErrorMessage("Something went wrong!"));  // default: An unexpected error occurred.
```
<!-- endSnippet -->

**The substitution is silent.** `new ErrorCode(code)` and `new Error(code, message)`
trim what you pass, and swap in the fallback when the result is empty. Neither throws,
and nothing is logged. So an `Error` whose message reads `An unexpected error
occurred.` is telling you a call site passed a blank message, not that the library hit
an unexpected error. Pass a real message at every call site — a fallback says nothing
about what actually failed.

Both configuration methods reject a blank argument themselves. `UseFallbackErrorCode`
and `UseFallbackErrorMessage` throw an `ArgumentException` when you pass null, empty or
whitespace, because a fallback that is itself unusable would leave nothing to fall back
to.

## Scoped Configuration

Use `MonadOptions.BeginScope` when you want different options for one region of
code — a single request, a test, or a block you are debugging. The scope applies
until you dispose it, and your global configuration is untouched.

<!-- snippet: configuration-a-scope -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Guides/Configuration.cs -->
```csharp
using (MonadOptions.BeginScope(options => options.UseFallbackErrorCode("Debug")))
{
    Result<int, Error> result = Result.Try<int>(() => int.Parse(input));

    _ = result;
}

// out here, your global configuration applies again
```
<!-- endSnippet -->

A scope accepts the same configuration methods as `Configure`, so it can override
any option:

<!-- snippet: configuration-scope-overrides -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Guides/Configuration.cs -->
```csharp
using (MonadOptions.BeginScope(options => options
           .UseErrorCodeFactory(new ShoutingErrorCodeFactory())
           .UseFallbackErrorMessage("Something went wrong while debugging.")))
{
    // ...
}
```
<!-- endSnippet -->

### What a scope does

* **Inherits what you do not set.** Options you leave alone keep the values they
  had when the scope opened.
* **Takes a snapshot.** Calling `Configure` while a scope is open does not change that
  scope. The new global value applies once the scope ends. This is not new in 7.0.0 — a
  scope has always held its own copy.
* **Nests.** Disposing the innermost scope restores the scope around it.
* **Restores only from the inside out.** A scope that is no longer the innermost one
  declines to restore anything when you dispose it, and reports itself instead. See
  below.
* **Isolates concurrent work.** A scope applies to the current asynchronous flow,
  so parallel work each sees its own options. This makes scopes safe to use in
  tests that run in parallel.

{% hint style="info" %}
A scope affects work you start inside it. It does not affect work that was already
running when you opened the scope.
{% endhint %}

{% hint style="warning" %}
**Dispose scopes in the reverse of the order you opened them.** A `using` block does
this for you. Nothing else guarantees it.
{% endhint %}

#### What happens when you dispose out of order

Since 7.0.0, a scope restores only when it is the innermost one still open. Disposing
it at any other time changes nothing and reports the mistake.

`Dispose` looks at the options in effect on the current flow and picks one of three
paths:

| What it finds | What it does |
| --- | --- |
| The options this scope installed, so this scope is the innermost one | Restores what came before it |
| The options this scope restored to, so it has already been disposed | Nothing, silently |
| Anything else | Nothing, and writes a `ScopeDisposedOutOfOrder` diagnostic event |

It never throws, on any path.

**The third path leaves the early-disposed scope's options in effect.** They stay live
until the inner scope that is still open is disposed, which then restores them as *its
own* predecessor — so those options outlive the scope that installed them. That is the
bug the event exists to tell you about.

Two more cases take the third path, both worth knowing:

* Disposing a scope from a different asynchronous flow than the one that opened it. A
  scope lives in the flow, so another flow's `Dispose` never sees it.
* Disposing a `default(MonadOptionsScope)`. Before 7.0.0 this dropped the flow back to
  your global configuration; now it reports like any other out-of-order disposal.

**Repeated disposal on the third path reports every time.** A scope that has already
declined cannot remember that it did, because it is a readonly struct, so each further
`Dispose` writes the event again. Deduplicate in your subscriber if that matters. The
"harmless twice" promise covers the restoring path only.

To see these events, see
[Watching for a scope disposed out of order](observability.md#watching-for-a-scope-disposed-out-of-order).

{% hint style="info" %}
[`Waystone.Monads.FluentValidation`](../packages/fluent-validation.md)
options are covered by the same scope, so you only ever open one.
{% endhint %}
