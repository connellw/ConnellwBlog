---
layout: post
title: Software is about Drawing Boxes
tags: architecture
---

That's what it all comes down to. Not to diminish anyone's long, hardworking career, but every design decision we make ultimately boils down to where we want to define a certain boundary. What concepts should we couple together? Which layers *"know of"* which? It's all a structure of how thoughts and ideas fit together.

We do it everywhere. We don't just write everything in the `Main` method; the languages give us ways to separate our concerns. We group lines of code into methods, then we group those into classes, classes into libraries, into microservices, then sometimes even into [domain boundaries](/domain-driven-boundaries).

It's very visual. We use shapes and patterns to describe different relationships. We draw boxes and give each of them some meaning, a name, and maybe even colours.

![Boxes and Arrows](/images/diagrams/boxes-no-arrows.png)

# Dependency

Once we've split everything up into boxes, we stitch it all back together again using some arrows. Libraries reference other libraries, classes depend on other classes, and methods call other methods. But it's not meaningless; the arrows represent a **direction of dependency**, where a box *knows of* the other box it points to.

We should ensure boxes don't *know of* each other, otherwise we will create a **circular dependency** where our modules become tightly coupled and we cannot use either without the other.

![Boxes and Arrows](/images/diagrams/boxes-and-arrows.png)

There are a few exceptions to this rule. Circular references are quite common in functional programming, such as mutual recursion when parsing expressions or grammar. But generally, the higher up the architectural levels you go, the more it is considered an anti-pattern. The bigger those boxes are, the more you'll want to reduce coupling.

- While it's possible for classes to reference each other, usually code quality tools will flag that as a code smell.
- With class libraries, the compiler will error if two reference each other.
- With microservices, this is a sign of a **distributed monolith**.

## Inverting the Control

Circular dependencies often come to existence due to a misconception that the direction of dependency must match the flow of control, but that is not the case. **Inversion of Control** can be achieved through **dependency injection**, where a class depends on an interface, but it doesn't know which implementation has been passed into it.

We can invert the control using any form of [callback](https://en.wikipedia.org/wiki/Callback_(computer_programming)) mechanism, such as **subscribing to an event**. A publisher doesn't know who is listening to its calls, rather the subscriber knows what the event looks like and requests that it is called back to whenever the event happens.

> Don't call us, we'll call you.
> -- The Hollywood Principle

With these techniques in our tool belts, it's possible to avoid circular dependencies and create a neat one-way dependency diagram of our entire architecture.

![Microservices direction of dependency](/images/diagrams/microservices-direction.png)

# Internal and Public

In every box, we decide which ideas should be exposed to the outside. A method has it's parameters and return type, a class has `private` methods, libraries have `internal` classes. At a solution level, we often use a single API project that presents our application to the rest of the network. We may even use a separate gateway API to present many APIs to an even wider audience.

This is the **contract** between the box and those that *know of* it - an interface that lives on the boundary. From the outside of the box, all we see is this public contract.

![Public contracts](/images/diagrams/public-contracts.png)

Being public is not to be taken lightly. Changing this interface could break consumers that reference it. If you're using [semantic versioning](https://semver.org/), a change to a public contract requires a major version bump. When we do need to change them, we usually have to maintain multiple versions and deprecate use of the old version so that clients can switch over.

For this reason, we should follow the **principle of least privilege**. Classes should be internal by default unless they need to be exposed. Having to expose contracts as public should make us think:

> Do I want to make this part of the public API? Is this worth sacrificing the freedom to modify this code so easily?

## Controlling the Contract

Generally I've found that information doesn't flow from the internals out to the public contracts unless it's flowing all the way out as an event or a callback. That is, you can internally implement a public interface, or you can raise an event from an internal class, but calling your own public API smells a bit to me. It feels like a circular reference again, so is something that personally I avoid.

# Layers and Libraries

Not all boxes are created equal. There are usually different ways to break down the same architecture. We think about **horizontal or vertical slices** and sometimes even the 2 dimensions on our screens aren't enough to visualise the many ways to cut up our code.

Sometimes we draw diagrams one way and write code in another. Class libraries give us a solid way to enforce a direction of dependency and introduce public contracts, which sounds like a great fit for our 'layers'. But horizontal slicing means we have independent features that can reference each other's internals and dependent frameworks.

Maybe the way we think about the problems doesn't always translate well to a codebase.  Or maybe we're just not splitting our code in the same way we think about them. We can avoid doing either and [package by component](http://www.codingthearchitecture.com/2015/03/08/package_by_component_and_architecturally_aligned_testing.html) instead. We can divide projects both horizontally and vertically where it makes sense to do so.

![Package by component](/images/diagrams/package-by-component.png).

We should use the features available in our languages and frameworks to reflect the way we think about our architecture. Diagrams are great, but quality code really is its own best documentation. Both play an important role in understanding and communicating the boundaries of our systems.