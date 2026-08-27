---
description: >-
  The diagnostics that ship inside Waystone.Monads, what each one means, and
  how to turn them up or off.
---

# Analyzer rules

## What this page is for

`Waystone.Monads` ships a Roslyn analyzer inside the package. Install or upgrade to
6.0.0 and you get these rules. You add no reference and configure nothing.

Every rule has an ID like `WM1002`. The first digit tells you how much it matters:

| Rules | Severity | What they report |
| --- | --- | --- |
| `WM1xxx` | Warning | Code that throws or quietly does the wrong thing at run time |
| `WM2xxx` | Suggestion | Working code that reads better another way |
| `WM3xxx` | Off | Migration aids you turn on while you adopt the library |

Warnings show up in your build. Suggestions show up in your IDE only, so they never
break a build that passes today. The `WM3xxx` rules stay off until you enable them.

A separate set of diagnostics uses a `WMG` prefix. Those come from the source
generator rather than the analyzer, they are all errors, and they only fire on an
enum you marked with `[ErrorCodeCatalog]`. They are on
[generated-error-codes.md](generated-error-codes.md "mention") instead of this page.

{% hint style="info" %}
This page lists the `WM` rules as at the version it was written for. For the set
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

## Runtime bugs

These rules are warnings. Each one marks code that compiles and then misbehaves.

### WM1001

**`Some` cannot hold null.** `Option.Some(x)` throws `ArgumentNullException`
when `x` is null, so `Option.Some(default(string)!)` always throws.

```diff
-Option<string> option = Option.Some(default(string)!);
+Option<string> option = Option.None<string>();
```

The rule fires only when it can prove the value is null without running your
program — a `null` literal, or `default(T)` where `T` is a reference type. Use
`Option.FromNullable` when the value merely might be null; `WM1005` covers that
case.

A `Some` may hold `0`, `false` and any other value-type default from 6.0.0
onwards. Null is the only value it rejects.

**Quick fix:** use `Option.None<T>()`.

### WM1002

**Null where an `Option` or `Result` belongs.** Both types are records, so the
compiler lets you write `null` anywhere one is expected. Your next member access
then throws `NullReferenceException`.

```diff
-Option<int> option = null;
+Option<int> option = Option.None<int>();
```

The rule covers assignment, `return` and arguments you pass to a method. It also
catches a null you wrote as `null!`:

```diff
-Accept(null!);
+Accept(Option.None<int>());
```

Annotating the target `Option<int>?` stops this rule, because the annotation says you
meant to allow null. It starts `WM1008` instead, which asks you to drop the
annotation. An `Option` already has an empty case, so a nullable one gives you two
ways to say the same thing. This holds wherever you write the annotation — a
parameter, a return type, or a tuple or array element.

**Quick fix for `Option<T>`:** use `Option.None<T>()`. `Result<TOk, TErr>` gets no
quick fix, because nothing in your code says whether you meant `Ok` or `Err`.

### WM1003

**The default of an `Option` or `Result` is null.** `default(Option<int>)` gives you
no empty option. It gives you `null`, for the reason `WM1002` explains.

```diff
-Result<int, string> result = default;
+Result<int, string> result = Result.Err<int, string>("not set");
```

**Quick fix for `Option<T>`:** use `Option.None<T>()`.

### WM1005

**You passed a possibly null value to `Some`.** `Some` rejects null. Pass a value
the compiler treats as maybe-null and the call throws whenever that value is
null.

```diff
-Option<string> option = Option.Some(value);
+Option<string> option = Option.FromNullable(value);
```

This rule reads the compiler's nullable flow state, so it says nothing in a project
that has nullable reference types turned off. That project faces the bug most. Turn
nullable on.

**Quick fix:** use `Option.FromNullable`.

### WM1006

**You discarded a `Result`.** A `Result` used as a statement throws nothing and
reports nothing, so the failure vanishes:

```csharp
Save();   // returns Result<int, Error>, and you just lost the Err case
```

Handle it, or return it to a caller who will:

```csharp
return Save().Match(value => value, error => 0);
```

The rule follows an `await`, including one you wrote with `.ConfigureAwait(false)`,
so it catches a discarded `Task<Result<TOk, TErr>>` as well.

Rust marks `Result` as `#[must_use]` for the same reason. C# has no attribute that
forces you to consume a return value, so this rule stands in for one.

**No quick fix.** Only you can decide what the failure should do.

### WM1008

**An `Option` or `Result` is declared nullable.** Both are records, so the compiler
accepts `Option<int>?`. That gives you three states where two mean anything: a value,
an empty option, and null. `None` is already the empty case you are reaching for.

```diff
-Option<int>? option = Find(id);
+Option<int> option = Find(id);
```

The rule reads the annotation itself rather than the compiler's nullable state, so it
fires whether or not you build with nullable reference types on.

It also finds the annotation when the type is nested inside another one. A tuple
element, an array element and a type argument all count, and the element can sit in
any position of the tuple:

```diff
-(Option<int>? a, int b) Make() => (null, 1);
+(Option<int> a, int b) Make() => (Option.None<int>(), 1);
```

`Result` behaves the same way. There the rule points you to `Err` rather than `None`.

**Quick fix:** drop the `?`.

### WM1011

**Your delegate returns a task and the method does not await it.** A synchronous
method calls your delegate and stores whatever comes back. Give it one that
returns a task and the monad holds the task, not the result.

```diff
-var result = Option.Try(() => FetchCountAsync());
+var result = await Option.TryAsync(() => FetchCountAsync());

-Option<Task<int>> doubled = option.Map(x => DoubleAsync(x));
+Option<int> doubled = await option.MapAsync(x => DoubleAsync(x));
```

Both calls compile. Neither awaits anything, so the work has not finished when
the monad is handed back, and anything it throws goes unobserved. For `Try` the
damage is worse: **your exception handling is gone entirely** — a throw escapes
to your caller instead of becoming a `None` or an `Err`, and your configured
exception logger never sees it.

Use the `Async` sibling of whatever you called. It awaits the delegate and holds
the result.

#### What it does and does not report

The rule asks **where the task ends up**, not whether a delegate produced one.

| Call | Produces | Reported |
| --- | --- | --- |
| `Option.Try(() => FetchAsync())` | `Option<Task<int>>` | Yes |
| `option.Map(x => FetchAsync(x))` | `Option<Task<int>>` | Yes |
| `result.MapErr(e => FormatAsync(e))` | `Result<int, Task<string>>` | Yes |
| `option.Match(x => FetchAsync(x), () => ZeroAsync())` | `Task<int>` | No |
| `option.MapOr(zeroTask, x => FetchAsync(x))` | `Task<int>` | No |
| `Option.Some(FetchAsync())` | `Option<Task<int>>` | No |

`Match` and `MapOr` hand the task straight back, so you can await it and nothing
is lost. `Option.Some(FetchAsync())` takes a value rather than a delegate, so you
built that task on purpose.

Only a task trapped **inside** an `Option` or a `Result` is a defect — you cannot
await it without unwrapping first, and most callers never do.

This is a warning rather than a suggestion because it fires on code that runs and
does the wrong thing. It is also the only protection you have against the v6
removal of the async `Try` overloads, which rebinds those call sites silently. See
[Silent change 1](../upgrading-and-deprecations/v5-to-v6.md#silent-change-1-try-with-an-async-factory).

**No quick fix, deliberately.** Renaming to the `Async` sibling leaves you with an
unawaited task, and no fixer can decide where your `await` belongs.

## Idioms

These rules are suggestions. Your code works. The rule points at a clearer way to
write it. You see them in your IDE, and they stay out of your build.

| ID | What it reports | Quick fix |
| --- | --- | --- |
| [`WM2001`](#wm2001) | `Unwrap` and `UnwrapErr`, which throw when there is no value | `UnwrapOrDefault()` |
| [`WM2002`](#wm2002) | `Expect`, which throws when its invariant does not hold | `UnwrapOrDefault()` |
| [`WM2003`](#wm2003) | A `throw` inside a member that returns `Result`, so the failure escapes the channel your signature promises | — |
| [`WM2004`](#wm2004) | An `IsSome` check with an `Unwrap` inside it, which asks the same question twice | — |
| [`WM2005`](#wm2005) | `Map` followed by `Flatten`, which is `AndThen` | `AndThen` |
| [`WM2006`](#wm2006) | A state check combined with an unwrap of the same value, which is `IsSomeAnd`, `IsNoneOr`, `IsOkAnd` or `IsErrAnd` | — |
| [`WM2007`](#wm2007) | `UnwrapOr` given the default of the type, which is `UnwrapOrDefault` | `UnwrapOrDefault()` |
| [`WM2008`](#wm2008) | An `Option` or `Result` compared to `null`, which reads like an absence check but is not one | The matching state check |
| [`WM2009`](#wm2009) | `Option<Option<T>>`, which has three states where only two mean anything | — |
| [`WM2010`](#wm2010) | `Result<T, T>`, whose two implicit conversions are ambiguous | — |
| [`WM2011`](#wm2011) | A declaration that names `Some`, `None`, `Ok` or `Err` instead of `Option` or `Result`, so it can hold only one of the two states | The base type |
| [`WM2012`](#wm2012) | A nullable member sitting alongside members that use `Option`, so one type has two ways of saying "absent" | — |
| [`WM2013`](#wm2013) | A discarded `Option`, as `WM1006` does for `Result` | — |
| [`WM2015`](#wm2015) | `UnwrapOrDefault` or `MapOrDefault` producing a value type, where the default is indistinguishable from a real result | `UnwrapOrNull()` or `MapOrNull()` |
| [`WM2016`](#wm2016) | An argument to `Or`, `And`, `UnwrapOr`, `MapOr` or `OkOr` that is not free to evaluate, so it runs even when it is discarded | The `Else` sibling |
| [`WM2017`](#wm2017) | A delegate that captures a local or a parameter, where a state overload would avoid the closure | — |
| [`WM2018`](#wm2018) | Two `[ErrorCodeCatalog]` enums that generate the same error code | — |
| [`WM2019`](#wm2019) | A generated error code that `ErrorCodes.txt` does not list | Update `ErrorCodes.txt` |
| [`WM2020`](#wm2020) | An `ErrorCodes.txt` entry no catalog generates | — |
| [`WM2021`](#wm2021) | `IsSome`, `IsNone`, `IsOk` or `IsErr` read through a property pattern, which hides the check from the rules that read it | — |

`WM2008` owns every null comparison and null pattern, so `WM1002` leaves those
alone. You get one diagnostic per site, not two.

`WM2007` and `WM2015` point opposite ways on purpose, and both are suggestions so you
can decide. `WM2007` says `UnwrapOr(0)` is `UnwrapOrDefault()`, which is shorter and
says what it means. `WM2015` then says that on a value type `UnwrapOrDefault()` hands
back `0`, which you cannot tell apart from a real zero, and offers `UnwrapOrNull()`.
Take the second one where the difference matters to you. Applying the `WM2007` quick
fix on a value type reports `WM2015` on the result, for the same reason. The same pair
applies to the quick fixes `WM2001` and `WM2002` offer.

`WM2003` ignores the same throws as `WM3002`, listed below.

We split `WM2001` and `WM2002` on purpose. `Expect` states an invariant and names it
in the message, which is fair where the invariant is real. So you can keep `WM2002`
on and turn `WM2001` off, or the reverse.

### WM2001

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

### WM2002

**`Expect` throws too, and names the invariant on the way out.** That is defensible
where the invariant is genuine, which is why this is a separate rule from `WM2001` —
you can keep one on and turn the other off.

```diff
-int count = result.Expect("the length was validated upstream");
+int count = result.UnwrapOr(0);
```

**Quick fix:** `UnwrapOrDefault()`.

### WM2003

**A `throw` inside a member returning `Result` goes around its own signature.** The
signature promises failures arrive as values, so a caller who handled `Err` still has
to wrap the call in `try` to be safe.

```diff
-if (!found) throw new InvalidOperationException("no such user");
+if (!found) return new Error("NoSuchUser", "no such user");
```

Ignores the same throws `WM3002` does, listed under Migration aids.

### WM2004

**An `IsSome` check with an `Unwrap` inside it asks the same question twice.** Nothing
enforces that the two answers agree; the reader is left to notice.

```diff
-if (option.IsSome) Console.WriteLine(option.Unwrap());
+option.Inspect(Console.WriteLine);
```

Use `Match` where both branches do work, `Inspect` where only the present one does.

### WM2005

**`Map` followed by `Flatten` is `AndThen`.** Mapping with a function that itself
returns an `Option` builds an `Option<Option<T>>`, and the flatten then takes apart
what the map just built.

```diff
-option.Map(FindParent).Flatten()
+option.AndThen(FindParent)
```

**Quick fix:** `AndThen`.

### WM2006

**A state check combined with an unwrap of the same value already has a name.**
`IsSomeAnd`, `IsNoneOr`, `IsOkAnd` and `IsErrAnd` take the predicate and supply the
value to it.

```diff
-if (option.IsSome && option.Unwrap() > 10)
+if (option.IsSomeAnd(x => x > 10))
```

Distinct from `WM2004`: that one is a guard around a block, this one is a single
boolean expression.

### WM2007

**`UnwrapOr` given the default of the type is `UnwrapOrDefault`.** The same result
without naming the type's default yourself.

```diff
-int count = option.UnwrapOr(0);
+int count = option.UnwrapOrDefault();
```

On a value type, applying this reports `WM2015` on the result. That is deliberate and
explained above.

**Quick fix:** `UnwrapOrDefault()`.

### WM2008

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

### WM2009

**`Option<Option<T>>` has three states and two meanings.** An absent outer and an
absent inner are different values that almost no caller acts on differently.

```diff
-Option<Option<User>> user = FindUser(id).Map(LoadProfile);
+Option<Profile> user = FindUser(id).AndThen(LoadProfile);
```

It usually means a `Map` wanted to be an `AndThen`, which is `WM2005`.

### WM2010

**`Result<T, T>` cannot use its own implicit conversions.** `Result` declares one
conversion from `TOk` and another from `TErr`. When those are the same type the
compiler cannot choose between them, so every implicit conversion becomes a compile
error — and `Ok` and `Err` become indistinguishable to a reader.

```diff
-Result<string, string> Parse(string input);
+Result<string, Error> Parse(string input);
```

### WM2011

**`Some`, `None`, `Ok` and `Err` are the cases, not the type.** A field declared
`Some<int>` can never be `None`, which is the entire point of `Option`.

```diff
-Some<int> total;
+Option<int> total;
```

**Quick fix:** the base type.

### WM2012

**A nullable member beside `Option` members gives one type two ways to say
"absent".** Callers then have to remember which convention applies to which member.

```diff
-string? DisplayName { get; }
+Option<string> DisplayName { get; }
```

Fires only on a type that already uses `Option` or `Result` somewhere. `WM3001` is the
version for a codebase that has not adopted the library at all.

### WM2013

**A discarded `Option` is a question nobody read the answer to.** Less harmful than
discarding a `Result`, which `WM1006` reports as a bug, but usually still a mistake.

```diff
-option.Filter(IsActive);
+Option<User> active = option.Filter(IsActive);
```

### WM2015

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

### WM2016

**An eager argument runs even when it is thrown away.** `And`, `Or`, `UnwrapOr`,
`MapOr` and `OkOr` evaluate their argument before they check whether the receiver
needs it. An expensive call or one with a side effect runs unconditionally.

```diff
-option.UnwrapOr(LoadFallbackFromDisk())
+option.UnwrapOrElse(() => LoadFallbackFromDisk())
```

The lazy siblings — `AndThen`, `OrElse`, `UnwrapOrElse`, `MapOrElse` and
`OkOrElse` — take a delegate and only call it when the other branch is taken.

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

### WM2017

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
[Where you can use it](core-functionality.md#where-you-can-use-it) for the list.

`Match` is the most expensive of them to call with a closure. Its two branches
share one display class but need a delegate each, so the call costs 152 bytes
rather than 88.

It reads the list off the receiver's own type rather than matching names, so a
delegate-taking method with no state overload — `ZipWith` and `Reduce` — never
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

### WM2018

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

See [generated-error-codes.md](generated-error-codes.md "mention") for what the
attribute generates.

### WM2019

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
[#reviewing-your-codes-as-a-list](generated-error-codes.md#reviewing-your-codes-as-a-list "mention").

### WM2020

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
[#making-a-divergence-fail-the-build](generated-error-codes.md#making-a-divergence-fail-the-build "mention").
{% endhint %}

### WM2021

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

## Migration aids

These two rules ship **off**. They report on code that has not adopted the library
yet, so in most codebases they fire everywhere. Turn them on while you convert, then
turn them off again.

| ID | What it reports |
| --- | --- |
| [`WM3001`](#wm3001) | A member that returns a nullable type, where `Option<T>` would make the absent case impossible to ignore |
| [`WM3002`](#wm3002) | A `throw`, where returning `Result<TOk, Error>` would state the failure in the signature |

Enable one in your `.editorconfig`:

```ini
[*.cs]
dotnet_diagnostic.WM3001.severity = suggestion
dotnet_diagnostic.WM3002.severity = suggestion
```

`WM3002` ignores the throws a `Result` would not improve: `ArgumentException` and
its subtypes, `NotImplementedException`, `NotSupportedException`,
`ObjectDisposedException`, a bare `throw;` rethrow, and any throw inside a lambda
you pass to `Option.Try` or `Result.Try`.

### WM3001

**A nullable return leaves the absent case easy to ignore.** `Option<T>` makes the
caller acknowledge it. Off by default, because in a codebase that has not adopted the
library this fires on very nearly every member.

```diff
-User? FindUser(int id);
+Option<User> FindUser(int id);
```

`WM2012` is the narrower, on-by-default version, for a type that already uses `Option`
elsewhere.

### WM3002

**A `throw` states a failure nowhere in the signature.** Returning
`Result<TOk, Error>` puts it there, where a caller cannot miss it. Off by default,
because it fires on every throw in a codebase that has not adopted `Result`.

```diff
-if (!found) throw new InvalidOperationException("no such user");
+if (!found) return new Error("NoSuchUser", "no such user");
```

It skips the throws listed above, which a `Result` would not improve. `WM2003` is the
on-by-default version, for a member that already returns `Result`.

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
