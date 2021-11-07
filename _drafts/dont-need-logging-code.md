---
layout: post
title: You Don't Need Logging Code
tags: aop logging decorator
---

This an opinion I've held and maintained for some years now. The title is a little disingenuous. What I actually mean is **classes should not be concerned with what they need to log**.

```csharp
public void DoImportantLogic(string someArg)
{
    var someObject = _objectGenerator.Create(someArg);

    _logger.Debug("About to do a thing.");

    try
    {
        _thingDoer.DoThing();
        _anotherDoer_.DoSomethingElse();

        _logger.Information("Done a thing.");
    }
    catch (Exception exception)
    {
        _logger.Error("Error doing thing.", exception);
        throw;
    }
}
```

- More drawing boxes
- Extra box for "stuff I want to log"
    - Just use the existing boxes.
    - Same as code comments

Single Responsibility. Separation of Concerns.



```csharp
public void DoImportantLogic(string someArg)
{
    var someObject = _objectGenerator.Create(someArg);

    _logger.Debug("About to do a thing.");

    try
    {
        _thingDoer.DoThing();
        _anotherDoer_.DoSomethingElse();

        _logger.Information("Done a thing.");
    }
    catch (Exception exception)
    {
        _logger.Error("Error doing thing.", exception);
        throw;
    }
}
```



The time consumed talking about it. What to log, when to log them, what level to log at.

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