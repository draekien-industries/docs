---
description: >-
  Mark an enum with [ErrorCodeProvider] and get its error codes as compile-time
  constants, generated at build time.
---

# Generated error codes

## What this page is for

`ErrorCode.FromEnum(InputErrors.Missing)` works out the string `"InputErrors.Missing"`
at run time, by reflecting over the enum. That means the string your callers see is
not written anywhere you can point at. You cannot use it in a `switch`, you cannot put
it in an attribute, and you cannot find every place that reads it.

Mark the enum with `[ErrorCodeProvider]` and `Waystone.Monads` generates those strings
as constants when you build. This ships inside the package from 6.2.0. You add no
reference and configure nothing.

## Marking an enum

```csharp
using Waystone.Monads.Results.Errors;

namespace Ordering;

[ErrorCodeProvider]
public enum OrderErrorCode
{
    NotFound,
    AlreadyShipped,
}
```

That gives you a new class next to the enum, `OrderErrorProvider`, in the same
namespace and with the same accessibility as the enum.

The name is the enum's with `ErrorProvider` on the end, and a trailing `Error` or
`ErrorCode` on the enum's own name taken off first so you do not read it twice.
`OrderErrorCode` and `OrderError` both give you `OrderErrorProvider`;
`PaymentFailure` gives you `PaymentFailureErrorProvider`.

## What you get

Three nested classes, one per shape:

```csharp
// The code as a compile-time constant.
OrderErrorProvider.ErrorCodeStrings.NotFound   // "OrderErrorCode.NotFound"

// The code as an ErrorCode.
OrderErrorProvider.ErrorCodes.NotFound         // ErrorCode { Value = "OrderErrorCode.NotFound" }

// An Error carrying that code.
OrderErrorProvider.Errors.NotFound("no order with that id")
```

`ErrorCodeStrings` gives you a `const string`, so you can use it anywhere C# wants a
constant — a `case` label, an attribute argument, a switch on a code that arrived over
the wire.

And three extension methods, for when you are holding a value rather than naming a
member:

```csharp
OrderErrorCode errorCode = Classify(order);

string asString = errorCode.ToErrorCodeString();
ErrorCode asErrorCode = errorCode.ToErrorCode();
Error asError = errorCode.ToError("no order with that id");
```

The nesting is what keeps your member names usable as-is. A member called
`NotFoundCode` becomes `ErrorCodeStrings.NotFoundCode`, and nothing collides.

## A value that is not a declared member

Casting an arbitrary integer to an enum is legal C#, so `(OrderErrorCode)99` is a value
you can be handed. The three extensions apply the same scheme to it:

```csharp
((OrderErrorCode)99).ToErrorCodeString(); // "OrderErrorCode.99"
```

## The generated code and your factory

The generated strings follow the same `{EnumName}.{MemberName}` scheme as
[#error-code-from-enum](errors-and-exceptions.md#error-code-from-enum "mention"), so a
constant and a call to `ErrorCode.FromEnum` give you the same string — as long as you
are on the default factory.

{% hint style="warning" %}
**A generated member never consults your `ErrorCodeFactory`.** If you install your own
through `MonadOptions.UseErrorCodeFactory`, `ErrorCode.FromEnum` starts returning your
codes and the generated members keep returning theirs. The generator runs at build
time and cannot run a factory you install at run time, so it always bakes in the
default scheme.

That holds for every generated member, including the fallback for an undeclared value.
Pick one and stay on it per enum: an `[ErrorCodeProvider]` enum's codes come from the
generator, and your factory keeps handling everything else.
{% endhint %}

## Renaming is a breaking change

The code is built from two names: the enum's and the member's. Rename either and every
consumer reading the code sees a different string, with nothing in the compiler to tell
you.

```csharp
[ErrorCodeProvider]
public enum OrderErrorCode   // rename this -> every code changes
{
    NotFound,                // rename this -> "OrderErrorCode.NotFound" changes
}
```

This is not new — `ErrorCode.FromEnum` has always worked this way, and `ErrorCode`'s
own guidance is that a code should not change between occurrences of the same error.
Treat an attributed enum as a published contract, the way you would treat a URL.

## Diagnostics

Four diagnostics come from the generator rather than from the analyzer, so they use a
`WMG` prefix. All four are errors: each marks a case where the generator would
otherwise produce code that does not compile, or a code you did not mean to publish.

| ID | What it reports |
| --- | --- |
| `WMG0001` | `[ErrorCodeProvider]` on a `[Flags]` enum |
| `WMG0002` | Two members of the enum sharing a value |
| `WMG0003` | A member named `ErrorCodeStrings`, `ErrorCodes` or `Errors` |
| `WMG0004` | `ErrorCode` or `Error` not resolvable in the compilation |

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
emits nested classes called `ErrorCodeStrings`, `ErrorCodes` and `Errors`. A member
with one of those names would produce a member sharing its enclosing type's name,
which is `CS0542`. Rename the member.

Every other name is fine, including `ToError`, `ToErrorCode` and `ToErrorCodeString` —
the extensions live on the outer class and your members live in the nested ones, so
they never meet.

### WMG0004

**The generator cannot find `ErrorCode` or `Error`.** You will only see this if the
generator is running on a project that does not reference `Waystone.Monads`, which
normally means a hand-wired analyzer reference. Reference the package.

## Reusing a code across two enums

Two attributed enums with the same name in different namespaces generate the same code
for every member name they share. `Ordering.OrderErrorCode.NotFound` and
`Shipping.OrderErrorCode.NotFound` both generate `"OrderErrorCode.NotFound"`, and nothing
reading the code can tell the two errors apart.

`WM2018` reports that. It is a suggestion, not a warning — see
[#wm2018](analyzer-rules.md#wm2018 "mention").
