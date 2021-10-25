---
layout: post
title: API with ASP.NET Middleware Pipeline
tags: aop cqrs
---

ASP.NET Core but without a controller.
- Maybe note Minimal APIs?

Middlewares
- Logging & Monitoring
- Error Handling
- Find endpoint and deserialize DTO
- Validation (FluentValidation)
- Map to Command/Query (AutoMapper)

Mediator
- DispatchEvents
- UnitOfWork