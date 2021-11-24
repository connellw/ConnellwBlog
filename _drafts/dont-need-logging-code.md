---
layout: post
title: You Don't Need Logging Code
tags: aop logging decorator
---

This an opinion I've held and maintained for some years now. The title is a little disingenuous. What I actually mean is that classes should not be concerned with what they need to log. Take this example.

```csharp
public void DoImportantLogic(string someArg)
{
    var someObject = _objectGenerator.Create(someArg);

    _logger.Debug("About to do a thing.");

    try
    {
        _setterUpper.DoInitialSetup(someObject.SomeProperty);
        _worker.DoSomeActualWork(someObject);

        _logger.Information("Done a thing.");
    }
    catch (Exception exception)
    {
        _logger.Error("Error doing a thing.", exception);
        throw;
    }
}
```

I see this sort of code far too often. Let me explain what's wrong with it.

### 1. There's an implicit boundary

It's all about [drawing boxes](/drawing-boxes). The developer who wrote those log lines defined the concept of "a thing", but only in the logs. If you saw the same thing with code comments, you would probably want to refactor the code into a new `DoAThing()` method. After all, [the best comment is a good name for a method or class](https://refactoring.guru/smells/comments).

So let's make our code structure match how we break down this problem in our heads, and let's introduce the concept of "a thing" in our code.

```csharp
public void DoImportantLogic(string someArg)
{
    var someObject = _objectGenerator.Create(someArg);

    _logger.Debug("About to do a thing.");

    try
    {
        DoAThing(someObject);

        _logger.Information("Done a thing.");
    }
    catch (Exception exception)
    {
        _logger.Error("Error doing a thing.", exception);
        throw;
    }
}

private void DoAThing(SomeObject someObject)
{
    _setterUpper.DoInitialSetup(someObject.SomeProperty);
    _worker.DoSomeActualWork(someObject);
}
```

### 2. There are too many constructor parameters

In his book Clean Code, Uncle Bob makes the case for an upper limit of three parameters for all functions.

> The ideal number of arguments for a function is zero (niladic). Next comes one (monadic) followed closely by two (dyadic). Three arguments (triadic) should be avoided where possible. More than three (polyadic) requires very special justification—and then shouldn't be used anyway.
> -- Robert C. Martin

Constructors are essentially a function that creates an instance of a class. The constructor parameters show us what the class depends on - they're usually the first thing you see when opening a class file and the signature of the constructor is an indicator of what a class' responsibility is. Is the class responsible for logging? Is that it's single responsibility? Not in this case.

### 3. We have not separated cross cutting concerns

Logging is called a **cross-cutting concern** because it is often used in many parts of our application: presentation, business logic, database code; they all want to write log lines. When we talk about separating concerns, the word 'concern' is not a coincidental homophone, this is the same thing. Cross-cutting concerns are certainly not exempt and should be separated like all other concerns.

### 4. Should the tests have to test logging?

Actually this is just points 2 and 3 repeated. Often I see the `ILogger` interface mocked out and ignored in unit tests. Just because it keeps the runtime happy. It's almost as if the class is not really *concerned* with logging, right?\

### 5. We spend so much time talking about it

I've worked in teams where we spend a *huge* chunk of our time writing log lines, discussing what to log, what level (Debug, Info, Warn) to log which things at, commenting on PRs suggesting adding another property to some log line. We spent even more time in meetings discussing 'logging standards' where we wrote several documents explaining the logging rules. What a colossal waste of time.

- Ensure properties are consistent.

Do you know what, this point is also Single Responsibility. We managed to write standards of sorts, so why couldn't we just write a piece of code that is responsible for logging, which stuck to those standards. **Once**.


```csharp
public void DoAThing(string someArg)
{
    var someObject = _objectGenerator.Create(someArg);

    _logger.Debug("About to do a thing.");

    try
    {
        _setterUpper.DoInitialSetup();
        _worker.DoAnImportantTask(someObject);

        _logger.Information("Done the thing.");
    }
    catch (Exception exception)
    {
        _logger.Error("Error doing the thing.", exception);
        throw;
    }
}
```



Too much boilerplate.

Cross Cutting Concerns.

# Decorators

The Decorator pattern 

Custom decorators for classes
DI Framework Support.
- Autofac's RegisterDecorator
Make it configurable.

# Pipelines

Middlewares. PipelineBehaviors.
DelegatingHandler

# ILWeaving vs Type Interception

PostSharp. Scary?

- Aspectos github

------

- Link from Domain-Driven Boundaries
> I have made the mistake of letting the domain model be influenced by technical requirements: maintainability, performance, connectivity with other domains, or [monitoring concerns](/dont-need-logging-code.md).