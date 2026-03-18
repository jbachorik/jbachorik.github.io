---
layout: post
title: "Connecting the Dots in a JFR Recording"
date: 2026-03-18
categories: [java, jfr, performance, profiling]
tags: [jfr, jfr-shell, decorateBy, profiling, jafar, cross-signal]
---

1. TOC
{:toc}

## One Recording, Many Silos

A JFR recording from a busy service is absurdly rich. CPU samples, allocation samples, GC pauses, exceptions, thread states, endpoint spans, lock contention events. All timestamped, all in the same file, all covering the same two minutes of life.

And yet, almost everyone analyzes them one event type at a time.

You look at CPU samples. You see hot methods. You look at allocations. You see big objects. You look at exceptions. You see error counts. Each view is correct, but each is also incomplete. The interesting questions are the ones that span two views at once. Which endpoint is responsible for GC pressure? Are the exceptions actually correlated with CPU waste, or are they harmless? Do threads allocate memory while they are parked and waiting?

These are not exotic questions. They are the first things a performance engineer wants to know when triaging a production incident. The data to answer them is right there in the recording. The problem has always been connecting it.

## The Hard Way

Suppose you have a recording with 44,000 CPU samples and 438,000 endpoint span events. You want to know how much CPU each endpoint consumes.

Both event types carry a `localRootSpanId` field that identifies the distributed trace they belong to. In theory, you just join them on that field. In practice, that means:

1. Export all execution samples with their span IDs
2. Export all endpoint events with their span IDs and endpoint names
3. Load both into a script or a spreadsheet
4. Join on span ID
5. Group by endpoint name
6. Count

It works. It's also tedious enough that nobody does it during an incident at 3am. You end up eyeballing thread names in the flamegraph and hoping they correlate with something useful.

If you want to cross allocations with exceptions on the same trace, that's another export, another join, another script. Each question is a small project. So in practice, people don't ask these questions. They stick to single-event-type queries and miss the connections.

## decorateBy

jfr-shell has an operator called `decorateBy` that does this join inline, as part of a query pipeline. You tell it which event type to use as context, which field to join on, and which fields to pull from it. The result is the original events enriched with fields from the second type, accessible as `$decorator.*` in downstream operators like `groupBy` or `select`.

There are two flavors. `decorateByTime` matches events that overlap in time on the same thread, useful for things like "which lock was this thread waiting on when this sample was taken." `decorateByKey` matches events that share a value in a specified field, useful for any kind of ID-based correlation: trace IDs, span IDs, request IDs, GC cycle IDs.

The query for "CPU budget per endpoint" looks like this:

```
events/datadog.ExecutionSample
  | decorateByKey(datadog.Endpoint,
      key=localRootSpanId, decoratorKey=localRootSpanId,
      fields=endpoint)
  | groupBy($decorator.endpoint)
```

One query. No exports, no scripts, no spreadsheets. Takes a few seconds to run. And because it's just another pipeline operator, you can chain it with everything else: filters, aggregations, top-N, sorting.

## A Recording to Play With

I had a recording from a gRPC-based profile analysis service. Two minutes, 16 megabytes. Here's what was in it:

- **44,454** CPU execution samples (Datadog async profiler)
- **13,492** method samples with trace correlation
- **4,186** exception samples
- **820** allocation samples
- **438,041** endpoint span events
- **116** GC pauses, including one full GC at 3.6 seconds
- 8 cpu-intensive worker threads, gRPC/Netty I/O threads, a Kafka consumer

The Datadog profiler puts a `localRootSpanId` on most of its events. This is the distributed trace root. When an execution sample, an allocation sample, and an endpoint event share the same `localRootSpanId`, they all belong to the same request. That is the join key for everything that follows.

## Five Queries

### 1. Where does the CPU actually go?

```
events/datadog.ExecutionSample
  | decorateByKey(datadog.Endpoint,
      key=localRootSpanId, decoratorKey=localRootSpanId,
      fields=endpoint)
  | groupBy($decorator.endpoint)
```

```
| endpoint                                 | count |
+------------------------------------------+-------+
| (untraced)                               | 23578 |
| com.datadoghq.profiling.analyzer.Analyz… | 19270 |
| processProfileRecord                     |  1418 |
| libstreamingFetch                        |   153 |
| SubmitSeries                             |    35 |
```

53% of CPU has no trace context at all. That's GC threads, Kafka housekeeping, JIT compilation, internal bookkeeping. The main gRPC `Analyze` endpoint takes 43%. And there's a `processProfileRecord` step eating 3.2% that you'd never notice in a flat flamegraph because it gets mixed in with everything else.

Before this query, you have 44,000 undifferentiated samples. After it, you have an endpoint-level CPU budget.

### 2. Which endpoint is driving GC?

```
events/datadog.ObjectSample
  | decorateByKey(datadog.Endpoint,
      key=localRootSpanId, decoratorKey=localRootSpanId,
      fields=endpoint)
  | groupBy($decorator.endpoint, agg=sum, value=weight)
```

```
| endpoint                                 | allocation weight |
+------------------------------------------+-------------------|
| com.datadoghq.profiling.analyzer.Analyz… |      1,237,346 KB |
| (untraced)                               |        161,972 KB |
| processProfileRecord                     |         94,296 KB |
| libstreamingFetch                        |          2,396 KB |
```

Same idea, different signal. The gRPC `Analyze` endpoint is responsible for 83% of all allocation weight: about 1.2 GB in two minutes. That's almost certainly what's feeding the 116 GC pauses and the 3.6-second full GC.

When you see 116 GC pauses in a summary, the natural question is "caused by what?" Allocation profiling alone tells you which object types are hot, but not which business operation produced them. This one query connects the two.

### 3. Wall-clock profile by trace operation

```
events/datadog.MethodSample
  | decorateByKey(datadog.Endpoint,
      key=localRootSpanId, decoratorKey=localRootSpanId,
      fields=operation)
  | groupBy($decorator.operation)
```

```
| operation            | count |
+----------------------+-------+
| grpc.server          | 10686 |
| processProfileRecord |  1017 |
| (unmatched)          |  1697 |
| libstreamingFetch    |    82 |
| SubmitSeries         |    10 |
```

`MethodSample` events are wall-clock samples. They capture what threads are doing regardless of whether they are on CPU or waiting. That makes this a different lens than query #1: it includes time threads spend parked, blocked, or sleeping. Decorating them with the endpoint's `operation` field shows that `grpc.server` dominates at 79% of traced wall-clock time. That's surprising, because the heavy lifting in this service is `processProfileRecord`, the step that actually analyzes incoming profiles. It only accounts for 7.5% here. So the question becomes: what's the gRPC layer doing with all that time? Is it serialization overhead, TLS, frame handling? The wall-clock view raises questions that a CPU-only profile would not.

### 4. How much CPU is spent on failing traces?

This one is my favorite.

```
events/datadog.ExecutionSample
  | decorateByKey(datadog.ExceptionSample,
      key=localRootSpanId, decoratorKey=localRootSpanId,
      fields=sampled)
  | groupBy($decorator.sampled)
```

```
| error-tainted? | count |
+----------------+-------+
| true           | 22529 |
| (clean/none)   | 21925 |
```

This doesn't correlate CPU with endpoints. It correlates CPU with exceptions. Every CPU sample whose trace also threw at least one exception gets tagged. Every sample from a clean trace, or from background work with no trace at all, falls into the other bucket.

The result: **50.7% of all CPU is spent on traces that also threw at least one exception.**

Now, a caveat. The profiler tracks all exceptions, caught and uncaught alike. A `try/catch` around a socket timeout still produces an exception sample. So "error-tainted" doesn't necessarily mean "failing." Some of those traces might be perfectly healthy code paths that just happen to use exceptions for control flow (looking at you, Java I/O).

Still, 50% is a striking number. It tells you that half your compute shares traces with exception activity. Whether that is harmless or a real problem, you now know where to look. Filter those traces by exception type and you will quickly separate the expected `EOFException` noise from the real errors.

### 5. Heap allocations by wall-clock thread state

```
events/datadog.ObjectSample
  | decorateByKey(datadog.MethodSample,
      key=localRootSpanId, decoratorKey=localRootSpanId,
      fields=state)
  | groupBy($decorator.state, agg=sum, value=weight)
```

```
| thread state | allocation weight |
+--------------+-------------------|
| RUNNABLE     |      1,046,390 KB |
| (unmatched)  |        261,461 KB |
| PARKED       |        188,159 KB |
```

This joins two completely different profiling signals: heap allocation samples and wall-clock thread state samples, correlated by trace. For each allocation, it grabs a wall-clock sample from the same trace and pulls the thread state at that moment.

A word of caution: this is a loose correlation. `decorateByKey` picks one wall-clock sample from the matching trace. It doesn't tell you what the thread was doing at the exact instant of the allocation. What it does tell you is whether a given trace was predominantly active or idle during the recording window.

With that in mind: 1 GB of heap allocations belong to traces that also show RUNNABLE wall-clock activity. No surprise there. But 188 MB belong to traces where the sampled state was PARKED. That doesn't prove the thread was parked when it allocated, but it flags those traces as worth a closer look. Why is a trace that spends time parked also producing meaningful heap pressure?

## Why This Matters

Every one of these queries takes two things that JFR already records and asks a question that neither can answer alone. CPU samples don't know which endpoint they serve. Allocation samples don't know if their trace threw an exception. Exception counts don't know how much CPU the failing traces consumed.

The data was always there. The recording has always contained all of it. What was missing was a way to connect the signals without leaving the query language and reaching for external scripts.

`decorateBy` is that connection. And it's not limited to Datadog-specific events. Standard JDK events work the same way. Decorate execution samples with `jdk.JavaMonitorEnter` by time overlap to find CPU under lock contention. Decorate allocation samples with GC phase events to find which allocations happen during collection. Decorate file reads with endpoint spans to attribute I/O to business operations. Any two event types in the same recording can be joined, as long as they share a time range or a key field.

The recording is always richer than a single-event-type query can show. Most of the interesting performance stories live in the space between two signals.

> **Sidebar: the bug that almost ate this post.**
> When I first ran these queries, every single one returned zero matches. It turned out there was a path-navigation bug in `DecoratedEventMap`: the `$decorator.field` path was being split into two steps (`"$decorator"` then `"field"`), but the map only recognized the full string `"$decorator.field"` as a single key. A [four-line fix](https://github.com/btraceio/jafar/commit/3cfad01) made the bare `"$decorator"` lookup return the decorator map directly, and everything started working. Lesson: when your feature silently returns empty results for every input, it's probably not a data problem.

---

*jfr-shell is part of [JAFAR](https://github.com/btraceio/jafar) and available via [JBang](https://jbang.dev) and [Maven Central](https://central.sonatype.com/).*
