---
layout: post
title: Look, No Controllers
tags: aop cqrs
---

Those that have worked with me will know I'm a big advocate for Aspect-Oriented Programming. I generally like to "bolt on" concerns that sit outside of the main logic using the **decorator pattern** or a behaviour that can wrap around a class or method, which follows the Open/Closed principle.

When following this principle, I've found that changing a public method signature is more of a chore. Adding new parameters means I have to update the decorators too, most of which will not even use the shiny new parameter anyway. Instead, these methods are more maintainable if we use a single object parameter and add properties to that object when needed.

```c#
public Task DoTheThing(ThingContext context)
```

So in controllers, rather than adding a new parameter with a `[FromRoute]` or `[FromQuery]` attribute, it's possible to just have one object and add the attributes to a property of that object. Following this rule means I don't break as many unit tests when I change the action's signature too.

But now all my controller actions look the same. They're all just boilerplate methods that follow the same step-by-step process. Which got me asking,
> Can I write this process once? Can I write a generic pipeline for every HTTP request? Can I get rid of the controllers?

I want each request to follow the same basic pipeline:
1. Bind the HTTP request to a model
2. Pass that through a validator
3. Send it to [MediatR](https://github.com/jbogard/MediatR)
4. Map the result to some response

I toyed with Minimal APIs, some kind of generic controller, but eventually I thought it might be fun to roll my own. Introducing **[Controlless](https://github.com/connellw/Controlless)**.

# Binding

All requests are just DTOs (data transfer objects). There are no controllers to perform the model binding though. Instead, just put the familiar-looking `[HttpGet]`, `[HttpPost]`, `[FromRoute]` and `[FromBody]` attributes straight onto the request object.

```c#
[HttpGet("/films/{id}/actors")]
public class GetFilmActorsRequest
{
    [FromRoute("id")]
    public string FilmId { get; set; }

    [FromQuery("page")]
    public int Page { get; set; }
}
```

Those attributes are actually replicas in the `Controlless` namespace. It would be pretty neat if this could hook straight into MVC's model binding.

# Handling

Once that request object exists, it is passed into whatever implementation of `IRequestHandler<GetFilmActorsRequest>` is registered, which should be familiar to developers who have used MediatR or NServiceBus.

This is where we're going to implement a generic pipeline as one generic class.

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

MediatR has [it's own pipeline](https://lostechies.com/jimmybogard/2014/09/09/tackling-cross-cutting-concerns-with-a-mediator-pipeline/), where usually I would will also dispatch domain events and commit the unit of work. We could easily to move our validation to that pipeline instead.

The main difference is that the MediatR request handler must return the defined response type, whereas the Controlless handler can return any object.

# Responding

Once a response object has been returned from a handler, we need a way to write it back to the HTTP stream. The most obvious is to register a generic `JsonResponseWriter` that by default serializes the response and is included by default.

For some response objects, however, we might want to set different status codes. For specific cases, we can register more specific `IResponseWriter<>` implementations which will be used as priority. The priority order is however the DI container works, which usually means you'll want to register the specific handler after the generic handler.

```c#
internal class ValidationFailureJsonResponseWriter : IResponseWriter<List<ValidationFailure>>
{
    public async Task Write(List<ValidationFailure> responseObject, HttpResponse response)
    {
        response.StatusCode = 400;
        await response.WriteAsJsonAsync(responseObject);
    }
}
```

# Exceptions and Logging

Errors from the handlers will propagate up to the [normal ASP.NET Core exception handlers](https://docs.microsoft.com/en-us/aspnet/core/web-api/handle-errors?view=aspnetcore-5.0).

Similarly, there's no special logging or tracing built into the framework. All of these concerns are cross-cutting and can be handled in the ASP.NET Core pipeline, such as with Middlewares.

Together, ASP.NET Core middlewares, Controlless and MediatR Pipeline Behaviors create a Aspect-Oriented feel and a very modular Web API solution that strictly obeys the Single Responsibility and Open/Closed principles.