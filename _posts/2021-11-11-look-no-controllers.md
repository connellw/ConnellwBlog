---
layout: post
title: Look, No Controllers
tags: aop cqrs
---

Those that have worked with me will know I'm a big advocate for Aspect-Oriented Programming. I generally like to "bolt on" concerns that sit outside of the main logic using the **decorator pattern** or some kind of behaviour that can wrap around a class or method, which follows the Open/Closed principle.

When following this principle, I've found that changing a public method signature is more of a chore. Adding new parameters means I have to update the decorators, most of which will not even use my shiny new parameter anyway. Instead, these methods are more maintainable if we use a single object parameter and add properties to that object when needed.

```c#
public Task DoTheThing(ThingContext context)
```

So in controllers, rather than adding a new parameter with a `[FromRoute]` or `[FromQuery]` attribute, I opt just have one object and add the attributes to a property of that object. Following this rule means I don't break as many unit tests by changing the action's signature too.

But now all my controller actions look the same. They're all just boilerplate methods that follow the same step-by-step process. Which got me asking,
> Can I write this process once? Can I write a generic pipeline for every HTTP request? Can I get rid of the controllers?

I want each request to follow the same basic pipeline:
1. Bind the HTTP request to a model
2. Pass that through a validator
3. Send it to [MediatR](https://github.com/jbogard/MediatR)
4. Map the result to some response

I toyed with Minimal APIs, some kind of generic controller, but eventually I thought it might be fun to roll my own. Introducing **[Controlless](https://github.com/connellw/Controlless)**.

# Binding

All requests are just DTOs (data transfer objects). There are no controllers to perform the model binding though. Instead, just put the `[RouteGet]` or `[RoutePost]` attributes straight onto the request object.

```c#
[RouteGet("/films/{id}/actors")]
public class GetFilmActorsRequest
{
    [FromRoute("id")]
    public string FilmId { get; set; }

    [FromQuery("page")]
    public int Page { get; set; }
}
```

That's it! The library [dynamically generates a controller for you](https://www.strathweb.com/2018/04/generic-and-dynamically-generated-controllers-in-asp-net-core-mvc/).

The model is passed into a controller action, so all of MVC's model binding attributes work the same.

# Handling

Once that request object is created, it is passed into whatever implementation of `IRequestHandler<GetFilmActorsRequest>` is registered - a pattern that should be familiar to developers who have used MediatR or NServiceBus.

This is where we're going to implement a generic pipeline as one generic class. All request models will then use this same generic implementation.

```c#
internal class GenericRequestHandler<T> : IRequestHandler<T>
{
    private readonly IValidator<T> _validator;
    private readonly IMediator _mediator;

    public GenericRequestHandler(IValidator<T> validator, IMediator mediator)
    {
        _validator = validator;
        _mediator = mediator;
    }

    public async Task<object> Handle(T request, CancellationToken ct)
    {
        var validationResult = _validator.Validate(request);

        if(validationResult.IsValid == false)
            return validationResult.Errors;

        return await _mediator.Send(request, ct);
    }
}
```

MediatR has [it's own pipeline](https://lostechies.com/jimmybogard/2014/09/09/tackling-cross-cutting-concerns-with-a-mediator-pipeline/), where usually I would also put cross cutting concerns such as dispatching domain events and committing the unit of work. We could easily to move our validation to that pipeline instead.

The main difference is that the MediatR request handler must return the defined response type, whereas the Controlless handler can return any object, so we can return the validation errors rather than throwing an exception.

# Responding

For some responses, we may want to use a different status code or add headers.

Returning from the handler is exactly the same as returning from a controller action in MVC. We could return any object we like, including return an `IActionResult`. We could even write a generic mapper than maps a response to an action result.

Or we could use an `IResultFilter`. You can place action filters on the request DTO itself, or register them globally. Everything works just like it would do in a controller.

```c#
public class ValidationFailureResultFilter : IResultFilter
{
    public void OnResultExecuting(ResultExecutingContext context)
    {
        if(context.Result is ObjectResult objectResult
            && objectResult.Value is List<ValidationFailure>)
        {
            context.HttpContext.Response.StatusCode = 400;
        }
    }

    public void OnResultExecuted(ResultExecutedContext context)
    {
    }
}
```

Or, specifically for status codes, there a result filter built into Controlless. Just plop the `[StatusCode(400)]` on your response object type.

# Exceptions and Logging

Errors from the handlers will propagate up to the [normal ASP.NET Core exception handlers](https://docs.microsoft.com/en-us/aspnet/core/web-api/handle-errors?view=aspnetcore-5.0), or another option is to use [exception filters](https://docs.microsoft.com/en-us/aspnet/core/mvc/controllers/filters?view=aspnetcore-5.0#exception-filters).

Similarly, there's no special logging or tracing built into the framework. All of these concerns are cross-cutting and can be handled in the ASP.NET Core pipeline, such as with Middlewares.

Together, ASP.NET Core middlewares, action filters, Controlless and MediatR Pipeline Behaviors create a Aspect-Oriented feel and a very modular Web API solution that strictly obeys the Single Responsibility and Open/Closed principles. We write all aspects of our API once, and the frameworks and pipeline reuse those rules for the whole API.