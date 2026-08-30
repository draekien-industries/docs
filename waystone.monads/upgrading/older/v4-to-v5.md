# v4.x to v5.x

{% hint style="info" %}
**This page describes a historical upgrade.** Its replacement code was correct for
`5.x`, and later majors have removed some of the API it names. Land on `5.x` first if you
are following these steps, then read the pages after this one in order.
[Deprecations](../deprecations.md) lists what has gone since.
{% endhint %}

<details>

<summary>Upgrade with an agent — copy this prompt</summary>

Pointed at your solution, in Claude Code or a similar tool.

```text
Upgrade this solution from Waystone.Monads v4 to v5. Report what you changed.

1. Build and capture every CS0029 and CS1503 involving Task and ValueTask on a
   Waystone async extension. In v5 several of those extensions return ValueTask<T>
   where they returned Task<T>.

2. For each one, prefer changing the declared type of the local or field to
   ValueTask<T>, or simply awaiting the value. Add .AsTask() only where the value is
   passed to something that demands a Task — Task.WhenAll is the usual case.
   .AsTask() allocates; do not reach for it first.

3. Find every async lambda passed to a Waystone async extension that returns a
   ValueTask<T>. v5 no longer accepts those. Change the lambda to return T or Task<T>.

4. Build again and report anything left. Do not suppress a diagnostic or add a
   null-forgiving ! to make one go away.
```

</details>

Simplified the API for chaining async extension methods on `Option<T>` and `Result<TOk, TErr>` , removing support for async lambdas that returned a `ValueTask<T>`. Additionally, optimized the return type of specific extensions that would return either a synchronous value or a Task to have a type of `ValueTask`.

```diff
-Task<string> output = result.MatchAsync(async x => await doWork(x), e => e.ToString());
+ValueTask<string> output = result.MatchAsync(async x => await doWork(x), e => e.ToString());
```
