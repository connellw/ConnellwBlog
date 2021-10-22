---
layout: post
title: CQRS, DDD and Event Sourcing
tags: cqrs ddd
---

Three independent concepts that are often thought of together.

... ven diagram ?

# Domain Events

In **Domain-Driven Design**, domain events are the things that happen that are of interest to a domain expert. They are written in the ubiquitous language of the domain.

```c#
public class SomethingHappenedEvent : IDomainEvent
{
    // props
}
```

These are raised when methods are called making changes to domain **aggregates**. When domain events are dispatched, their handlers usually make more changes to other aggregates.

```c#
internal class SomethingHappenedEventHandler : IDomainEventHandler<SomethingHappenedEvent>
{
    // ctor

    public async Task Handle(SomethingHappenedEvent domainEvent)
    {
        var otherAggregate = _otherRepository.Get(domainEvent.SomeProperty);

        otherAggregate.DoAnotherThing();
    }
}
```

## Transaction Boundary

Whether or not domain events should be handled within the same data transaction is disagreed amongst architects. Some argue that the aggregate is the consistency boundary.

> Any rule that spans Aggregates will not be expected to be up-to-date at all times. Through event processing, batch processing, or other update mechanisms, other dependencies can be resolved within some specific time.
> -- <cite>Eric Evans</cite>

However, Jimmy Bogard proposes [dispatching domain events immediately before committing the transaction](https://lostechies.com/jimmybogard/2014/05/13/a-better-domain-events-pattern/), so the side-effects of handling the events are all committed together as one unit of work, enforcing immediate consistency. I like this approach myself. It's simpler to consider the application layer boundary as the transaction boundary. I send my application a command, all the work is done, the transaction is committed, the command has been fulfilled.

![CQRS command sequence-ish diagram](/images/diagrams/sequence-ish-command.png)

## MediatR Pipeline

When all commands behave this way, it's easy to define a single behaviour, writing this once and not violating the Single Responsibility Principle.

We can implement our own [decorator pattern](https://www.dofactory.com/net/decorator-design-pattern). Or, if we use [MediatR](https://github.com/jbogard/MediatR) to send commands to our application, we can implement a **pipeline behavior**.

```c#
internal class CommandPipelineBehavior : IPipelineBehavior
{
    // ctor

    public async Task<TResponse> Handle(TRequest request, CancellationToken cancellationToken, RequestHandlerDelegate<TResponse> next)
    {
        await next();

        await _eventDispatcher.DispatchAll();

        await _unitOfWork.Commit();
    }
}
```

## After Commit

Sometimes work must be done *after* the transaction has been committed, such as calling external APIs. In these cases, Kamil Grzybek proposes [a separate layer for domain event notifications](http://www.kamilgrzybek.com/design/how-to-publish-and-handle-domain-events/). This is a nice distinction and gives us the choice to handle inside or outside the transaction in each handler.

Because the transaction has already been committed, we will need to **schedule a job** to ensure the notification is processed. Otherwise the notification handling could fail, or our application could crash, and our application could be left in an inconsistent state. We can implement an Outbox pattern, which writes the notification to a data table as part of the same transaction and processed it later. One way of achieving this is by implementing a **generic domain event handler** for all domain events.

```c#
internal class DomainEventNotificationOutboxHandler<TEvent> : IDomainEventHandler<TEvent>
    where TEvent : IDomainEvent
{
    // ctor

    public async Task Handle(TEvent domainEvent)
    {
        _outbox.AddNotification(domainEvent);
    }
}
```

A separate process may routinely dispatch events from this outbox to the notification handlers. Exceptions in these handlers will not cancel the transaction, because it has already been committed. We will need to retry if these fail and raise an alert if it continues.

## Eventual Consistency

When we make further changes outside of the transaction boundary, either to our own aggregates or to third party systems via API calls, the command returns to us before all the work is done.

- Changes are propagated
- Commands that return acceptance not fulfillment ?

# Integration Events

Integration events are similar to domain events, but serve a different purpose.

[some link](https://docs.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/domain-events-design-implementation#domain-events-versus-integration-events)

- Must be serialised
- Makes sense to schedule job inside transaction

- Public Contracts maybe?

## Event Stream

- Event store
- Generic domain event handler

# Event Sourcing

- Internal Read Store = different tables, within transaction, consistent

- External read store = different databases, outside transaction, resilient, eventually consistent

# Eventual Consistency

- Commands that return acceptance not fulfillment