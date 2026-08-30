---
description: >-
  The current major. Three changes keep compiling and change what your code
  does; the rest break the build.
icon: seven
---

# Upgrade to v7

7.0.0 is the current major. It is the largest release the library has had, and most
of it breaks loudly — the compiler finds it for you.

The part that does not is what to read first.

## At a glance

| | |
| --- | --- |
| Changes that keep compiling | 3 from v6, 6 if you are coming from v5 |
| Changes that break the build | 11 |
| Removals with no warning release | 1, in `Waystone.Monads.FluentValidation` |
| Analyzer rules added | `WM2022` |
| Analyzer rules removed | `WM2010` |

## Pick your path

| You are on | Read |
| --- | --- |
| 6.x | [From v6.x](from-v6.md) |
| 5.x | [From v5.x](from-v5.md) |
| 4.x or older | [Older upgrades](../older/README.md) up to 5.x first, then [From v5.x](from-v5.md) |

## The supporting pages

* **[Breaking changes](breaking-changes.md)** — every v7 break as a reference table,
  naming the compiler diagnostic each one produces, and whether a code fix exists.
  Read this if you want the size of the job before you pick a path.
* **[Deprecations](../deprecations.md)** — what 7.0.0 marks for removal in 8.0.0.

Both v7 pages open with a collapsed agent prompt for that path.
