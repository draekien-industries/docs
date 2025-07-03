# Configuration

This library contains some configurable behaviours. Configuration is available via the `Configure` method of the `MonadOptions` class. It is recommended to configure each option below _once_ in the lifecycle of your application.

## Logging

There are several places in this library where exceptions are silently handled and transformed into non-throwing types. You can configure a custom logging action to inspect these exceptions as they are handled by the library.

```csharp
MonadOptions.Configure(options => options.UseExceptionLogger((ex) => {
    Console.WriteLine(ex); // replace with your logger's log method
}));
```

## Error Code Generation

There are a few factory methods included in the library for generating `ErrorCode` instances from `Enum` and from `Exception` instances. To customise how these error codes are generated, create a class inheriting from `ErrorCodeFactory` and override the methods you wish to customise. Then create an instance and pass it into the `MonadOptions` instance via the `UseErrorCodeFactory`.

```csharp
public class MyErrorCodeFactory : ErrorCodeFactory
{
    public override ErrorCode FromEnum(Enum @enum)
    {
        // your implementation
    }
    
    public override ErrorCode FromException(Exception exception)
    {
        // your implementation
    }
}

MonadOptions.Configure(options => options.UseErrorCodeFactory(new MyErrorCodeFactory()));
```

## Error Code and Message Fallbacks

There may be exception circumstances which cause the `string` used to create the `ErrorCode` or the message of the `Error` classes to be null or white-space. In these situations, a set of fallbacks are used. These fallbacks can be configured.

```csharp
MonadOptions.Configure(options => {
    options.FallbackErrorCode = "unknown"; // default is `Unspecified`
    options.FallbackErrorMessage = "Something went wrong!" // default is `An unspecified error has occurred.`
});
```
