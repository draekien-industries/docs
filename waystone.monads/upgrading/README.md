---
description: >-
  What changes between major versions, what you have to do about it, and what is
  on its way out.
icon: arrow-up-right-dots
---

# Overview

## What version are you on now?

| You are on | Read |
| --- | --- |
| 6.x | [From v6.x](v7/from-v6.md) |
| 5.x | [From v5.x](v7/from-v5.md) — the combined path, not the two pages in sequence |
| 4.x or older | [Older upgrades](older/README.md), then come back here |
| 7.0.0 already | [Deprecations](deprecations.md) — what is going away next |

**If you are skipping v6, take the combined page.** The two sets of changes interact,
and one v6 change had a warning that only existed in 5.5.x — so a reader coming from
5.4 or earlier gets it with no signal at all.

## The three entries in this group

* **[Upgrade to v7](v7/README.md)** — the current major. Start with
  [Breaking changes](v7/breaking-changes.md) if you want to know how bad it is before
  you pick a path.
* **[Deprecations](deprecations.md)** — API that still works today but is going away,
  with the version that removes it. Read this before you upgrade, not after.
* **[Older upgrades](older/README.md)** — every hop from v1 to v6, newest first.

## Every upgrade page has an agent prompt

Each of the seven upgrade pages opens with a collapsed block holding a prompt for that
hop, ready to copy into Claude Code or a similar tool. It is collapsed by default, so
a reader doing the upgrade by hand scrolls past one line.

Three rules apply to every one of them. They are in each prompt, and they are worth
knowing before you run one:

* Never suppress a diagnostic, add a pragma to disable one, or add a null-forgiving
  `!` to make an error go away. Every diagnostic in an upgrade has a real fix.
* Do not change behaviour to make a test pass. A test that fails after an upgrade is
  either a silent change you missed or a real finding.
* Do not reformat, rename, or refactor anything the upgrade does not require.

## What a prompt cannot do for you

{% hint style="warning" %}
**Step 1 of the v7 prompts needs your judgement.** The prompt asks the agent to find and report every
silent change, and to decide only the ones with a mechanical answer.

Two have no mechanical answer:

* **A projection that can return null.** Whether the null meant "absent" or was never
  supposed to happen decides whether you convert to `Option.FromNullable` or fix the
  caller. Only someone who knows the domain can say.
* **Coming from 5.x, an `IsNone` branch on a value-type option.** `Option.Some(0)` now
  gives you a `Some`. Whether that branch was standing in for "zero" needs someone who
  knows what the code means. See
  [Silent change 3](older/v5-to-v6.md#silent-change-3-some-accepts-value-type-defaults).

Expect the agent to bring these to you. If it decided them on its own, that is a
finding.
{% endhint %}

## If you would rather not use an agent

Every step in a prompt maps to a section on the upgrade page it sits on, and
[Breaking changes](v7/breaking-changes.md) is the same v7 information as a reference
table naming the diagnostic each break produces.
