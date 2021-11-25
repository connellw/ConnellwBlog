---
layout: post
title: Using Logs to Refactor
tags: logging refactoring
---

One of my pet peeves in software is seeing so much boilerplate logging code everywhere. It mainly frustrates me because it's usually very easy to avoid, and avoiding it almost always leads to better code. It's as if logs encourage poor separation of concerns.

- Sometimes it's difficult to refactor. This is because it's already spaghetti code. Methods scope variables etc.

```c#
public bool IsReady()
{
    if(certainObject.HasSomeFlag)
    {
        _logger.Information("Flag is set. Not ready.");

        return false;
    }

    try
    {
        _processAThing.YeahYeah();

        _logger.Information("This was fine. We're ready.");

        return true;
    }
    catch (SpecificException exception)
    {
        _logger.Error("Specific error doing a thing.");
        
        return false;
    }
}
```

```c#
public ReadyState GetReadyState()
{
    if(certainObject.HasSomeFlag)
    {
        return ReadyState.HasThatFlag;
    }

    try
    {
        _processAThing.YeahYeah();

        return ReadyState.Ready;
    }
    catch (SpecificException exception)
    {        
        return ReadyState.SpecificError;
    }
}
```