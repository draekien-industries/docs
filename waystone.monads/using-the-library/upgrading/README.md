---
description: What changes between major versions, and what you have to do about it.
---

# Upgrading

Pick the page for the jump you are making. If you are skipping versions, read each
page in order — the changes stack.

| Upgrade | What you have to change |
| --- | --- |
| [v1.x to v2.x](v1-to-v2.md) | `Bind` becomes `Try`, and error logging moves to global config |
| [v2.x to v3.x](v2-to-v3.md) | Async overloads gain an `Async` suffix and move to extension namespaces |
| [v3.x to v4.x](v3-to-v4.md) | `MonadsGlobalConfig` becomes `MonadOptions` |
| [v4.x to v5.x](v4-to-v5.md) | Some async extensions return `ValueTask` instead of `Task` |
| [v5.x to v6.x](v5-to-v6.md) | Three changes that keep compiling and change what your code does |

The v5.x to v6.x page is the one to read closely. Every other upgrade on this list
breaks the build when you get it wrong. That one has three changes that do not —
your code keeps compiling and quietly means something different.

See [Deprecations](../deprecations.md) for API that is on its way out but still
works today.
