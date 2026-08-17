---
description: >-
  The 21 diagnostics that ship inside Waystone.Monads, what each one means, and
  how to turn them up or off.
---

# Analyzer rules

## What this page is for

`Waystone.Monads` ships a Roslyn analyzer inside the package. Install or upgrade to
5.3.0 and you get these rules with no extra reference and no configuration.

Each rule has an ID like `WM1002`. The first digit says how much it matters:

| Rules | Severity | What they report |
| --- | --- | --- |
| `WM1xxx` | Warning | Code that throws or silently misbehaves at runtime |
| `WM2xxx` | Suggestion | Working code that reads better another way |
| `WM3xxx` | Off | Migration aids you turn on while adopting the library |

Warnings appear in your build. Suggestions appear in your IDE only, so they do not
change a build that already passes. The `WM3xxx` rules are disabled until you
enable them.

{% hint style="warning" %}
If you build with `TreatWarningsAsErrors`, a `WM1xxx` rule that fires will fail
your build after you upgrade. That is deliberate — every one of them marks code
that throws or produces the wrong value at runtime. Read the rule before you
suppress it.
{% endhint %}

## Runtime bugs

These are warnings. Each marks code that compiles and then does the wrong thing.

### WM1001

**Some cannot hold a default value.** `Option.Some(x)` throws
`InvalidOperationException` when `x` equals `default(T)`. `Option.Some(0)`,
`Option.Some(false)` and `Option.Some(Guid.Empty)` are guaranteed to throw.

```diff
-Option<int> option = Option.Some(0);
+Option<int> option = Option.None<int>();
```

The rule only fires on constants and the well-known aliases for a default —
`Guid.Empty`, `DateTime.MinValue`, `DateTimeOffset.MinValue`, `TimeSpan.Zero`,
`IntPtr.Zero` and `UIntPtr.Zero`. `Option.Some(count)` is left alone, because
whether `count` is zero cannot be known without running the program.

**Fix available:** replace with `Option.None<T>()`.

### WM1002

**Null assigned where an Option or Result is expected.** Both types are records,
so the compiler permits `null` wherever one is expected. The next member access
then throws `NullReferenceException`.

```diff
-Option<int> option = null;
+Option<int> option = Option.None<int>();
```

The rule covers assignment, `return`, and passing an argument, including when the
null is written `null!`:

```diff
-Accept(null!);
+Accept(Option.None<int>());
```

A parameter or return type you have explicitly annotated `Option<int>?` is not
reported. Annotating it says you meant to allow null there.

**Fix available for `Option<T>`:** replace with `Option.None<T>()`. There is no fix
for `Result<TOk, TErr>`, because nothing in the code says whether you meant `Ok` or
`Err`.

### WM1003

**The default of an Option or Result is null.** `default(Option<int>)` is not an
empty option. It is `null`, for the same reason as `WM1002`.

```diff
-Result<int, string> result = default;
+Result<int, string> result = Result.Err<int, string>("not set");
```

**Fix available for `Option<T>`:** replace with `Option.None<T>()`.

### WM1004

**A default value converts to None.** `Option<T>` has an implicit conversion from
`T` that returns `None` when the value equals `default(T)`. So this does not
produce a `Some` holding zero — it produces `None`, silently:

```csharp
Option<int> option = 0;   // this is None, not Some(0)
```

Nothing throws, and nothing warns without this rule. Write the case you meant:

```diff
-Option<int> option = 0;
+Option<int> option = Option.None<int>();
```

**Fix available:** make it explicit as `Option.None<T>()`.

### WM1005

**A possibly null value is passed to Some.** `Some` rejects a value equal to
`default(T)`, and the default of a reference type is null. Passing a value the
compiler considers maybe-null throws when it is null at run time.

```diff
-Option<string> option = Option.Some(value);
+Option<string> option = Option.FromNullable(value);
```

This rule reads the compiler's nullable flow state, so it is silent in a project
with nullable reference types disabled. That is the project most exposed to the
bug — another reason to turn nullable on.

**Fix available:** replace with `Option.FromNullable`.

### WM1006

**The Result of this call is discarded.** A `Result` used as a statement throws
nothing and reports nothing, so the failure disappears:

```csharp
Save();   // returns Result<int, Error>, and the Err case is now lost
```

Handle it or return it:

```csharp
return Save().Match(value => value, error => 0);
```

Rust marks `Result` `#[must_use]` for the same reason. C# has no equivalent
attribute for a return value you must consume, so this rule stands in for it.

**No fix available** — what to do with the failure is the decision the rule is
asking you to make.

## Idioms

These are suggestions. The code works; the rule points at a clearer way to write
it. They show up in your IDE and stay out of your build.

| ID | Reports | Fix |
| --- | --- | --- |
| `WM2001` | `Unwrap` and `UnwrapErr` throw when there is no value | `UnwrapOrDefault()` |
| `WM2002` | `Expect` throws when its invariant does not hold | `UnwrapOrDefault()` |
| `WM2003` | A `throw` inside a member that returns `Result`, which bypasses the failure channel the signature promises | — |
| `WM2004` | An `IsSome` check with an `Unwrap` inside it, which asks the same question twice | — |
| `WM2005` | `Map` followed by `Flatten`, which is `FlatMap` | `FlatMap` |
| `WM2006` | A state check combined with an unwrap of the same value, which is `IsSomeAnd`, `IsNoneOr`, `IsOkAnd` or `IsErrAnd` | — |
| `WM2007` | `UnwrapOr` given the default of the type, which is `UnwrapOrDefault` | `UnwrapOrDefault()` |
| `WM2008` | An `Option` or `Result` compared to `null`, which reads as an absence check but is not one | The matching state check |
| `WM2009` | `Option<Option<T>>`, which has three states where two are meaningful | — |
| `WM2010` | `Result<T, T>`, whose two implicit conversions are ambiguous | — |
| `WM2011` | A declaration naming `Some`, `None`, `Ok` or `Err` instead of `Option` or `Result`, which can only hold one of the two states | The base type |
| `WM2012` | A nullable member alongside members that use `Option`, so one type has two conventions for absence | — |
| `WM2013` | A discarded `Option`, as `WM1006` does for `Result` | — |

`WM2003` ignores the throws listed under `WM3002` below, for the same reasons.

`WM2001` and `WM2002` are separate rules on purpose. `Expect` states an invariant
and names it in the message, which is defensible where the invariant is real, so
you can leave `WM2002` on and turn `WM2001` off, or the reverse.

## Migration aids

These two are **disabled**. They report on code that has not adopted the library
yet, so in most codebases they fire everywhere. Turn them on while you convert,
and turn them off when you are done.

| ID | Reports |
| --- | --- |
| `WM3001` | A member returning a nullable type, where `Option<T>` would make the absent case impossible to ignore |
| `WM3002` | A `throw`, where a `Result<TOk, Error>` return would state the failure in the signature |

To enable one, add it to your `.editorconfig`:

```ini
[*.cs]
dotnet_diagnostic.WM3001.severity = suggestion
dotnet_diagnostic.WM3002.severity = suggestion
```

`WM3002` deliberately ignores throws that a `Result` would not improve:
`ArgumentException` and its subtypes, `NotImplementedException`,
`NotSupportedException`, `ObjectDisposedException`, a bare `throw;` rethrow, and
any throw inside a lambda passed to `Option.Try` or `Result.Try`.

## Changing a rule

Every rule is configured the standard way, through `.editorconfig`. Raise one:

```ini
[*.cs]
dotnet_diagnostic.WM2001.severity = warning
```

Silence one:

```ini
[*.cs]
dotnet_diagnostic.WM1005.severity = none
```

Silence one at a single site:

```csharp
#pragma warning disable WM1001
Option<int> option = Option.Some(0);
#pragma warning restore WM1001
```

Scope a rule to part of your solution by putting the `.editorconfig` in that
directory. Test projects are a common case — `WM2001` on `Unwrap` is noisier in a
test than in production code, where an unwrap that throws fails the test anyway.

You cannot remove the analyzer while keeping the library, because it ships in the
same package. `.editorconfig` is the way to turn rules off.
