---
layout: post
title: Visualising Software in Shapes
tags: architecture
---

- Note that these layers are not necessarily csproj.

A lot of software engineering is about drawing boxes, deciding how to break down the code we write. We don't just write everything in the `Main` method in `Program.cs`; we set boundaries, create abstractions, and divide things into single responsibilities.

In C#, we group code into methods. Then we group those into classes and decide that some methods are `private`. We do the same again, grouping classes into libraries, deciding that some are `internal`, and then the same again with libraries within an application.

.... diagram ....

Once we've split everything up into boxes, we stitch it all back together again using some arrows. Libraries reference other libraries, classes depend on other classes, and methods call other methods. But it's not meaningless; the arrows here represent a **direction of dependency**, where a box "knows of" the other box it is pointing to. At least, it knows the `public` things from that box, the **public contracts**, not the internals.

# Layered Architecture



# Hexagonal Architecture

Ports and Adapters was originally called the Hexagonal Architecture. There is nothing special about the number six here, it is just a nice way to visualise the architecture. It's diagrams tend to use a hexagon in the middle, but the shape doesn't matter; it can just as easily be drawn as a circle.

..... bit on driving and driven adapters ?? .....

# Olympic Rings

I had a go myself once.

# Clean Architecture

# Onion Architecture

Onion Architecture is just Ports and Adapters architecture, but the business logic layer is further divided into domain logic and application logic.

## Castle Architecture

## Explicit Architecture

