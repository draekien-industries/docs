---
description: >-
  The WMGxxxx and WMSCxxxx codes. Each one marks something a generator cannot
  emit, or emitted differently from how you meant it.
icon: triangle-exclamation
---

# Generator diagnostics

These come from a source generator rather than from the analyzer. Two generators ship,
and each has its own prefix.

| Prefix | Generator | Page section |
| --- | --- | --- |
| `WMG` | Error code catalogs | [WMG](#wmg-error-code-catalogs) |
| `WMSC` | Schemas | [WMSC](#wmsc-schemas) |

{% hint style="info" %}
`WMSC` is not `WMS`. The `WMS` codes are analyzer rules from
`Waystone.Monads.Shouldly`, listed on
[Assertion rules](../analyzers/assertion-rules.md). The prefixes are four characters
and three characters, and suppressing the wrong one silences a rule you wanted.
{% endhint %}

## WMG: error code catalogs

Six diagnostics, all errors. Each marks a case where the generator would otherwise
produce code that does not compile, or a code you did not mean to publish.

| ID | What it reports |
| --- | --- |
| [`WMG0001`](#wmg0001) | `[ErrorCodeCatalog]` on a `[Flags]` enum |
| [`WMG0002`](#wmg0002) | Two members of the enum sharing a value |
| [`WMG0003`](#wmg0003) | A member named `Names`, `Codes` or `Errors` |
| [`WMG0004`](#wmg0004) | `ErrorCode` or `Error` not resolvable in the compilation |
| [`WMG0005`](#wmg0005) | A `Format` the generator cannot parse |
| [`WMG0006`](#wmg0006) | A `Format` that leaves out `{member}` |

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
generates one string for the whole enum, so the codes stop telling the members apart.
Include `{member}`.


## WMSC: schemas

Eight diagnostics from the schemas generator. Five are errors — the schema cannot be
generated, or the code cannot work. Three warn about code that compiles and runs and
is probably not what you meant.

See [Schemas](../packages/schemas.md) for the package itself.

| ID | Severity | What it reports |
| --- | --- | --- |
| [`WMSC0001`](#wmsc0001) | Error | A schema, or a type containing it, is not `partial` |
| [`WMSC0002`](#wmsc0002) | Error | No accessible parameterless constructor |
| [`WMSC0003`](#wmsc0003) | Error | A member named `Instance`, `Schema` or `FieldSet` |
| [`WMSC0004`](#wmsc0004) | Error | The `Into` lambda's arity does not match the field count |
| [`WMSC0005`](#wmsc0005) | Warning | `Refine` is handed a field that produces a value |
| [`WMSC0006`](#wmsc0006) | Error | An asynchronous rule reached from a field set |
| [`WMSC0007`](#wmsc0007) | Warning | A field-set call the generator did not recognise |
| [`WMSC0008`](#wmsc0008) | Warning | A field path taken from an expression, not a name |

### WMSC0001

**A generator adds members through a second declaration of the same class**, which the
compiler accepts only where every type in the nesting chain is `partial`.

`class QuestSchema : SchemaConfig<QuestDto, Quest>` gets nothing generated. Add
`partial`.

The diagnostic reports against the type that is missing the modifier, which is not
always the schema. A schema nested inside `public class Endpoints` needs `Endpoints`
to be `partial` too, and that is the one you have to edit.

Nothing else reports this. A schema that is not `partial` is perfectly legal C#; it
just silently receives no `Instance`.

### WMSC0002

**The generated `Instance` is a static property initialised with `new`.**

`SchemaConfig` supplies a protected parameterless constructor, so a derived schema
inherits one — until it declares a constructor of its own, at which point the implicit
one disappears with no diagnostic of its own.

Given `public QuestSchema(IQuestBoard board)` and nothing else, `Instance` cannot be
constructed. Either add a parameterless constructor, or take what the schema needs
from the input it parses rather than from a constructor. A schema that genuinely needs
a dependency is usually a schema that wants
[`CheckAsync`](../packages/schemas/asynchrony.md) composed around it instead.

### WMSC0003

**The generator reopens your class and writes three names into it** — `Instance`, a
nested `Schema`, and a `FieldSet` struct per field count. A hand-written member of any
of those names is a duplicate definition.

Rename yours, or delete it and use the generated one.

Type parameters do not separate them. A nested type collides with an existing member of
the same name whatever its arity, so `class FieldSet<T>` collides with the generated
`FieldSet<T1, T2>`.

The compiler reports this collision too, but against the generated file — which is not
a file anyone can edit.

`Schema` and `FieldSet` are only checked where a ladder is actually being emitted. A
schema that never calls `Schema.Fields` may keep a member of either name.

### WMSC0004

**The generated `Into` takes one parameter per field**, so a lambda of any other arity
cannot bind to it.

Three fields in `Schema.Fields` and `.Into((a, b) => …)` is a mismatch. Give the lambda
one parameter per field, in the order the fields are listed.

The compiler already rejects this, as a delegate conversion failure against a generated
type you never wrote and cannot open. That message names neither the field count nor
the file that decided it, which is why this rule exists — to say the thing you need
rather than to catch something new.

The diagnostic reports at the `Into` call, not at the field list. The field list is
what you meant; the lambda is what disagrees with it.

### WMSC0005

**`Refine` takes the non-generic `Field` base, which drops the value side.** It accepts
any field and keeps only its violations.

That is the right shape for `Schema.Forbidden` and `Schema.Extend`, which yield
`Checked` and have no value to contribute. It is a silent mistake for a field that
parses something somebody expected to find on the result.

`Refine(Schema.Required(subject.Email, Guild.Email))` checks the email and throws it
away. List it in `Schema.Fields` instead, so it reaches the `Into` lambda.

**A known false positive.** Gating on a value without keeping it is legitimate — a
confirmation field that has to be a well-formed email but is never stored is the
obvious case. That is why this warns rather than fails. Suppress it at the call site
if that is what you meant.

### WMSC0006

**`SchemaConfig.Configure` returns a value rather than a task**, so a field set
evaluates synchronously even when the caller uses `ParseAsync`. An asynchronous rule
reached that way throws `InvalidOperationException`.

Nothing in the type system says so, because `CheckAsync` returns the same schema type a
synchronous rule does.

Two ways out:

* Use `Check` if the rule can answer from the value alone.
* Compose the schema outside the field set and parse it with `ParseAsync`. See
  [Asynchrony](../packages/schemas/asynchrony.md#compose-it-around-the-outside-instead).

This is an error even though the code generates and compiles. There is no reading of it
that works: the rule either throws or is skipped, and it never does its job.

The diagnostic reports at the `CheckAsync` call, which is the one place you can act on.
The schema holding it is fine, and so is every other rule in the chain.

### WMSC0007

**`Schema.Fields` is the member being generated**, so it binds to nothing while the
generator is deciding whether to emit it. The receiver therefore has to be recognised
as written rather than resolved.

An alias, a renamed import, or a call with no receiver at all carries nothing to
recognise, so no ladder is generated. Write the receiver as `Schema`, qualified by the
type that contains it if you need to.

**A known false positive.** The generator matches the receiver by name, so this fires
on *any* unbound call to a member named `Fields` — including one that has nothing to do
with a field set and failed to bind for its own reasons. The compiler is already
reporting that call, so this rule warns rather than adding a second build failure to a
build that has one.

### WMSC0008

**A field's path comes from `CallerArgumentExpression`**, which hands the runtime the
argument's source text and nothing else.

A member access reduces to the member's name, and that is the case the whole design is
built around: `subject.Title` gives `title`. Anything else keeps its punctuation. A
method call, an indexer, a literal or a null-forgiving operator gives a path like
`subject.Total.ToString()`, and that text then reaches your logs and your API responses
alongside the violation.

Add `.Named("total")` to report it under a name a caller can act on.

**A known false positive.** The derived path is only *usually* wrong. If you are not
showing violations to anybody outside your own process, you may not care what the
expression reduces to. That is why this warns rather than fails.
