---
description: >-
  Packages that add to Waystone.Monads itself. None is required, and none
  changes how the library behaves.
icon: puzzle-piece
---

# Add-ons

These extend `Waystone.Monads` itself. They keep their own `Waystone.Monads.*`
namespaces, ship from the same repository on the same version number, and are purely
additive. You install one because you want what it adds, not because an upgrade made
you.

Packages that bridge to a library from somewhere else are under
[Integrations](integrations.md).

## Pick a package

| If you need to | Install | Page |
| --- | --- | --- |
| Parse untrusted input into a domain type | `Waystone.Monads.Schemas` | [Schemas](schemas.md) |
| Write `from … select` over an `Option` or a `Result` | `Waystone.Monads.Linq` | [LINQ](linq.md) |
| See the exceptions the library swallows | `Waystone.Monads.Extensions.Logging` | [Logging](logging.md) |

## Start with Schemas

[Schemas](schemas.md) is the largest of the three, and the one most codebases have a
use for. It parses untrusted input into a type you could not have built without
passing, and reports every failure at once rather than the first. The
[Schemas guide](../guides/schemas.md) builds one end to end and explains why you
would want to.

## None of them changes the library

No package here changes the behaviour of anything in `Waystone.Monads`.

* Two add vocabulary. Remove [Schemas](schemas.md) or [LINQ](linq.md) and the code
  that used it stops compiling, and nothing else moves.
* One changes what reaches your logs. [Logging](logging.md) reports the exceptions
  the library already swallowed. It does not stop them being swallowed.

[Observability](../guides/observability.md) covers the signals that need no package
at all.
