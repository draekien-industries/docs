---
description: >-
  What [ErrorCodeCatalog] emits: three nested classes, three extensions, and the
  fallback for a value that is not a declared member.
icon: table-list
---

# Error code catalogs

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

`Errors.NotFound(message)` and `ToError(message)` build the `Error` for you, so they
inherit how `Error` treats a message: it is trimmed, and a blank one is replaced by
your [configured fallback](../guides/configuration.md#error-code-and-message-fallbacks) rather
than rejected. Neither throws on a blank message, so pass a real one.

## A value that is not a declared member

Casting an arbitrary integer to an enum is legal C#, so `(OrderErrorCode)99` is a value
you can be handed. The three extensions apply the same scheme to it:

```csharp
((OrderErrorCode)99).ToErrorCodeName(); // "OrderErrorCode.99"
```

## Reusing a code across two enums

Two attributed enums with the same name in different namespaces generate the same code
for every member name they share. `Ordering.OrderErrorCode.NotFound` and
`Shipping.OrderErrorCode.NotFound` both generate `"OrderErrorCode.NotFound"`, and nothing
reading the code can tell the two errors apart. So can two differently named enums that
share a format: `"order.{member:kebab}"` on both `OrderErrorCode` and `ShipmentError`
makes `NotFound` collide.

`WM2018` reports that. It is a suggestion, not a warning — see
[#wm2018](../analyzers/idioms.md#wm2018 "mention").
