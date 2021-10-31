---
layout: post
title: API with ASP.NET Middleware Pipeline
tags: aop cqrs
---

Those that have worked with me will know I'm a big advocate for Aspect-Oriented Programming. I generally like to "bolt on" concerns that sit outside of the main logic into decorators or any pattern that can wrap around a class or method, which follows the Open/Closed principle.

When following this principle and writing many decorators, I've found that changing a public method signature is more of a chore. Adding new parameters means I have to update the decorators too, most of which will not even use the shiny new parameter anyway. Instead, these methods are more maintainable if we use a single object parameter and add properties to that object when needed.

```c#
c# example
```

This leads me to controllers. Rather than adding a new parameter with a `[FromRoute]` or [FromQuery]` attribute, it's possible to just have one object and add the attributes to a property of that object. Following this rule means I don't break as many unit tests when I change the Action's signature too.

But now all my controller actions look the same. They're all now just boilerplate methods that follow the same rules:
1. MVC model binding passes in a request object.
2. Validate that object with FluentValidation
3. Map that object to a MediatR request and send it
4. Catch errors and return a generic error response
5. Map the MediatR response to an API response and return that

Which got me asking. Can I write this once? Can I write up that pipeline somewhere, and have this stuff automatically happen for every request?

I toyed with Minimal APIs, some kind of generic controller, but eventually I thought it might be fun to roll my own. It might not even be too complicated, especially if I can hook into MVC's model binding. Introducing [Uncontrollable](https://github.com/connellw/Uncontrollable).

# Binding

All requests are just DTOs (data transfer objects).

```c#
public class UpdateThingDetailsRequest
{
    public string ThingId { get; set; }

    public string Name { get; set; }

    public string Type { get; set; }
}
```

For each request, register a **RequestBinder** which maps parts of the request body, route, or query.

I think it would be cool in future to hook into MVC's model binding and have this all work automatically using the `[FromBody]`, `[FromRoute]` and `[FromQuery]` attributes.

# Handling

## Validation

## MediatR

MediatR also has it's own pipeline. Perhaps we actually want to move our validation there.

- Dispatch domain events
- Commit unit of work

# Responding

Once a response object has been returned from a handler, we need a way to write it back to the HTTP stream. The most obvious is to register a generic `JsonResponseWriter` that by default serializes the response.

For some response objects, however, we might want to set different status codes. For specific cases, we can register more specific `IResponseWriter<>` implementations which will be used as priority. The priority order is however the DI container works, which usually means you'll want to register the specific handler after the generic handler.

# Exceptions

If the handler throws an exception

------

ASP.NET Core but without a controller.
- Maybe note Minimal APIs?

Middlewares
- Logging & Monitoring
- Error Handling
- Find endpoint and deserialize DTO
- Validation (FluentValidation)
- Map to Command/Query (AutoMapper)

Mediator
- DispatchEvents
- UnitOfWork