---
description: Teach your coding agent to write Waystone.Monads the way it is meant to be written.
icon: robot
layout:
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

# Agent skills

Coding agents reach for `null` checks and `try`/`catch` by default, because that
is what most of the C# they learned from does. Give an agent this library and it
will often write `if (option.IsSome) { option.Unwrap(); }` — code that compiles,
passes review at a glance, and throws away everything you adopted the library
for.

The `waystone-monads` **skill** fixes that. It is a document the agent loads when
it notices you are writing `Option<T>` or `Result<TOk, TErr>`, and it teaches the
composition style, the traps, and the analyzer rules that catch them.

You do not need it to use the NuGet package. Install it if you want an agent to
write idiomatic code without being told how every time.

## Install it

Pick whichever suits the agent you use.

### As a Claude Code plugin

Run both commands inside Claude Code:

```
/plugin marketplace add draekien-industries/waystone-dotnet
/plugin install waystone-dotnet@waystone-dotnet
```

The plugin name and the marketplace name are both `waystone-dotnet`, which is why
it appears twice. Installing the plugin gets you the skill plus anything else the
plugin gains later.

### As a standalone skill

Use [`npx skills`](https://agentskills.io) if you use a different agent, or if you
want the skill without the plugin:

```sh
npx skills add draekien-industries/waystone-dotnet --skill waystone-monads
```

That installs into the current project. Add `-g` to install it for every project
instead. Run the same command with `--list` and no `--skill` to see everything the
repository offers.

## What it teaches

The skill is built from this library's own analyzer rules and sample project
rather than from general advice about monads. It covers:

| Area | What the agent learns |
| --- | --- |
| Composition | Chain `Map`, `AndThen`, `Filter` and `OrElse`; collapse once at the end |
| Traps | Nested `Match`, `IsSome` guarding an `Unwrap`, `null` on a monad, a discarded `Result` |
| Choosing a type | When absence is an `Option`, when failure is a `Result`, and when to throw instead |
| Nesting | What `Result<Option<T>, E>` means, and how `Transpose` moves between the two shapes |
| Async | Keeping a chain intact across an `await` rather than breaking it into locals |
| Error codes | Building failures through the generated `{EnumName}Catalog.Errors` factories |
| Rust habits | Which reflexes carry over, and which ones hurt in shipped C# |

You do not invoke it. The agent loads it when the work matches, so it applies to
code you ask for in passing as much as to a task you set up deliberately.

## It does not replace the analyzer

The skill and the analyzer solve two halves of one problem, and you want both.

* The **skill** shapes what an agent writes. It has no way to check the result.
* The **analyzer** checks what anyone wrote, whether an agent or a person. It
  cannot write anything.

An agent following the skill still produces code the analyzer reports, and a
clean build is the bar. See
[Analyzers](../analyzers/README.md) for what
each rule means.

{% hint style="info" %}
**Update it the way you update a package.** The skill describes the library at a
point in time, so a stale copy will teach an API that has moved on. Run
`npx skills update` for a standalone install, or
`/plugin update waystone-dotnet` inside Claude Code — that one names the plugin
and needs a restart to take effect.
{% endhint %}
