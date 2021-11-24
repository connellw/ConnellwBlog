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

##### 6. Log and rethrow

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

ASP.NET Core has Middlewares or Filters. Both of these ideas allow you to wrap around the call to the `next()` method in the pipeline, such as a controller action. This is the same idea. A global `LoggingActionFilter` is the same as a logging decorator for every controller.

```c#

```

Middlewares. PipelineBehaviors.
DelegatingHandler

# Type Interception

- Aspectos github
- Too much boilerplate. Boilerplate vs magic.

# ILWeaving

PostSharp. Scary?
- Too much boilerplate. Boilerplate vs magic.

------

- Link from Domain-Driven Boundaries
> I have made the mistake of letting the domain model be influenced by technical requirements: maintainability, performance, connectivity with other domains, or [monitoring concerns](/dont-need-logging-code.md).

- Make a point about extracting being too difficult then it's probably already spaghetti.