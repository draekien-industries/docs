---
icon: two
---

# v1.x to v2.x

{% hint style="info" %}
**This page describes a historical upgrade.** Its replacement code was correct for
`2.x`, and later majors have removed some of the API it names. Land on `2.x` first if you
are following these steps, then read the pages after this one in order.
[Deprecations](../deprecations.md) lists what has gone since.
{% endhint %}

<details>

<summary>Upgrade with an agent — copy this prompt</summary>

Pointed at your solution, in Claude Code or a similar tool.

```text
Upgrade this solution from Waystone.Monads v1 to v2. Report what you changed.

1. Rename every Option.Bind and Result.Bind factory call to Try. Parameters and
   behaviour are otherwise unchanged.

2. Delete the exception-handling callback from every Option.Try call. Option.Try no
   longer takes one. Result.Try still does — leave those.

3. Add one MonadsGlobalConfig.UseExceptionLogger call at start-up, pointed at the
   solution's existing logger, so the exceptions the library handles are still
   reported. Show me where you put it.

4. Build and report anything left.
```

</details>

## Renamed `Bind` to `Try`

The `Option.Bind` and `Result.Bind` factory methods have been renamed to `Try` to better adhere to functional programming concepts.

`Bind` is often associated with `FlatMap`, a way of composing functions together in a pipeline. This renaming removes the confusion.

```diff
-Option.Bind(() => CreateSome(), ex => Console.WriteLine(ex));
+Option.Try(() => CreateSome());

-Result.Bind(() => CreateOk(), ex => HandleEx(ex));
+Result.Try(() => CreateOk(), ex => HandleEx(ex));
```

## Introduced `MonadsGlobalConfig`

This configuration allows the setting of a global error logger that will be invoked whenever an exception is caught and handled by the library.

```csharp
MonadsGlobalConfig.UseExceptionLogger((ex) => {
    Console.WriteLine(ex); // replace with your logger's log method, e.g. serilog
});
```

## Removed local error handling for Option

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

