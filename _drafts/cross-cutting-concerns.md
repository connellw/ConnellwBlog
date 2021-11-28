---
layout: post
title: Cross-cutting Concerns
tags: aop logging decorator
---

With all of these, logging is the example used, but there are many other cross-cutting concerns you can add to these pipelines such as validation, caching, retrying, tracing. The concerns don't need to be cross-cutting, for example I like to commit my database transaction after MediatR requests are handled, for which I register a Pipeline Behavior.
