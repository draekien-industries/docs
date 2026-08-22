---
description: >-
  Mark an enum with [ErrorCodeCatalog] and get its error codes as compile-time
  constants, generated at build time.
---

# Generated error codes

## What this page is for

`ErrorCode.FromEnum(InputErrors.Missing)` works out the string `"InputErrors.Missing"`
at run time, by reflecting over the enum. That means the string your callers see is
not written anywhere you can point at. You cannot use it in a `switch`, you cannot put
it in an attribute, and you cannot find every place that reads it.

Mark the enum with `[ErrorCodeCatalog]` and `Waystone.Monads` generates those strings
as constants when you build. This ships inside the package from 6.2.0. You add no
reference and configure nothing.

{% hint style="warning" %}
`ErrorCode.FromEnum` is obsolete from 6.2.0 and removed in 7.0.0, so this page
describes the only supported way to get an error code from an enum. See
[deprecations.md](deprecations.md "mention") for the migration.
{% endhint %}

## Marking an enum

```csharp
using Waystone.Monads.Results.Errors;

namespace Ordering;

[ErrorCodeCatalog]
public enum OrderErrorCode
{
    NotFound,
    AlreadyShipped,
}
```

That gives you a new class next to the enum, `OrderErrorCodeCatalog`, in the same
namespace and with the same accessibility as the enum.

The name is the enum's own name with `Catalog` on the end, and nothing is taken off it.
`OrderFailure` gives you `OrderFailureCatalog`; `OrderErrorCode` gives you
`OrderErrorCodeCatalog`. If that reads twice, rename the enum — two enums whose names
differ always get two catalogs whose names differ.

## What you get

Three nested classes, one per shape:

```csharp
// The code as a compile-time constant.
OrderErrorCodeCatalog.Names.NotFound   // "OrderErrorCode.NotFound"

// The code as an ErrorCode.
OrderErrorCodeCatalog.Codes.NotFound   // ErrorCode { Value = "OrderErrorCode.NotFound" }

// An Error carrying that code.
OrderErrorCodeCatalog.Errors.NotFound("no order with that id")
```

`Names` gives you a `const string`, so you can use it anywhere C# wants a
constant — a `case` label, an attribute argument, a switch on a code that arrived over
the wire.

And three extension methods, for when you are holding a value rather than naming a
member:

```csharp
OrderErrorCode errorCode = Classify(order);

string asName = errorCode.ToErrorCodeName();
ErrorCode asErrorCode = errorCode.ToErrorCode();
Error asError = errorCode.ToError("no order with that id");
```

The nesting is what keeps your member names usable as-is. A member called
`NotFoundCode` becomes `Names.NotFoundCode`, and nothing collides.

## A value that is not a declared member

Casting an arbitrary integer to an enum is legal C#, so `(OrderErrorCode)99` is a value
you can be handed. The three extensions apply the same scheme to it:

```csharp
((OrderErrorCode)99).ToErrorCodeName(); // "OrderErrorCode.99"
```

## Choosing the code format

`OrderErrorCode.NotFound` is the default, not the only option. Set `Format` on the
attribute and the generated codes follow it:

```csharp
[ErrorCodeCatalog(Format = "order.{member:kebab}")]
public enum OrderErrorCode
{
    NotFound,
    AlreadyShipped,
}
```

```csharp
OrderErrorCodeCatalog.Names.NotFound       // "order.not-found"
OrderErrorCodeCatalog.Names.AlreadyShipped // "order.already-shipped"
```

Everything the format does happens at build time, so what you get out is still a
`const string`.

### The format language

Two placeholders, and everything else is literal text:

| Placeholder | Substitutes |
| --- | --- |
| `{enum}` | The enum's name, `OrderErrorCode` |
| `{member}` | The member's name, `NotFound` |

Either one takes an optional casing after a colon — `{member:kebab}`:

| Casing | `NotFound` | `HTTPNotFound` | `Error404` |
| --- | --- | --- | --- |
| *(none)* | `NotFound` | `HTTPNotFound` | `Error404` |
| `kebab` | `not-found` | `http-not-found` | `error-404` |
| `snake` | `not_found` | `http_not_found` | `error_404` |
| `lower` | `notfound` | `httpnotfound` | `error404` |
| `upper` | `NOTFOUND` | `HTTPNOTFOUND` | `ERROR404` |

`kebab` and `snake` split the identifier into words; `lower` and `upper` only change
case. A word boundary falls where a lowercase or a digit meets an uppercase, before the
last uppercase of a run that runs into a lowercase, and between letters and digits. An
underscore already in the name counts as a boundary rather than doubling up, so
`Already_Shipped` gives `already-shipped`.

Write a literal brace by doubling it: `{{` and `}}`.

The default is `{enum}.{member}`, which is what an enum that sets nothing gets.

### One format for the whole assembly

Put the attribute on the assembly and every attributed enum in the project uses it:

```csharp
[assembly: ErrorCodeFormat("{enum:kebab}/{member:kebab}")]
```

An enum's own `Format` wins over the assembly's, so the assembly attribute sets the
house style and an individual enum departs from it where it needs to.

## The format is your contract, not your factory

The generated strings come from the format and nothing else. In particular:

{% hint style="warning" %}
**A generated member never consults your `ErrorCodeFactory`.** If you install your own
through `MonadOptions.UseErrorCodeFactory`, `ErrorCode.FromEnum` starts returning your
codes and the generated members keep returning theirs. The generator runs at build time
and cannot run a factory you install at run time.

That holds for every generated member, including the fallback for an undeclared value.
If you are using a custom factory today to shape the codes an enum produces, say the
same thing with `Format` instead — you get the same strings as constants, and one
answer rather than two.
{% endhint %}

Shaping enum codes is the part of `ErrorCodeFactory` the format replaces, and that part
is obsolete from 6.2.0: `ErrorCodeFactory.FromEnum` and `ErrorCode.FromEnum` both
report `CS0618` and are removed in 7.0.0. See
[deprecations.md](deprecations.md "mention"). `FromException` is not affected.

## Reviewing your codes as a list

An error code is a wire contract, and the thing that makes one hard to hold onto is that
a rename changes it silently. You can make the whole set reviewable by committing it.

Add an `ErrorCodes.txt` to the project and list it as an additional file:

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

### Making a divergence fail the build

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

The code is built from two names: the enum's and the member's. Rename either — or edit
the format — and every consumer reading the code sees a different string, with nothing
in the compiler to tell you.

```csharp
[ErrorCodeCatalog]
public enum OrderErrorCode   // rename this -> every code changes
{
    NotFound,                // rename this -> "OrderErrorCode.NotFound" changes
}
```

This is not new — `ErrorCode.FromEnum` has always worked this way, and `ErrorCode`'s
own guidance is that a code should not change between occurrences of the same error.
Treat an attributed enum as a published contract, the way you would treat a URL.

## Diagnostics

Six diagnostics come from the generator rather than from the analyzer, so they use a
`WMG` prefix. All six are errors: each marks a case where the generator would
otherwise produce code that does not compile, or a code you did not mean to publish.

| ID | What it reports |
| --- | --- |
| `WMG0001` | `[ErrorCodeCatalog]` on a `[Flags]` enum |
| `WMG0002` | Two members of the enum sharing a value |
| `WMG0003` | A member named `Names`, `Codes` or `Errors` |
| `WMG0004` | `ErrorCode` or `Error` not resolvable in the compilation |
| `WMG0005` | A `Format` the generator cannot parse |
| `WMG0006` | A `Format` that leaves out `{member}` |

### WMG0001

**A flags enum has no single code per value.** `OrderErrorCode.NotFound |
OrderErrorCode.AlreadyShipped` is one value whose `ToString()` is `"NotFound,
AlreadyShipped"`, and there is no sensible code for it. Use a plain enum for errors,
and model a combination as its own member if you need one.

### WMG0002

**Two members with the same value are one value.** Given `NotFound = 1` and
`Missing = 1`, the two are indistinguishable at run time, so neither has a code of its
own. Give them different values, or delete the alias.

### WMG0003

**A member named after a generated class would produce invalid C#.** The generator
emits nested classes called `Names`, `Codes` and `Errors`. A member
with one of those names would produce a member sharing its enclosing type's name,
which is `CS0542`. Rename the member.

Every other name is fine, including `ToError`, `ToErrorCode` and `ToErrorCodeName` —
the extensions live on the outer class and your members live in the nested ones, so
they never meet.

### WMG0004

**The generator cannot find `ErrorCode` or `Error`.** You will only see this if the
generator is running on a project that does not reference `Waystone.Monads`, which
normally means a hand-wired analyzer reference. Reference the package.

### WMG0005

**The format does not parse.** An unclosed placeholder, a stray `}`, a name that is not
`{enum}` or `{member}`, or a casing that is not `kebab`, `snake`, `lower` or `upper`.
The message names the position and what it expected. This reports on the attribute that
set the format, whether that is the enum's or the assembly's.

### WMG0006

**A format without `{member}` gives every member the same code.** `"{enum:kebab}"`
generates one string for the whole enum, so the codes stop identifying anything.
Include `{member}`.

## Reusing a code across two enums

Two attributed enums with the same name in different namespaces generate the same code
for every member name they share. `Ordering.OrderErrorCode.NotFound` and
`Shipping.OrderErrorCode.NotFound` both generate `"OrderErrorCode.NotFound"`, and nothing
reading the code can tell the two errors apart. So can two differently named enums that
share a format: `"order.{member:kebab}"` on both `OrderErrorCode` and `ShipmentError`
makes `NotFound` collide.

`WM2018` reports that. It is a suggestion, not a warning — see
[#wm2018](analyzer-rules.md#wm2018 "mention").
