---
title: "Incidents Are Hard"
date: 2025-12-21T16:44:24Z
draft: true
showtoc: true
---

Most software engineers will remember the first incident they got involved
with. You might have been roped in by a more senior engineer to get some
exposure, or perhaps you had a part in causing it. Either way, you were thrown
into the chaos and had to work out what was going on.

## Staying calm

One of the most important things you can do in an incident scenario is try and
remain calm. Incidents caused by a change that has just been released to
production and is causing some level of customer impact can rapidly descend
into chaos as different engineers try to work out what's going on.

Allowing yourself to remain calm and slow down enables you to make more
measured decisions and think more logically, which is especially important when
considering a system which is entirely logical. Plenty of simple incidents have
been worsened by stressed engineers making rash decisions.

As you trawl through logs and metrics, or find yourself on the cusp of making
an important decision, try and consider what you would do outside of an
incident scenario.

No matter how the situation feels at the time, remind yourself that larger
companies have had worse incidents with significantly more impact and survived.
It's highly likely that life will continue after this is over.

## Directing focus

With the level of chaos that incidents can induce, it's likely that you'll have
overlapping discussions where the focus changes often. Some engineers may be
working towards understanding the root cause, whereas others may be attempting
to mitigate the impact downstream.

You need to allow both groups to speak in order to facilitate synchronous
communication and make faster progress, but it's also important to allow both
groups to focus. It becomes confusing when the conversation jumps between some
traces that have just been discovered and a potential mitigation plan, leaving
neither group time to discuss.

In larger incidents separate breakout calls can be useful for this, however
they increase the cost of communication between the two groups to evaluate
progress as it moves from synchronous to asynchronous. Another option is for
engineers to switch calls, but this risks them losing context of their area or
joining halfway through discussions.

## Commanding the incident

Frequently, incidents can be prolonged because no one wants to make a decision
about the direction to take them in. Groups of engineers might be debating over
whether they should do a rollback or a fix-forward, making no progress towards
either.

At some point, someone needs to take charge and make those difficult decisions.
They don't even need to be familiar with the problem domain, in fact sometimes
this can be an aid as they won't begin trying to solve the problem themselves.

Their role as the incident commander is, broadly, to tell people what to do.
Make the decisions about who should be working on what. Assign a small group to
understand the root cause. Assign some others to prepare a rollback of the
change. One engineer to raise a revert of the problematic change and another to
inform the operations teams about what impact customers might be seeing.

Again, none of this requires domain knowledge. It just needs someone to get in
there and have the confidence to make decisions.

Separating out the participants into different groups also allows them to work
in parallel. you might find that a rollback isn't possible but at least you
were ready to go if the option presented itself. You might discover that the
customer impact is less than expected, but having the revert build through
earlier is always useful.

## Live scribing

Some incidents will be slow burners, where an issue has been discovered but it
has been around for a while. In order to find these and decide they are worthy
of an incident as opposed to a regular bug fix, plenty of analysis will have
been done in the area before the incident is created.

Others will be off the back of a change that has just been released, and these
may be much faster paced as we work to do all of that analysis as fast as
possible. This is typically done synchronously (perhaps a Slack huddle or a
Zoom call).

Unfortunately, this makes it difficult for bystanders to understand what is
being discussed and where the incident is at. It also results in a lot of catch
up talk, where someone joins the call and needs to be brought up to speed with
everything that has already been investigated.

This is where having an engineer acting as the scribe role comes in handy.
Their job, similar to the incident commander's specialism, is to join the call
and just write down whatever is being discussed. This allows a wider audience
to consume the state of the incident asynchronously through their messages,
enabling them to join the call if there's an area they are familiar with or
catch up if they join late.

Scribing is difficult though! Your aim is to listen to the call while
simultaneously filtering out the relevant information that a wider audience
might want. The audience may change each incident (depending on how technically
focused it is) or you may need to provide multiple update streams for different
interested parties.

Good scribes are a great asset in incidents though. They can enable the right
people to join at the right time, avoid duplicated work and allow many streams
of the business to be involved all at once. Their output is incredibly useful
when you reflect on the incident and begin to pick apart your resolution
process as part of a review, identifying bottlenecks or gaps for next time.

## Root cause analysis

When incidents break out, plenty of engineers will dive into the root cause
analysis and attempt to work out why this is happening. It's part of our
engineering mindset that wants to solve the problem as fast as possible. All of
this is great, but only if you find it quickly.

Every so often, groups will get stuck trying to find out the root cause of a
problem and this distracts them from understanding the impact of what's going
on. Perhaps you're seeing errors in a mobile API, which sounds bad, but you
don't find out that it was only in the background of customer requests until
later. That's important information that dictates how fast it needs to be
resolved.

Sometimes it's worth time-boxing the root cause analysis while the incident is
active. Provided you have good observability tools, you'll have plenty of time
once the impact is over to dive into the logs, metrics and traces to analyse
what went wrong and how it cascaded. It might be worth focusing efforts on
resolving the incident first.

## Overestimating severity

One of the first actions done as part of an incident is usually assigning a
severity, which provides a very broad overview of "how bad is this problem".
There is a little bit of nuance here, a severity of 3 might indicate an issue
affecting a large number of customers in a minor way, or a small number of
customers in a major way. Categorisation is dependent on how the company
defines it.

When you start one of these faster paced incidents, you might not fully
understand the impact yet and thus assigning a severity can be difficult. You
haven't had time to work out whether the errors in the logs are being presented
to customers or preventing payments from going through. You just know there's a
problem.

In this case, it's usually easier to overestimate the severity at the start and
then reduce it as the incident progresses and more information comes to light.
High severity incidents get more attention (in both good and bad ways), which
increases the likelihood that the right people will get involved earlier.

When impact assessment decides that an issue is higher priority than expected,
it can be difficult to get the attention of more senior engineers if they have
already seen the incident and mentally dismissed it. Overestimating the
severity is better than not having the right people in the incident when you
need them.
