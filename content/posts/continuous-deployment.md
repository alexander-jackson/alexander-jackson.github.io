---
title: "Continuous Deployment"
date: 2022-10-01T16:49:33+01:00
draft: true
showtoc: true
---

Writing code is great and all, but you know what's better than writing it?
Continuously deploying said code, without any manual steps. Push a commit to
GitHub and a few minutes later, that change is running in production.

## Why would I want every change going straight to production?

Don't get me wrong, this isn't an approach that works for a lot of projects. If
you're a large organisation or company, you likely have some form of demo
environment where changes go first, either for a few minutes or a few months
depending on the culture of the business. Once changes have sat there for a
while, there's another release process that puts them into production.

Sometimes this process happens often and is rather dull. Sometimes it's a once
a quarter or once a year job, and there's a whole load of fanfare when it
finally goes out (plus a lot of bug fixing). But this article isn't for those
companies (unless this is how they want to operate).

This is aimed mainly at developers like me, who have one or many small side
projects they enjoy tinkering around with. Where it doesn't matter so much if
something goes wrong, either because no one is really using it that often, or
you can cope with some issues before rolling back or fixing forward.

For those developers, it's incredibly satisfying to be able to tweak something
and push it straight to production so you can use it yourself or add a feature
someone has asked for. It also makes it simpler for others working on the same
project, as their changes can be automatically deployed as well without waiting
for your manual input.
