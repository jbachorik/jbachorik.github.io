---
layout: post
title:  "A Stroll Down Memory Lane: Navigating the intricacies of observing Java Heap usage"
date:   2021-05-26 12:00:00 +0200
categories: java jvm
---

As I've been working diligently on the final touches for the 'Live Heap Profiling' feature, I found myself needing to access 
the current Java heap usage from native code. Despite initial expectations that this would be straightforward, the actual
code manifested into an intricate series of workarounds, or as I may euphemistically call it, a hot mess of hacks.

Unfortunately, there isn't a standardized method for querying the Java heap usage from the native code via either JVMTI or JNI proper.

## Introducing Live Heap Profiling

Let's start with an overview of what I aim to accomplish with the 'Live Heap Profiling' feature.

The goal of this feature is to produce a sample of all live objects on the heap (live refers to objects not yet garbage collected). 
At a conceptual level, this is relatively simple, especially when using the JVMTI Allocation Sampler (introduced in JDK 11) 
to guide the sampling process. The process becomes complex, however, when there is a need to 'reconstruct' the live heap 
size from this data.

Users nowadays expect comprehensive information. They aren't satisfied with just raw samples; they desire to understand 
how these samples relate to the total byte count. Addressing this demand introduces a multitude of challenges.

The [JVMTI Allocation Sampler](https://openjdk.org/jeps/331) has been a critical tool in our standard 'Allocation Profiling 'feature,
helping us estimate the number of bytes each sample covers. We took inspiration from Go runtime's implementation of a similar 
feature, a logical choice considering the JVMTI Allocation Sampler is a ported version of Go's allocation sampler.

However, the 'upscaling' method we used for the live heap samples can't be applied in this case. The issue arises from 
the fact that garbage collection (GC) results in its own subsampling by collecting some instances referenced by the 
allocation samples. This subsampling process lacks known statistical properties that I know of. Yet, the proportions 
between the upscaling factors of the preserved allocation samples should give us a decent approximation of the 'live' 
heap used by those samples. This solution seems fairly straightforward, except it requires access to the Java heap usage, 
ideally right after a GC cycle has finished.

## Integrating Java Heap Usage

Typically, we obtain the Java heap usage by using JMX and executing `ManagementFactory.getMemoryMXBean().getMemoryUsage()`.  
This will yield an instance containing values for initial, committed, used, and maximum heap size. Although this assumes 
JMX is available and initialized, that's generally a safe assumption nowadays.

Next, we need to make this value accessible to the profiler. As the profiler is a native library, we would need to use 
JNI to transfer the value over. This could be feasible if we don't need to execute this action frequently. So far, so good.

However, this only provides us with an 'undead' size value - the size of the heap containing both live objects and those 
not yet processed by the GC, essentially 'undead'. While not optimal, it's a decent starting point.

## Next Up, The Live Heap Size

Once we've set up the code to capture the heap usage and report it to the profiler, we can upscale the live heap samples
to more accurately reflect the actual live heap. This provides users with a clearer picture of what's going on.

But we can still push for better - after all, we're talking about 'Live Heap Profiling' here.

In an ideal world, another field in the memory usage object returned by the JMX call would provide us with the used heap 
size right after the most recent GC cycle. This could give us an accurate estimate of the live heap size as seen by the GC 
itself. But, alas, this field doesn't exist.

A knee-jerk solution would be to subscribe to JMX notifications on all reported `GarbageCollectorMXBean` instances and 
capture the heap usage right after each GC cycle. Although theoretically possible, JMX notifications aren't guaranteed 
to be delivered. Moreover, they tend to generate a lot of superfluous objects that keep the GC busy and could distort 
the profiling results. Adding to the pile of issues, the notifications are delayed, and the usage does not accurately 
reflect the 'clean' state post-GC cycle.

Luckily, there's another mechanism for observing GC activity in JVMTI. One can register a 
[callback](https://docs.oracle.com/en/java/javase/17/docs/specs/jvmti.html#GarbageCollectionFinish) which executes each 
time GC completes its cycle. And, by a fortunate coincidence, we're already using this callback in our profiler to update 
the information about the 'liveness' of the allocation samples.

So, the GC notification mechanism is effective and lightweight (unless you unintentionally wreak havoc with 
[Weak Global References](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#weak_global_references) 
in that callback, causing GC to go haywire). The only missing piece of the puzzle is to capture the heap usage value once 
we know a GC cycle has just finished.

However, this turned out to be a Herculean task. After combing through the JVMTI and JNI documentation, I'm convinced 
that there is a glaring oversight in the APIs, preventing native code from querying the Java heap usage. This discovery 
led me to explore alternative solutions.

### Could We Call Back to Java?

Considering we can get the memory usage via JMX, perhaps we could initiate a series of Java upcalls to retrieve the heap usage.

But, this is a flawed concept. It's ill-advised to initiate Java upcalls from a GC callback due to the process being 
resource-intensive (parameter marshalling, crossing JVM boundary and exception handling are all not exactly free) and 
potentially problematic, especially when transitioning back to Java from a GC callback.

### Perhaps Almost Calling Back to Java?

Interestingly, JMX delegates to a native method to gather the heap usage information. Thus, the information is accessible 
from the native code; there's just no clear-cut way to retrieve it from outside the JVM itself.

Did you know it's possible to[intercept](https://docs.oracle.com/en/java/javase/17/docs/specs/jvmti.html#NativeMethodBind)
the binding of Java native methods to their native counterparts? You can do this from JVMTI, and this interception provides 
you with the address of the native function to bind to. Pretty neat, right?

We can exploit this interception to establish a function pointer to the native method that generates the MemoryUsage object 
for JMX. Once we have the function pointer, we can call the function, use JNI to break down the returned object into its 
primitive components, and utilize that information. This eliminates the need for Java upcalls and is relatively safe to 
execute even from the GC callback.

So, this seems to be a viable option. It does, however, have its shortcomings. The profiler must be initialized before 
the `MemoryMXBean.getMemoryUsage()` method is first invoked, otherwise, the native method would have already been bound. 
Also, the profiler must force the binding by calling that method upon initialization; otherwise, the function pointer 
won't be captured unless some other code calls `MemoryMXBean.getMemoryUsage()`.

These are some potential drawbacks, but perhaps we can manage them.

Yet, could we do even better?

### Welcome, VMStructs

While examining the JDK sources, I realized that the[CollectedHeap](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/gc/shared/collectedHeap.hpp) 
class provides most of the information I need. But how do I access this internal class, and particularly, a usable instance of it?

Interestingly, there's a VMStructs class (defined in[vmStructs.hpp](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/runtime/vmStructs.hpp) 
and [vmStructs.cpp](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/runtime/vmStructs.cpp) files), originally 
created for "Sun Solaris Studio" (similar to the renowned AsyncGetCallTrace). This class has a well-known memory layout, 
described in the[vmStructs.hpp](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/runtime/vmStructs.hpp) file, 
which can be used to access many intriguing JVM internals.

The[async-profiler](https://github.com/async-profiler/async-profiler), which our[Java profiler](https://github.com/DataDog/java-profiler) 
heavily relies on, is already using VMStructs. Without it, most of the wizardry the [async-profiler](https://github.com/async-profiler/async-profiler) 
performs wouldn't be feasible.

You can get the address of the [CollectedHeap](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/gc/shared/collectedHeap.hpp) 
instance from VMStructs, identified as the `_collectedHeap` field definition (not the C++ class field). In reality, 
the address points to a [CollectedHeap](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/gc/shared/collectedHeap.hpp) 
subclass instance (e.g., G1CollectedHeap), but this is an implementation detail. The key point is that we can reach the
[CollectedHeap](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/gc/shared/collectedHeap.hpp) instance to obtain 
the heap memory usage.

Once we have the heap instance, we need to be able to call its methods. However, there are no exported header files we could use, 
so we will extend what the [async-profiler](https://github.com/async-profiler/async-profiler) does and try resolving symbols from the 
`libjvm` library. This process is manual and requires identifying the C++ mangled symbols corresponding to the source file 
methods we're trying to invoke. This task is made even more tedious by the fact that the symbols can and do vary between 
JDK versions and vendors.

After some investigation, I identified the following symbols to use:

*   `_ZN13CollectedHeap12memory_usageEv` for JDK 17+
*   `_ZN13CollectedHeap19create_heap_summaryEv` for JDK 11

I didn't attempt to find the symbol for JDK 8 because live heap profiling isn't supported on JDK 8, but the process would
be the same if you ever need this functionality for that particular Java version.

With these function pointers and the[CollectedHeap](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/gc/shared/collectedHeap.hpp) 
instance, we can actually make the calls to get the heap usage. Naturally, these calls will return JDK version-specific 
objects, whose layouts would need to be copied from the JDK sources, unless you're willing to manually interpret the 
memory bytes as field values.

And there you have it! Well, it wasn't exactly a walk in the park. It took me a while to piece together all the components 
I needed for this approach to work. However, it is operational and quite lightweight - as long as you're running on a Hotspot VM.

Unfortunately, other VMs are not supported as they lack the necessary symbols, VMStructs, or both.

### What More Can We Extract From CollectedHeap?

While examining the [CollectedHeap](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/gc/shared/collectedHeap.hpp)
class to determine which methods should be used to retrieve heap usage, I noticed two very intriguing fields in JDK 17:

*   `capacity_at_last_gc`
*   `used_at_last_gc`

These fields contain historical GC information updated with each GC cycle.

This seems to be exactly what I was looking for - data on live heap size at the time of the last GC cycle! Regrettably, 
this information is only available for JDK 17+ - but it's better than nothing.  
In fact, this information is also available in JDK 11, but only for some of the `CollectedHeap` subclasses.  
Making use of that would require some form of runtime introspection to determine which 
[CollectedHeap](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/gc/shared/collectedHeap.hpp) subclass instance 
is stored in the `_collectedHeap` field, and that's a challenge I haven't tackled yet.

## Bringing It All Together

Recall when I referred to the final implementation of this functionality as a hot mess of hacks?

Here's the reason - to support the range of JDK versions and vendors, we need to ensure compatibility with all three 
modes: native-binding interception to get the JMX MemoryUsage equivalent, natively getting heap usage in the GC callback, 
and using the historical GC information from[CollectedHeap](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/gc/shared/collectedHeap.hpp)
when it's available. We need to try and capture the data for all these modes and then dispatch to the supported mode just in time.

In summary, the best performance will be available for JDK 17+ with Hotspot. Acceptable performance will be achieved with 
JDK 11 (again, with Hotspot), and any other setup will at least mostly work.

While the solution is not without its downsides and might appear daunting, it works well in practice. Now, we have live
heap profiling at our disposal, providing detailed insight into the runtime heap and effectively enhancing the capabilities of 
our [Java profiler](https://github.com/DataDog/java-profiler) once the changes are properly tested and merged in.