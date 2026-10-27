---
layout: post
title: "BTrace 3.0: Same Old Trick, New Bag"
# Draft: set the real date (and move to _posts/YYYY-MM-DD-btrace-3-0-0.md) when 3.0.0 is out.
date: 2026-11-01
categories: [java, btrace, jvm, observability]
tags: [btrace, jvm, bytecode, instrumentation, extensions, oneliner, mcp]
---

1. TOC
{:toc}

## A bit of history

BTrace is old. Not "legacy" old, but old enough that when I wrote about [unattended execution](/posts/btrace-unattended) back in 2020 I was already calling its origins "more than 10 years ago". The core trick has not changed since: you write a small, restricted Java class, point it at a running JVM, and the agent weaves your probe into the bytecode of the methods you care about. No restart, no recompilation of the application, and the verifier makes sure your probe cannot take the JVM down with it.

The trick still works. What has aged is everything around it.

The last 2.x release, 2.2.6, went out in November 2024. By then the list of things that bothered me about BTrace was long enough that no amount of patch releases would fix it:

- Three jars (`btrace-agent.jar`, `btrace-boot.jar`, `btrace-client.jar`) that had to sit next to each other on disk. Fine for a tarball, a disaster for [jbang](https://www.jbang.dev/) or Maven.
- Asking a one-line question ("which method is slow?") meant writing a class with imports, annotations and `BTraceUtils.` on every call.
- A package name, `org.openjdk.btrace`, pointing to a place BTrace has not lived in for years.
- GPLv2 with the Classpath Exception. Perfectly fine license, perfectly reliable way to start a conversation with someone's legal department.
- Every new JDK release meant waiting for an ASM release, bumping it, and hoping the instrumentation still produced the right bytecode.
- The `libs=` profiles, which worked by quietly appending jars to the global classpath of the target application. What could possibly go wrong.
- An agent started with `-javaagent` happily listening on port 2020 for anyone who wanted to deploy a probe.

So, 3.0.0 is a major release in the proper sense of the word. It is also the first release where I would describe BTrace as pleasant to use rather than merely useful. Let me walk through what changed, roughly in the order you will bump into it.

## New address

BTrace now lives in `io.btrace`. The sources, the Maven coordinates (`io.btrace:btrace`), the runtime API, all of it. And the license is now [Apache 2.0](https://github.com/btraceio/btrace/blob/develop/LICENSE).

If you have 2.x scripts as source, there is a one-line mechanical rename, and a [helper script](https://github.com/btraceio/btrace/blob/develop/scripts/migrate-btrace-script.sh) that does it in bulk with `.bak` backups. If you have *compiled* 2.x probes sitting in a deployment somewhere, you do nothing at all. The agent scans a persisted probe for `org/openjdk/btrace/` references at load time and, only if it finds any, rewrites every class, method, field, descriptor and annotation reference to `io/btrace/` in memory. The file on disk stays as it is. The same path also upgrades truly ancient probes compiled against `com/sun/btrace`, because apparently those still exist.

## One jar

The three-jar layout is gone. `btrace.jar` is the agent, the client and the launcher at once; its manifest declares `io.btrace.boot.Loader` as `Premain-Class`, `Agent-Class` and `Main-Class`.

The obvious way to do this would be to dump everything into one jar with `Boot-Class-Path: .` and call it a day. That would also push roughly 1500 classes (ASM, the compiler, the instrumentation engine, the client) into the bootstrap classloader of every application BTrace touches. Namespace pollution, ASM version conflicts with the application, memory for nothing.

Instead, the jar uses *classdata masking*, a trick I borrowed from the [dd-trace-java](https://github.com/DataDog/dd-trace-java) agent, which has been shipping one jar with hidden class data for years. Only a small core API (about 110 classes) is stored as regular `.class` files and is visible to the bootstrap loader. Everything else (about 1600 classes) is stored under `META-INF/btrace/` with a `.classdata` extension. The JVM does not consider those to be classes at all, so they are invisible to the application. A tiny `MaskedClassLoader` loads them on demand when the agent or the client actually needs them. Same jar, two very different views of its contents. The details, including why this was needed for jbang to work at all, are in the [Masked JAR Architecture](https://github.com/btraceio/btrace/blob/develop/docs/architecture/MaskedJarArchitecture.md) document.

The practical upshot: `jbang io.btrace:btrace:3.0.0 <pid> script.java` just works, `sdk install btrace` just works, and `docker pull ghcr.io/btraceio/btrace:3.0.0` gets you an image (there are `-alpine` and `-distroless` variants too) you can copy the distribution out of as a layer.

## Stop writing files for one-line questions

This is the one I use the most. BTrace 3.0 has oneliners, shamelessly inspired by DTrace:

```sh
btrace -n 'OrderService::* @return if duration>200ms { print method, time }' <PID>
```

<video controls muted playsinline preload="metadata" style="max-width: 100%;">
  <source src="{{ site.baseurl }}/assets/images/btrace-3-0-0/01-oneliner.mp4" type="video/mp4">
</video>

The grammar is small: a class pattern, a method pattern, a location (`@entry`, `@return`, `@error`), an optional filter, and a list of actions (`print`, `count`, `time`, `stack`). Patterns take wildcards or `/regex/`. The filter can look at `duration` or at `args[N]`.

```sh
# who is constructing exceptions, and from where
btrace -n 'java.lang.Exception::<init> @entry { print self, stack(5) }' <PID>

# how often is this cache hit
btrace -n 'com.myapp.Cache::get @entry { count }' <PID>

# the swallowed exception, with the call path
btrace -n 'OrderService::validateOrder @error { print method, stack }' <PID>
```

There is no magic here. The client turns the oneliner into a regular `@BTrace` class, compiles it, and deploys it like any other probe. You can even see the generated source. The property has to reach the `btrace` client, not the application, and the launcher passes anything you append to the command line straight to the client's argument parser, so set it through `JAVA_TOOL_OPTIONS`:

```sh
JAVA_TOOL_OPTIONS="-Dbtrace.oneliner.dump=true" btrace -n 'OrderService::processOrder @return { print method, time }' <PID>
```

A oneliner is a real probe with the training wheels on, which also means it goes through the same verifier as everything else.

When you outgrow a oneliner, the next step is a script file, and that got shorter too. The flat DSL drops the `BTraceUtils.` prefix, and the compiler injects `import static io.btrace.BTrace.*` and `import io.btrace.core.annotations.*` into any script that does not import the DSL itself. The minimal script is now literally this:

```java
@BTrace
public class OrderTiming {
    @OnMethod(clazz = "OrderService", method = "processOrder", location = @Location(Kind.RETURN))
    public static void onReturn(@ProbeMethodName String method, @Duration long duration) {
        println(method + " took " + str(duration / 1_000_000) + "ms");
    }
}
```

No imports. Compare that to what a 2.x script looked like and you will understand why I kept putting off writing probes for small questions.

## Extensions, or how BTrace learned to speak OTel

<!-- DRAFT NOTE: the btrace-otel extension currently lives only on the claude/project-review-HBcMK
     branch (commit 7fd17c56), still under org.openjdk.btrace, and is NOT on develop or in the
     3.0.0 bundled set. Land it (or move it to btraceio/btrace-extensions) before publishing, or
     reword this section as "coming after 3.0.0". The snippet below is adapted to the 3.0 API. -->

Let me start with the feature and work backwards to the plumbing, because the plumbing only makes sense once you see what it is for.

In 2026 nobody wants a tracing tool that prints to a console and calls it a day. The console is where you *start*; the data has to end up next to everything else, which these days means OpenTelemetry. So here is a BTrace probe that turns method timings into OTLP spans and metrics:

```java
@BTrace
public class CheckoutSpans {
  @Injected
  private static OTel otel;

  @OnMethod(clazz = "com.example.OrderService", method = "checkout", location = @Location(Kind.RETURN))
  public static void onCheckout(@Duration long durationNanos) {
    otel.span("OrderService.checkout", durationNanos);
    otel.counter("orders.checkout", 1);
  }
}
```

The extension runs inside the traced JVM, so it is configured there, with plain system properties on the application's command line. Tell it where your OTLP/HTTP collector listens and what to call the service, allow the extension, and start the application:

```sh
java -javaagent:btrace.jar=allowExtensions=btrace-otel \
     -Dbtrace.otel.endpoint=http://otel-collector:4318 \
     -Dbtrace.otel.service.name=order-service \
     MyApp
```

The endpoint defaults to `http://localhost:4318`, the service name to `btrace`, and the exporter posts to the standard `/v1/traces` and `/v1/metrics` paths under that base URL. Deploy the probe above with `btrace <PID> CheckoutSpans.java` and the spans show up in whatever backend you already run. No SDK in the application, no agent other than BTrace, and the probe can be removed the same way it was added. Notice there is no start and no end call. BTrace already knows when the method returned and how long it took, so the span is built after the fact from the duration, and each one lands in its own trace. Spans and counters are queued on the hot path and shipped by a background exporter thread over OTLP/HTTP in batches, so the instrumented method pays for an enqueue and nothing else.

Now the uncomfortable part. Look at what that extension needs in order to work: an HTTP client, a JSON encoder, a queue, and a daemon thread that opens network connections *from inside the JVM you are observing*. In 2.x there was exactly one way to get code like that into a probe: the `libs=` profiles, which appended jars to the global classpath of the target application and hoped nothing conflicted. On top of that, calling into such a jar required unsafe mode, euphemistically called "trusted", so the verifier was off too. A "safe" tracing tool mutating the application classpath and switching off its own safety net so that it can open sockets. I could never write that sentence with a straight face, so the profiles are gone, and I do not miss them.

The replacement is the extension framework, and OTel is the reason it looks the way it does.

An extension is a pair of jars. A small API jar (the `OTel` interface, nothing else) lands on the bootstrap classpath so probes can link against it. The implementation (the HTTP code, the queues, the exporter thread) is loaded in isolation and bridged to the probe through an `invokedynamic` call site. The application never sees it, and the implementation never sees the application's classes either. `@Injected` is the whole API surface from the script's point of view.

Then permissions. Every extension declares what it needs, and `btrace-otel` declares `NETWORK` and `THREADS`, one for the socket and one for the exporter thread. Both are privileged permissions, and privileged extensions do not link unless the *operator* of the target JVM allows it, either as an agent argument (the `allowExtensions=btrace-otel` in the command above) or in a policy file.

There is deliberately no client-side flag. The person deploying a probe does not get to grant themselves a network socket on a JVM they do not own. The BTrace Gradle plugin even computes most of an extension's permission set by statically scanning the implementation for JDK APIs that imply one, so an extension cannot quietly forget to mention that it uses reflection or threads. When an extension is blocked, the agent says so, and says why:

```
$ btrace -le <PID>
Failed Extensions:
  1. btrace-metrics: Blocked privileged extension. Required=[THREADS, REFLECTION, CLASSLOADER]
```

The same machinery carries the rest of the bundled extensions, and the one I expect people to reach for first is `btrace-metrics`, HdrHistogram under the hood, which is also the replacement for the removed 2.x `Aggregations` API:

```java
@BTrace
public class LatencyHistogram {
  @Injected(optional = true)
  private static MetricsService metrics;

  private static HistogramMetric chargeCardLatency;

  @OnMethod(clazz = "OrderService", method = "chargeCard", location = @Location(Kind.RETURN))
  public static void onChargeCardReturn(@Duration long durationNanos) {
    if (chargeCardLatency == null) {
      chargeCardLatency = metrics.histogramMillis("chargeCard.latency");
    }
    chargeCardLatency.record(durationNanos / 1_000_000);
  }

  @OnTimer(5000)
  public static void report() {
    HistogramSnapshot h = chargeCardLatency.snapshot();
    println("chargeCard p50=" + h.p50() + "ms p95=" + h.p95() + "ms p99=" + h.p99() + "ms");
  }
}
```

Note the `optional = true`. When the operator has not allowed the extension, the probe still deploys and the field is bound to a stub that throws on every call, so the failure is visible and local, not fatal. If you would rather have the calls silently do nothing, ask for a no-op shim instead, either per field with `@Injected(optional = true, mode = SHIM)` or for every optional injection in the target JVM with `-Dbtrace.extension.shimMode=shim`. Without `optional = true` there is no fallback at all and the deployment fails fast.

OTel and metrics are two of the extensions that ship with 3.0. The others are StatsD export, runtime contracts (latency budgets and rate limits checked on live calls), a handful of utilities, and three aimed at AI workloads: LLM call tracing (tokens, latency, estimated cost per model), RAG quality (vector store query latency and similarity scores), and a GPU bridge for ONNX Runtime, DJL and TensorFlow inference timing. The AI ones exist because "which method is slow" has, for a lot of applications, turned into "which model call is slow and how many tokens did it burn". The answer is produced the same way as everywhere else in this post: a probe on the client method, with nothing added to the application and nothing left behind when the probe goes.

Around all of this there is a `btracex` CLI (with a small TUI) to list and inspect extensions and to edit the policy, and an [extension registry](https://github.com/btraceio/btrace-extensions) for third-party ones.

## One jar, part two: the fat agent

Extensions on disk are fine on a laptop. They are not fine on a Spark executor, a Hadoop node, or a distroless container, where there is no `$BTRACE_HOME`, no `extensions/` directory, no shared filesystem, and nobody who wants to copy anything anywhere. Those environments can cope with exactly one thing: a `-javaagent` jar.

So 3.0 ships a Gradle plugin, `io.btrace.fat-agent`, that builds that one jar:

```groovy
plugins {
    id 'io.btrace.fat-agent' version '3.0.0'
}

btraceFatAgent {
    baseName = 'demo-btrace-agent'
    btraceVersion = '3.0.0'

    embedExtensions {
        file('btrace-otel.zip')          // an extension package
        project(':my-custom-extension')  // or a subproject in the same build
        maven('com.acme:acme-ext:1.2')   // or published Maven coordinates
    }
}
```

`./gradlew fatAgentJar` stages the masked BTrace engine, then every extension you listed, and writes a jar whose manifest still says `Premain-Class: io.btrace.boot.Loader`, so it attaches like plain `btrace.jar`.

The interesting part is how the extensions go in, because it is the classdata masking trick again, applied one level up. An extension's API classes (the interfaces a probe links against) are copied into the jar as regular `.class` files, so the bootstrap loader sees them through `Boot-Class-Path`. The implementation classes are renamed to `.classdata` and filed under the extension's own directory, `META-INF/btrace-extensions/btrace-metrics/io/btrace/metrics/MetricsServiceImpl.classdata` for example. The JVM ignores them, and each extension gets its own small classloader that resolves class names only below that prefix. That per-extension prefix is not decoration: it is what lets two extensions each ship a `com.example.Helper` without one silently running the other's bytecode. Each embedded extension also gets a `META-INF/btrace-extensions/<id>/extension.properties`, and the manifest carries a `BTrace-Embedded-Extensions` list that the agent reads at startup instead of scanning a directory. Same isolation as an installed extension, just with the directory folded into the jar.

Two details I like. First, the published `io.btrace:btrace:3.0.0` artifact *is* a fat agent: it is built by this same plugin and already embeds the bundled extension set, which is why `jbang` users get `MetricsService` without installing anything. Your own `embedExtensions` entries are deduplicated by id against that list, so the plugin is really for adding what the engine does not carry: your own extension, or a third-party one. Second, embedding does not launder permissions. The plugin reads `BTrace-Extension-Permissions` from the extension's API jar manifest, refuses to build at all if the attribute is missing rather than defaulting to "no permissions", and copies it into the embedded metadata. At runtime the embedded extension goes through exactly the same privileged gate as a filesystem one, so the deploy line still has to say what is allowed:

```sh
java -javaagent:demo-btrace-agent.jar=allowExtensions=btrace-otel MyApp
```

You can also bake probes in. A `bundledProbes { }` block points at a directory of compiled probe classes (the output of `btracec`, in its package tree) and names the ones to embed by their binary name:

```groovy
btraceFatAgent {
    bundledProbes {
        from layout.buildDirectory.dir('compiled-probes').get().asFile
        include 'com.example.OrderStartupProbe'
    }
}
```

The class ends up at `META-INF/btrace-probes/com/example/OrderStartupProbe.class`, and an agent argument selects which embedded probe starts with the JVM:

```sh
java -javaagent:demo-btrace-agent.jar=probes=com.example.OrderStartupProbe,output=stdout MyApp
```

Probe names are exact binary names, and a name that is not in the jar fails startup loudly rather than being ignored, which also keeps `probes=` from reaching outside the `META-INF/btrace-probes/` namespace. For a Spark job that means one jar in `spark.executor.extraJavaOptions`, with the extension and the startup probe inside it, and nothing else to distribute. The full walkthrough, including what the staged jar looks like when you `unzip -l` it, is [tutorial 08](https://github.com/btraceio/btrace/blob/develop/docs/tutorials/08-fat-agent.md).

## Attaching to production without sweating

Here is the 2.x situation, honestly: a JVM started with `-javaagent:btrace-agent.jar` opened port 2020 and would accept a probe from anyone who could reach it. Dynamic attach was gated by the OS (same user, same host), but the prepared endpoint was not gated by anything.

In 3.0, *prepared mode* (an agent started at launch with no script, waiting for a client) looks like this:

```sh
java -javaagent:/path/to/btrace.jar=port=0 MyApp
```

The agent binds a loopback address only. There is a `bindAddress=` option, but it exists to pick the loopback family (`127.0.0.1` or `::1`); give it a wildcard or a non-loopback address and the agent rejects it, logs one line naming the option, and starts without a command server while the application runs on. The agent then generates an owner-protected token file, and publishes the address, the port and the token file path as JVM properties. A 3.0 client reads those through the Attach API and authenticates before every command. The token never shows up as a process argument or a system property. `port=0` lets the OS pick a free port, so there is nothing to collide with. A 2.x client cannot talk to a 3.0 prepared endpoint, and that is the point.

The other half of this is `btrace doctor`:

```
$ btrace doctor <PID>
BTrace doctor for PID 89156
Status: READY (exit 0)
Target JDK: 17.0.12 (Oracle Corporation)
Attach API: available
Target attach: accessible
Dynamic agent loading: not tested (doctor never loads an agent)
BTrace endpoint: ready at 127.0.0.1:58360
Authentication: required; credential file readable
Actions: read target system properties and detached; no agent load or BTrace command connection was attempted.
```

Doctor reads the published properties through the Attach API and detaches. It never loads an agent and never opens the command socket, which makes it safe to run against anything, including the JVM you are not supposed to touch. The exit codes are stable (`0` ready, `2` preparation required, `3` inaccessible), so a deployment script can gate on the result, and `btrace doctor <PID> --json` prints the same report as JSON for anything that would rather parse it than grep it. When the answer is "not ready", it tells you what to change on the *next* deployment rather than leaving you to guess.

## Leaving without a trace

Attaching is only half of the production story. The other half is what happens when you leave. Anyone who has run a Java agent in production has met the variant that cannot be removed: the bytecode stays patched, a stub keeps getting called on every invocation, and the only way to get the original performance back is a restart, which is exactly what you attached an agent to avoid. For a tool that is meant to be attached for ten minutes and then forgotten, that is a disqualifying property.

Here is the whole cycle on a deliberately unfair target: a single thread calling one cheap method in a tight loop, reporting its own throughput every two seconds. Watch the number drop when the probe lands, and watch it come back after the client exits.

<video controls muted playsinline preload="metadata" style="max-width: 100%;">
  <source src="{{ site.baseurl }}/assets/images/btrace-3-0-0/07-production-safe-attach.mp4" type="video/mp4">
</video>

Two things make the recovery work. Probe handlers in 3.0 are dispatched through `invokedynamic` call sites, and when the client exits those call sites become no-ops. At the same time the agent retransforms the instrumented classes back to their original bytecode. The first mechanism covers the window in which the old bytecode is still executing; the second removes it. The drop itself is the worst case on purpose: the probe handler is empty, so everything you see is call-site dispatch on a method invoked tens of millions of times per second, and a real production method is called orders of magnitude less often. An agent you cannot remove is a liability. This one leaves.

## The ASM ceiling

This one is for the people who, like me, have had to explain to someone why their shiny new JDK broke the tracing tool.

BTrace's instrumentation engine is built on ASM. ASM only parses class files up to the major version it knows about and throws on anything newer. Every JDK release therefore meant: wait for ASM, bump ASM, re-validate the generated bytecode, release. Six months later, repeat.

BTrace 3.0 keeps ASM as the default backend, with a *validated* ceiling of class file version 71 (Java 27). That number is bumped deliberately, after the instrumentation output has been checked on a new class file version, not automatically with every ASM dependency update. Anything above the ceiling is routed to a second backend built on the JDK's own [ClassFile API](https://openjdk.org/jeps/484) (`java.lang.classfile`, final since JDK 24). The JDK always understands the class file format it ships with, so this backend is forward-compatible by construction. Today that means JDK 28 early-access builds (class file version 72) get instrumented without BTrace knowing anything about them in advance.

The selection is embarrassingly simple:

```java
static InstrumentationBackend select(int classFileMajorVersion) {
  if (!ASM.supports(classFileMajorVersion) && CLASSFILE_API != null) {
    return CLASSFILE_API;
  }
  return ASM;
}
```

The ClassFile API backend is compiled against Java 24 and loaded reflectively, so the rest of the agent stays Java 8 compatible and the jar is not even a multi-release jar. On an agent running on JDK < 24 the reflective load fails cleanly and you get the ASM backend only, which is exactly what you had before. All probe kinds and handler parameters are supported on the new backend, with one documented gap: `@Duration` on `SYNC_ENTRY`/`SYNC_EXIT` handlers. The [backend document](https://github.com/btraceio/btrace/blob/develop/docs/architecture/InstrumentationBackends.md) has the rest.

## Wire protocol

Less visible, still worth a paragraph. The client and the agent used to talk Java serialization. 3.0 defaults to a binary V2 protocol: [3-6x faster](https://github.com/btraceio/btrace/blob/develop/docs/architecture/Version2ProtocolArchitecture.md) command transmission and 2-5x smaller payloads, per the protocol benchmarks.

Every connection starts with a negotiation: the receiving side looks for the `BTR2` magic prefix and falls back to V1 if it is missing. So a 2.x client talks to a 3.0 agent and vice versa on dynamic attach, with no configuration. The one wrinkle: a 3.0 client connecting to a 2.x agent waits for a V2 reply that never comes before falling back, which costs about 5 seconds on the first connect. If you know the agent is 2.x, `-Dbtrace.comm.protocol=v1` on the client skips the wait.

## Let the robot do it

I have written about [jfr-mcp](/posts/jfr-mcp-serve) before, so this will not surprise anyone. BTrace has an MCP server, and it now lives in the [btraceio/agent-plugins](https://github.com/btraceio/agent-plugins) repository as the `btrace-observability` plugin, together with a set of skills for latency analysis, failure analysis, probe lifecycle and data safety.

```text
/plugin marketplace add btraceio/agent-plugins
/plugin install btrace-observability@btraceio-agent-plugins
```

The server runs through jbang, downloads the single `io.btrace:btrace` jar on first use (the masked jar earning its keep again), and exposes tools to list JVMs, deploy oneliners and scripts, send events and remove probes. You describe a symptom, the assistant writes a probe shaped like `OrderService::* @return if duration>200ms { print method, time }`, attaches it, reads the output, and removes it.

The part I consider important is that the assistant gets fewer privileges than you have, not more. The MCP tools take a PID, a probe and a port, and nothing else: there is no parameter that turns the verifier off, so the assistant cannot put a JVM into trusted mode even if it wanted to. Privileged extensions still need the operator's policy on the agent side, and the plugin guidance pushes towards narrow probes (an exact method in production, a bounded class for discovery). Read the tool calls, not just the answer. The one thing worth saying no to is an assistant that suggests you run `btrace -u` by hand.

## What breaks

A major version is allowed to break things, and this one does. The complete list is in the [migration guide](https://github.com/btraceio/btrace/blob/develop/docs/Migration-2.x-to-3.0.md); the short version:

| What you have | What happens |
|---|---|
| Compiled 2.x probes | Nothing. Rewritten in memory at load time. |
| 2.x script sources | Rename `org.openjdk.btrace` to `io.btrace`, or delete the imports entirely. |
| `btrace-agent.jar` / `-boot` / `-client` in launch scripts | Point them at the single `btrace.jar`. |
| `libs=<profile>` | Removed. The agent logs an error and loads nothing. Migrate to an extension. |
| `Aggregations` API | Removed. Use `btrace-metrics` for histograms and stats, plain maps and counters for the rest. |
| 2.x client against a 3.0 prepared agent | Rejected. Prepared mode is authenticated now; upgrade the client. |
| Targets on Java 8-16 | Still work for all of 3.x, with a one-time deprecation warning. Gone in 4.0. |

That last row deserves a sentence. BTrace 3.0 runs against Java 8 through 27 and beyond, but I am not going to pretend that testing every probe kind against JVMs released between 2014 and 2021 is a good use of anyone's time in 2026. If the warning bothers you, `-Dbtrace.suppressJavaDeprecationWarning=true` turns it off.

## Try it

```sh
# jbang, no installation
jbang catalog add --name btraceio https://raw.githubusercontent.com/btraceio/jbang-catalog/main/jbang-catalog.json
jbang btrace@btraceio -n 'com.myapp.*::* @return if duration>100ms { print method, duration }' $(pgrep -f myapp)

# or any of these
sdk install btrace
docker pull ghcr.io/btraceio/btrace:3.0.0
curl -LO https://github.com/btraceio/btrace/releases/download/v3.0.0/btrace-v3.0.0-bin.tar.gz
```

The repository ships a set of [replayable demos](https://github.com/btraceio/btrace/tree/develop/demos), each with a `run.sh`, and a [tutorial series](https://github.com/btraceio/btrace/tree/develop/docs/tutorials) that walks the same ground in ten-minute pieces. The recordings in this post come straight from those demos.

BTrace has been asking the same question for well over fifteen years: what is this JVM doing, right now, without me restarting it? 3.0 does not change the question. It just stops making you work so hard to ask it.

---

*BTrace is developed at [btraceio/btrace](https://github.com/btraceio/btrace) under the Apache 2.0 license. The 3.0.0 release notes are [here](https://github.com/btraceio/btrace/blob/develop/.github/release-notes/v3.0.0.md).*
