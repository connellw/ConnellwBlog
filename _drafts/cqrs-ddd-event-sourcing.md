---
layout: post
title: CQRS, DDD and Event Sourcing
tags: cqrs ddd
---

- Maybe split out DDD bits, but reference them from another post?
- Make this about:
    - Domain Events
    - Integration Events
    - Transaction Boundary
        - Blue circle
        - MediatR Pipeline
    - Event Sourcing
    - Eventual Consistency



These three concepts are often used together, but why?

# Domain Events

In **Domain-Driven Design**, domain events are the things that happen that are of interest to a domain expert. They are written in the ubiquitous language of the domain.

```c#
public class SomethingHappenedEvent : IDomainEvent
{
    // props
}
```

These are raised when methods are called making changes to domain **aggregates**. When domain events are dispatched, their handlers usually make more changes to other aggregates.

## Transaction Boundary

Whether or not domain events should be handled within the same data transaction is disagreed amongst architects.

Jimmy Bogard proposes [dispatching domain events immediately before committing the transaction](https://lostechies.com/jimmybogard/2014/05/13/a-better-domain-events-pattern/), so the side-effects of handling the events are all committed together as one unit of work.

I like this approach myself. I like to consider the commands that enter my application as a transaction boundary themselves.

## MediatR Pipeline

- Behaviors

## After Commit

However, there are times such as calling external APIs that require work to be done after the transaction has been committed. In these cases, Kamil Grzybek proposes [a separate layer for domain event notifications](http://www.kamilgrzybek.com/design/how-to-publish-and-handle-domain-events/).

- What if the notification handling keeps failing then the service crashes?

## Eventual Consistency

- Changes are propagated
- Commands that return acceptance not fulfillment ?

# Integration Events

Integration events are similar to domain events, but serve a different purpose.

[some link](https://docs.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/domain-events-design-implementation#domain-events-versus-integration-events)

- Must be serialised
- Makes sense to schedule job inside transaction

- Public Contracts maybe?

## Event Stream

- Generic domain event handler

# Event Sourcing

- Internal Read Store = different tables, within transaction, consistent

- External read store = different databases, outside transaction, resilient, eventually consistent

# Eventual Consistency

- Commands that return acceptance not fulfillment