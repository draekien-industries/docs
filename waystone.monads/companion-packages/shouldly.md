---
description: >-
  Shouldly assertions that take an Option or a Result, so a failing test names
  the state it found.
---

# Waystone.Monads.Shouldly


{% hint style="warning" %}
**This page describes `7.0.0-beta.x`, a pre-release.** NuGet gives you `6.x` unless you ask for a pre-release:

```
dotnet add package Waystone.Monads.Shouldly --prerelease
```

Or set the version yourself: `<PackageReference Include="Waystone.Monads.Shouldly" Version="7.0.0-beta.*" />`.

The API can still change before `7.0.0` is stable.
{% endhint %}

## What it fixes

Assert on a piece of a monad and the failure message is about that piece, not about the
monad. This is the shape most test suites have:

```csharp
result.IsOk.ShouldBeTrue();
```

When it fails, Shouldly tells you that `True` was expected and `False` was found. It
cannot tell you what the error was, because by the time the assertion runs the `Result`
is gone — `IsOk` handed it a `bool`.

```csharp
result.ShouldBeOk();
```

This one fails with:

```
result
    should be Ok
    but was
Err("failed")
```

The `Unwrap` shape is worse than the `IsOk` shape. `result.Unwrap().ShouldBe(42)`
throws from `Unwrap` before the assertion runs at all, so the test fails on a panic
and the message is about unwrapping rather than about what you expected.

## Install it

```
dotnet add package Waystone.Monads.Shouldly --prerelease
```

The package targets `netstandard2.0`, depends on Shouldly 4.3.0, and brings
`Waystone.Monads` with it.

**You need no new `using`.** The assertions are declared in the `Shouldly` namespace,
so a test file that already has `using Shouldly;` picks them up as soon as the package
is referenced.

{% hint style="info" %}
**Why not the `Waystone.Monads.Shouldly` namespace?** Because a namespace of that name
would shadow the global `Shouldly` for every file inside `namespace Waystone.Monads`,
and break every plain `using Shouldly;` in that tree. Living in `Shouldly` is also what
makes the package invisible to a reader: there is nothing new to import.
{% endhint %}

## The assertions

Ten assertions, each on three receiver shapes — the monad, a `Task` of it, and a
`ValueTask` of it.

### Option

| Assertion | Asserts | Returns |
| --- | --- | --- |
| `ShouldBeSome()` | Holds a value | The value |
| `ShouldBeNone()` | Holds no value | Nothing |
| `ShouldBeSomeValue(expected)` | Holds exactly `expected` | The value |

### Result

| Assertion | Asserts | Returns |
| --- | --- | --- |
| `ShouldBeOk()` | Succeeded | The Ok value |
| `ShouldBeErr()` | Failed | The error |
| `ShouldBeOkValue(expected)` | Succeeded with exactly `expected` | The Ok value |
| `ShouldBeErrValue(expected)` | Failed with exactly `expected` | The error |

### Every one returns what it found

So you keep asserting with Shouldly's own vocabulary, on a value you now know is
there:

```csharp
Order order = result.ShouldBeOk();
order.Total.ShouldBe(42);
```

Or in one line:

```csharp
result.ShouldBeOk().Total.ShouldBe(42);
```

`ShouldBeNone` is the exception. There is nothing to hand back, so it returns `void`.

## Asserting on a task

Every assertion has an `Async` sibling declared on `Task<...>` and `ValueTask<...>`
receivers, so you do not have to parenthesise the `await`:

```diff
-(await LoadAsync()).ShouldBeSome();
+await LoadAsync().ShouldBeSomeAsync();
```

{% hint style="danger" %}
**Await it.** An `Async` assertion you forget to await passes without asserting
anything, and the test goes green. This is why they carry an `Async` suffix instead of
being overloads of the same name — the compiler warns you about an unawaited call, and
it can only do that if the call is distinguishable.

`WMS2002` finds these rewrites for you. See [Assertion rules](../using-the-library/analyzer-rules.md#assertion-rules).
{% endhint %}

## Custom messages

Every assertion takes an optional `customMessage`, which is added to the failure rather
than replacing it:

```csharp
result.ShouldBeOk("the seed data should have loaded");
```

```
result
    should be Ok
    but was
Err("failed")


Additional Info:
    the seed data should have loaded
```

The second optional parameter, `actualExpression`, is filled in by the compiler — it is
what puts `result` at the top of that message. Do not pass it. If you need to pass a
custom message positionally you will hit it, so pass the message by name.

## The analyzers ship in the same package

Installing this package also installs `WMS2001` and `WMS2002`, which find the
assertions this page replaces and rewrite them for you. Both are suggestions, both are
on by default, and both are batch-fixable — **Fix all occurrences in Project** converts
a suite in one pass.

They are documented with the rest of the rules, on
[Assertion rules](../using-the-library/analyzer-rules.md#assertion-rules).
