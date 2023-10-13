---
layout: post
title:  "Seeing In Context: A journey to contextualized JFR profiles (part 2)"
date:   2023-10-03 15:00:00 +0200
categories: java jvm jfr openjdk profiling performance
---

1. TOC
{:toc}

# Applying context to JFR events

In the [previous blog post]({% post_url 2023-09-13-seeing-in-context_1 %}) I talked about the reasons we want to have
the profiling (well, observability, in general) data contextualized, how we, at Datadog, tried different approaches to
recording the context and ended up with something which is more than acceptable.

The downside of doing the context capture in a 3rd party library as opposed to directly in JFR is that none of the built-in
JFR events will be contextualized. And this means that we are not able to use those events efficiently together with our
profiler events which are contextualized.

## A bit of history

The idea of context is not new to JFR. It was discussed first a long time ago, when JFR was still a part of JRockit, 
and it was called 'thread coloring', according to [Marcus Hirt](https://hirt.se). But as times went there were other, 
more important features to implement and the context/thread coloring was pushed further and further. In 2021 [Marcus Hirt](https://hirt.se)
filed an [OpenJDK ticket name 'Thread Coloring/Profiling Labels'](https://bugs.openjdk.org/browse/JDK-8264516) and in 
2022 a former colleague of mine, [Ludovic Henry](https://blog.ludovic.dev/) even created a JEP draft for the JFR context.

## It's prototype time

This time, instead of directly opening a new JEP, I decided to create a mostly functional prototype of the JFR context
implementation on top of the current JDK trunk. The prototype is building on all the experience we got implementing and
using the context in our profiler library and the main requirements are:
- reading and writing context must be fast
- context attributes are all strings 
- the amount of memory reserved per thread for context must be deterministic and strictly bound
- the Java API to use the context must be easy to use from injected code, eg. no lambdas

The main parts of the JFR context implementation are:
- Java API to define, register and set the context
- Augmented event types and event serialization to include context data
- Context storage

The prototype lives at a [feature branch of OpenJDK fork](https://github.com/DataDog/openjdk-jdk/tree/jb/jfr_context), 
and it is still under development, although it is kept in buildable state at all times.

### Getting our hands dirty
#### Java APIs for JFR Context
In order to bring some structure into various contexts across the application the context types are to be described in
a similar way to how the custom JFR events are described.
{% highlight java %}
@Name("tracer-context") // the context type name
@Description("Tracer context type, comprised of [traceid, spanid] tuple") // description
public class TracerContextType extends ContextType {

  // attributes are defined as plain public fields annotated by at least @Name annotation
  @Name("traceid")
  @Description("The UUID identifying a single trace")
  public String traceid;

  @Name("spanid")
  public String spanid;

  // it is possible to define a convenience method to set the composite context in one call
  public TracerContextType withValues(String traceid, String spanid) {
    this.traceid = traceid;
    this.spanid = spanid;
    return this;
  }

  // one can define a handy constructor to set the context values at the initialization
  public TracerContextType(String traceid, String spanid) {
    this.traceid = traceid;
    this.spanid = spanid;
  }
}
{% endhighlight %}

Before a certain context type can be used it needs to be registered. An arbitrary number of context types can be 
registered as long as the cumulative number of attributes is within a certain limit (8 attributes in the prototype).

If the context type has been registered, the registration method will return `true`. 

{% highlight java %}
// first you need to register the context type
// all registrations must be done before JFR is initialized
boolean registeredCtx = FlightRecorder.registerContext(TracerContextType.class);

TracerContextType ctx = new TracerContextType();
// it is also possible to query the context type instance about whether is registered/active
if (ctx.isActive()) {
  // set the context attributes and apply the context 
  ctx.withValues("trace-1", "span-1").set();
}
// do some work, emit events

// set a different context
// the activity check can be ommitted as applying a non-active context is an effective noop 
ctx.withValues("trace-1", "span-2").set();
// do some more work, emit more events

// work is done, completely remove the context
ctx.unset();
}
{% endhighlight %}

Since registering the context before JFR is initialized can be really tricky, especially with the possibility of starting
JFR recording at the same time the application starts up, there is an alternative method to allow just-in-time registration
of context types.
This approach is based on `ServiceLoader` and the actual service type is `ContextRegistration`.

{% highlight java %}
public class MyContextRegistration implements ContextType.Registration {
  public Stream<Class<? extends ContextType>> types() {
    return Stream.of(TracerContextType.class);
  }
}
{% endhighlight %}

The `MyContextRegistration` will get discovered by `FlightRecorder` before it is initialized and the `TracerContextType`
will be registered for usage.

#### Augmented event types
Any JFR event type defined by the Java API can be marked as context aware by annotating it with `@ContextAware` annotation.

For the built-in events defined in OpenJDK's `metadata.xml` the way to make an event type context aware is to set the 
attribute `withContext` to `true`
{% highlight xml %}
<Event name="JavaMonitorEnter" category="Java Application" label="Java Monitor Blocked" thread="true" stackTrace="true" withContext="true">
  <Field type="Class" name="monitorClass" label="Monitor Class" />
  <Field type="Thread" name="previousOwner" label="Previous Monitor Owner" />
  <Field type="ulong" contentType="address" name="address" label="Monitor Address" relation="JavaMonitorAddress" />
</Event>
{% endhighlight %}

There are two alternatives for recording the context in an event instance
- add all context attributes as extra event attributes
- ads a simple 'list' attribute for context and then store the context attributes as the context list

In either case, for the best user experience a separate event describing the context attributes should be emitted for
each chunk.

#### Context storage

The context storage is thread-local in nature. The context for a particular thread will always be read/mutated by 
a single thread only. Another peculiarity is that the context needs to be writable from Java and readable from the native
code and the access must be very cheap. 

Although in the profiler library we are using a large chunk of continuous memory to represent a sparse map of context-per-thread
which allows us to create a `DirectByteBuffer` over the memory with light-weight views for each thread the approach taken
in the prototype is different. The context is represented as a small chunk in the thread-local data attached to the OS thread.

The reason is that the thread-local data is slightly easier to reason about and if it turns out that the performance is 
acceptable it may stay like that. So far, the preliminary benchmarks are not showing any reason for not staying with the
per-thread chunks.

##### Challenges of virtual threads

With the virtual threads in picture we need to consider the cost of storing the context copy for each virtual thread
when it is unmounted and restoring the context when it is mounted. This comes as extra memory and extra CPU cycles 
to maintain the context copy. On the other hand, the memory overhead would be quite negligible compared to the heap-persisted
virtual thread stack. The context copy cost would be dominated by memory bandwidth and the mechanical sympathy of the
context placement. For now, the virtual thread support should be considered as highly experimental and will definitely
be adjusted when the JEP related discussion starts. Also, the current implementation throws off some JVMTI asserts about
frame counts where, apparently, although virtual threads are implemented in Java calling an arbitrary Java method from
the transition handling code will break the assumptions. This all will have to be resolved before moving on with the context
implementation.

##### What about Scoped Values?

The [Scoped Values](https://openjdk.org/jeps/429) (currently incubating) shares some features with the proposed JFR context -
namely attaching 'free variables' or 'context' to a particular semantic or runtime scope. Unfortunately, using it directly
for the purpose of supporting of the JFR context is not really possible fo two reasons:
- it is fully implemented in Java space so JFR built-in events would need to make upcall each time they are emitted and this
  would have terrible performance
- the API assumes having the code running with the bound scoped values in a form of lambda or method reference in order to
  clearly demarcate the scope - this makes it extremely hard to use from code injected by the distributed tracer implementations

On the other hand, it should be possible to utilize the [Scoped Values](https://openjdk.org/jeps/429) mechanism when one would
create the code with the context in mind, eg. by having a scoped value of a type that would be a subtype of `ContextType`
and this would act as an integration point between the JFR context and scoped values. 

An example of such an integration might look like this:
{% highlight java %}
private static final ScopedValue<TracerContextType> CONTEXT = ScopedValue.newInstance();

ScopedValue.runWhere(CONTEXT, new TraceContexType("trace-2", "span-1"), () -> {
  CONTEXT.get().set();
  // do work, emit events
  CONTEXT.get().unset();
});
{% endhighlight %}


### Looking for early feedback

As you can see the JFR Context API was vastly simplified compared to the original JEP. The main reason is that the scope-local-like
APIs are extremely complex to implement properly with good enough performance. Also, if we want the scope-local experience
it should be possible to combine the real Scope Local API with the JFR Context API instead of trying to duplicate it directly
in the JFR Context API.

The main point of the proposed API is to be as similar to the custom JFR Event API as possible to allow users to quickly 
grasp the concepts and start being productive in no time.

I would love to hear some feedback on this proposal - you can reach me on X where my handle is [@BachorikJ](https://twitter.com/BachorikJ).

And just to reiterate, the prototype is available [here](https://github.com/DataDog/openjdk-jdk/tree/jb/jfr_context)


_[Continue to part 3]({% post_url 2023-10-13-seeing-in-context_3 %})_
