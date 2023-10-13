---
layout: post
title:  "Seeing In Context: A journey to contextualized JFR profiles (part 3)"
date:   2023-10-12 15:00:00 +0200
categories: java jvm jfr openjdk profiling performance
---

1. TOC
{:toc}

# Integrating with existing tracers

I've already shown the [basic concepts of profiling context]({% post_url 2023-09-13-seeing-in-context_1 %}) and talked 
about [how the proposed JFR API would look like]({% post_url 2023-10-03-seeing-in-context_2 %}).

Although having a shiny new API to deal with the profiling context is nice, we cannot forget about the existing code 
already dealing with the context - the distributed tracers. Whether we talk about[OpenTelemetry](https://opentelemetry.io/),
[OpenTracing](https://opentracing.io/), [OpenCensus](https://opencensus.io/), or any of the proprietary implementations, 
the proposed JFR API must allow for an easy, cheap, and non-intrusive integration.

## Searching for integration points 

A brief look at [OpenTelemetry](https://opentelemetry.io/) and  [DD Tracer](https://github.com/DataDog/dd-trace-java) confirmed 
that they are already dealing with the context problem in a broader way than what the proposed JFR API is supposed to tackle. 
Namely, the distributed tracers are taking care of maintaining the context stack and propagating the context around via extensive 
instrumentation. And the propagation is not restricted to the same JVM but, as the 'distributed' part suggests, they are
able to transfer the context to other processes, hosts, and networks just as easily.

The JFR API proposal was never meant to deal with the distributed context in all of its nuances - rather it is supposed 
to be a public and supported way any tracer implementation could use to make the JFR recordings aware of the context 
they are working with.

The fact that the tracers are already maintaining their own context rules out using the custom ContextType directly as 
that would require either a parallel machinery to maintain the context stack and perform the context propagation or 
setting up mirroring parts of the tracer context to the custom ContextType, potentially resulting in much increased 
allocation rates and memory copies. As we all can see, this approach would not meet any of the 'easy', 'cheap', 
or 'non-intrusive' requirements.

## Exposing ContextAccess

Instead of relying on the stateful context representation via a custom `ContextType`, I propose providing a `ContextAccess`
which can be generated for any type (POJO) with the only requirement of the class being annotated by `@Name` and at least
one field or method returning a value being annotated by `@Name` as well.

Once such `ContextAccess` instance is obtained, it can be used to `set` or `unset` the context from a specific instance of
the given type. The access implementation will use the information about which fields/methods are annotated by `@Name` to
properly fill the context slots by the values extracted from those fields and methods.

#### A note about picking the annotation type
The `@Name` annotation was picked as something non-intrusive and is guaranteed to exist in all supported versions of OpenJDK,
as long as it was compiled with JFR enabled.

An alternative would be to use specific annotations for context type declaration, but that would require an ASM-based annotation 
detection as opposed to the currently used reflection-based one. This is because the annotations coming from a newer JDK 
would not be resolvable in older JDKs.

## Supporting primitive types for context values

In order to facilitate zero-conversion integrations, it is necessary to increase the set of supported context attribute types.
If we stick to supporting only `string`, the amount of conversions between, e.g., long values and strings would definitely
become rather costly, as it would have to be done every single time the context is manipulated.

Instead, the definition of a context type is relaxed to allow attributes of any primitive type as well as string/charsequence.
This comes at no extra cost since JFR already supports all the primitive types anyway.

### DDSpanContext example

#### Modify DDSpanContext class

{% highlight java %}
// expose only `rootspanid', `spanid` and `operation_name` attributes.
@Name("dd-context") // add this annotation such that type can be accepted for context 
public class DDSpanContext
  implements AgentSpan.Context, RequestContext, TraceSegment, ProfilerContext {
  // bunch of fields defining the context - all of them private

  @Name("rootspanid") // context attribute getter method
  public long getRootSpanID() {
    return rootSpanID;
  }

  @Name("spanid")
  public long getSpanID() {
    return spanID;
  }

  @Name("operation_name")
  public CharSequence getOperationName() {
     return operationName;
  }

  // more context stuff
}
{% endhighlight %}

#### Obtain and use ContextAccess instance

{% highlight java %}
DDSPanContext context = new DDSpanContext(...);
ContextAccess<DDPspanContext> access = ContextAccess.forType(DDSpanContext.class);
...
// context is all set up, just activate it
access.set(context);
...
access.unset(); // deactivation is not instance specific
{% endhighlight %}

If you want to see the actual working integration, you can find this [draft PR](https://github.com/DataDog/dd-trace-java/pull/6013)
quite interesting. The PR description contains the guide for building the patched DD tracer agent to run against the patched OpenJDK 21.

The original JFR context prototype had to be backported from OpenJDK 22 because the DD tracer cannot be built and used on 
OpenJDK 22 due to the lack of support in [ASM](https://asm.ow2.io/). This is because [ASM](https://asm.ow2.io/) can support 
only released JDK versions.

_The captured JFR would look something like this in JMC_

![JFR recording with context]({{ site.baseurl }}/assets/images/2023-10-13-seeing_in_context_3/JFR_context_JMC.png)

# Wrap up

In this blog series, I have attempted to build a case for introducing the notion of context to JFR events. To support 
my case, I also proposed the API form and provided a prototype implementation for OpenJDK, as well as a PoC for 
integration with the Datadog tracer agent.

My impression is that the current API form suits the purpose of building context-aware applications and libraries, 
and it also integrates well with existing context-tracking solutions.

The implementation itself is prototypical, covering mostly only the happy paths. However, it should be sufficient to 
gather initial feedback about the feasibility of this approach and to serve as the foundation for the JEP preparation phase.

As always, if you wish to try out the API or consider adding an integration for your favorite tracer, please reach out 
to me on X via [@BachorikJ](https://twitter.com/BachorikJ).