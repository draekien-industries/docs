---
description: Get started with using Waystone.Monads.FluentValidation in under a minute.
icon: bullseye-arrow
---

# Quickstart

{% stepper %}
{% step %}
### Install the library

```sh
dotnet add package Waystone.Monads.FluentValidation
```
{% endstep %}

{% step %}
### Create a validator

{% hint style="info" %}
See [Fluent Validation: Creating your first validator](https://docs.fluentvalidation.net/en/latest/start.html) for more detailed information
{% endhint %}

<pre class="language-csharp"><code class="lang-csharp"><strong>record UserInput(int Range, string Search);
</strong>class UserInputValidator : AbstractValidator&#x3C;UserInput> 
{
    public UserInputValidator()
    {
        RuleFor(x => x.Range).GreaterThan(0);
        RuleFor(x => x.Search).NotEmpty();
    }
}
</code></pre>
{% endstep %}

{% step %}
### Validate

```csharp
UserInput input = new(1, "bob");
UserInputValidator validator = new();
Result<UserInput, ValidationErr> result = input.Validate(validator);
// or
Result<UserInput, ValidationErr> result = await input.ValidateAsync(validator, cancellationToken);
```

{% hint style="info" %}
See [Waystone.Monads](https://app.gitbook.com/o/eVVl3k56v8DR211ydaly/s/nQgeZ1m9pTmEbKceyEkw/ "mention") to learn how to work with a `Result`
{% endhint %}
{% endstep %}
{% endstepper %}

