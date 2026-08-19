---
description: >-
  A Some may hold 0, false and Guid.Empty in v6. Your code keeps compiling and
  changes meaning, so read this before you upgrade.
---

# v5.x to v6.x

{% hint style="warning" %}
v6.0.0 has not shipped. This page exists so you can start looking for the
affected code now. The analyzer rule `WM1010` ships in 5.5.0 for the same
reason.
{% endhint %}

## What changes

In v5, a `Some` cannot hold the default of its type. `Option.Some(0)` throws, and
`Option<int> x = 0;` gives you `None`.

In v6, only `null` is rejected. `Option.Some(0)` gives you a `Some` holding `0`,
and so does `Option<int> x = 0;`.

We made this change because `Option<int>` could not represent part of its own
domain. Zero is an ordinary integer. `Option<bool>` was close to useless, because
`false` is an ordinary bool.

## Why this upgrade is different

Nothing fails to compile. No signature changes. Your build stays green and four
common expressions start returning a different value.

| Expression | v5 | v6 |
| --- | --- | --- |
| `Option.Try(() => ComputeCount())` when the count is `0` | `None` | `Some(0)` |
| `Option<int> x = someInt;` when `someInt` is `0` | `None` | `Some(0)` |
| `Option.FromNullable(nullableInt)` when the value is `0` | `None` | `Some(0)` |
| `option.Map(x => x - x)` | `None` | `Some(0)` |

The same applies to `false`, `'\0'`, `Guid.Empty`, `DateTime.MinValue`,
`DateTimeOffset.MinValue`, `TimeSpan.Zero`, `IntPtr.Zero` and any enum's zero
member.

## What to do before you upgrade

1. **Search for `Option<` over a value type.** Every one is a candidate.
2. **Check each `IsNone` branch on those options.** Ask whether it is treating a
   zero as "no result". If it is, that branch stops running in v6.
3. **Turn the `WM1010` warnings into a to-do list.** Upgrade to 5.5.0 first. The
   analyzer marks every call site where it can prove the value is a default, and
   tells you what v6 will do there instead. It cannot reach the four expressions
   in the table above — those need the search in step 1.
4. **Delete any `catch (InvalidOperationException)` around `Option.Some`.** It
   becomes dead code. A null now throws `ArgumentNullException` instead.

## Null still throws

`Option.Some(null!)` throws in v6, as it did in v5. Only the exception type
changes, from `InvalidOperationException` to `ArgumentNullException`.

Use [`Option.FromNullable`](../core-functionality.md) when the value may be null.
That has not changed.

## Analyzer rules that go away

Three rules encoded the old invariant, so they are wrong in v6 and stop shipping:

| Rule | Why it goes |
| --- | --- |
| `WM1004` | The implicit conversion no longer maps a value-type default to `None` |
| `WM1009` | `Option<bool>` is no longer broken, so the rule's premise is false |
| `WM1010` | It exists only to warn you about this upgrade |

`WM1001` stays. It narrows to the null case, which is the half that survives.

Their IDs are never reused. If you have a `.editorconfig` entry or a
`#pragma warning disable` naming one of them, it becomes a no-op — safe to leave,
tidier to remove.
