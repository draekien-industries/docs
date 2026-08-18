---
description: >-
  API that still works today but will be removed in the next major release, and
  what to use instead.
---

# Deprecations

## What this page is for

Everything listed here still compiles and still behaves exactly as it did before.
Each entry is marked `[Obsolete]`, so your build reports a `CS0618` warning when
you call it.

Migrate before you upgrade to **v6.0.0**, where these members are deleted.

{% hint style="info" %}
Packages in the Waystone family share one version number, so a v6.0.0 of
`Waystone.Monads` means a v6.0.0 of every package.
{% endhint %}

## Option.FlatMap

**Deprecated in:** 5.4.0 · **Removed in:** 6.0.0 · **Replacement:** `AndThen`

| Deprecated | Replacement |
| --- | --- |
| `Option<T>.FlatMap<TOut>(Func<T, Option<TOut>>)` | `Option<T>.AndThen<TOut>(Func<T, Option<TOut>>)` |
| `FlatMapAsync` on `Option<T>`, `Task<Option<T>>` and `ValueTask<Option<T>>` | `AndThenAsync` |

### How to migrate

Rename the call. The parameters, the behaviour and the return type do not change.

```diff
-Option<int> option = Find(id).FlatMap(Parse);
+Option<int> option = Find(id).AndThen(Parse);
```

`WM2014` reports every call site for you, and its quick fix does the rename.

### Why this changed

`Result<TOk, TErr>` already spelled this operation `AndThen`, so one library had two
names for one idea. Rust, which both types follow, calls it `and_then`. `AndThen` is
the name that agrees with the rest of the library.

`FlatMap` stays as a member that forwards to `AndThen`, so nothing changes at run
time until v6.0.0 removes it.

## Try overloads that accept an async factory

**Deprecated in:** 5.2.0 · **Removed in:** 6.0.0 · **Replacement:** `TryAsync`

| Deprecated | Replacement |
| --- | --- |
| `Option.Try<T>(Func<Task<T>>, …)` | `Option.TryAsync<T>(Func<Task<T>>, …)` |
| `Result.Try<TOk, TErr>(Func<Task<TOk>>, Func<Exception, TErr>, …)` | `Result.TryAsync<TOk, TErr>(Func<Task<TOk>>, Func<Exception, TErr>, …)` |

### How to migrate

Rename the call. The parameters, the behaviour and the return type do not change.

```diff
-Option<int> option = await Option.Try(() => FetchAsync());
+Option<int> option = await Option.TryAsync(() => FetchAsync());

-Result<int, string> result = await Result.Try(() => FetchAsync(), ex => ex.Message);
+Result<int, string> result = await Result.TryAsync(() => FetchAsync(), ex => ex.Message);
```

### Why this changed

A lambda whose body is a `throw` expression converts to both `Func<T>` and
`Func<Task<T>>`. When both overloads are called `Try`, the compiler cannot choose
between them, and you have to declare the delegate type to break the tie:

```csharp
// ambiguous, will not compile
Result<int, Error> result = Result.Try<int>(() => throw new InvalidOperationException());

// what you had to write instead
Result<int, Error> result = Result.Try<int>(new Func<int>(() => throw new InvalidOperationException()));
```

Giving the async overloads their own name removes the ambiguity, so the call site
above compiles as written.

### What is not affected

The synchronous overloads keep the `Try` name and are not deprecated:

- `Option.Try<T>(Func<T>, …)`
- `Result.Try<TOk, TErr>(Func<TOk>, Func<Exception, TErr>, …)`
- `Result.Try<TOk>(Func<TOk>, …)`

{% hint style="info" %}
`Result.TryAsync<TOk>(Func<Task<TOk>>, …)` — the overload that defaults the error
type to `Error` — was introduced as `TryAsync` and never had a `Try` spelling.
There is nothing to migrate.
{% endhint %}

See [Async](async.md) for the full async surface, and
[Upgrading](upgrading.md) for the upgrades that have already shipped.
