# v2.x to v3.x

{% hint style="info" %}
**This page describes a historical upgrade.** Its replacement code was correct for
`3.x`, and later majors have removed some of the API it names. Land on `3.x` first if you
are following it literally, then read the pages after this one in order.
[Deprecations](deprecations.md) lists what has gone since.
{% endhint %}

Renamed async overloads for methods to have the `Async` suffix

```diff
-await option.Map(...);
+await option.MapAsync(...);
```

Fundamentally changed how async overloads are declared. They are now extension methods instead of existing in the Option/Result instance. This enables method chaining on `Task<Result<T, E>>` and `Task<Option<T>>`.

```diff
-var a = await option.Map(...);
-var b = await a.Map(...);
+var c = await option.MapAsync(...).MapAsync(...);
```

Moved async overloads into `Results.Extensions` and `Options.Extensions` namespaces.

```diff
using Waystone.Monads.Results;
using Waystone.Monads.Options;
+using Waystone.Monads.Results.Extensions;
+using Waystone.Monads.Options.Extensions;
```

