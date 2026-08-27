# v4.x to v5.x

{% hint style="info" %}
**This page describes a historical upgrade.** Its replacement code was correct for
`5.x`, and later majors have removed some of the API it names. Land on `5.x` first if you
are following it literally, then read the pages after this one in order.
[Deprecations](deprecations.md) lists what has gone since.
{% endhint %}

Simplified the API for chaining async extension methods on `Option<T>` and `Result<TOk, TErr>` , removing support for async lambdas that returned a `ValueTask<T>`. Additionally, optimized the return type of specific extensions that would return either a synchronous value or a Task to have a type of `ValueTask`.

```diff
-Task<string> output = result.MatchAsync(async x => await doWork(x), e => e.ToString());
+ValueTask<string> output = result.MatchAsync(async x => await doWork(x), e => e.ToString());
```
