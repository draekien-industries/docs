---
description: Keep exceptions exceptional
icon: hand-wave
cover: https://gitbookio.github.io/onboarding-template-images/header.png
coverY: 0
layout:
  cover:
    visible: true
    size: full
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

# Welcome

Waystone.Monads is a lightweight, idiomatic C# library that implements two fundamental functional types: `Option<T>` and `Result<T, E>`. It is inspired by Rust's `std::option` and `std::result` types and brings their power and functionality into the world of C#.

## Why this library exists

Most C# codebases default to `null` and exceptions for absence and failure. That's fine, until it isn't.

{% hint style="warning" %}
`null` and exceptions result in guard clauses everywhere, unpredictable runtime crashes, and unclear API intent.
{% endhint %}

Waystone.Monads replaces that with explicit types that make the intent clear at the type level:

* `Option<T>` means a value might be there.
* `Result<T, E>` means a computation might fail.

## Who should use this

You should use this library if:

* You want to eliminate `null` and exceptions from business logic
* You want your exceptions to be exceptional
* You prefer expressive, explicit code over defensive boilerplate
* You appreciate functional patterns but still want to write C#

{% hint style="success" %}
If you've ever used `Option` and `Result` in Rust or F#, you'll fee right at home. If you haven't, you'll pick it up quickly - and wonder how you ever lived without it.
{% endhint %}
