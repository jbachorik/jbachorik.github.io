---
layout: post
title: "Let the AI Debug It: JFR Analysis Over MCP"
date: 2026-02-24
categories: [java, jfr, mcp, performance]
tags: [jfr, jfr-mcp, mcp, claude, ai, profiling, jafar]
---

1. TOC
{:toc}

## The Pitch You Didn't Ask For

A JFR recording is one of the richest artifacts a JVM can produce. CPU samples, GC pauses, memory allocations, thread states, lock contention, I/O latency, class loading, JIT compilations, exceptions — all timestamped, all in one file. The problem was never the data. The problem is that a 200MB recording from a production JVM that melted down at 3am contains *all of it at once*, and the human staring at it has to know which questions to ask, in what order, and how to connect the answers.

Or you could let an AI do that part.

Not "let an AI hallucinate a diagnosis from the filename." Actually let it open the recording, query events, correlate thread states with lock contention, extract resource utilization metrics, run systematic performance methodologies, and explain what it found — with evidence.

That's what [jfr-mcp](https://github.com/jbachorik/jafar/tree/master/jfr-mcp) does. It's a [Model Context Protocol](https://modelcontextprotocol.io/) server that exposes the full depth of JFR analysis as a set of tools that any MCP-capable AI agent (Claude, for instance) can call directly. No copy-pasting terminal output. No narrowing down to a single event type and hoping it's the right one. The AI gets the whole recording, the whole query language, and established performance analysis frameworks to work with.

## What MCP Actually Is (30-Second Version)

MCP is a protocol that lets AI agents call external tools in a structured way. The agent sees a catalog of available tools, each with a name, description, and parameter schema. When the agent decides it needs to, say, open a JFR file, it emits a tool call with the right parameters, the MCP server executes it, and the result flows back as structured data.

Think of it as giving the AI a CLI it can actually use, except it's JSON over stdio instead of bash over a terminal.

## JBang: Zero-Install Java Distribution

Before we get into the MCP server itself, a word about how it's distributed.

jfr-mcp is packaged as a fat JAR — single file, all dependencies baked in. But "download a JAR, find the right Java, set the classpath, run it" is the kind of ceremony that kills adoption. So it's published to a [JBang](https://jbang.dev) catalog instead.

JBang is a launcher that resolves, caches, and runs Java applications with zero setup. It downloads the right JDK if you don't have one, fetches the artifact from Maven Central, caches it locally, and runs it. First invocation takes 10–30 seconds (it's pulling ~70MB of JDK plus the JAR). Every subsequent run starts in under 2 seconds from cache.

If you don't have JBang yet:

```bash
# macOS
brew install jbangdev/tap/jbang

# Linux / macOS (without Homebrew)
curl -Ls https://sh.jbang.dev | bash -s - app setup

# Windows
scoop install jbang
# or: choco install jbang
# or: iex "& { $(iwr https://ps.jbang.dev) } app setup"

# SDKMAN (any platform)
sdk install jbang
```

Or skip all that — the one-liner install script handles JBang installation too:

```bash
curl -Ls https://raw.githubusercontent.com/btraceio/jafar/main/jfr-mcp/install.sh | bash
```

This installs JBang if needed, then installs the `jfr-mcp` command. One script, no prerequisites.

The catalog alias is `jfr-mcp@btraceio` for stable releases and `jfr-mcp-dev@btraceio` for development snapshots. You can run directly without installing:

```bash
jbang jfr-mcp@btraceio          # run without installing
jbang app install jfr-mcp@btraceio   # install as a command, then:
jfr-mcp                              # run by name
```

To update, force a reinstall:

```bash
jbang app install --force jfr-mcp@btraceio
```

For dev snapshots where Maven Central caching may serve stale versions:

```bash
jbang --fresh jfr-mcp-dev@btraceio
```

## Two Transports: STDIO and HTTP

The MCP server speaks two protocols. Which one you want depends on who's talking to it.

### STDIO Mode

```bash
jfr-mcp --stdio
```

In STDIO mode, the server reads JSON-RPC messages from stdin and writes responses to stdout. No network port, no HTTP, no SSE. The server runs as a subprocess of whatever launched it — which is exactly how Claude Desktop and Claude Code expect to talk to MCP servers.

This is the mode you want for AI integration. The client (Claude) spawns the server process, sends tool calls over stdin, reads results from stdout, and kills the process when the conversation ends. Each invocation is a fresh instance. No port conflicts, no "is the server already running" questions.

### HTTP/SSE Mode

```bash
jfr-mcp                          # default: port 3000
jfr-mcp -Dmcp.port=8080          # custom port
```

In HTTP mode, the server starts a Jetty instance exposing two endpoints:

- `/mcp/sse` — Server-Sent Events stream (establishes a session)
- `/mcp/message` — JSON-RPC message endpoint

This is for web-based MCP clients, custom integrations, or manual testing with `curl`. The server auto-detects if the port is already in use and exits silently — so you won't accidentally start two instances fighting over port 3000.

For most people reading this post, you want `--stdio`.

## Wiring It to Claude

### Claude Code

One command:

```bash
claude mcp add jafar -- jbang jfr-mcp@btraceio --stdio
```

That registers an MCP server named `jafar` that Claude Code will start using JBang in STDIO mode. Next time you start a conversation, Claude has 13 new JFR analysis tools available. No restart needed.

### Claude Desktop

Edit `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) or `%APPDATA%\Claude\claude_desktop_config.json` (Windows):

```json
{
  "mcpServers": {
    "jafar": {
      "command": "jbang",
      "args": ["jfr-mcp@btraceio", "--stdio"]
    }
  }
}
```

Restart the app. The hammer icon in the input area should now list the jfr tools.

For development snapshots, swap the alias:

```json
{
  "mcpServers": {
    "jafar": {
      "command": "jbang",
      "args": ["--fresh", "jfr-mcp-dev@btraceio", "--stdio"]
    }
  }
}
```

The `--fresh` flag tells JBang to bypass its cache and fetch the latest snapshot — useful when you're testing changes that were published minutes ago.

### Building from Source

If you prefer to skip JBang entirely:

```bash
cd jafar
./gradlew :jfr-mcp:shadowJar
java -jar jfr-mcp/build/libs/jfr-mcp-*-all.jar --stdio
```

Requires Java 25+. The Claude Desktop config for a manual JAR looks like:

```json
{
  "mcpServers": {
    "jafar": {
      "command": "java",
      "args": ["-jar", "/path/to/jfr-mcp-0.15.0-all.jar", "--stdio"]
    }
  }
}
```

## Why JFR Deserves Better Tooling

Most people who touch JFR recordings end up tunneling in on one event type. Open the file, look at CPU samples, maybe check GC pauses, close the file. That's like reading the first chapter of a detective novel and declaring the butler did it.

A JFR recording is a correlated, timestamped snapshot of *everything the JVM was doing*. CPU and native method samples. GC phases and heap summaries. Thread lifecycle events. Monitor contention. File and socket I/O. Object allocations. Class loading. JIT compilation phases. Exception throws. And the power of JFR is that all of these events share a timeline — you can correlate a GC pause with the thread states that were blocked during it, or trace an allocation spike to the exact call path that triggered it.

The MCP server is designed to treat the recording holistically. It doesn't just expose "query this event type." It exposes systematic analysis frameworks that cross-cut multiple event types and produce a coherent picture.

## The Toolbox

The server exposes 13 tools organized in two layers.

### Core Tools: Opening, Querying, Learning

| Tool | What it does |
|------|--------------|
| `jfr_open` | Opens a recording. Returns session ID, event count, timestamp range. |
| `jfr_close` | Closes a session (or all sessions). |
| `jfr_list_types` | Lists event types present in the recording. Optional scan for actual counts. |
| `jfr_query` | Runs a JfrPath query against any event type. The surgical instrument. |
| `jfr_help` | Returns JfrPath documentation so the AI can learn the query language on the fly. |

`jfr_query` accepts the same JfrPath expressions that [jfr-shell](https://github.com/jbachorik/jafar) uses:

```
events/jdk.ExecutionSample | groupBy(eventThread.javaName) | top(10)
events/jdk.GCPhasePause | count()
events/jdk.FileRead[bytes>1000] | groupBy(path)
```

The AI doesn't need to know JfrPath in advance. It calls `jfr_help` first, reads the syntax reference, and constructs queries from there. This is one of those details that sounds minor but matters a lot in practice — the agent is self-teaching, not hardcoded.

### Analysis Tools: Methodologies, Not Just Queries

This is where the holistic approach lives. These tools don't just run a single query — they correlate across event types and apply structured analysis frameworks.

| Tool | What it does |
|------|--------------|
| `jfr_summary` | Recording overview: duration, event counts, GC stats, CPU sampling density, exception patterns. The table of contents. |
| `jfr_use` | [USE Method](https://www.brendangregg.com/usemethod.html) analysis: Utilization, Saturation, Errors across CPU, memory, threads, and I/O. |
| `jfr_tsa` | [Thread State Analysis](https://www.brendangregg.com/tsamethod.html): how threads distribute across RUNNABLE, WAITING, BLOCKED, with lock correlation. |
| `jfr_diagnose` | Automated triage. Inspects the recording and triggers the right analyses based on what it finds. |
| `jfr_hotmethods` | CPU-intensive method ranking from leaf frames. |
| `jfr_flamegraph` | Aggregated stack traces in folded or tree format. Top-down or bottom-up. |
| `jfr_callgraph` | Caller-callee relationship graph. DOT or JSON. |
| `jfr_exceptions` | Exception grouping by type, throw site, and frequency. |

`jfr_diagnose` is the entry point for the "I don't know what's wrong" case. The server inspects the recording — is there high exception volume? GC pressure? CPU saturation? Thread contention? — and runs the relevant analyses automatically, composing `jfr_use`, `jfr_tsa`, and other tools as needed. It's the tool an AI reaches for first when you say "something is slow and I don't know why."

## What a Conversation Looks Like

Here's a realistic exchange. You drop a JFR file in front of Claude and ask what's wrong:

> **You:** Open `/tmp/checkout-service.jfr` and tell me why latency spiked.

Behind the scenes, Claude works through the recording methodically:

1. **`jfr_open`** — opens the recording, gets session ID and timestamp range
2. **`jfr_use`** — runs USE Method analysis. CPU utilization is 92%, thread saturation is high (long monitor waits), memory shows frequent GC pauses. Now it knows *which resources* are stressed.
3. **`jfr_tsa`** — runs Thread State Analysis. 40% of checkout-handler threads are BLOCKED, not RUNNABLE. The bottleneck isn't just CPU — threads are waiting on something. TSA correlates this with `jdk.JavaMonitorEnter` events and identifies `InventoryCache.lock` as the contention point.
4. **`jfr_hotmethods`** — of the threads that *are* running, `com.example.cart.PriceCalculator.recomputeTotals` dominates CPU samples
5. **`jfr_query`** — targeted follow-up: `events/jdk.JavaMonitorEnter[monitorClass='InventoryCache'] | top(5, by=duration)` confirms the lock is held for 200ms+ during cache refresh

Claude then synthesizes: USE analysis shows CPU saturation and thread contention. TSA identifies that checkout threads spend 40% of their time BLOCKED on `InventoryCache.lock`. The lock holder is `PriceCalculator.recomputeTotals`, which is also the CPU hotspot. The latency spike correlates with cache refresh intervals. Recommendation: decouple the cache refresh from the read path.

Notice the structure: USE first to identify *which* resources are stressed, TSA to understand *how* threads are affected, then targeted queries to confirm the specific mechanism. Each finding cross-references a different slice of the same recording. That's the holistic approach — not "look at CPU samples" but "systematically check every resource class, then drill into the ones that hurt."

## The Self-Teaching Trick

When you first connect the MCP server, the AI has never seen JfrPath. It doesn't need to have. The conversation typically goes:

1. AI calls `jfr_help` with topic `overview`
2. Reads the syntax reference (event paths, filters, pipeline operators, aggregation functions)
3. Starts constructing queries

This is possible because `jfr_help` returns comprehensive documentation broken into topics: `overview`, `filters`, `pipeline`, `functions`, `examples`, `event_types`. The AI reads what it needs, when it needs it.

It won't always write a perfect query on the first try. But the error messages from `jfr_query` are informative enough that the AI can self-correct — adjust a field name, fix a filter syntax, try a different aggregation. It's the same loop a human goes through, just compressed into a few hundred milliseconds of API calls.

## USE and TSA: Standing on the Shoulders of Giants

The most important design decision in jfr-mcp is not which JFR events it can parse. It's that the analysis tools implement *real performance engineering methodologies* rather than ad-hoc "let me grep for big numbers."

### The USE Method

Brendan Gregg's [USE Method](https://www.brendangregg.com/usemethod.html) is a systematic approach to resource analysis: for every resource in the system, check three things — **U**tilization, **S**aturation, and **E**rrors. It's simple, exhaustive, and it prevents the common trap of fixating on one metric while ignoring the actual bottleneck.

`jfr_use` applies this to four resource classes extracted from the JFR recording:

| Resource | Utilization | Saturation | Errors |
|----------|-------------|------------|--------|
| **CPU** | `jdk.CPULoad`, `jdk.ThreadCPULoad` | Run queue depth | — |
| **Memory** | Heap usage from `jdk.GCHeapSummary` | GC pause frequency and duration | Allocation failures |
| **Threads/Locks** | Active thread count | `jdk.JavaMonitorEnter` wait times, `jdk.ThreadPark` durations | — |
| **I/O** | `jdk.FileRead`, `jdk.FileWrite`, `jdk.SocketRead`, `jdk.SocketWrite` throughput | I/O wait times | I/O errors |

A single `jfr_use` call cross-references all of these event types and returns a structured report with actionable insights. When an AI runs this as a first step, it immediately knows which resource classes are healthy and which need investigation — before looking at a single stack trace.

### Thread State Analysis

Gregg's [TSA Method](https://www.brendangregg.com/tsamethod.html) asks a different question: instead of "which resource is stressed," it asks "what are threads actually *doing* with their time?" A thread is either running on-CPU, waiting for I/O, waiting for a lock, sleeping, or parked. The distribution across these states tells you whether your application is CPU-bound, I/O-bound, lock-bound, or just idle.

`jfr_tsa` groups execution samples by thread state — RUNNABLE, WAITING, TIMED_WAITING, BLOCKED — and breaks down the distribution globally and per-thread. When it finds threads spending significant time in BLOCKED state, it correlates with `jdk.JavaMonitorEnter` and `jdk.JavaMonitorWait` events to identify *which locks* are responsible and *who is holding them*.

This is the kind of analysis that, done manually, requires querying three or four different event types and cross-referencing timestamps. The MCP server does it in one call and returns structured results the AI can reason about.

### Why This Matters for AI-Driven Analysis

An AI without methodology is a pattern matcher looking for big numbers. An AI *with* methodology has a playbook: run USE to classify resource bottlenecks, run TSA to understand thread behavior, then drill into the specific event types that USE and TSA flagged. `jfr_diagnose` orchestrates exactly this — it inspects the recording characteristics, runs the appropriate methodologies, and chains into targeted follow-ups based on what it finds.

The AI doesn't need to be an expert in JFR event types. It needs to follow a framework that an expert designed. The frameworks are baked into the tools.

## What It Doesn't Do

A few things to be upfront about:

- **No live attach.** The MCP server works with recorded `.jfr` files, not live JVMs. You still need to capture the recording first (`jcmd`, JDK Mission Control, or continuous profiling infrastructure).
- **No visualization.** Results come back as structured data and text, not rendered charts. The AI can read and reason about them; if you want a flamegraph SVG, pipe the folded output from `jfr_flamegraph` into your favorite renderer.
- **The AI can be wrong.** It's working with real data and real methodologies, but its interpretations are still AI-generated. Treat them as a knowledgeable colleague's first take, not as gospel. The difference is that every claim is backed by a query you can re-run yourself.

## Why Not Just Use jfr-shell Directly?

Good question. If you know JfrPath well and have a hypothesis to test, jfr-shell (especially with the [TUI]({% post_url 2026-02-23-jfr-shell-tui %})) is probably faster. You know what you're looking for, you write the query, you get the answer.

The MCP server shines when:

- **You don't know where to start.** The AI can run `jfr_diagnose`, form hypotheses, and chase leads without you directing every step.
- **You want a summary for someone else.** Ask the AI to write up findings. It'll produce a narrative backed by USE/TSA data, which is more useful in a Slack thread than a raw query dump.
- **You're reviewing multiple recordings.** Open three recordings from different services, ask the AI to compare thread behavior across them. It'll manage the sessions and cross-reference results.
- **You're tired.** It's 3am, the pager went off, and you need to triage fast. Let the AI do the first pass while you get coffee.

jfr-shell gives you precision. The MCP server gives you leverage. They're not competing — jfr-mcp uses the same parser, the same query engine, and the same analysis primitives under the hood.

## Try It

```bash
# One-liner: installs JBang (if needed) + jfr-mcp
curl -Ls https://raw.githubusercontent.com/btraceio/jafar/main/jfr-mcp/install.sh | bash

# Or manually via JBang
jbang app install jfr-mcp@btraceio

# Wire it to Claude Code
claude mcp add jafar -- jbang jfr-mcp@btraceio --stdio

# Then in any conversation:
# "Open /path/to/recording.jfr and diagnose the performance issue"
```

---

*jfr-mcp is part of [JAFAR](https://github.com/jbachorik/jafar) and available via [JBang](https://jbang.dev) and [Maven Central](https://central.sonatype.com/). Source on [GitHub](https://github.com/jbachorik/jafar/tree/master/jfr-mcp).*
