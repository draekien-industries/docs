# v2.x to v3.x

{% hint style="info" %}
**This page describes a historical upgrade.** Its replacement code was correct for
`3.x`, and later majors have removed some of the API it names. Land on `3.x` first if you
are following these steps, then read the pages after this one in order.
[Deprecations](../deprecations.md) lists what has gone since.
{% endhint %}

<details>

<summary>Upgrade with an agent — copy this prompt</summary>

Pointed at your solution, in Claude Code or a similar tool.

```text
Upgrade this solution from Waystone.Monads v2 to v3. Report what you changed.

1. Add the Async suffix to every awaited Waystone call — Map becomes MapAsync, AndThen
   becomes AndThenAsync, and so on. The synchronous names no longer have async
   overloads.

2. Add using Waystone.Monads.Options.Extensions; and
   using Waystone.Monads.Results.Extensions; to every file that now fails to resolve
   one of those names. The async overloads are extension methods in v3, not instance
   members.

3. Collapse the intermediate awaits the old shape needed. Because the extensions apply
   to Task<Option<T>> and Task<Result<TOk, TErr>>, several steps chain from one await
   instead of one await each.

4. Build and report anything left.
```

</details>

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

