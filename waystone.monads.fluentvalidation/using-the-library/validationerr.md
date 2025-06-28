# ValidationErr

This library wraps the `ValidationResult` provided by [FluentValidation](https://docs.fluentvalidation.net/en/latest/index.html) inside the `ValidationErr` class. `ValidationErr` is designed to capture and expose validation failures in a way that integrates smoothly with the functional patterns provided by [Waystone.Monads](https://app.gitbook.com/o/eVVl3k56v8DR211ydaly/s/nQgeZ1m9pTmEbKceyEkw/ "mention").

{% hint style="info" %}
`ValidationErr` isolates the failure case of the `ValidationResult`
{% endhint %}

## Creation

A `ValidationErr` can be created by invoking the `Create` factory method and providing it with a `ValidationResult`. The factory will return an [Option\<T>](https://app.gitbook.com/s/nQgeZ1m9pTmEbKceyEkw/using-the-library/option-less-than-t-greater-than "mention") of `ValidationErr` which will be `Some` if the provided `ValidationResult` is invalid.

```csharp
class GreaterThanZeroValidator : AbstractValidator<int>
{
    public GreaterThanZeroValidator()
    {
         RuleFor(x => x).GreaterThanZero();   
    }
}

int x = -1;
GreaterThanZeroValidator validator = new();
ValidationResult result = validator.Validate(x);
Option<ValidationErr> maybeValidationErr = ValidationErr.Create(result);
//                    ^? Some(ValidationErr)
```

{% hint style="info" %}
In most cases, you will want to use the [#validate](validating-values.md#validate "mention") or [#validateasync](validating-values.md#validateasync "mention") extension methods instead.
{% endhint %}

## Properties

`ValidationErr` provides access to the underlying validation result's failure properties:

* `Errors`: the list of validation failures as exposed by the `ValidationResult`
* `RuleSetsExecuted`: the array of rulesets executed as exposed by the `ValidationResult`

## Methods

`ValidationErr` provides the following helper methods:

* `ToDictionary()`: converts the `ValidationResult` into a dictionary
* `ToString()`: converts the `ValidationResult` into a string
* `AsValidationResult()`: provides access to the underlying `ValidationResult`
