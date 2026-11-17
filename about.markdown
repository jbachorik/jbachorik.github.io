---
layout: page
title: About
permalink: /about/
---

Occasional rants, stories from the trenches and some whining on top.

Been in this world for too long to take things seriously.

I welcome your problems, with enthusiasm!

---

## News

### BTrace 3.0.0-RC1 is out: trace a live JVM in one line, no restart

*17 November 2026*

BTrace attaches to a running Java application, injects a small probe, and removes it when you leave. 3.0 is the biggest release since 2.2.6: 165 commits touching 1,869 files. That is why it starts as a release candidate. Whatever you try in RC1 is what 3.0.0 will be, unless you find something wrong.

**What is new**

- Oneliners, no script file needed:

  ```sh
  btrace -n 'OrderService::* @return if duration>200ms { print method, time }' <pid>
  ```

- One jar, now Apache 2.0, with the namespace moved to `io.btrace`
- Extensions that run isolated and need permissions granted by whoever operates the JVM, not by whoever deploys the probe
- Authenticated attach, a doctor command to check readiness, and probe budgets a client cannot raise
- Instrumentation for class files newer than ASM supports, through the JDK's ClassFile API
- An MCP server, so a coding agent can attach a narrow probe, read the numbers and remove it. The post walks through a recorded session where it finds two defects inside a library with no source.
- Probes compiled with 2.x run unchanged

**Try it.** A release candidate is not on SDKMAN or the jbang alias yet, so use the explicit coordinates:

```sh
jbang io.btrace:btrace:3.0.0-RC1 -n '<oneliner>' '~myapp'
```

**How this got built.** Most of this exists because of agentic development. It let me test crazy ideas quickly, and once an idea proved out, add far more tests and checks than I could ever have written by hand in evenings and weekends. The ideas and the mistakes are mine. The verification is what I could not have afforded before.

- [The write-up](/posts/btrace-3-0-0)
- [Project site](https://btrace.io/btrace/)
- [Found a problem? Open an issue](https://github.com/btraceio/btrace/issues)
