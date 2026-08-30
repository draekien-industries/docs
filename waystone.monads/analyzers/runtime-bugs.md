---
description: >-
  The WM1xxx rules. Each one marks code that compiles and then throws or quietly
  does the wrong thing at run time.
icon: bug
---

# Runtime bugs

These rules are warnings. Each one marks code that compiles and then misbehaves.

| ID | What it reports |
| --- | --- |
| [`WM1001`](#wm1001) | `Option.Some` given a value that is provably null, which always throws |
| [`WM1002`](#wm1002) | `null` written where an `Option` or `Result` belongs |
| [`WM1003`](#wm1003) | `default` on an `Option` or `Result`, which is null rather than the empty case |
| [`WM1005`](#wm1005) | `Option.Some` given a value the compiler treats as maybe-null |
| [`WM1006`](#wm1006) | A discarded `Result`, so the failure vanishes |
| [`WM1008`](#wm1008) | An `Option` or `Result` declared nullable, which adds a third state |
| [`WM1011`](#wm1011) | An async delegate passed to a synchronous method, so the task is never awaited |

The gaps are real. `WM1004`, `WM1007`, `WM1009` and `WM1010` shipped in 5.x and
were removed in 6.0.0. A removed id is never reused, so a suppression you wrote
for one of them is dead but harmless.

## WM1001

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

## WM1002

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

## WM1003

**The default of an `Option` or `Result` is null.** `default(Option<int>)` gives you
no empty option. It gives you `null`, for the reason `WM1002` explains.

```diff
-Result<int, string> result = default;
+Result<int, string> result = Result.Err<int, string>("not set");
```

**Quick fix for `Option<T>`:** use `Option.None<T>()`.

## WM1005

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

## WM1006

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

## WM1008

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

## WM1011

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

## What it does and does not report

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
[Silent change 1](../upgrading/older/v5-to-v6.md#silent-change-1-try-with-an-async-factory).

**No quick fix, deliberately.** Renaming to the `Async` sibling leaves you with an
unawaited task, and no fixer can decide where your `await` belongs.

