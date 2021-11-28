---
layout: post
title: You Don't Need Logging Code
tags: aop logging decorator
---

This an opinion I've held and maintained for some years now. The title is a little disingenuous. What I actually mean is that classes should not be concerned with what they need to log. Take this example.

```csharp
public MyResult DoImportantLogic(string someArg)
{
    var someObject = _objectGenerator.Create(someArg);

    _logger.Debug("About to do a thing.");

    try
    {
        _setterUpper.DoInitialSetup(someObject.SomeProperty);
        var result = _worker.DoSomeActualWork(someObject);

        _logger.Information("Done a thing.");

        return result;
    }
    catch (Exception exception)
    {
        _logger.Error("Error doing a thing.", exception);
        throw;
    }
}
```

I see this sort of code far too often. Let me explain what's wrong with it.

##### 1. There's an implicit boundary

It's all about [drawing boxes](/drawing-boxes). The developer who wrote those log lines defined the concept of "a thing", but only in the logs. If we saw the same thing with comments, we would want to refactor into a new `DoAThing()` method, so I feel we should do the same here. The structure of our code should match how we break down the problem in our minds.

It's not just about grouping things together and hiding them in a different part of the file. A method scopes the variables declared within that section of code so they cannot interfere with those outside. Extracting a method forces us to explicitly define which arguments are passed into the method and which are returned from it. The boundary is clearer and more rigid, which helps us avoid spaghetti code as it develops.

##### 2. There are too many constructor parameters

In his book Clean Code, Uncle Bob makes the case for an upper limit of three parameters for all functions.

> The ideal number of arguments for a function is zero (niladic). Next comes one (monadic) followed closely by two (dyadic). Three arguments (triadic) should be avoided where possible. More than three (polyadic) requires very special justification—and then shouldn't be used anyway.
> -- Robert C. Martin

Constructors are essentially a function that creates an instance of a class. The constructor parameters show us what the class depends on - they're usually the first thing you see when opening a class file and the signature of the constructor is an indicator of what a class' responsibility is. Is the class responsible for logging? Is that part of its single responsibility? Not in this case.

##### 3. We have not separated cross cutting concerns

Logging is called a **cross-cutting concern** because it is often used in many parts of our application: presentation, business logic, database code; they all want to write log lines. When we talk about [separation of concerns](https://en.wikipedia.org/wiki/Separation_of_concerns), the word 'concern' is not a coincidental homophone - it's the same thing.

Sometimes it's difficult to define what a concern really is when we're trying to separate them, but not this one - examples of cross-cutting concerns are all over the internet, and logging is usually the first.

##### 4. Should the tests have to test logging?

Often I see the `ILogger` interface mocked out and ignored in unit tests. Just because it keeps the runtime happy. It's almost as if the class is not really *concerned* with logging, right?

##### 5. We spend so much time talking about it

I've worked in teams where we spend a *huge* chunk of our time writing log lines, discussing what to log, what level (Debug, Info, Warn) to log which things at, commenting on PRs suggesting adding another property. We spent time discussing logging standards and wrote documents explaining them. And we still got it wrong anyway - we'd have to raise new PRs to rename a log property to make it consistent with another part of the application. What a colossal waste of time.

Do you know what, this is about separating concerns too. We managed to write the rules in one place. Couldn't we implement them in one place? Perhaps one class that has the single responsibility of logging? And only speak about it when we work on the code that is concerned with logging?

##### 6. There's so much boilerplate

There's so much of it! Sometimes more than half the lines of code are related to logging. It clutters the codebase so important logic gets obfuscated and is harder to read.

Especially if we've defined standards for how to write this stuff, we end up seeing very similar logging code everywhere. Sometimes we glance over the familiar-looking code when reviewing, which could lead to missing bugs.

##### 7. Log and rethrow

How does the calling code know if you've already logged the exception? You'll probably end up logging this same exception in multiple places if you follow this anti-pattern everywhere.

Exceptions propagate all the way up the call stack until they are caught. Let them bubble up to a catch block that can either handle the exception properly, or one that is so high up it has no choice but to log it.

# Refactoring

First, let's address point #1 and make our code structure match how we break down this problem in our heads by introducing the concept of "a thing" in our code using a `DoAThing` method.

```c#
private MyResult DoAThing(SomeObject someObject)
{
    _setterUpper.DoInitialSetup(someObject.SomeProperty);
    return _worker.DoSomeActualWork(someObject);
}
```

Wonderful. Now, where should I put the logging? Inside this method? Or in the one that calls this? What would it look like if I create yet another method that has all the logging in it?

```c#
private MyResult DoAThingAndLogIt(SomeObject someObject)
{
    _logger.Debug("About to do a thing.");

    try
    {
        var result = DoAThing(someObject);

        _logger.Information("Done a thing.");

        return result;
    }
    catch (Exception exception)
    {
        _logger.Error("Error doing a thing.", exception);
        throw;
    }
}
```

Note that both those methods accept a `SomeObject` parameter and return a `MyResult`. All we've done is wrap around the method and logged stuff. It would be easy to switch between those methods in the calling code!

```c#
public MyResult DoImportantLogic(string someArg)
{
    var someObject = _objectGenerator.Create(someArg);

    return DoAThingAndLogIt(someObject);
}
```

# Decorators

You might've noticed that all 3 of those methods used different fields from the parent class. So what is the responsibility of the class? This might be an indicator that all 3 of these should be in separate classes, each with their relevant fields injected into their constructors.

The innermost method `DoAThing` only needs two dependencies for its parent class.

```c#
public class ThingDoer : IThingDoer
{
    public DoAThing(ISetterUpper setterUpper, IWorker worker) { /* set fields */ }

    public MyResult DoAThing(SomeObject someObject)
    {
        _setterUpper.DoInitialSetup(someObject.SomeProperty);
        return _worker.DoSomeActualWork(someObject);
    }
}
```

For the logging method in the middle, we need a dependency for the `IThingDoer` above, and another for `ILogger`.

Remember the methods have the same inputs and outputs. If we name the methods the same thing, this middle class conforms to the same `IThingDoer` interface above. This is the [Decorator Pattern](https://refactoring.guru/design-patterns/decorator).

```c#
public class LoggingThingDoer : IThingDoer
{
    public LoggingThingDoer(IThingDoer innerThingDoer, ILogger logger) { /* set fields */ }

    public MyResult DoAThing(SomeObject someObject)
    {
        _logger.Debug("About to do a thing.");

        try
        {
            var result = DoAThing(someObject);

            _logger.Information("Done a thing.");

            return result;
        }
        catch (Exception exception)
        {
            _logger.Error("Error doing a thing.", exception);
            throw;
        }
    }
}
```

Our outermost class depends on `IThingDoer`, but it doesn't know whether that is just the `ThingDoer` or the `LoggingThingDoer`.

```c#
public class ImportantLogicDoer
{
    public DoAThing(IObjectGenerator objectGenerator, IThingDoer thingDoer) { /* set fields */ }

    public MyResult DoAThing(SomeObject someObject)
    {
        var someObject = _objectGenerator.Create(someArg);

        return DoAThingAndLogIt(someObject);
    }
}
```

So we decide whether the logging is included outside of all of this, when we are wiring up our dependencies.

```c#
new LoggingThingDoer(new ThingDoer(new SetterUpper(), new Worker()), new Logger())
```

Of course you don't have to new-up the dependencies. [Autofac supports adding decorators with an easy one-liner](https://autofac.readthedocs.io/en/latest/advanced/adapters-decorators.html#decorators) and similar things are possible with other DI containers.

```c#
builder.RegisterDecorator<LoggingThingDoer, IThingDoer>();
```

You could even make this configurable if you like. For example, you could register some verbose logging decorators only when a certain flag is set in config.

# Pipelines

We've solved problems 1-4 so far, but it can be quite cumbersome to write decorators for every class you'd usually log from. We're still going to review and talk about what should be logged, get the wrong level or property name, etc. We might need something more generic.

Depending on what you want to "wrap around", there might already be a way to write one logging class for many uses.

ASP.NET Core has [Middlewares](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware/) or [Filters](https://docs.microsoft.com/en-us/aspnet/core/mvc/controllers/filters). Both of these ideas allow you to wrap around the call to the `next()` method in the pipeline, such as a controller action. This is the same idea. A global `LoggingActionFilter` is the same as a logging decorator for every controller.

```c#
public class LoggingActionFilter : IAsyncActionFilter
{
    private readonly ILogger _logger;

    public LoggingActionFilter(ILogger logger)
    {
        _logger = logger;
    }
    
    public async Task OnActionExecutionAsync(ActionExecutingContext context, ActionExecutionDelegate next)
    {
        _logger.Debug("Handling HTTP request.");

        try
        {
            await next();

            _logger.Information("Handled HTTP request.");
        }
        catch (Exception exception)
        {
            _logger.Error("Error handling HTTP request.", exception);
            throw;
        }
    }
}
```

MediatR and NServiceBus can have [Pipeline Behaviors](https://lostechies.com/jimmybogard/2014/09/09/tackling-cross-cutting-concerns-with-a-mediator-pipeline/) registered, which is the same idea where the `next()` function calls deeper into the pipeline.

```c#
public class LoggingBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
{
    private readonly ILogger _logger;

    public LoggingBehavior(ILogger logger)
    {
        _logger = logger;
    }

    public async Task<TResponse> Handle(TRequest request, CancellationToken cancellationToken, RequestHandlerDelegate<TResponse> next)
    {
        _logger.Debug($"Handling {typeof(TRequest)}.");

        try
        {
            var result = await next();

            _logger.Information($"Handled {typeof(TRequest)}.");

            return result;
        }
        catch (Exception exception)
        {
            _logger.Error($"Error handling {typeof(TRequest)}.", exception);
            throw;
        }
    }
}
```

For outbound HTTP requests you can use [HttpClient Message Handlers](https://docs.microsoft.com/en-us/aspnet/web-api/overview/advanced/httpclient-message-handlers) and implement a `DelegatingHandler`. Again, same idea.

In Entity Framework you can implement a `DbCommandInterceptor` to do the same thing, although there are [alternative approaches tailored to logging](https://docs.microsoft.com/en-us/ef/core/logging-events-diagnostics/) which would be better.

# Type Interception

Okay, but what if there isn't a pipeline available? What if it's just a classic call to an interface I've defined in the same library? Now we're getting deeper into the realm of **Aspect-Oriented Programming**.

This is where we add some magic and, I admit, it can get a little scary, but in the pursuit of ridding our codebase of boilerplate logging code, we must accept the trade-off:

[![What would you like to complain about? Too much magic. Too much boilerplate.](/images/quotes/magic-vs-boilerplate.png)](https://twitter.com/phillip_webb/status/705909774001377280)

The [Castle.Core](https://github.com/castleproject/Core) library provides a DynamicProxy framework that can **create dynamic runtime types that decorate other classes**. These proxy types use interceptors for every method call, which have similar features to the filters/behaviors/handlers above.

```c#
public class LoggingInterceptor : IInterceptor
{
    private ILogger _logger;

    public LoggingInterceptor(ILogger logger)
    {
        _logger = logger;
    }

    public void Intercept(IInvocation invocation)
    {
        _logger.Debug($"Making method call to {invocation.Method.Name}.");

        try
        {
            invocation.Proceed();

            _logger.Information($"Completed method call to {invocation.Method.Name}.");

            return invocation.ReturnValue;
        }
        catch (Exception exception)
        {
            _logger.Error($"Error sending HTTP request to {request.RequestUri}.", exception);
            throw;
        }
    }
}
```

You may have noticed that interceptors are not asynchronous, but you can get around this with some [reflection magic](https://github.com/castleproject/Core/blob/master/docs/dynamicproxy-async-interception.md) too.

Something this generic would need a custom approach to dealing with which properties to log. You could use reflection to iterate through properties on the method parameters and return value, log them all, use a convention-based approach, or perhaps check for a custom attribute indicating that you want that property to be logged.

The `Castle.Core.DynamicProxy.IProxyGenerator` interface is used to create the proxy types at runtime. [Autofac supports wiring this up automatically](https://autofac.readthedocs.io/en/latest/advanced/interceptors.html) though its DynamicProxy extension.

```c#
builder.RegisterType<LoggingInterceptor>()
       .As<IInterceptor>();

builder.RegisterType<SomeType>()
       .As<ISomeInterface>()
       .EnableInterfaceInterceptors();
```

Type interception relies on creating runtime types where the decorator pattern can be implemented, so not all method calls can be intercepted.
- **Interface methods** can be intercepted because the runtime type implements the decorator pattern. It implements the interface and also encapsulates the inner type that it forwards its calls to. Use `.EnableInterfaceInterceptors()` for this approach.
- **Virtual methods on non-sealed classes** can also be intercepted, because the proxy type derives from the base type, overrides those methods, and calls the `base` method. Use `.EnableClassInterceptors()` for this.

# Abstracting Even Further

Haven't we gone far enough? Well, maybe, but in for a penny.

Maybe now you have a handful of very similar looking behaviors, interceptors, middlewares, etc. Maybe you have logging absolutely everywhere.

![Log all of the things](/images/memes/log-all-the-things.png)

## Adapters

It's possible to abstract all of these pipelines and write adapters for a single aspect interface. This is the premise behind [Aspectos](https://github.com/connellw/Aspectos); a mini library I put together. All you do is implement `IAspect` once, then write that up to all of the behaviors and middlewares listed above.

```c#
public class LoggingAspect : IAspect
{
    private readonly ILogger _logger;

    public LoggingAspect(ILogger logger)
    {
        _logger = logger;
    }

    public async Task InvokeAsync(IInvocationContext context)
    {
        _logger.Debug($"Making method call to {context.Method.Name}.");

        try
        {
            await context.InvokeAsync();

            _logger.Information($"Completed method call to {invocation.Method.Name}.");

            return invocation.ReturnValue;
        }
        catch (Exception exception)
        {
            _logger.Error($"Error sending HTTP request to {request.RequestUri}.", exception);
            throw;
        }
    }
}
```

## IL Weaving

Type interception wasn't scary enough for you? Want to log some `private` method calls? [PostSharp modifies the MSIL instructions](https://www.postsharp.net/aop.net/msil-injection) after the C# code has been compiled. Your source code remains exactly the same, without the logging code, but the compiled code is as if the log lines were written directly into your class.

# Is Everywhere Useful?

But should you? I'd actually say, probably not. We still haven't addressed problem #7 yet - the log and rethrow anti-pattern.

In my opinion, logging all these verbose lines feels old-fashioned. I tend to write web applications, not console applications. My error details will be returned to my API when using a development environment. For trace information, we have debuggers nowadays. The scale we work at in production might mean such verbose logging is impractical anyway.

Instead, I'd prefer **one log line per request scope**, whether that's an HTTP request into my application, pulling a message from a bus, or handling a scheduled job.

Only log when the processing has finished, ensuring relevant information about the request and whether it was successful is included. I don't need an "about to do a thing" line at the start. I only ever find these are needed for one of two reasons:

1. To time how long something takes. Just time it within the application and add an `elapsed_time` property to line at the end to achieve the same thing.

2. To debug scenarios where the starting line is logged but not the end. This is normally because:
    - An exception was thrown or something happened to divert the control flow away from logging the ending line. Use a `finally` block.
    - You're stuck in an infinite loop somewhere. You could log a timeout for the request here.
    - Your application has totally crashed. This is a real edge case.

There may be a little value in covering the edge cases, but in my opinion, it's not worth the trade-off. I prefer cleaner code, cleaner tests and cleaner logs. If ever I feel I need detailed trace information, I can still use the logs I have to recreate a request locally or write a failing test.