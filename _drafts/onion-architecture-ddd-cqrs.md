---
layout: post
title: Onion Architecture with DDD and CQRS
tags: architecture ddd cqrs
---

A lot of software engineering is about drawing boxes, deciding how to break down the code we write. We don't just write everything in the `Main` method in `Program.cs`; we set boundaries, create abstractions, and divide things into single responsibilities.

In C#, we group code into methods. Then we group those into classes and decide that some methods are `private`. We do the same again, grouping classes into libraries, deciding that some are `internal`, and then the same again with libraries within an application.

.... diagram ....

Once we've split everything up into boxes, we stitch it all back together again using some arrows. Libraries reference other libraries, classes depend on other classes, and methods call other methods. But it's not meaningless; the arrows here represent a **direction of dependency**, where a box "knows of" the other box it is pointing to. At least, it knows the `public` things from that box, not the internals.

# Inverting Project Dependency

Many developers will be familiar with the N-Tier project architecture. Usually 3 layers: presentation, business logic and data access. We say the data access layer doesn't *know of* the business logic that is making calls to it; it just puts things in a database. Similarly, the business logic doesn't *know of* the API. In this architecture, the direction of dependency goes in the same direction as the **flow of control**, which is the direction method calls are made. You can think of control as *calling* or *driving*; which code tells which what to do and when to do it.

.... N tier diagram ....

These arrows do not have to point in the same direction. For example, if a `Service` class in the business logic layer *knows of* an `IRepository` interface, and the `Repository` implementation also *knows of* the `IRepository` interface, all we need to do is move the interface to the business logic layer, and suddenly the data access layer must have a reference to the business logic layer. We've inverted the project dependency.

.... flipped diagram ....

## Ports and Adapters

This is the fundamental principle behind the Ports and Adapters architecture. By inverting that project dependency, the **business logic has no dependencies**. It becomes easily testable, as there are no databases, no HTTP requests; it's pure C# code.

The business logic defines the `IRepository` and the entity objects that it uses as a public **port**. It says:
> Here is a port that I know how to use. Please, someone implement this port.

Other projects can implement the interfaces by creating **adapters**. We could create an `EfRepository` that implements our business logic's port and wraps up Entity Framework's `DbSet`. This is the Adapter pattern.

There aren't even transitive dependencies to the infrastructure this way, so you can't accidentally use functionality from `EntityFramework` in the business logic layer.

Ports and Adapters was originally called the Hexagonal Architecture. There is nothing special about the number six here, it is just a nice way to visualise the architecture. It's diagrams tend to use a hexagon in the middle, but the shape doesn't matter; it can just as easily be drawn as a circle.

Onion Architecture is just Ports and Adapters architecture, but the business logic layer is further divided into domain logic and application logic.

# The Domain Layer

At the heart of our application is a single project with no dependencies. For this, we will use **Domain-Driven Design**. This is not a requirement of Onion Architecture, but it is a convinient way to divide our logic.

Before introducing the building blocks of DDD, it's important to mention that DDD is not fundamentally about the technical detail, but is focused around the domain model and how the language and structure of the code should match the business domain. It's a collaboration between technical and domain experts.

A great way to develop this language is **Event Storming**, where the domain experts tell a story of what happens in their domain. Throughout the story they will describe events that are of interest to them, which we model as **Domain Events**.

Domain Events are written in past tense, such as `AccountRegistered` or `PaymentTaken`, because they have already happened at the time we initialise them. Once they're initialised, the objects are immutable; you cannot go back and change the past.

Events will usually represent some kind of change to the state of a domain **entity**.

Entities and value objects are grouped together into clusters of objects called **aggregates**. An aggregate provides a consistency boundary for several objects and can enforce the business rules of the domain.

Aggregates are stored in **repositories**; an abstraction for data storage. It's a port that the domain defines, that it expects to be implemented in another project that references it.

And finally, the **unit of work** is another abstraction, this time for a data transaction. All changes made to aggregates and repositories are committed together as one.

# The Application Layer



# The Infrastructure Layer



# TL;DR

I call controllers in the presentation layer, which send commands to my application layer, which load aggregates from the domain layer, using repositories which are implemented in the infrastructure layer as an adapter to the write store.

Aggregates are made up of entities and value objects. They handle all changes and raise domain events, which are handled in the application layer, either by making further changes to aggregates, or by writing to a read store, which can later be queried by controllers or resilient jobs, such as publishing integration events, writing to an external read store, or calling external APIs.