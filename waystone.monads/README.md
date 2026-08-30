---
description: Keep exceptions exceptional
icon: hand-wave
cover: https://gitbookio.github.io/onboarding-template-images/header.png
coverY: 0
layout:
  cover:
    visible: true
    size: full
  title:
    visible: true
  description:
    visible: true
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
---

# Welcome

Waystone.Monads gives C# two types that say what your code actually means.

* `Option<T>` — the value might not be there.
* `Result<T, E>` — the work might fail.

Both are ordinary C# types. There is no framework to adopt and nothing to wire
up. You add a package, you change a return type, and the compiler starts telling
you about the cases you used to find at runtime.

## Who this is for

You are writing C# and you are tired of two things: `null` reaching places it
should not, and exceptions being used for outcomes that are not exceptional.

The library replaces both with values you can return, pass around, and compose.
Absence and failure stop being surprises hidden inside a method body, and start
being part of the signature you already read.

If you have used `Option` and `Result` in Rust or F#, you already know the shape.
If you have not, [Why monads](start-here/why-monads.md) walks through it with no
prior knowledge assumed.

## Where to go next

| If you want to | Go to |
| --- | --- |
| Install the package and see both types work | [Quickstart](start-here/quickstart.md) |
| Understand why this beats `null` and `try`/`catch` | [Why monads](start-here/why-monads.md) |
| Teach your coding agent to write it properly | [Agent skills](start-here/agent-skills.md) |

## What comes in the box

Installing `Waystone.Monads` gets you the two types, a Roslyn analyzer, and a
source generator. You configure none of it. The analyzer flags the mistakes
people make with these types, and the generator turns an enum into a set of
error codes. See [Analyzer rules](analyzers/README.md) and
[Generated error codes](using-the-library/generated-error-codes.md).

Optional packages sit beside the library — Shouldly assertions, LINQ query
syntax, JSON converters, and more. None is required, and none changes how
`Waystone.Monads` behaves. See
[Companion packages](companion-packages/README.md).

## Links

* [Source on GitHub](https://github.com/draekien-industries/waystone-dotnet) — the `Waystone.Monads` package lives in `src/Waystone.Monads`
* [Waystone.Monads on NuGet](https://www.nuget.org/packages/Waystone.Monads)
* [Report an issue](https://github.com/draekien-industries/waystone-dotnet/issues) — for the library or for these docs
