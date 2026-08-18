---
description: >-
  The 26 diagnostics that ship inside Waystone.Monads, what each one means, and
  how to turn them up or off.
---

# Analyzer rules

## What this page is for

`Waystone.Monads` ships a Roslyn analyzer inside the package. Install or upgrade to
5.4.0 and you get these rules. You add no reference and configure nothing.

Every rule has an ID like `WM1002`. The first digit tells you how much it matters:

| Rules | Severity | What they report |
| --- | --- | --- |
| `WM1xxx` | Warning | Code that throws or quietly does the wrong thing at run time |
| `WM2xxx` | Suggestion | Working code that reads better another way |
| `WM3xxx` | Off | Migration aids you turn on while you adopt the library |

Warnings show up in your build. Suggestions show up in your IDE only, so they never
break a build that passes today. The `WM3xxx` rules stay off until you enable them.

{% hint style="warning" %}
Do you build with `TreatWarningsAsErrors`? Then a `WM1xxx` rule that fires breaks
your build after you upgrade. We chose that on purpose. Every one of these rules
marks code that throws or returns the wrong value at run time. Read the rule before
you suppress it.
{% endhint %}

## Runtime bugs

These rules are warnings. Each one marks code that compiles and then misbehaves.

### WM1001

**`Some` cannot hold a default value.** `Option.Some(x)` throws
`InvalidOperationException` when `x` equals `default(T)`. So `Option.Some(0)`,
`Option.Some(false)` and `Option.Some(Guid.Empty)` always throw.

```diff
-Option<int> option = Option.Some(0);
+Option<int> option = Option.None<int>();
```

The rule fires only on constants and on the well-known names for a default:
`Guid.Empty`, `DateTime.MinValue`, `DateTimeOffset.MinValue`, `TimeSpan.Zero`,
`IntPtr.Zero` and `UIntPtr.Zero`. It skips `Option.Some(count)`, because it cannot
tell whether `count` is zero without running your program.

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

Annotating the parameter or the return type `Option<int>?` stops this rule, because
the annotation says you meant to allow null. It starts `WM1008` instead, which asks
you to drop the annotation. An `Option` already has an empty case, so a nullable one
gives you two ways to say the same thing.

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

### WM1004

**A default value converts to `None`.** `Option<T>` converts implicitly from `T`,
and that conversion returns `None` when the value equals `default(T)`. So this line
does not give you a `Some` holding zero. It gives you `None`, and it says nothing:

```csharp
Option<int> option = 0;   // this is None, not Some(0)
```

Nothing throws. Without this rule, nothing warns you either. Write the case you
meant:

```diff
-Option<int> option = 0;
+Option<int> option = Option.None<int>();
```

**Quick fix:** say `Option.None<T>()` outright.

### WM1005

**You passed a possibly null value to `Some`.** `Some` rejects any value equal to
`default(T)`, and the default of a reference type is null. Pass a value the compiler
treats as maybe-null and the call throws whenever that value is null.

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

### WM1007

**A type derives from `Option` or `Result`.** Both types have exactly two cases, and
every member that switches on them handles two. A third case is invisible to `Match`,
so it silently takes whichever branch its base case falls into.

```diff
-internal sealed record Cached<T>(T Value) : Some<T>(Value);
+internal sealed record Cached<T>(Option<T> Value);
```

Compose the monad instead of inheriting from it. Hold one as a member, as above.

Both hierarchies close in v6, so a derived type stops compiling then. That is why
this rule is a warning even though it has no quick fix: you need the notice before
the upgrade, and no fixer can rewrite your type and every one of its callers.

**No quick fix.** The correction changes your type and cascades to its callers.

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

**Quick fix:** drop the `?`.

### WM1009

**An `Option` over a type whose zero is meaningful.** `Option.Some(x)` throws when
`x` equals `default(T)`. So an `Option<bool>` cannot hold `false`, and an
`Option<Colour>` cannot hold whichever member you numbered `0`. Part of the type's
own domain is unreachable.

```csharp
Option<bool> option = Option.Some(false);   // throws
```

For `bool`, use a three-state enum and let its zero member carry what you wanted
`None` to say:

```diff
-Option<bool> approved = ...;
+enum Approval { Pending = 0, Approved = 1, Rejected = 2 }
```

For an enum, renumber it so no meaningful member is `0`:

```diff
-enum Colour { Red = 0, Green = 1 }
+enum Colour { Red = 1, Green = 2 }
```

The rule covers `bool` and enums that have a zero member. It says nothing about
`Option<int>`, where zero is usually a real value you would rather model another way,
and nothing about an enum whose members all start at `1`.

**No quick fix.** Renumbering an enum or replacing a `bool` changes your call sites.

## Idioms

These rules are suggestions. Your code works. The rule points at a clearer way to
write it. You see them in your IDE, and they stay out of your build.

| ID | What it reports | Quick fix |
| --- | --- | --- |
| `WM2001` | `Unwrap` and `UnwrapErr`, which throw when there is no value | `UnwrapOrDefault()` |
| `WM2002` | `Expect`, which throws when its invariant does not hold | `UnwrapOrDefault()` |
| `WM2003` | A `throw` inside a member that returns `Result`, so the failure escapes the channel your signature promises | — |
| `WM2004` | An `IsSome` check with an `Unwrap` inside it, which asks the same question twice | — |
| `WM2005` | `Map` followed by `Flatten`, which is `AndThen` | `AndThen` |
| `WM2006` | A state check combined with an unwrap of the same value, which is `IsSomeAnd`, `IsNoneOr`, `IsOkAnd` or `IsErrAnd` | — |
| `WM2007` | `UnwrapOr` given the default of the type, which is `UnwrapOrDefault` | `UnwrapOrDefault()` |
| `WM2008` | An `Option` or `Result` compared to `null`, which reads like an absence check but is not one | The matching state check |
| `WM2009` | `Option<Option<T>>`, which has three states where only two mean anything | — |
| `WM2010` | `Result<T, T>`, whose two implicit conversions are ambiguous | — |
| `WM2011` | A declaration that names `Some`, `None`, `Ok` or `Err` instead of `Option` or `Result`, so it can hold only one of the two states | The base type |
| `WM2012` | A nullable member sitting alongside members that use `Option`, so one type has two ways of saying "absent" | — |
| `WM2013` | A discarded `Option`, as `WM1006` does for `Result` | — |
| `WM2014` | `FlatMap`, which is obsolete and goes away in v6 | `AndThen` |
| `WM2015` | `UnwrapOrDefault` or `MapOrDefault` producing a value type, where the default is indistinguishable from a real result | `UnwrapOrNull()` or `MapOrNull()` |

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

## Migration aids

These two rules ship **off**. They report on code that has not adopted the library
yet, so in most codebases they fire everywhere. Turn them on while you convert, then
turn them off again.

| ID | What it reports |
| --- | --- |
| `WM3001` | A member that returns a nullable type, where `Option<T>` would make the absent case impossible to ignore |
| `WM3002` | A `throw`, where returning `Result<TOk, Error>` would state the failure in the signature |

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
