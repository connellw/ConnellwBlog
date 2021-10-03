---
layout: post
title: Introducing Reflectious
tags: firestorm
---

Whilst developing Firestorm, I noticed I was often combining Expression Trees with Reflection code. I'd find a static LINQ method, pass it a lambda expression argument and throw the whole thing into Expression.Call. 

This stuff is happening all over the architecture, but most noticeably in core libraries that I didn't want referencing each other. I had a few copy-n-paste utility classes replicated, but we shouldn't be repeating ourselves! 

That was it, my heart was now set on making a new library for these utilities. And whilst I was at it, I was going to make all that common Reflection code neater and cleaner. 

I'd been enjoying using Fluent APIs like those found in Entity Framework Core, ASP.NET Core's Startup.cs and FluentAssertions. I'd even written a fluent API in Firestorm. So why not keep it going, eh? 

The way we use reflection favours a builder approach. We usually grab a TypeInfo object, use it to get a MethodInfo object, maybe make it generic to get another MethodInfo object, then Invoke that. We usually use each object once to get another object, that we use once, and so on. 

But there's different styles going on. We're using standard instance methods, static utility classes and casting. Consider this code: 

```c#
var type = typeof(List<>).MakeGenericType(stubType);
var list = Activator.CreateInstance(type);
int count = (int)type.GetProperty("Count").GetValue(list);
```

Now I think that can be better written as something like: 

```c#
int count = typeof(List<>).Reflect()
        .MakeGeneric(stubType)
        .WithNewInstance()
        .GetProperty("Count")
        .OfType<int>()
        .GetValue();
```

And that pretty much the premise! The concept extends into use with expressions, creating delegates and dependency injection and could possibly reach into the realm of assembly scanning.