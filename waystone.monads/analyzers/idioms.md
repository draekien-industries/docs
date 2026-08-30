---
description: >-
  The WM2xxx rules. Your code works; each rule points at a clearer way to write
  it.
icon: wand-magic-sparkles
---

# Idioms

These rules are suggestions. Your code works. The rule points at a clearer way to
write it. You see them in your IDE, and they stay out of your build.

| ID | What it reports | Quick fix |
| --- | --- | --- |
| [`WM2001`](#wm2001) | `Unwrap` and `UnwrapErr`, which throw when there is no value | `UnwrapOrDefault()` |
| [`WM2002`](#wm2002) | `Expect`, which throws when its invariant does not hold | `UnwrapOrDefault()` |
| [`WM2003`](#wm2003) | A `throw` inside a member that returns `Result`, so the failure escapes the channel your signature promises | None |
| [`WM2004`](#wm2004) | An `IsSome` check with an `Unwrap` inside it, which asks the same question twice | None |
| [`WM2005`](#wm2005) | `Map` followed by `Flatten`, which is `AndThen` | `AndThen` |
| [`WM2006`](#wm2006) | A state check combined with an unwrap of the same value, which is `IsSomeAnd`, `IsNoneOr`, `IsOkAnd` or `IsErrAnd` | None |
| [`WM2007`](#wm2007) | `UnwrapOr` given the default of the type, which is `UnwrapOrDefault` | `UnwrapOrDefault()` |
| [`WM2008`](#wm2008) | An `Option` or `Result` compared to `null`, which reads like an absence check but is not one | The matching state check |
| [`WM2009`](#wm2009) | `Option<Option<T>>`, which has three states where only two mean anything | None |
| [`WM2010`](#wm2010) | **Retired in 7.0.0.** `Result<T, T>`, whose two implicit conversions were ambiguous | None |
| [`WM2011`](#wm2011) | A declaration that names `Some`, `None`, `Ok` or `Err` instead of `Option` or `Result`, so it can hold only one of the two states | The base type |
| [`WM2012`](#wm2012) | A nullable member sitting alongside members that use `Option`, so one type has two ways of saying "absent" | None |
| [`WM2013`](#wm2013) | A discarded `Option`, as `WM1006` does for `Result` | None |
| [`WM2015`](#wm2015) | `UnwrapOrDefault` or `MapOrDefault` producing a value type, where the default is indistinguishable from a real result | `UnwrapOrNull()` or `MapOrNull()` |
| [`WM2016`](#wm2016) | An argument to `Or`, `And`, `UnwrapOr`, `MapOr` or `OkOr` that is not free to evaluate, so it runs even when it is discarded | The `Else` sibling |
| [`WM2017`](#wm2017) | A delegate that captures a local or a parameter, where a state overload would avoid the closure | None |
| [`WM2018`](#wm2018) | Two `[ErrorCodeCatalog]` enums that generate the same error code | None |
| [`WM2019`](#wm2019) | A generated error code that `ErrorCodes.txt` does not list | Update `ErrorCodes.txt` |
| [`WM2020`](#wm2020) | An `ErrorCodes.txt` entry no catalog generates | None |
| [`WM2021`](#wm2021) | `IsSome`, `IsNone`, `IsOk` or `IsErr` read through a property pattern, which hides the check from the rules that read it | None |
| [`WM2022`](#wm2022) | A `Task`-returning method group passed to `AndThenAsync` or `OrElseAsync`, whose step returns a `ValueTask` | Wrap it in an async lambda |

There is no `WM2014`. It shipped in 5.4.0 as a `FlatMap` rename aid and was
removed in 6.0.0. `WM2010` is listed above because build output from 6.x still
links to it; nothing in 7.0.0 reports it.

`WM2008` owns every null comparison and null pattern, so `WM1002` leaves those
alone. You get one diagnostic per site, not two.

`WM2007` and `WM2015` point opposite ways on purpose, and both are suggestions so you
can decide. `WM2007` says `UnwrapOr(0)` is `UnwrapOrDefault()`, which is shorter and
says what it means. `WM2015` then says that on a value type `UnwrapOrDefault()` hands
back `0`, which you cannot tell apart from a real zero, and offers `UnwrapOrNull()`.
Take the second one where the difference matters to you. Applying the `WM2007` quick
fix on a value type reports `WM2015` on the result, for the same reason. The same pair
applies to the quick fixes `WM2001` and `WM2002` offer.

`WM2003` ignores the same throws as [`WM3002`](migration-aids.md#wm3002).

We split `WM2001` and `WM2002` on purpose. `Expect` states an invariant and names it
in the message, which is fair where the invariant is real. So you can keep `WM2002`
on and turn `WM2001` off, or the reverse.

## WM2001

**`Unwrap` throws when there is nothing to unwrap.** It converts an absence you had
already captured in the type back into an unhandled exception.

```diff
-int count = option.Unwrap();
+int count = option.UnwrapOr(0);
```

`UnwrapOrElse` defers the fallback until it is needed, and `Match` handles both
branches explicitly. On a value type the quick fix reports `WM2015`, for the reason
given above.

**Quick fix:** `UnwrapOrDefault()`.

## WM2002

**`Expect` throws too, and names the invariant on the way out.** That is defensible
where the invariant is genuine, which is why this is a separate rule from `WM2001` —
you can keep one on and turn the other off.

```diff
-int count = result.Expect("the length was validated upstream");
+int count = result.UnwrapOr(0);
```

**Quick fix:** `UnwrapOrDefault()`.

## WM2003

**A `throw` inside a member returning `Result` goes around its own signature.** The
signature promises failures arrive as values, so a caller who handled `Err` still has
to wrap the call in `try` to be safe.

```diff
-if (!found) throw new InvalidOperationException("no such user");
+if (!found) return new Error("NoSuchUser", "no such user");
```

Ignores the same throws `WM3002` does, listed under Migration aids.

## WM2004

**An `IsSome` check with an `Unwrap` inside it asks the same question twice.** Nothing
enforces that the two answers agree; whoever reads the code has to notice.

```diff
-if (option.IsSome) Console.WriteLine(option.Unwrap());
+option.Inspect(Console.WriteLine);
```

Use `Match` where both branches do work, `Inspect` where only the present one does.

## WM2005

**`Map` followed by `Flatten` is `AndThen`.** Mapping with a function that itself
returns an `Option` builds an `Option<Option<T>>`, and the flatten then takes apart
what the map just built.

```diff
-option.Map(FindParent).Flatten()
+option.AndThen(FindParent)
```

**Quick fix:** `AndThen`.

## WM2006

**A state check combined with an unwrap of the same value already has a name.**
`IsSomeAnd`, `IsNoneOr`, `IsOkAnd` and `IsErrAnd` take the predicate and supply the
value to it.

```diff
-if (option.IsSome && option.Unwrap() > 10)
+if (option.IsSomeAnd(x => x > 10))
```

Distinct from `WM2004`: that one is a guard around a block, this one is a single
boolean expression.

## WM2007

**`UnwrapOr` given the default of the type is `UnwrapOrDefault`.** The same result
without naming the type's default yourself.

```diff
-int count = option.UnwrapOr(0);
+int count = option.UnwrapOrDefault();
```

On a value type, applying this reports `WM2015` on the result. That is deliberate and
explained above.

**Quick fix:** `UnwrapOrDefault()`.

## WM2008

**Comparing an `Option` or `Result` to `null` reads as an absence check and is not
one.** Neither type is ever null in correct use, so the comparison is always false and
the case you meant to test goes untested.

```diff
-if (option == null)
+if (option.IsNone)
```

Covers `is null` and `is not null` patterns as well as `==` and `!=`. `WM2008` owns
every null comparison, which is why `WM1002` leaves them alone — one diagnostic per
site, not two.

**Quick fix:** the matching state check.

## WM2009

**`Option<Option<T>>` has three states and two meanings.** An absent outer and an
absent inner are different values that almost no caller acts on differently.

```diff
-Option<Option<User>> user = FindUser(id).Map(LoadProfile);
+Option<Profile> user = FindUser(id).AndThen(LoadProfile);
```

It usually means a `Map` wanted to be an `AndThen`, which is `WM2005`.

## WM2010

**Retired in 7.0.0. This rule no longer ships.** The anchor stays because build output
from earlier versions links here.

`Result` used to declare one implicit conversion from `TOk` and another from `TErr`.
Where those were the same type the compiler could not choose between them, so every
implicit conversion became a compile error — and `Ok` and `Err` were indistinguishable
to a reader. 7.0.0 removes both conversions, so there is nothing left for the rule to
report. A `Result<T, T>` is still worth avoiding for the second reason.

```diff
-Result<string, string> Parse(string input);
+Result<string, Error> Parse(string input);
```

## WM2011

**`Some`, `None`, `Ok` and `Err` are the cases, not the type.** A field declared
`Some<int>` can never be `None`, which is the entire point of `Option`.

```diff
-Some<int> total;
+Option<int> total;
```

**Quick fix:** the base type.

## WM2012

**A nullable member beside `Option` members gives one type two ways to say
"absent".** Callers then have to remember which convention applies to which member.

```diff
-string? DisplayName { get; }
+Option<string> DisplayName { get; }
```

Fires only on a type that already uses `Option` or `Result` somewhere. `WM3001` is the
version for a codebase that has not adopted the library at all.

## WM2013

**A discarded `Option` is a question nobody read the answer to.** Less harmful than
discarding a `Result`, which `WM1006` reports as a bug, but usually still a mistake.

```diff
-option.Filter(IsActive);
+Option<User> active = option.Filter(IsActive);
```

## WM2015

**On a value type, `UnwrapOrDefault` hands back `0` for the absent case.** `T?` on a
type parameter constrained only to `notnull` is an annotation, not a `Nullable<T>`, so
nothing distinguishes "there was no value" from a real zero. The message names
the value you get back, so it reads "hands back 0, the default of 'int'", and it
renders an enum's zero member by name.

```diff
-int? count = option.UnwrapOrDefault();
+int? count = option.UnwrapOrNull();
```

Perfectly legitimate where you do want the default, which is why this informs rather
than warns. `MapOrDefault` and `MapOrNull` work the same way.

**Quick fix:** `UnwrapOrNull()` or `MapOrNull()`.

## WM2016

**An eager argument runs even when it is thrown away.** `And`, `Or`, `UnwrapOr`,
`MapOr` and `OkOr` evaluate their argument before they check whether the receiver
needs it. An expensive call or one with a side effect runs unconditionally.

```diff
-option.UnwrapOr(LoadFallbackFromDisk())
+option.UnwrapOrElse(() => LoadFallbackFromDisk())
```

The lazy siblings (`AndThen`, `OrElse`, `UnwrapOrElse`, `MapOrElse` and
`OkOrElse`) take a delegate and only call it when the other branch is taken.

The rule reports what it cannot prove is **free**, not what it can prove is
expensive, because only the first is decidable. It stays silent on a constant, a
bare local, parameter, field or property read, and `default`. It also stays
silent on an expression built entirely out of those, so it leaves
`fallback + 1`, `defaults[0]` and `flag ? a : b` alone.

It fires on a call, a `new` and an `await`. It also fires on a user-defined
operator or implicit conversion. Both are ordinary method calls, however short
they look — `option.UnwrapOr(count)` reports when `count` is an `int` and the
option holds a type you can implicitly convert an `int` to.

The message tells you which of two reasons applies:

| The message says | What to do |
| --- | --- |
| `and computing it may be expensive` | Weigh it. The rule cannot tell a cheap call from a costly one |
| `and evaluating it changes state` | Act on it. An increment or an assignment runs on every call, needed or not |

Moving a state-changing argument to the `Else` sibling changes what your code
does, not just what it costs.

{% hint style="info" %}
**A known false positive.** The rule cannot tell a cheap call from an expensive
one, so it fires on `option.UnwrapOr(GetZero())` as readily as on a database
round-trip. When it does, you pay a delegate allocation to avoid nothing. That is
why it is a suggestion. Ignore it where the call is trivial.
{% endhint %}

A bare property read is skipped whatever the receiver, including one whose getter
computes. We cannot tell an auto-property from a computed one when it comes from
another assembly, and a rule that behaved differently depending on which assembly
declared a property would be worse than one that skips them all.

**Quick fix:** wrap the argument in a lambda and call the `Else` sibling.

## WM2017

**Your delegate captures, and there is an overload that would not.** A lambda that
reads a local or a parameter from the enclosing method allocates a display class
every time the call site runs.

```diff
-option.Map(value => value + offset)
+option.Map(offset, static (value, state) => value + state)
```

The state overload takes your data as its first argument and hands it to the
delegate, so the delegate closes over nothing and the compiler caches it. The
closure costs 88 bytes at every call — 24 for the display class, 64 for the
delegate.

The rule covers every method that has a state overload, which is nearly every
delegate-taking method on both types. See
[Where you can use it](../reference/state-overloads.md#where-you-can-use-it) for the list.

`Match` is the most expensive of them to call with a closure. Its two branches
share one display class but need a delegate each, so the call costs 152 bytes
rather than 88.

It reads the list off the receiver's own type rather than matching names, so a
delegate-taking method with no state overload (`ZipWith` and `Reduce`) never
gets pointed at one that does not exist.

It stays quiet when:

- **The lambda captures only `this`** — a bare field read, or a call to another
  method on the same type. That allocates a delegate rather than a display class,
  a much smaller cost, and reporting it would fire on most ordinary code.
- **The lambda captures nothing.** The compiler already caches it in a static
  field, so the state overload would buy you nothing.
- **You are already on the state overload.**

**No quick fix.** The obvious rewrite reuses the captured name as the new
parameter, which shadows the enclosing local. That is fine from C# 8 but is
`CS0136` on C# 7.3, and this analyzer reaches consumers on every language version.

## WM2018

**Two enums generate the same error code.** An `[ErrorCodeCatalog]` enum builds each
code from a format, and by default that format is the enum's name and the member's name,
so two enums sharing a name in different namespaces generate the same code for every
member name they share.

```csharp
namespace Ordering;

[ErrorCodeCatalog]
public enum OrderErrorCode { NotFound }   // "OrderErrorCode.NotFound"

namespace Shipping;

[ErrorCodeCatalog]
public enum OrderErrorCode { NotFound }   // "OrderErrorCode.NotFound" -- the same code
```

Whoever reads the code cannot tell which error happened. Namespaces separate the two
enums in your source and do not separate the codes.

The rule reports on the second declaration in alphabetical order, and names both
members and the shared code in the message. It reports once per colliding member, not
once per pair of enums, so an enum with three shared members gives you three
diagnostics.

The rule keys on the generated code, not on the enum's name, so a `Format` moves what
it sees. Two differently named enums that share `"order.{member:kebab}"` collide and are
reported; two enums sharing a name with different formats do not collide and are not.

**No quick fix.** The fix is a rename or a different format, and which of the two enums
should keep the code is not something the analyzer can work out.

See [Error code catalogs](../source-generation/error-code-catalogs.md) for what the
attribute generates.

## WM2019

**A generated error code is missing from the registry.** A project that commits an
`ErrorCodes.txt` has opted into reviewing its error codes as a list, so a code that is
not on the list is a wire contract nobody read a diff for.

```csharp
[ErrorCodeCatalog(Format = "order.{member:kebab}")]
public enum OrderErrorCode
{
    NotFound,        // "order.not-found" -- listed
    AlreadyShipped,  // "order.already-shipped" -- not listed, reported here
}
```

Reported on the member, because that is the thing you can act on.

**Quick fix: Update `ErrorCodes.txt`.** It rewrites the whole file from the compilation
— every missing code added, every stale entry removed, sorted, your leading comment
block kept. One invocation is enough however many diagnostics there are.

A project with no `ErrorCodes.txt` never sees this rule. See
[#reviewing-your-codes-as-a-list](../source-generation/reviewing-codes.md#reviewing-your-codes-as-a-list "mention").

## WM2020

**The registry lists a code nothing generates.** The other direction: an entry left
behind by a rename or a deletion, claiming a code the project no longer produces.

```
order.already-shipped
order.cancelled          <- nothing generates this any more
order.not-found
```

Reported against `ErrorCodes.txt` itself, at the line, because nothing in your source
corresponds to it.

**No quick fix.** Roslyn does not offer fixes for a diagnostic reported at the end of a
compilation, which this has to be — whether an entry is stale cannot be known until every
enum in the project has been seen. In practice the `WM2019` fix removes stale entries
too, so the two travel together; delete the named line by hand if it is your only
divergence.

{% hint style="warning" %}
This rule's severity cannot be set from a path-matched `.editorconfig` section, not even
`[*]`. It needs a global analyzer config — see
[#making-a-divergence-fail-the-build](../source-generation/reviewing-codes.md#making-a-divergence-fail-the-build "mention").
{% endhint %}

## WM2021

**A property pattern is a state check the other rules cannot see.**
`option is { IsSome: true }` asks exactly what `option.IsSome` asks. It reads as
though pattern matching is doing something for you here, and it is not — the
monad exposes no value to destructure, so the pattern only reaches the same
boolean by a longer route.

```diff
-if (option is { IsSome: true }) { return option.Unwrap(); }
+return option.UnwrapOr(0);
```

The rule fires wherever a property subpattern reads `IsSome`, `IsNone`, `IsOk`
or `IsErr` on an `Option` or a `Result`: an `is` expression, a negated one, a
`switch` arm, a `when` clause, and a subpattern nested inside another type's
pattern.

```diff
-return option switch { { IsSome: true } => 1, _ => 0 };
+return option.MapOr(0, _ => 1);
```

The value you test for makes no difference. `{ IsSome: false }` asks the same
question as `{ IsSome: true }` and hides it the same way.

{% hint style="info" %}
**Why this is its own rule.** `WM2004` and `WM2006` both match a property *read*.
A subpattern is not one, so writing the check as a pattern used to silence them
without changing what the code does. That is what this rule reports — not the
pattern's style, but the fact that it opts your call site out of the rules that
would otherwise read it.
{% endhint %}

Fixing this rule leaves you with `option.IsSome`, which `WM2004` may then report
if an unwrap follows it. That chain is deliberate. Each rule states one true
thing, and the second only becomes visible once the first is resolved — the same
way `WM2007` and `WM2015` pair up.

**No quick fix.** The rewrite depends on where the pattern sits. An `is`
expression becomes a property read, a negated one becomes a negated read, and a
`switch` arm needs restructuring into a `Match`. A fix that handled only the
first would leave the two shapes most worth correcting untouched, while implying
the rule had been dealt with.

## WM2022

**`CS0411` here is not a generics problem.** `AndThenAsync` and `OrElseAsync` take a
step that returns `ValueTask<Option<T>>` or `ValueTask<Result<TOk, TErr>>`. Hand one a
method group that returns `Task<...>` and the conversion fails, so the compiler reports
a type inference failure — a message that names neither `ValueTask` nor the parameter,
and sends you looking at your type arguments. This rule says the part `CS0411` leaves
out.

```diff
-Task<Option<Order>> LoadOrder(int id) => ...;
+ValueTask<Option<Order>> LoadOrder(int id) => ...;

 // option is an Option<int>
 ValueTask<Option<Order>> order = option.AndThenAsync(LoadOrder);
```

There are two corrections, and which you want depends on who owns the step.

**Change the step to return `ValueTask`** where the step is yours. Prefer this. Every
async member of this library returns a `ValueTask`, so a step that does the same drops
straight into another chain as a step of its own.

**Wrap it** where the step is third-party and you cannot change its signature:

```csharp
option.AndThenAsync(async id => await LoadOrder(id));
```

**The quick fix offers the wrap only**, though the message names both corrections.
Retyping the step is safe only where it is already `async` — a `Task.FromResult` body
does not convert — and it changes a signature every other caller of that member sees.
A fix reading one call site cannot judge either, so the message states both and leaves
the choice to you.

## What it does and does not report

The rule fires on `AndThenAsync` and `OrElseAsync` only, on both `Option` and
`Result`, and on both the monad and the awaited-receiver extensions.

It fires only where the call **failed to bind**. That is the whole point: a call that
compiles needs no advice, and a rule that could not read a failed call would have
nothing to report. So `WM2022` never breaks a build that passes today — the build is
already broken when it fires, which is also why it is a suggestion rather than a
warning.

It reports **method groups**, not lambdas. An async lambda already infers a
`ValueTask` from its body, so there is nothing to correct.

The quick fix declines on an **overloaded** method group. The lambda's parameter name
comes from the method's own, and there is no reason to prefer one overload's spelling
of it over another's.

