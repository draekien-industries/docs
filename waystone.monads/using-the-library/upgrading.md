# Upgrading

## v1.x - v2.x

### Renamed `Bind` to `Try`

The `Option.Bind` and `Result.Bind` factory methods have been renamed to `Try` to better adhere to functional programming concepts.

`Bind` is often associated with `FlatMap`, a way of composing functions together in a pipeline. This renaming removes the confusion.

```diff
-Option.Bind(() => CreateSome(), ex => Console.WriteLine(ex));
+Option.Try(() => CreateSome());

-Result.Bind(() => CreateOk(), ex => HandleEx(ex));
+Result.Try(() => CreateOk(), ex => HandleEx(ex));
```

### Introduced `MonadsGlobalConfig`

This configuration allows the setting of a global error logger that will be invoked whenever an exception is caught and handled by the library.

```csharp
MonadsGlobalConfig.UseExceptionLogger((ex) => {
    Console.WriteLine(ex); // replace with your logger's log method, e.g. serilog
});
```

### Removed local error handling for Option

Removed the local handle error callback on the `Option.Try` methods in favour of the `MonadsGlobalConfig`.

```diff
// program.cs
+MonadsGlobalConfig.UseExceptionLogger((ex) => {
+    Console.WriteLine(ex); // replace with your logger's log method, e.g. serilog
+});

// usage
-Option.Bind(() => CreateSome(), ex => Console.WriteLine(ex));
+Option.Try(() => CreateSome());
```

## v2.x - v3.x

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

## v3.x - v4.x

Replaced `MonadsGlobalConfig` with `MonadOptions` to enable better DX when configuring library behaviours.

```diff
-MonadsGlobalCongig.UseExceptionLogger(...);
+MonadOptions.Configure(options => {
+    options.UseExceptionLogger(...)
+           .UseErrorCodeFactory(...)
+    options.FallbackErrorCode = "error.unknown";
+    options.FallbackErrorMessage = "An unknown error has occurred.";
+});
```

The `IErrorCodeFormatter<T>` interface has been removed in favour of `ErrorCodeFactory` so that it can be applied once during your app's life-cycle, instead of during each invocation of the error code creation methods. You can override the default formatting by inheriting `ErrorCodeFactory` and invoking `MonadOptions.UseErrorCodeFactory()`.

```diff
-class MyErrorCodeFormatter<MyErrorCodeEnum> : IErrorCodeFormatter<MyErrorCodeEnum>;
-ErrorCode.FromEnum(MyErrorCodeEnum.BadRequest, new MyErrorCodeFormatter<MyErrorCodeEnum>());
+class MyErrorCodeFactory : ErrorCodeFactory;
+MonadOptions.Configure(options => options.UseErrorCodeFactory(new MyErrorCodeFactory()));
```

## v4.x - v5.x

Simplified the API for chaining async extension methods on `Option<T>` and `Result<TOk, TErr>` , removing support for async lambdas that returned a `ValueTask<T>`. Additionally, optimized the return type of specific extensions that would return either a synchronous value or a Task to have a type of `ValueTask`.

```diff
-Task<string> output = result.MatchAsync(async x => await doWork(x), e => e.ToString());
+ValueTask<string> output = result.MatchAsync(async x => await doWork(x), e => e.ToString());
```
