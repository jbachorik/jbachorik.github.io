---
layout: post
title:  "A different shade of context"
date:   2024-04-26 09:00:00 +0200
categories: java jvm openjdk
---

1. TOC
{:toc}

## Oh, not again ...
Yes, you remember correctly. I did write a quite lengthy series of blog posts about my proposal
for [JFR context implementation]({% post_url 2023-09-13-seeing-in-context_1 %}). However, it turned
out that the proposed changes were too intrusive and would never be accepted.

Things looked bleak for a while, because without the context in JFR, we would still need to rely
on a separate JVMTI agent to do the cool things we can do, thanks to the context.

## The plot twist
During my trip to Stockholm where my colleague [Richard Startin](https://richardstartin.github.io/) and I talked
about [future-proofing the profiling support in the JVM](https://www.youtube.com/watch?v=10L-7fb4SWk&list=PLUQORQEatnJezysGP4J-EZm34u-OyILC2&index=7),
we also managed to secure a meeting with my former colleagues and, I dare say friends, [Markus Grönlund](https://inside.java/u/MarkusGronlund/)
and [Erik Gahlin](https://inside.java/u/ErikGahlin/). They are the main force behind JFR and it made a lot of sense to meet them.

We ended up talking for many more hours than originally planned, and it was a very fruitful discussion. When trying to figure
out what to do with the context in JFR, Erik suddenly asked: "Why don't we just have special events to convey the context?".

## The idea
The idea was simple: we would introduce a new event meta type, `@Contextual`, which would be used to convey the context.

This idea is also pretty old and was originally dismissed because, in order to capture rapidly changing context, as would
be the case for async or reactive applications, we would need to generate a lot of events. And that would be a problem -
the recording size would blow up, and the transfer and processing costs would skyrocket.

Here, Markus and Erik stopped and asked - "Hm, but what if we emit the contextual event only when it applies to at least one
other event?".

The word *"applies"* here translates to an event being committed between calls to `begin()` and `end()` methods
of the contextual event.

And sure, with this little tweak we would be able to attach an arbitrary context to virtually any JFR event. The number and types
of the context fields would be limited only by what the user will be willing to pay in terms of memory and storage
costs.

## Juicing the idea
What if we don't stop there? If we have contextual events we could also use the presence of the context as an alternative
to the event threshold which is in use today. While thresholding allows focusing on the outliers, it is many times the
'death by a hundred cuts' that is the real problem. But in that case, thresholding would actually mask the real problem, as
the short-lived events would never cross the threshold and be reported.

But if, instead of checking the event duration to cross the threshold, we could check if there is a contextual event that applies to the event,
this would create a 'magnifying glass' effect, where all the fine details would be preserved as long as there is a context.
And, considering that the context is present for operations of special interest to the user, it would be a perfect match.

## The proposal
Putting all of this together I set out creating an early [prototype of the contextual events](https://github.com/openjdk/jdk/pull/18689).
The prototype is still in the very PoC stage, but it already shows a lot of promise. I have patched the [Datadog Java tracer](https://github.com/DataDog/dd-trace-java)
to turn the existing `TimelineEvent` type to a contextual event and ran a bunch of applications to see how it behaves.

And it does what is expected - the context is easily attributable to various events, like CPU, Wallclock, or Allocation samples, but not only that.
It is also possible to associate the context with built-in events like MonitorWait, etc.

In order to test the thresholding alternative, I set the threshold for `ThreadPark` and `JavaMonitorWait` events to 0ms (no threshold, and something you really don't want to do in production).
The results were astonishing - the recording was not bloated, the processing was not overwhelmed, and the events were still there, providing the fine-grained details of the application's behavior.

## What's next
Currently, we are collecting feedback from the community and early adopters. So far, there hasn't been much of the feedback,
but I am hoping that this blog post will change that.

If you have any thoughts, ideas, or concerns, please feel free to use
the [PR](https://github.com/openjdk/jdk/pull/18689) to comment directly there. Or, if you prefer, you can leave a comment here.

---
_(This is a copy of the detailed description of the proposal from the PR - just for the record if the PR would change in future)_
## Gnarly details

### Design

#### Contextual annotation

A contextual event will be demarked by `@Contextual` annotation. This annotation wil be a simple indication
that this particular event type is supposed to provide context to other events and tooling can handle it as such.

All custom fields of such annotated event type will then constitute the context.

#### Context driven behaviour

Although having the `@Contextual` annotation will allow the tooling to associate the context with other
JFR events, there are more ways they can be utilized.

###### Conditionally emit events

The contextual events can be used to guard annotations of events which are too costly to emit unconditionally
and using the durational thresholds would introduce too strong bias. An example would be `JavaMonitorWait` event.

If left unchecked, the emission rate of `JavaMonitorEvent` can overwhelm the recording. What's worse is that
the majority of the recorded events will provide very little additional information. Turning on the durational
threshold will improve the situation, but will introduce bias where the JFR will not be able to point out too much
time spent waiting on a lock, if each wait is shorter than the threshold. In addition to that, this event type
might be frequently emitted from thread pools where threads are just waiting for work.

If the emission is bound to the presence of a context (contextual event) which will be activated only when
an important work (what is important work will usually be defined by the user) is being done, providing laser
focus on fine-grained details of the application's behaviour.

###### Record only activated context

We are talking about an activated context (contextual event) when there is at least one other event committed
on the same thread between calling `begin()` and `end()` of the contextual thread. We can also think about
the context being 'triggered' by the regular events.

The concept of 'active' context is beneficial in lowering the overhead related to recording the context -
eg. for the distributed tracers with context propagation it is possible to generated millions of contextual
events per minute for certain frameworks (async and reactive ones are pretty notorious). This creates a huge
pressure both when the recording is written and also when it needs to be processed. And most of these events
will be literally useless because there would be no events the context could be applied to.


##### Controlling the behaviour via settings

The proposal is to use the standard JFR event settings mechanism to affect the behaviour of both
contextual and regular events.

There will be a new setting called `select` and the following permitted values:
- `if-context`   - the regular event will be emitted only if a context is present
- `if-triggered` - the contextual event will be emitted only if the context is triggered
- `all`          - no context related restrictions are applied

The `if-context` option is valid only for non-contextual events.
The `if-triggered` option is valid only for contextual events.
The `all` option is valid for any event.

If an invalid option is provided, JFR will log a warning and the setting will be set to `all`.

The `select` setting is to be used in conjunction with other filtering mechanisms, like `threshold`.

### Implementation

#### `@Contextual` annotation

The annotation implementation is pretty straightforward and there is nothing special going on there.

#### Activated context

In order to support selective emission of the contextual events only when they are activated the event
class must be instrumented and a synthetic field named `^ctxOffset` must be inserted there.

The field is used to track the number of events written while this context is open. The actual number
does not matter, we just need to make sure we can tell there is at least one written event.

This information is then used in the `shouldCommit()` method of the contextual event type which needs
to be changed to consult `^ctxOffset` field and return false if that field is `0`. That is, if the
event's settings contains `select=if-triggered`. Otherwise, the behaviour of `shouldCommit()` is not
affected.

The `^ctxOffset` field is updated from `EventWriter`, incrementing it on a new event commit.
