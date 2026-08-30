# v3.x to v4.x

{% hint style="info" %}
**This page describes a historical upgrade.** Its replacement code was correct for
`4.x`, and later majors have removed some of the API it names — `UseExceptionLogger`,
`ErrorCode.FromEnum` and the `ErrorCodeFactory.FromEnum` override are all gone in
`7.0.0`. `UseErrorCodeFactory` itself remains, but it can no longer shape enum codes.
Land on `4.x` first if you are following these steps, then read the pages in order.
[Deprecations](deprecations.md) lists what has gone since.
{% endhint %}

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

