---
description: >-
  What changes between major versions, what you have to do about it, and what is
  on its way out.
---

# Overview

This group holds two kinds of page.

* **[Deprecations](deprecations.md)** — API that still works today but is going away, with the version that removes it. Read this before you upgrade, not after.
* **The upgrade pages** — one per major version jump, listing what breaks and what to change.

Pick the page for the jump you are making. If you are skipping versions, read each
page in order — the changes stack.

| Upgrade | What you have to change |
| --- | --- |
| [v1.x to v2.x](v1-to-v2.md) | `Bind` becomes `Try`, and error logging moves to global config |
| [v2.x to v3.x](v2-to-v3.md) | Async overloads gain an `Async` suffix and move to extension namespaces |
| [v3.x to v4.x](v3-to-v4.md) | `MonadsGlobalConfig` becomes `MonadOptions` |
| [v4.x to v5.x](v4-to-v5.md) | Some async extensions return `ValueTask` instead of `Task` |
| [v5.x to v6.x](v5-to-v6.md) | Three changes that keep compiling and change what your code does |
| [v6.x to v7.0.0](v6-to-v7.md) | Three more silent changes, and eleven that break the build |
| [v5.x to v7.0.0](v5-to-v7.md) | Both of the above at once, and the order to do them in |

**If you are skipping v6, take the combined page rather than reading the other two in
sequence.** The two sets of changes interact, and one v6 change had a warning that only
existed in 5.5.x — so a reader coming from 5.4 or earlier gets it with no signal at all.

Two supporting pages sit beside these:

* [Every v7 break](v7-breaks.md) — the same v7 information as a reference table, naming
  the compiler diagnostic each break produces.
* [Upgrade with an agent](v7-agent-prompt.md) — one prompt covering both v7 paths, and
  the two decisions it deliberately leaves to you.

The v5.x to v6.x and v6.x to v7.0.0 pages are the ones to read closely. Older upgrades on
this list break the build when you get them wrong. Those two have changes that do not —
your code keeps compiling and quietly means something different.
