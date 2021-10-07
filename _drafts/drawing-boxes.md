---
layout: post
title: Software is about Drawing Boxes
tags: architecture
---

A lot of software engineering is about drawing boxes, deciding how to break down the code we write. We don't just write everything in the `Main` method in `Program.cs`; we set boundaries, create abstractions, and divide things into single responsibilities.

In C#, we group code into methods. Then we group those into classes and decide that some methods are `private`. We do the same again, grouping classes into libraries, deciding that some are `internal`, and then the same again with libraries. At each level we decide what is internal and what becomes our **public contracts**.

![Boxes and Arrows](/images/diagrams/boxes-no-arrows.png)

Once we've split everything up into boxes, we stitch it all back together again using some arrows. Libraries reference other libraries, classes depend on other classes, and methods call other methods. But it's not meaningless; the arrows represent a **direction of dependency**, where a box "knows of" the other box it is pointing to. At least, it knows the public contracts, not the internals.

We should ensure boxes don't *know of* each other, otherwise we will create a **circular dependency**. The compiler will error if we attempt this between projects. With other boxes, it's possible, but not recommended, due to the high-coupling it creates.

![Boxes and Arrows](/images/diagrams/boxes-and-arrows.png)

# Inverting Project Dependency

Many developers will be familiar with the N-Tier project architecture. Usually 3 layers: presentation, business logic and data access. We say the data access layer doesn't *know of* the business logic that is making calls to it; it just puts things in a database. Similarly, the business logic doesn't *know of* the API. In this architecture, the direction of dependency goes in the same direction as the **flow of control**, which is the direction method calls are made. You can think of control as *calling* or *driving*; which code tells which what to do and when to do it.

.... N tier diagram ....

These arrows do not have to point in the same direction. For example, if a `Service` class in the business logic layer *knows of* an `IRepository` interface, and the `Repository` implementation also *knows of* the `IRepository` interface, all we need to do is move the interface to the business logic layer, and suddenly the data access layer must have a reference to the business logic layer. We've inverted the project dependency.

.... flipped diagram ....

## Ports and Adapters

This is the fundamental principle behind the Ports and Adapters architecture. By inverting that project dependency, the **business logic has no dependencies**. It becomes easily testable as there are no databases, no HTTP requests; it's pure C# code.

There aren't transitive dependencies to the infrastructure this way either, so you can't even accidentally use functionality from `EntityFramework` in the business logic layer.

..... bit on driven and dirving adapters .....


# Visual Shapes

Ports and Adapters was originally called the Hexagonal Architecture. There is nothing special about the number six here, it is just a nice way to visualise the architecture.

# Presentation

- Documentation
- Integration Events

# Microservices

