---
layout: post
title: Domain-Driven Boundaries
tags: ddd architecture
---

At its core, Domain-Driven Design is about building shared models as a collaboration between domain experts and technical experts. When engineers speak the same language as the rest of the business, we can build an architecture that aligns with how the business works.

## Bounded Context

In [Martin Fowler’s post on Bounded Contexts](https://martinfowler.com/bliki/BoundedContext.html), he explains that instead of building one unified model to describe the entire business, it's more cost-effective to divide the system into several contexts.

Some concepts exist in multiple contexts, but may have different models or use different terms. Equally, some terms may describe different concepts within different contexts. Each context uses its own **Ubiquitous Language** within its boundary.

![Bounded Contexts example](/images/diagrams/bounded-contexts.png)

## Identifying Boundaries

We [create boundaries all the time](/drawing-boxes); it’s a key part of designing software. We put things in methods, classes, libraries, microservices. We make things private, protected, internal. It’s one of the most important things we do.

But whilst we’re very familiar with creating our own boundaries, practicing DDD means our architecture should use the boundaries that already exist in the business. The challenge for us is to understand those business boundaries, develop models with the domain experts, then design systems based on those models. Otherwise our technical boundaries just become constraints that make solving business problems difficult and expensive.

To achieve this, the business experts must be involved in the modelling process. Together we can use processes like [Context Mapping](https://www.infoq.com/articles/ddd-contextmapping/) to help identity boundaries and [Event Storming](https://en.wikipedia.org/wiki/Event_storming) to build the Ubiquitous Language.

## Microservices

Once upon a time, developers built large monoliths that model the whole business. DDD helped divide the code in that solution based on how the business works. Contexts provide language boundaries, aggregates provide consistency boundaries, and value objects can define small boundaries for validation rules.

Nowadays, [identifying Bounded Contexts can help design a microservices architecture](https://docs.microsoft.com/en-us/dotnet/architecture/microservices/architect-microservice-container-applications/identify-microservice-domain-model-boundaries). It makes sense to use solution and network boundaries to build the logical boundaries that DDD describes.

## Tech-Driven Domains

It's too easy to get caught out by this convenient parallel, break down microservices like we're so used to doing with classes and libraries, then assert that those boundaries are now business domains.

I have made the mistake before of inventing language to solve technical problems in the model. Creating abstractions for maintainability, connectivity, persistence concerns, or [monitoring concerns](/dont-need-logging-code).

The problem with this is that, if our domain boundaries or models are driven by the technical design, we’re not speaking the same language any more; we’ve started to use technical jargon again. It’s harder to understand requirements, it’s harder to explain problems, harder to compromise, to prioritise, and ultimately it's much more costly to build software.