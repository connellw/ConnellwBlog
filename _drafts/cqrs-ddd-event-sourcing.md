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
    - Event Sourcing
    - Eventual Consistency

# The Domain Layer with DDD

At the heart of our application is a single project with no dependencies. For this, we will use **Domain-Driven Design**. This is not a requirement of Onion Architecture, but it is a convinient way to divide our logic.

Before introducing the building blocks of DDD, it's important to mention that DDD is not fundamentally about the technical detail, but is focused around the domain model and how the language and structure of the code should match the business domain. It's a collaboration between technical and domain experts.

A great way to develop this language is **Event Storming**, where the domain experts tell a story of what happens in their domain. Throughout the story they will describe events that are of interest to them, which we model as **Domain Events**.

Domain Events are written in past tense, such as `AccountRegistered` or `PaymentTaken`, because they have already happened at the time we initialise them. Once they're initialised, the objects are immutable; you cannot go back and change the past.

When an event is raised, the domain doesn't *know* who's listening. It calls out to anyone who has subscribed to the event in advanced, but it doesn't know who. The control flows out of the domain to an event handler, but it is the handler that *knows of* the event. Again, we have inverted the dependency. Like Ports and Adapters, raising events is another technique that allows the control to flow in the opposite direction to the project reference.

...... sequence-ish diagram for domain ......

Events usually represent a change to the state of a domain **entity**. Entities and other domain objects are grouped together into clusters called **aggregates**, which provide a consistency boundary and can enforce the business rules of the domain.

Aggregates are stored in **repositories**; an abstraction for data storage. It's a port that the domain defines, that it expects to be implemented in another project that references it.

And finally, the **unit of work** is another abstraction, this time for a data transaction. All changes made to aggregates and repositories are committed together as one.

I like to think of the data abstractions as sitting in a thin layer just on the edge of the domain layer. In Onion Architecture, dependencies go inwards, so my repositories *know of* my aggregates, but not the other way round.

..... diagram with red circle, repo layer, and building blocks? .....

# The Application Layer with CQRS

For the layer just outside of the domain layer, we're going to build using **Command Query Responsibility Segregation**. This means our application layer will be made up of two sub-systems: a command side, responsible for executing tasks and making changes to our domain; and a query side, responsible only for getting data.

Both sides will be entirely separated from each other. They won't share any models or classes. This is essentially the **opposite of CRUD**. I found this difficult to grasp at first, because I was designing RESTful APIs as the front-end. I instictly thought of writing a certain resource to a location, and the getting that same shape back out of that location, which is breaking the only rule of CQRS; separate the read and write concerns.

## The Command Side

For commands, think task-based, rather than resource-based. What are we actually trying to do? Once we've worked out what the command looks like, that becomes the public contract for our application layer. The command is sent into this layer and handled by a `CommandHandler`, which is responsible for:

1. Creating or loading an aggregate from a `Repository`
2. Calling methods on that aggregate to make changes.
3. Committing the `IUnitOfWork`.

```csharp
public class OpenTinCommandHandler
{
    private readonly ITinRepository _tinRepository;
    private readonly IUnitOfWork _unitOfWork;

    public OpenTinCommandHandler(ITinRepository repository, IUnitOfWork unitOfWork)
    {
        _repository = repository;
        _unitOfWork = unitOfWork;
    }

    public async Task Handle(OpenTinCommand command)
    {
        var tin = await _repository.Get(command.TinId);

        tin.Open();

        await _unitOfWork.Commit();
    }
}
```

Note that this command returns only a `Task`. We're not going to return any business data. Certainly not our read models.

However, we could throw an exception here. We can return data about the command execution itself. We might prefer to return a custom `CommandResult` object with error information included. But definitely not any business data; that is what queries are for.

A command either succeeded or it didn't; it cannot return that some of the command worked but then it failed. If multiple changes are made on an aggregate, or if domain events caused by the changes are handled and make further changes, everything must be committed as one unit of work.

```csharp
CommandHandler example with multiple and dispatch
```

I like to view the whole application layer as this transaction boundary. From outside, we just send a command, and it either succeeds or it doesn't.

### Dispatching Events

![CQRS command sequence-ish](/images/diagrams/sequence-ish-command.png)

### MediatR

## The Query Side

The other half of our application will be handle reads. Think about what we want information we need to know, then we can directly return that ViewModel.

Query objects look very similar to Commands, and are handled similarly with a `QueryHandler` class.

However, in this side, we don't want to use our repositories, aggregates or entities; they are our write models. We certainly don't want to be returning them from our queries, because consumers could use them to make changes to the system. Instead, we just project our query results straight onto the response object.

```csharp
QueryHandler example
```

We have the freedom to do powerful things here. We could use Dapper to execute queries that join several tables together. We could even read from a totally different database to our write side, called a **read store**.

.... sequence-ish diagram with blue and projection? .....

CQRS gives us the power to scale the two concerns independently. We can optimise a query that uses joins by moving to use a denormalised table designed for the query instead. The table can be sourced by handling domain events, so that the query results are saved at the time the command is executed, instead of on-the-fly.

### Eventual Consistency

??

## Jobs and Resiliency

Writing to the **read store** can be done as part of the same database transaction to ensure consistency between the read and write sides. This is done by making changes to another table in a `DomainEventHandler`, which is handled within the same `UnitOfWork` as the command execution.

```csharp
DomainEventHandler writing to read store
```

This isn't possible with a totally different database. Instead, we must **schedule a job** to write the changes after the transaction has committed. The scheduling itself must be done within the transaction, so I like to view this as just writing to another read store (the jobs store) which is later queried by a job processor. When the job is executed, the changes have already made and we cannot throw an error when processing the job. The job must therefore be processed with a retry mechanism to ensure it is processed.

```csharp
DomainEventHandler scheduling a job
```

...... diagram with 3 building blocks of application layer too ......

## Integration Events

- Outside of the blue circle

### Event Stream