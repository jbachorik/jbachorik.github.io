---
layout: post
title:  "Improved JFR allocation profiling in JDK 16"
date:   2021-01-24 12:00:00 +0200
categories: java jfr
---

JFR (JDK Flight Recorded) has been providing the support for on-the fly allocation profiling for a while with 
the help of **ObjectAllocationInNewTLAB** and **ObjectAllocationOutsideTLAB** events.

<div style="background-color: #f5f5f5;padding: 25px;border-radius: 5px">
<b><i>Short excursion - what is TLAB?</i></b>
<hr/>

TLAB stands for Thread Local Allocation Buffer and it is a region inside Eden, which is exclusively assigned to a thread. 
Each thread has its own TLAB. Thanks to that, as long as objects are allocated in TLABs, there is no need for any type 
of synchronization.  
Allocation inside TLAB is a simple pointer bump(that’s why it’s sometimes called pointer bump allocation).
<br/><br/>
For more detailed info I recommend reading the following articles [
<a href="https://dzone.com/articles/thread-local-allocation-buffers">https://dzone.com/articles/thread-local-allocation-buffers</a>, 
<a href="https://shipilev.net/jvm/anatomy-quarks/4-tlab-allocation">https://shipilev.net/jvm/anatomy-quarks/4-tlab-allocation</a>] - 
or simply search the web for `java TLAB` and follow one of many blogs, articles and StackOverflow entries available out there.
</div>

**ObjectAllocationInNewTLAB** event is emitted each time the TLAB is filled up and contains the information about the instance type, the instance size and the full stack trace where the allocation happened.

**ObjectAllocationOutsideTLAB** event is fired when the size of the instance to be allocated is bigger than the TLAB size. Also for this event JFR provides the instance type, size and the stacktrace where the allocation was initiated.

### TLAB based allocation sampling

By utilising the properties of these two events, namely the fact that they are emitted **only** when TLAB is filled up or the instance size is bigger than TLAB we can relatively cheaply obtain a heap allocation profile which can help us to identify allocation hot-spots.

#### Expectations vs. reality

At [DataDog Continous Profiler](https://docs.datadoghq.com/tracing/profiler/) we have been using the described technique to provide continuous, always on heap allocation profiling with great success. Until we started receiving reports of significant performance degradation from our internal teams. And the performance was restored the moment they disabled the TLAB related events and as such our allocation profiling.

Well, it turns out that some applications can generate a huuuuge amount of TLAB events simply due to the enormous number of allocations. And while collecting stacktraces in JFR is pretty swift, when the number of collections crosses a certain number it becomes certainly visible. The performance problems were made more prominent thanks to [a bug in the stacktrace hashode calculation](https://bugs.openjdk.java.net/browse/JDK-8250928) which has been fixed since then by my DataDog colleague Gabriel Reid and backported to all relevant JDK versions (JDK 8u282, 11.0.10 or 15.0.2 mentioning a few).

In addition to this performance regression the TLAB events contributed to a very significant increase in the recording size - again causing problems for our continuous profiler which needs to deal with hundreds of thousands such profiles daily. And to solve the size problem we would need a new way of collecting allocation samples which would guarantee a maximum number of samples per recording (to have a predictable recording size) while still providing statistically accurate picture.

# Rate limited allocation profiling

This is an incremental improvement on top of the TLAB based allocation profiling. It is using the same data source (TLAB filled up, outside of TLAB allocation) but is applying an adaptive throttling mechanism to guarantee the maximum number of samples per recording (or time unit, more generally speaking).

The samples are emitted as **ObjectAllocationSample** events with _throttled_ emission rate. The emission rate is customisable in JFC templates and defaults to 150 samples per second for the 'default' JFR profile and 300 samples per second for the 'profile' JFR profile. 

The generic throttling option has been built into JFR but is currently in use only by the **ObjectAllocationSample** event.

## Throttling sampler - implementation details

The main objective is to reduce the number of events emitted per time unit while maintaining statistical relevancy. Typically, this could be done by eg. [reservoir sampling](https://en.wikipedia.org/wiki/Reservoir_sampling) where a fixed size of random elements is maintained for the duration of the specified time unit.  
Unfortunately, this simple approach would not work for JFR events - each event is associated with its stacktrace at the moment when the event is committed. Committing the event, in turn, writes the event into the global buffer and effectively publishes it. Because of the inability to split the event stack collection and publication a slightly more involved algorithm had to be devised.

### JFR Adaptive Sampler

JFR adaptive sampler provides a generic support for controlling emission rate of JFR events. It is an 'online' sampler - meaning that the decision whether an event is to be sampled or not is done immediately without requiring any additional data structures like it is in the case of reservoir sampling. At the same time the sampler dynamically adjusts the effective sampling rate in order not to cross the maximum emission rate within the given time range (eg. per second) but still provide a good number of samples in 'quieter' periods.

Conceptually, the adaptive sampler is based on [PID controller](https://en.wikipedia.org/wiki/PID_controller) theory although it is missing the derivational part making it more similar to [PI controller](https://en.wikipedia.org/wiki/PID_controller#PI_controller). Also, due to the additional limits imposed by the fact that the sampler is going to be used in latency sensitive parts of JVM the implementation had to be embellished with several heuristics not found in the standard [discrete implementation](https://en.wikipedia.org/wiki/PID_controller#Discrete_implementation).

#### Quantization

The adaptive sampler work is quantised into a series of discrete time windows. 
An effective sampling rate (represented as a sample probability 0<= P<sub>w</sub><=1) is assigned to each time window 
and that rate will stay constant for the duration of that particular time window. 
That means that the probability by which an event is picked to sample is constant within a time window.

#### Dynamic sample probability

In order to adapt to the changes in the incoming data the effective sample probability is recalculated after each time 
window using an estimated number of incoming elements (population) in the next window (based on the historical trend),
the target number of samples per window (derived from the user supplied target rate) and 'sample budget'.

The formula to recompute the sample probability for window 'w' is:

<span style="font-size:larger;">P<sub>w</sub> = (S<sub>target</sub> + S<sub>budget</sub>) / N<sub>w,k</sub></span>
- _P<sub>w</sub>_ = effective sample probability
- _N<sub>w,k</sub>_ = estimated population size calculated as exponential weighted moving average over k previous windows
- _S<sub>target</sub>_ = target number of samples per window = user supplied target rate / window duration in seconds
- _S<sub>budget</sub>_ = sample budget = sum(S<sub>i</sub> - S<sub>target</sub>)<sup>w-1</sup><sub>w-x</sub>

_Note: budget is calculated per fixed one second interval and 'x' is the number of windows since the beginning of that interval_

#### Sample budget
As already mentioned in the previous paragraph the adaptive sampler is employing the concept of 'sample budget'. This budget
serves as a 'shock absorber' for situations when the prediction is diametrically different from the reality. It is exploiting 
the relaxation in terms of obeying the sampling rate only for time intervals longer than X windows - meaning that the actual
effective sampling rates observed in each separate time window within this interval can be significantly higher or lower than 
the requested sampling rate.  
Hence the 'budget' term - windows producing fewer samples than the requested number will 'store' unused samples which can 
later be used by other windows to accommodate occasional bursts.

## Conclusion
The introduction of a throughput management mechanism in JFR allows getting fine details about the application behavior 
without the risk of being overwhelmed by the sheer number of JFR events.
The results of our preliminary tests of the setups previously completely unable to run with the allocation profiling 
events turned on are very exciting - JFR with event emission rate controller is able to provide a clear statistical picture 
of the allocation activity while keeping the recording size at a very manageable level thanks to the limit imposed on 
the number of captured TLAB events.

Also, let's ponder the fact that the event rate emission control (throttling) is not TLAB event specific and can be used 
for other event types as well if there is such demand. Although I am quite sure there will be no more throttled events 
in JDK 16 which is in its stabilisation phase as of writing of this blog it might be the good time to take a look at 
the potential candidates for JDK 17.