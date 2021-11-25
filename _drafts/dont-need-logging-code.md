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

It's all about [drawing boxes](/drawing-boxes). The developer who wrote those log lines defined the concept of "a thing", but only in the logs. If you saw the same thing with code comments, you would probably want to refactor the code into a new `DoAThing()` method. The structure of our code should match how we break down the problem in our heads. Someone has broken down part of this problem into the concept of "a thing", so I think this should be its own method.

It's not just about grouping things together and hiding them in a different part of the file. A method scopes the variables declared within that section of code so they cannot interfere with those outside. Extracting a method forces us to explicitly define which arguments are passed into the method and which are returned from it. The boundary is clearer and more rigid, which helps us avoid spaghetti code as it develops.

##### 2. There are too many constructor parameters

In his book Clean Code, Uncle Bob makes the case for an upper limit of three parameters for all functions.

> The ideal number of arguments for a function is zero (niladic). Next comes one (monadic) followed closely by two (dyadic). Three arguments (triadic) should be avoided where possible. More than three (polyadic) requires very special justification—and then shouldn't be used anyway.
> -- Robert C. Martin

Constructors are essentially a function that creates an instance of a class. The constructor parameters show us what the class depends on - they're usually the first thing you see when opening a class file and the signature of the constructor is an indicator of what a class' responsibility is. Is the class responsible for logging? Is that it's single responsibility? Not in this case.

##### 3. We have not separated cross cutting concerns

Logging is called a **cross-cutting concern** because it is often used in many parts of our application: presentation, business logic, database code; they all want to write log lines. When we talk about [separation of concerns](https://en.wikipedia.org/wiki/Separation_of_concerns), the word 'concern' is not a coincidental homophone - it's the same thing.

Sometimes it's difficult to define what a concern really is when we're trying to separate them, but not in this case - examples of cross-cutting concerns are all over the internet, and logging is usually the first given.

##### 4. Should the tests have to test logging?

Actually this is just points 2 and 3 repeated. Often I see the `ILogger` interface mocked out and ignored in unit tests. Just because it keeps the runtime happy. It's almost as if the class is not really *concerned* with logging, right?

##### 5. We spend so much time talking about it

I've worked in teams where we spend a *huge* chunk of our time writing log lines, discussing what to log, what level (Debug, Info, Warn) to log which things at, commenting on PRs suggesting adding another property. We spent time discussing logging standards and wrote documents explaining them. And we still got it wrong anyway - we'd have to raise new PRs to rename a log property to make it consistent with another part of the application. What a colossal waste of time.

Do you know what, this is about separating concerns too. We managed to write the rules in one place. Couldn't we implement them in one place? Perhaps one class that has the single responsibility of logging? And never speak about it unless we are modifying the code that is responsible for logging?

##### 6. There's so much boilerplate

There's so much of it! Sometimes more than half the lines of code are related to logging. It clutters the codebase in such a way that important logic gets obfuscated and is harder to read.

Especially if we've defined standards for how to write this stuff, we end up seeing very similar logging code everywhere. Sometimes we tune it out and glance past it when reviewing it because we're so familiar with the general look of it.

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

Note that the method's inputs and outputs are exactly the same as the inner method. All we've done is wrap around the method and logged stuff. It would be easy to toggle between the logged and non-logged version of this method in the calling code!

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

We've solved problems 1-4 so far, but it can be quite cumbersome to write decorators for every class that you want to write logs from and we're still going to end up seeing all these decorators appear in PRs. We're still going to talk about what should be logged, get the wrong level or property name, etc. We need something more generic.

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

For outbound HTTP requests you can use [HttpClient Message Handlers](https://docs.microsoft.com/en-us/aspnet/web-api/overview/advanced/httpclient-message-handlers) and implement a `DelegatingHandler`. Again, same idea. In fact, Logging is even an example Microsoft give in their tutorial.

```c#
public class LoggingHandler : DelegatingHandler
{
    ILogger _logger;

    public LoggingHandler(ILogger logger)
    {
        _logger = logger;
    }

    protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
    {
        _logger.Debug($"Sending HTTP request to {request.RequestUri}.");

        try
        {
            var response = await base.SendAsync(request, cancellationToken);

            _logger.Information($"Received response from {request.RequestUri}.");

            return response;
        }
        catch (Exception exception)
        {
            _logger.Error($"Error sending HTTP request to {request.RequestUri}.", exception);
            throw;
        }
    }
}
```

With all of these, logging is the example used, but there are many other cross-cutting concerns you can add to these pipelines such as validation, caching, retrying, tracing. The concerns don't need to be cross-cutting, for example I like to commit my database transaction after MediatR requests are handled, for which I register a Pipeline Behavior.

# Type Interception

Okay, but what if there isn't a pipeline available? What if it's just a classic call to an interface I've defined in the same library? This is where we mix in a bit of magic and, I admit, it can get a little scary, but in the pursuit of ridding our codebase of boilerplate logging code, we must accept the trade-off:

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

- hook up with Autofac.

You may have noticed that interceptors are not asynchronous, but you can get around this with some [reflection magic](https://github.com/castleproject/Core/blob/master/docs/dynamicproxy-async-interception.md) too.

Type interception relies on creating runtime types where the decorator pattern can be implemented, so not all method calls can be implemented.
- **Interface methods** can be intercepted because the runtime type implements the decorator pattern. It implements the interface and also encapsulates the inner type that it forwards its calls to.
- **Virtual methods on non-sealed classes** can also be intercepted, because the proxy type derives from the base type and overrides those methods. These types have the ability to wrap around the inner method and call the `base.` method.

# IL Weaving

Still want to log those `private` method calls? Type interception wasn't scary enough for you? [PostSharp modifies the MSIL instructions](https://www.postsharp.net/aop.net/msil-injection) after the C# code has been compiled. Your source code remains exactly the same, without the logging code, but the compiled code is as if the log lines were written directly into your class.

# Abstracting Even Further

Haven't we gone far enough? Well, maybe, but in for a penny.

Maybe now you have a handful of very similar looking behaviors, interceptors, middlewares, etc. Maybe you have logging absolutely everywhere.

![Log all of the things](/images/memes/log-all-the-things.png)

It's possible to abstract all these behaviors into one super-behavior. This is the premise behind [Aspectos](https://github.com/connellw/Aspectos); a mini library I put together. All you do is implement a single `IAspect`, then you can hook that up to all of the behaviors and middlewares listed above.

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

But should you? I'd actually say, probably not. We still haven't addressed problem #7 yet - the log and rethrow anti-pattern.

In my opinion, logging all these verbose lines feels old-fashioned. I tend to write web applications, not console applications. My error details will be returned to my API when using a development environment. For trace information, we have debuggers nowadays. The scale we work at in production might mean such verbose logging is impractical anyway.

Instead, I'd prefer **one log line per request scope**, whether that's an HTTP request into my application, pulling a message from a bus or queue, or handling a scheduled job.

I don't need an "about to do a thing" line before the request. I only ever find these are needed for one of two reasons:
- To time how long something takes. Just add an `elapsed_time` property to the line that logs when the request has completed to achieve this.
- To debug scenarios where the starting line is logged but not the end. This is because:
    - An exception was thrown or something happened to divert the control flow away from logging the ending line. Use a `finally` block.
    - You're stuck in an infinite loop somewhere.
    - Your application has totally crashed. This is a real edge case.

------

end of post

------

- Link from Domain-Driven Boundaries
> I have made the mistake of letting the domain model be influenced by technical requirements: maintainability, performance, connectivity with other domains, or [monitoring concerns](/dont-need-logging-code.md).

- Make a point about extracting being too difficult then it's probably already spaghetti.