---
description: >-
  The WMSxxxx rules. They ship in Waystone.Monads.Shouldly and fire on test
  assertions only.
icon: vial
---

# Assertion rules

These rules ship in `Waystone.Monads.Shouldly`, not in the core package. Add that
package to a test project and you get them:

```
dotnet add package Waystone.Monads.Shouldly
```

Their ids start with `WMS`, a second namespace beside `WM`. `WM` ids are validated
against the core analyzer assembly, so a rule shipped from another package cannot take
one. The `2` means what it means everywhere else on this page: working code that reads
better another way, reported as a suggestion, on by default. There is no `WMS1` tier and
there should not be one — every rule here fires on a test that already passes.

| ID | What it reports | Quick fix |
| --- | --- | --- |
| [`WMS2001`](#wms2001) | An assertion on `IsSome`, `IsOk` or `Unwrap` instead of on the monad | The matching monad assertion |
| [`WMS2002`](#wms2002) | An `await` wrapped in parentheses so a synchronous assertion can run | The `Async` assertion |

Both fixes are batch-fixable, so **Fix all occurrences in Project** clears a test suite
in one pass. Run it more than once. `WMS2001` rewrites `(await task).IsSome.ShouldBeTrue()`
into `(await task).ShouldBeSome()`, which is then `WMS2002`'s input, and a batch fixer
lands only non-overlapping fixes per pass.

## WMS2001

**Assert on the monad, not on a piece of it.** `IsSome` and `IsOk` yield a `bool`, and
`Unwrap` yields the contained value. Either way the assertion that follows never sees
the monad, so a failing test cannot tell you what it found.

```diff
-option.IsSome.ShouldBeTrue();
+option.ShouldBeSome();
```

The `IsSome` version fails with "expected True, was False". The `ShouldBeSome` version
names the `None`. The `Unwrap` version is worse: `Unwrap` throws before the assertion
runs, so the test fails on a panic rather than on an assertion, and the message is about
`Unwrap` rather than about your expectation.

```diff
-option.Unwrap().ShouldBe(42);
+option.ShouldBeSomeValue(42);
```

The rule reads both halves of the pair, so `ShouldBeFalse` picks the opposite
assertion:

| You wrote | The fix writes |
| --- | --- |
| `option.IsSome.ShouldBeTrue()` or `option.IsNone.ShouldBeFalse()` | `option.ShouldBeSome()` |
| `option.IsNone.ShouldBeTrue()` or `option.IsSome.ShouldBeFalse()` | `option.ShouldBeNone()` |
| `result.IsOk.ShouldBeTrue()` or `result.IsErr.ShouldBeFalse()` | `result.ShouldBeOk()` |
| `result.IsErr.ShouldBeTrue()` or `result.IsOk.ShouldBeFalse()` | `result.ShouldBeErr()` |
| `option.Unwrap().ShouldBe(x)` | `option.ShouldBeSomeValue(x)` |
| `result.Unwrap().ShouldBe(x)` | `result.ShouldBeOkValue(x)` |
| `result.UnwrapErr().ShouldBe(x)` | `result.ShouldBeErrValue(x)` |

`UnwrapErr` on an `Option` is not in that table because it does not compile — an
`Option` has no error half.

## What it does not report

**`ShouldBeOfType<Some<T>>()` never reports.** Those sites are usually testing the
closed hierarchy itself, and nothing in the syntax separates that from an incidental
type check, so rewriting them would delete the only coverage of it. This is excluded by
design, not by omission.

**A comparison overload carrying its own options never reports.** That covers
`ShouldBe(expected, tolerance)`, the comparer overload, and the ignore-order overload.
Those arguments describe how to compare a bare value, and they have no counterpart on an
assertion that takes the monad. `ShouldBeTrue` and `ShouldBeFalse` need no equivalent
exclusion — a `bool` has nothing to configure.

{% hint style="info" %}
**Two diagnostics on the `Unwrap` line is one problem.** `WMS2001` overlaps `WM2001`
there on purpose. The spans differ: `WM2001` reports the panicking call, this rule the
whole assertion. Applying the `WMS2001` fix resolves both, because the rewrite is
what removes the `Unwrap`. Suppressing either to silence the pair would leave a consumer
who does not use this package with no signal at all.
{% endhint %}

The reported span is the whole assertion, and it is **not** faded in your IDE the way
most `WM2` rules with a fix are. Fading it would read as "this line can go", and the fix
replaces your assertion rather than removing it.

## WMS2002

**Drop the parentheses around the `await`.** Member access binds tighter than `await`,
so asserting on a task's result forces you to parenthesise it. Every assertion in this
package is also declared on `Task` and `ValueTask` receivers, so you do not have to.

```diff
-(await LoadAsync()).ShouldBeSome();
+await LoadAsync().ShouldBeSomeAsync();
```

The fix appends `Async` to the assertion and moves the `await` outward.

## What it does not report

**`(await task.ConfigureAwait(false)).ShouldBeSome()` never reports.** Read this one
before you conclude the rule is broken — `ConfigureAwait` is what this library's own
documented style produces everywhere, so this is the shape you are most likely to have
written. `ConfigureAwait` returns an awaitable that is not a task, and this package
declares no assertion on it, so moving the `await` outward would leave the rewrite with
no receiver.

**A chained assertion never reports.** In `(await task).ShouldBeSome().Name`, something
else reads the assertion's result. Moving the `await` outward would bind `.Name` to the
assertion's task rather than to its value, so the rewrite would change what the test
does.

The rule is scoped to an `await` of `Task<T>` or `ValueTask<T>` written directly, and to
an assertion whose result nothing else reads. Both exclusions exist for the same reason:
outside them, moving the `await` changes what is awaited.

