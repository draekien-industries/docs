# v4.x to v5.x

Simplified the API for chaining async extension methods on `Option<T>` and `Result<TOk, TErr>` , removing support for async lambdas that returned a `ValueTask<T>`. Additionally, optimized the return type of specific extensions that would return either a synchronous value or a Task to have a type of `ValueTask`.

```diff
-Task<string> output = result.MatchAsync(async x => await doWork(x), e => e.ToString());
+ValueTask<string> output = result.MatchAsync(async x => await doWork(x), e => e.ToString());
```
