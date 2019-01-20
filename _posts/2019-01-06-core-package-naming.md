---
layout: post
title: Core Package Naming Convention
---

I've noticed a few NuGet packages reference an underlying `.Core` package from their solution. Microsoft themselves have been using this convention in recent packages [Microsoft.AspNetCore.Mvc.Core](https://www.nuget.org/packages/Microsoft.AspNetCore.Mvc.Core/), [Microsoft.AspNetCore.SignalR.Core](https://www.nuget.org/packages/Microsoft.AspNetCore.SignalR.Core/) and [Microsoft.EntityFrameworkCore.Sqlite.Core](https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.Sqlite.Core/).

However, [this answer on StackOverflow](https://stackoverflow.com/a/146578/369247) from years ago says this is a **bad naming convention**.

The difference is about **Assemblies** vs **Namespaces**.

The `Microsoft.AspNetCore.Mvc.Core` **namepsace** doesn't seem to exist. The classes are all simply in `Microsoft.AspNetCore.Mvc`. The [Microsoft.AspNetCore.SignalR.Core.csproj on GitHub](https://github.com/aspnet/SignalR/blob/master/src/Microsoft.AspNetCore.SignalR.Core/Microsoft.AspNetCore.SignalR.Core.csproj) has the `<RootNamespace>` set to just `Microsoft.AspNetCore.SignalR`.

I've usually applied the same logic as namespaces to my assembly names, but I'm beginning to like the convention of having `.Core` assemblies and having the root assembly name (i.e. without `.Core` or any extension) being a metapackage that references all the sub-assemblies. Microsoft seem to be taking this approach now.

So I've renamed some projects in Firestorm. For example, [Firestorm.Engine.Core](https://github.com/connellw/Firestorm/tree/master/src/Firestorm.Engine.Core) and [Firestorm.Endpoints.Core](https://github.com/connellw/Firestorm/tree/master/src/Firestorm.Endpoints.Core). These are to be exposed as NuGet packages too, but as with Mvc and SignalR, the packages would usually be referenced by the main package your application would reference.