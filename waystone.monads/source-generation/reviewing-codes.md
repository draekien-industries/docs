---
description: >-
  Commit your error codes as a list, and make a divergence fail the build rather
  than a rename change a wire contract silently.
icon: list-check
---

# Reviewing generated codes

## Reviewing your codes as a list

An error code is a wire contract, and the thing that makes one hard to hold onto is that
a rename changes it silently. You can make the whole set reviewable by committing it.

Add an `ErrorCodes.txt` to the project and list it as an `AdditionalFiles` item:

```xml
<ItemGroup>
    <AdditionalFiles Include="ErrorCodes.txt"/>
</ItemGroup>
```

That is the whole opt-in — the same shape as `PublicAPI.Shipped.txt` for the public API
analyzers. A project without the file never sees either rule.

The file is one code per line. Blank lines are ignored and a line starting with `#` is a
comment:

```
# Every error code this project publishes. Reviewed on change.
order.already-shipped
order.not-found
```

Two rules then keep it honest:

| ID | What it reports |
| --- | --- |
| `WM2019` | An enum member generates a code the file does not list |
| `WM2020` | The file lists a code no catalog generates |

`WM2019` has a code fix, **Update ErrorCodes.txt**, which rewrites the whole file from
the compilation: it adds every missing code and removes every stale entry in one pass,
sorted, keeping your leading comment block. So a rename shows up as one removed line and
one added line, and you read it in the diff before you commit it.

{% hint style="info" %}
`WM2020` has no code fix of its own. It is reported against `ErrorCodes.txt` rather than
against any of your source, and Roslyn does not offer fixes for a diagnostic reported at
the end of a compilation. In practice this does not come up: the `WM2019` fix removes
stale entries too. A project whose *only* divergence is a stale entry deletes the line
the message names.
{% endhint %}

## Making a divergence fail the build

Both rules ship as suggestions, so by default they show in the IDE and not in CI. If you
have committed the file you probably want the opposite. `WM2019` responds to an ordinary
`.editorconfig`:

```ini
[*.cs]
dotnet_diagnostic.WM2019.severity = warning
```

{% hint style="warning" %}
**`WM2020` does not.** It is reported against `ErrorCodes.txt`, which has no syntax tree,
and Roslyn resolves `dotnet_diagnostic` severities per syntax tree — so a path-matched
section is never consulted for it, including `[*]`. Raising it takes a global analyzer
config:

```ini
is_global = true
dotnet_diagnostic.WM2019.severity = warning
dotnet_diagnostic.WM2020.severity = warning
```

Put that in a `.globalconfig` next to the project. A global config covers both rules, so
it is the simpler thing to write even though only one of them needs it.
{% endhint %}

## Renaming is a breaking change

The code is built from two names: the enum's and the member's. Rename either (or edit
the format) and every consumer reading the code sees a different string, with nothing
in the compiler to tell you.

```csharp
[ErrorCodeCatalog]
public enum OrderErrorCode   // rename this -> every code changes
{
    NotFound,                // rename this -> "OrderErrorCode.NotFound" changes
}
```

This is not new — `ErrorCode.FromEnum` worked the same way, and `ErrorCode`'s
own guidance is that a code should not change between occurrences of the same error.
Treat an attributed enum as a published contract, the way you would treat a URL.

