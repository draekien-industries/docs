---
icon: hand-wave
---

# Welcome

`Waystone.Monads.FluentValidation` is a library that seamlessly integrates the expressive validation capabilities of [FluentValidation](https://docs.fluentvalidation.net/en/latest/index.html) with the robust error handling patterns of [Waystone.Monads](https://app.gitbook.com/o/eVVl3k56v8DR211ydaly/s/nQgeZ1m9pTmEbKceyEkw/ "mention"). This library is designed to streamline how you validate objects and manage validation results in a clear, functional, and type-safe way.

## What's Included?

{% hint style="info" %}
It is recommended to familiarise yourself with [Waystone.Monads](https://app.gitbook.com/o/eVVl3k56v8DR211ydaly/s/nQgeZ1m9pTmEbKceyEkw/ "mention") before continuing.
{% endhint %}

This library provides a set of extensions that allow you to execute a validator on any value. These extension methods wrap the result of the validator in a [Result\<T, E>](https://app.gitbook.com/s/nQgeZ1m9pTmEbKceyEkw/result-less-than-t-e-greater-than "mention"), making it effortless to handle both `Ok` and `Err` cases in your application.

## Who should use this?

You should use this library if:

* You want to use both [FluentValidation](https://docs.fluentvalidation.net/en/latest/index.html) and [Waystone.Monads](https://app.gitbook.com/o/eVVl3k56v8DR211ydaly/s/nQgeZ1m9pTmEbKceyEkw/ "mention") in your application
* You want to plug in validation into your monadic workflows without having to manually plumb it all together
* You want to kickstart the monadic chain by executing a validator
* You want to keep validation and error-handling logic clear and concise

## Get Started

Explore the documentation to learn about the inner workings of this library.
