---
description: >-
  Store an Option in a single nullable column, where None is NULL. Read the
  query limits first.
---

# Waystone.Monads.EntityFrameworkCore


{% hint style="warning" %}
**This page describes `7.0.0-beta.x`, a pre-release.** NuGet gives you `6.x` unless you ask for a pre-release:

```
dotnet add package Waystone.Monads.EntityFrameworkCore --prerelease
```

Or set the version yourself: `<PackageReference Include="Waystone.Monads.EntityFrameworkCore" Version="7.0.0-beta.*" />`.

The API can still change before `7.0.0` is stable.
{% endhint %}

## Read this before you adopt it

Storage works. Querying mostly does not.

An `Option<T>` property saves and loads correctly. But you cannot filter on it
the way you filter on an ordinary property. Most of what you would write in a
`Where` throws.

{% hint style="danger" %}
If you need to query the column often, do not use `Option<T>` for that property.
Use a nullable property and convert at the edge of your code.
{% endhint %}

Here is every form, and what each one does.

| You write | What happens |
| --- | --- |
| `var wanted = Option.Some(31);`<br>`Where(p => p.Age == wanted)` | Works. `WHERE "Age" = @p` |
| `Where(p => p.Age == Option.Some(31))` | **Throws** |
| `Where(p => p.Age.IsSome)` | **Throws** |
| `var none = Option.None<int>();`<br>`Where(p => p.Age == none)` | Runs, and **finds nothing** |
| `Where(p => EF.Property<int?>(p, "Age") != null)` | Works. `WHERE "Age" IS NOT NULL` |
| `Where(p => EF.Property<int?>(p, "Age") == null)` | Works. `WHERE "Age" IS NULL` |

Every row is covered by a test in the repository.

### An inline `Option.Some(...)` does not work

Assign the option to a local first, then compare against the local.

```csharp
// throws
context.People.Where(person => person.Age == Option.Some(31));

// works
Option<int> wanted = Option.Some(31);
context.People.Where(person => person.Age == wanted);
```

The two lines look almost the same. Only the second one runs.

EF Core needs a value it can turn into a SQL parameter. It sees `Option.Some(31)`
as a method call it cannot run in the database, so it gives up.

### Comparing against a `None` finds nothing

This one is quiet, which makes it the worst of the three.

```csharp
Option<int> none = Option.None<int>();

// runs, returns zero rows, never throws
context.People.Where(person => person.Age == none);
```

It becomes `WHERE "Age" = NULL`. In SQL nothing is equal to `NULL`, not even
`NULL`. So the query is valid and matches no row.

To find your `None` rows, ask the column directly:

```csharp
context.People.Where(person => EF.Property<int?>(person, "Age") == null);
```

### Everything else on the option throws

`IsSome`, `IsNone`, `Match`, `Unwrap` — none of them work inside a query.

EF Core cannot turn a call on `Option<T>` into SQL. You get an exception when the
query runs, not a wrong answer, so you find out immediately.

## Install it

```
dotnet add package Waystone.Monads.EntityFrameworkCore --prerelease
```

The package targets `net8.0` and `net10.0`. It supports
`Microsoft.EntityFrameworkCore >= 8.0.11 && < 11.0.0`. Bring your own version
inside that range.

There is no `netstandard2.0` asset. EF Core does not ship one either, so a
`netstandard2.0` project could not use this package anyway.

## Use it

Add one line to `OnModelCreating`:

```csharp
using Microsoft.EntityFrameworkCore;
using Waystone.Monads.Options;

public sealed class AppDbContext : DbContext
{
    public DbSet<Person> People => Set<Person>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        modelBuilder.UseWaystoneOptionConversions();
    }
}

public sealed class Person
{
    public int Id { get; set; }
    public Option<string> Nickname { get; set; } = Option.None<string>();
    public Option<int> Age { get; set; } = Option.None<int>();
}
```

That one call finds every `Option<T>` property in the model, whatever type it
holds. You get two nullable columns of the held types:

```sql
CREATE TABLE "People" (
    "Id"       INTEGER NOT NULL CONSTRAINT "PK_People" PRIMARY KEY AUTOINCREMENT,
    "Nickname" TEXT    NULL,
    "Age"      INTEGER NULL
);
```

Call it last. It reads the entity types already in the model, so put your own
configuration above it. A property you have already given a converter is left
alone.

### Why not `ConfigureConventions`?

`ConfigureConventions` only takes closed types. You would need one
`Properties<Option<int>>()` line per `T` you use, and any `T` you forgot would
silently go unmapped. The sweep finds them all.

## What gets stored

A default value is stored and read back as a `Some`. `Some(0)` is not `None`.

| Property value | Column | Reads back as |
| --- | --- | --- |
| `Option.Some(31)` | `31` | `Option.Some(31)` |
| `Option.Some(0)` | `0` | `Option.Some(0)` |
| `Option.Some("")` | `''` | `Option.Some("")` |
| `Option.None<int>()` | `NULL` | `Option.None<int>()` |
| `null` | `NULL` | `Option.None<int>()` |

The last row is a kindness rather than a promise. The CLR default of `Option<T>`
is `null`, not `None`, so a property you forget to initialise holds a null. The
package treats that as a `None` instead of throwing. Initialise your properties
anyway — only the database round trip repairs it.

## Where the types live

The package shadows Entity Framework Core's own namespaces, so its types sit
where you already look for them.

| Member | Namespace |
| --- | --- |
| `UseWaystoneOptionConversions` | `Microsoft.EntityFrameworkCore` |
| `ReferenceTypeOptionConverter<T>`, `ValueTypeOptionConverter<T>` | `Microsoft.EntityFrameworkCore.Storage.ValueConversion` |
| `OptionValueComparer<T>` | `Microsoft.EntityFrameworkCore.ChangeTracking` |

The package and assembly are still called
`Waystone.Monads.EntityFrameworkCore`. Only the namespaces shadow.

## Configuring one property yourself

Use this when a property needs a column name the sweep would not give it.

```csharp
modelBuilder.Entity<Person>()
            .Property(person => person.Nickname)
            .HasConversion(
                new ReferenceTypeOptionConverter<string>(),
                new OptionValueComparer<string>())
            .IsRequired(false)
            .HasColumnName("nick");

modelBuilder.Entity<Person>()
            .Property(person => person.Age)
            .HasConversion(
                new ValueTypeOptionConverter<int>(),
                new OptionValueComparer<int>())
            .IsRequired(false);
```

Three things the sweep does for you, which you now have to do yourself.

**Pick the converter that matches the held type.** A reference type takes
`ReferenceTypeOptionConverter<T>`. A value type takes
`ValueTypeOptionConverter<T>`. Pick the wrong one and the code does not compile,
so this is an annoyance rather than a trap.

**Call `IsRequired(false)`.** Without it the column comes out `NOT NULL`, and
saving a `None` fails at the database.

**Pass the comparer.** Without it EF Core builds its own, which may not notice a
property changing from `Some(1)` to `Some(2)`. That change would then never reach
the database. One comparer class covers both reference and value types.

### Why two converter classes?

Because C# forces it.

Under a `notnull` constraint, `T?` means `T` — not `Nullable<T>`. So a single
class would give `Option<int>` a non-nullable column and write a `None` as `0`.
That is silent data loss, so the package splits the two cases instead.

## A nullable option is not supported

`Option<T>?` has no representation here.

The column has one `NULL`. The property wants two kinds of absent. There is no
way to tell "no value in this row" apart from "`None`". If you have that shape,
do not use `Option<T>` for that property.

## There is no `Result` converter

`Result<TOk, TErr>` is out of scope, and will stay out of scope.

`TOk` and `TErr` are different types, usually unrelated. One column cannot hold
both. You would need a discriminator column plus two nullable columns, and the
choices involved differ for every entity. That is a mapping decision, not a value
converter.

A result describes how an operation went. It is not persisted state. If you do
want to store one, map explicit columns and build the `Result` in your own code.
