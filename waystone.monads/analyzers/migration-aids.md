---
description: >-
  The WM3xxx rules. They ship off, and you turn them on while you convert a
  codebase to the library.
icon: arrow-right-arrow-left
---

# Migration aids

These two rules ship **off**. They report on code that has not adopted the library
yet, so in most codebases they fire everywhere. Turn them on while you convert, then
turn them off again.

| ID | What it reports |
| --- | --- |
| [`WM3001`](#wm3001) | A member that returns a nullable type, where `Option<T>` would make the absent case impossible to ignore |
| [`WM3002`](#wm3002) | A `throw`, where returning `Result<TOk, Error>` would state the failure in the signature |

Enable one in your `.editorconfig`:

```ini
[*.cs]
dotnet_diagnostic.WM3001.severity = suggestion
dotnet_diagnostic.WM3002.severity = suggestion
```

`WM3002` ignores the throws a `Result` would not improve: `ArgumentException` and
its subtypes, `NotImplementedException`, `NotSupportedException`,
`ObjectDisposedException`, a bare `throw;` rethrow, and any throw inside a lambda
you pass to `Option.Try` or `Result.Try`.

## WM3001

**A nullable return leaves the absent case easy to ignore.** `Option<T>` makes the
caller acknowledge it. Off by default, because in a codebase that has not adopted the
library this fires on nearly every member.

```diff
-User? FindUser(int id);
+Option<User> FindUser(int id);
```

`WM2012` is the narrower, on-by-default version, for a type that already uses `Option`
elsewhere.

## WM3002

**A `throw` states a failure nowhere in the signature.** Returning
`Result<TOk, Error>` puts it there, where a caller cannot miss it. Off by default,
because it fires on every throw in a codebase that has not adopted `Result`.

```diff
-if (!found) throw new InvalidOperationException("no such user");
+if (!found) return new Error("NoSuchUser", "no such user");
```

It skips the throws listed above, which a `Result` would not improve. `WM2003` is the
on-by-default version, for a member that already returns `Result`.

