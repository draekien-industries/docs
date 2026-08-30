---
description: >-
  The WMGxxxx codes. All six are errors, and each one marks a catalog the
  generator cannot emit.
icon: triangle-exclamation
---

# Generator diagnostics

Six diagnostics come from the generator rather than from the analyzer, so they use a
`WMG` prefix. All six are errors: each marks a case where the generator would
otherwise produce code that does not compile, or a code you did not mean to publish.

| ID | What it reports |
| --- | --- |
| [`WMG0001`](#wmg0001) | `[ErrorCodeCatalog]` on a `[Flags]` enum |
| [`WMG0002`](#wmg0002) | Two members of the enum sharing a value |
| [`WMG0003`](#wmg0003) | A member named `Names`, `Codes` or `Errors` |
| [`WMG0004`](#wmg0004) | `ErrorCode` or `Error` not resolvable in the compilation |
| [`WMG0005`](#wmg0005) | A `Format` the generator cannot parse |
| [`WMG0006`](#wmg0006) | A `Format` that leaves out `{member}` |

## WMG0001

**A flags enum has no single code per value.** `OrderErrorCode.NotFound |
OrderErrorCode.AlreadyShipped` is one value whose `ToString()` is `"NotFound,
AlreadyShipped"`, and there is no sensible code for it. Use a plain enum for errors,
and model a combination as its own member if you need one.

## WMG0002

**Two members with the same value are one value.** Given `NotFound = 1` and
`Missing = 1`, the two are indistinguishable at run time, so neither has a code of its
own. Give them different values, or delete the alias.

## WMG0003

**A member named after a generated class would produce invalid C#.** The generator
emits nested classes called `Names`, `Codes` and `Errors`. A member
with one of those names would produce a member sharing its enclosing type's name,
which is `CS0542`. Rename the member.

Every other name is fine, including `ToError`, `ToErrorCode` and `ToErrorCodeName` —
the extensions live on the outer class and your members live in the nested ones, so
they never meet.

## WMG0004

**The generator cannot find `ErrorCode` or `Error`.** You will only see this if the
generator is running on a project that does not reference `Waystone.Monads`, which
normally means a hand-wired analyzer reference. Reference the package.

## WMG0005

**The format does not parse.** An unclosed placeholder, a stray `}`, a name that is not
`{enum}` or `{member}`, or a casing that is not `kebab`, `snake`, `lower` or `upper`.
The message names the position and what it expected. This reports on the attribute that
set the format, whether that is the enum's or the assembly's.

## WMG0006

**A format without `{member}` gives every member the same code.** `"{enum:kebab}"`
generates one string for the whole enum, so the codes stop telling the members apart.
Include `{member}`.

