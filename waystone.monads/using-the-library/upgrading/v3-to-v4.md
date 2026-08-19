# v3.x to v4.x

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

