---
description: Every major version hop before v7, newest first.
icon: box-archive
---

# Older upgrades

One page per major version hop, newest first. If you are walking several hops, read
up this list from where you are.

| Upgrade | What you have to change |
| --- | --- |
| [v5.x to v6.x](v5-to-v6.md) | Three changes that keep compiling and change what your code does |
| [v4.x to v5.x](v4-to-v5.md) | Some async extensions return `ValueTask` instead of `Task` |
| [v3.x to v4.x](v3-to-v4.md) | `MonadsGlobalConfig` becomes `MonadOptions` |
| [v2.x to v3.x](v2-to-v3.md) | Async overloads gain an `Async` suffix and move to extension namespaces |
| [v1.x to v2.x](v1-to-v2.md) | `Bind` becomes `Try`, and error logging moves to global config |

Each page opens with a collapsed agent prompt for that hop.

Every page here describes an upgrade that landed on a version older than 7.0.0, so
the replacement code it names may itself have been removed since.
[Deprecations](../deprecations.md) is the current list.

Going to 7.0.0 from 5.x? Do not read `v5.x to v6.x` and then the v7 page. Take
[From v5.x](../v7/from-v5.md), which covers both at once and gives the order.
