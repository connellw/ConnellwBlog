---
layout: post
title: Software is about Drawing Boxes
tags: architecture
---

That's what it all comes down to. Not to diminish anyone's long, hardworking career, but every design decision we make ultimately boils down to where we want to define a certain boundary. What concepts should we couple together? Which layers "know of" which? It's all a structure of how thoughts and ideas fit together.

We do it everywhere. We don't just write everything in the `Main` method; the languages provide structures to separate our concerns. We group lines of code into methods, then we group those into classes, classes into libraries, into microservices, then sometimes even into [domain boundaries](/domain-driven-boundaries).

It's very visual. We use shapes and patterns to describe different relationships. We draw boxes and arrows, give them different colours.

![Boxes and Arrows](/images/diagrams/boxes-no-arrows.png)

# Dependency

Once we've split everything up into boxes, we stitch it all back together again using some arrows. Libraries reference other libraries, classes depend on other classes, and methods call other methods. But it's not meaningless; the arrows represent a **direction of dependency**, where a box *knows of* the other box it is pointing to.

We should ensure boxes don't *know of* each other, otherwise we will create a **circular dependency**. The compiler will error if we attempt this between projects. With other boxes, it's possible, but not recommended, due to the high-coupling it creates.

![Boxes and Arrows](/images/diagrams/boxes-and-arrows.png)

- Mention mutual recursion in functional programming
- Mention avoiding dependencies using events or dependency inversion
- Avoid at a higher level?

# Internal and Public

In every box, we decide which ideas should be exposed to the outside. A method has it's parameters and return type, a class has `private` methods, libraries have `internal` classes. Even at a solution level, we often use a single API project that presents our application to the rest of the network.

Being public is not to be taken lightly. This is the **contract** between the box and those that *know of* it - an interface that lives on the boundary. Changing this interface could break consumers that reference it. If you're using semver, a change to a public contract requires a major version bump. When we do need to change them, it's common to have to maintain multiple versions and deprecate use of the older version.

For this reason, we should follow the principle of least privilege. Classes should be internal by default unless they need to be exposed. Having to expose contracts as public should make us think:

> Do I want to make this part of the public API? Is this worth sacrificing the freedom to modify this code so easily?

# Layers and Libraries

Not all boxes are created equal. There are usually different ways to break down the same architecture. We think about **horizontal or vertical slices** and sometimes even the 2 dimensions on our screens aren't enough to visualise the many ways to cut up our code.

Sometimes we draw diagrams one way and write code in another. Maybe the way we think about the problems doesn't always translate to a codebase. Class libraries give us a solid way to enforce a direction of dependency and introduce public contracts, which sounds like a great fit for our 'layers'. But horizontal slicing means we have independent features that can reference each other's internals and dependent frameworks.

Or maybe we're just not splitting our code in the same way we think about them. We can avoid doing either and [package by component](http://www.codingthearchitecture.com/2015/03/08/package_by_component_and_architecturally_aligned_testing.html) instead.

!(Package by component)[].

- Mention presentation layer flow of control?



---------

---------

---------



- link from Domain-Driven Boundaries post:

We [create boundaries all the time](); it’s a key part of designing software. We put things in methods, classes, libraries, microservices. We make things private, protected, internal. It’s one of the most important things we do.

- Also link from Onion Architecture post:

A lot of software engineering is about drawing boxes, deciding how to break down the code we write. We don't just write everything in the `Main` method in `Program.cs`; we set boundaries, create abstractions, and divide things into single responsibilities.

### Inverting Project Dependency

Many developers will be familiar with the N-Tier project architecture. Usually 3 layers: presentation, business logic and data access. We say the data access layer doesn't *know of* the business logic that is making calls to it; it just puts things in a database. Similarly, the business logic doesn't *know of* the API. In this architecture, the direction of dependency goes in the same direction as the **flow of control**, which is the direction method calls are made. You can think of control as *calling* or *driving*; which code tells which what to do and when to do it.

.... N tier diagram ....

These arrows do not have to point in the same direction. For example, if a `Service` class in the business logic layer *knows of* an `IRepository` interface, and the `Repository` implementation also *knows of* the `IRepository` interface, all we need to do is move the interface to the business logic layer, and suddenly the data access layer must have a reference to the business logic layer. We've inverted the project dependency.

.... flipped diagram ....

### Ports and Adapters

This is the fundamental principle behind the Ports and Adapters architecture. By inverting that project dependency, the **business logic has no dependencies**. It becomes easily testable as there are no databases, no HTTP requests; it's pure C# code.

There aren't transitive dependencies to the infrastructure this way either, so you can't even accidentally use functionality from `EntityFramework` in the business logic layer.

..... bit on driven and dirving adapters .....


### Visual Shapes

Ports and Adapters was originally called the Hexagonal Architecture. There is nothing special about the number six here, it is just a nice way to visualise the architecture.

### Presentation

- Documentation
- Integration Events