---
layout: post
title: Log After, Not Before
tags: aop logging decorator
notes: https://tuhrig.de/my-logging-best-practices/
---

I don't need an "about to do a thing" line at the start. I only ever find these are needed for one of two reasons:

1. To time how long something takes. Just time it within the application and add an `elapsed_time` property to line at the end to achieve the same thing.

2. To debug scenarios where the starting line is logged but not the end. This is normally because:
    - An exception was thrown or something happened to divert the control flow away from logging the ending line. Use a `finally` block.
    - You're stuck in an infinite loop somewhere. You could log a timeout for the request here.
    - Your application has totally crashed. This is a real edge case.