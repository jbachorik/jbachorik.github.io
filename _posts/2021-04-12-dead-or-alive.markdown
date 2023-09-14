---
layout: post
title:  "Dead or Alive  (A short tale of Java heap live set)"
date:   2021-04-12 12:00:00 +0200
categories: java jfr
---

While working on my now closed (as not done) [PR](https://git.openjdk.java.net/jdk/pull/2579) to provide new JFR events 
containing an approximation of the heap live set size I happened to learn a bit about how the liveness was tracked in 
various GC implementation and how usable (or not) was the heap live set size estimation.

When talking about liveness and heap live set I mean the object graph that GC deems reachable and alive. 
Logically, anything that is not reachable is considered dead.

In this post I will try to summarize my findings in a concise way, mostly as a reminder for me when I need to come 
back to this topic some day, but also in hopes that the others might find it useful as well.

# TL;DR

If you are not interested in all the gory details you can skip this section.

Usually, the GC implementations are not tracking the liveness information explicitly, and it becomes available only shortly 
after a major GC cycle and becomes rapidly outdated.

But it turns out that using a knowledge of the inner workings of various GCs it is possible to create a somewhat 
representative estimate even after each minor GC cycle.

As mentioned above, the way to calculate the estimate is specific to each GC implementation.

## Epsilon GC

Ok. This one is extremely trivial. Anything on the heap is considered live for all that this GC implementation knows. 
Epsilon GC is not performing any garbage collection and therefore has no way of determining the liveness of objects stored on the heap.

## Serial GC

Here we can utilize the fact that mark-copy (minor GC) phase is naturally compacting so the number of bytes after copy 
is 'live' and that the mark-sweep (major GC) implementation keeps an internal info about objects being 'dead' but 
excluded from the compaction effort.

The ‘dead wood’ as those objects are being referred to are unreachable instances which should be removed and the space 
they occupy should be compacted but doing so would have an unreasonable cost when compared to the memory space released 
by such operation.

All this means that we can use the ‘dead-wood’ size to derive the old-gen live set size as used bytes minus the cumulative 
size of the 'dead-wood'.

## Parallel GC

For Parallel GC the live set size estimate can be calculated as the sum of used bytes in all regions after the last GC cycle.

This seems to be a safe bet because this collector is always compacting and the number of used bytes corresponds to the 
actual live set size.

## G1 GC

G1 GC is already keeping liveness information per region.

During concurrent mark processing phase we can utilize the region liveness info and just sum up the liveness information 
from all the relevant regions - by adding the live size summation into `G1UpdateRemSetTrackingBeforeRebuild::do_heap_region()` method.

For mixed and full GC cycles we can use the number of used bytes as reported at the end of `G1CollectedHeap::gc_epilogue()` method.

The downside of G1 GC is that concurrent mark is not happening very frequently (only if IHOP > 45% by default) and full 
GC cycles are basically a failure mode, meaning that the liveness information can go stale quite often.

## Shenandoah GC

Shenandoah, as well as G1, is keeping the per-region liveness info.

This can be easily exploited by hooking into `ShenandoahHeuristics::choose_collection_set()` method which already does 
full region iteration after each mark phase where the per-region liveness would be just summed up.

## ZGC

Actually, ZGC is readily available to provide the information about the live set size at the end of the most recent mark phase. 
The `ZStatHeap` class is already holding the liveness info - it just needs to be made public.

# Conclusion

The 'traditional' GC implementations like Serial GC or Parallel GC are trying really hard to avoid computing the liveness 
information. Usually, the only time they will guarantee an up-to-date and complete liveness information is right after 
full GC has happened and VM is still in safepoint. Once VM leaves the safepoint that information becomes stale almost immediately. 
If one would decide to do a full-heap inspection to calculate the live set size at an arbitrary time point it would have 
to be done in another JVM safepoint and it could easily take several hundred milliseconds, compromising the application 
responsiveness. 

G1 GC keeps the liveness information per region - but that information will usually be updated very infrequently as it is 
done during marking run. And that means that in order to get up-to-date liveness info it is necessary to perform a full-heap 
inspection with all the previously mentioned drawbacks. Good news is that both ZGC and Shenandoah are keeping fairly up-to-date 
liveness info which can be used almost immediately. In their current implementations (at the time of writing this article) 
the liveness information was not exposed publicly but there are no technical reasons why it could not be done - 
either via custom JFR events and/or GC specific MX Beans.