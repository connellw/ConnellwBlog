---
layout: post
title: Why Firestorm?
---

[Firestorm](https://github.com/connellw/Firestorm) is a little thing I was working on that turned into a bigger thing I'm still working on a year later. As it grew, it spawned another little thing that'll probably turn into a bigger thing too. Rinse and repeat. Sound familiar?

Ah, the never-ending cycle of the undisiplined late-night programmer. *"Who needs user stories when I'm the only user?"* I admit, I thought the project would be much smaller than it is. Just everything was fitting so well together; all these crazy ideas seemed to have a perfectly logical place to live.

I like to split my solutions into many projects and group them with solution folders. But I'd imagine most readers would be horrified to learn that `Firestorm.sln` currently houses **32 source projects**. That's not including the test projects.

The documentation explains [the solution architecture](http://firestorm.readthedocs.io/en/latest/contrib/solution-architecture/), so I won't bang on about it too much here. But here is a section from [What is Firestorm?](http://firestorm.readthedocs.io/en/latest/intro/what-is-firestorm/) that summaries how it works:

> Your code ([Stems](http://firestorm.readthedocs.io/en/latest/stems/stems-intro.md) or [Fluent](http://firestorm.readthedocs.io/en/latest/fluent/fluent-intro.md)) builds a set of `Expression` and `Delegate` objects that are used in your API. The pieces of the jigsaw, if you will.
> 
> The [client's HTTP request](http://firestorm.readthedocs.io/en/latest/endpoints/basic-requests.md) puts those pieces together to build an `IQueryable`.
> 
> The query is executed by your ORM framework, such as Entity Framework.

And that pretty much sums up all the cool stuff I get to play with.

It meant I had to learn to write ASP.NET Core Middleware and OWIN Middleware, learn the differences and abstract them. The same with EF 6 vs EF Core 2. The **Endpoints** part forced me to research REST conventions to follow the same semantics and return the correct types.

But my favourite part has been writing the **Engine**, which has given me a pretty decent understanding of LINQ and Expression Trees if I do say so myself. The Engine is basically code that turns a series of `Expression<Func<T1, T2>>` objects into an `IQueryable<T>`. It has to create runtime types, build predicate expressions with comparison operators, use deferred execution and asyncronous foreach loops. It's been one hell of a challenge.

A lot of the **Stems** library uses Reflection to get your lambda expressions from the members you've decorated with the attributes. A fair chunk of utility code was shared with the Engine and it wasn't long before I realised I was creating a suite of Reflection extensions. I found some more repeatative Reflection code throughout the Firestorm source and low and behold, **Reflectious was born**.

I'll write again in a few years time, when Reflectious' great granddaughter, an almost-complete quantum machine learning particle collider, causes me to quit programming for good and focus on my fingerboarding career.