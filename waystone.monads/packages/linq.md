---
description: C# query syntax over Option and Result, with no change in behaviour.
---

# LINQ

`Waystone.Monads.Linq` — C# query syntax over `Option` and `Result`.

## What it adds

`Select`, `SelectMany` and `Where`, under those names, so a `from … select` query
compiles over an `Option<T>` or a `Result<TOk, TErr>`.

```csharp
using Waystone.Monads.Linq;

Option<Quote> quote =
    from customer in FindCustomer(id)
    from address in customer.PostalAddress
    from rate in RateFor(address)
    select Price(customer, rate);
```

Every step there returns an `Option`. If any one of them is `None`, the query stops and
the result is `None` — you never write the check.

That is the same thing `AndThen` does, spelled differently:

```csharp
Option<Quote> quote = FindCustomer(id)
    .AndThen(customer => customer.PostalAddress
        .AndThen(address => RateFor(address)
            .Map(rate => Price(customer, rate))));
```

Pick whichever reads better to you. Three or more steps that each need the earlier
values is where query syntax wins, because the nesting flattens out.

## When to reach for it

Reach for it when a chain of `AndThen` calls has grown enough that the names of the
intermediate values stop being obvious. A `from … select` query names each step.

Skip it for a single step. `option.Map(x => x + 1)` is already shorter than the
query that does the same thing.

## Install it

```
dotnet add package Waystone.Monads.Linq
```

Then add one line to the file that needs it:

```csharp
using Waystone.Monads.Linq;
```

That is the whole opt-in. It works alongside `using System.Linq;` — the names do not
collide, because these are extensions on `Option<T>` and `Result<TOk, TErr>` rather
than on `IEnumerable<T>`.

The package is separate so the core library stays free of the LINQ names. Nothing in
`Waystone.Monads` gains or loses a member when you install it.

## What each clause maps to

| You write | It calls | Which is |
| --- | --- | --- |
| `select` | `Select` | `Map` |
| a second and later `from` | `SelectMany` | `AndThen`, then `Map` |
| `where` | `Where` | `Filter` |

Every member forwards to the core member and adds no behaviour. Two spellings, one
implementation. Use `Map` in a method-syntax chain and `select` in a query — do not
mix them in the same expression.

## Result has no `where` clause

This is the one place a query over a `Result` is poorer than a query over an `Option`,
and it is a decision rather than an omission.

```csharp
// Does not compile.
Result<int, string> positive =
    from n in GetResult()
    where n > 0
    select n;
```

Discarding an Ok value would have to invent the error that replaces it, and a signature
taking an error factory is not the one query syntax binds a `where` clause to. So there
is no `Where` on `Result` at all.

Two ways round it:

* Filter before you enter the query.
* Use `Filter` on an `Option`, then `OkOr` to get back to a `Result` with an error you
  chose.

`Option` has `Where`, and it behaves as you would expect — a value that fails the
predicate becomes `None`.

## Every step must share one error type

A query over `Result` threads one `TErr` through the whole chain, because that is what
`AndThen` does. A step that fails with a different error type has to be mapped onto the
shared one first, with `MapErr`, before it can join the query.

There is no LINQ name for projecting the error half. `MapErr` is the only spelling.

## There are no async shapes

Query expressions cannot await, so there is no `SelectAsync` and no
`SelectManyAsync`. Async chaining is already covered by `MapAsync` and `AndThenAsync`
in the core package — see [Async](../guides/async.md).

## This is not `AsEnumerable`

Both let an `Option` meet LINQ. They do opposite things.

| | Stays a monad? | Then what |
| --- | --- | --- |
| `Waystone.Monads.Linq` | **Yes** | `Select` on an `Option<T>` gives you an `Option<TOut>` |
| `AsEnumerable()` | **No** | Gives you an `IEnumerable<T>` of zero or one items, and you are in `System.Linq` from there |

`AsEnumerable` is the way *out*, and it loses things on the way. On a `Result` it drops
the error — an `Err` becomes an empty sequence, with nothing left to say why. Use it
when you genuinely want a sequence, usually to concatenate several options together.

This package is the way to *stay in*. Nothing is materialised, nothing is enumerated,
and the error survives.

## It costs no more than the method it forwards to

The three-argument `SelectMany` (the one a multi-clause `from` binds to) threads both
of your delegates through the core state-passing overloads using `static` lambdas, so it
captures nothing. A query does not allocate a closure that the equivalent `AndThen`
chain would avoid.

## One naming wrinkle

If you call these as methods with named arguments, note that the factory parameters
follow this library's chain-step naming (`optionFactory` and `resultFactory`) while
`resultSelector` keeps the name LINQ gives it. Positional calls and query syntax are
unaffected.

## What it does not do

* It adds no async shapes. C# query syntax has no `await`, so there is nothing to
  bind to.
* It gives `Result` no `where` clause, because filtering out an `Ok` has no error
  to fall back on.
* It changes no behaviour. Every clause forwards to a method that already existed.
