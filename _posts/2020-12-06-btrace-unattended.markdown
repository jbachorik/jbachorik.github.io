---
layout: post
title:  "BTrace Unattended Execution"
date:   2020-12-06 12:00:00 +0200
categories: java btrace
---

1. TOC
{:toc}

# BTrace Unattended Execution
## A bit of history
BTrace origins go back more than 10 years and it shows that the modus operandi at that time was to have one JVM at 
a time and do all the experiments and debugging on that single JVM.
The standard workflow for BTrace dynamic attach was decided to be:

1. Identify the JVM to attach to
2. Attach to that JVM and deploy the probe(s)
3. Stay connected until you are satisfied with the results

While probably quite ok for the one JVM situation it becomes quite a hindrance when trying to operate in ad-hoc mode for 
a bunch of JVMs running on several hosts at once (yes, k8s, talking about you). The lack of unattended execution basically 
makes BTrace unusable in modern environments - if dynamic attach is required.

## Time to fix
So, after a period of procrastination I finally decided to add unattended mode of execution to BTrace. It was both easy and 
hard task at the same time - the binary I/O protocol BTrace is using is very easy to extend but the underlying management of 
client 'sessions' had to be refactored slightly to allow disconnecting and reconnecting to a session without killing it. 
But nothing that could not have been done during a particularly grey and rainy COVID lockdown weekend.

Here we go with a few improvements to the BTrace client which should make it much easier to use BTrace in fire&forget mode - 
which, I think, will become more and more popular with the JFR support when one can easily define a dynamic JFR event type and 
deploy a probe to generate that event and leave the probe running, turning on and of the JFR recording as necessary.

### 'Detach client' client command
In addition to 'Exit' option in the BTrace CLI it is now possible to detach from the running session.
Upon detaching a unique probe ID is displayed which can be used to later reconnect to that probe.
!['Detach' Command]({{ site.baseurl }}/assets/images/2020-12-06-btrace-unattended/detach_command.png)

### 'List probes' client command
This command will list any probes which were deployed and the clients left them detached.
!['List Probes' Command]({{ site.baseurl }}/assets/images/2020-12-06-btrace-unattended/list_probes.png)

### List probes from command line
Use `btrace -lp <pid>` to list the probes in detached mode in a particular JVM
!['List Probes' Result]({{ site.baseurl }}/assets/images/2020-12-06-btrace-unattended/list_results.png)

### Reconnect to a detached probe
Use `btrace -r <probe id> <pid>` to reconnect to a detached probe and start receiving probe data.

The detached probes are maintaining a circular buffer for the latest data so you can get a bit of history after reconnecting as well.

### Attach a probe and disconnect immediately
Useful shortcut for scripting BTrace deployments when the probe is deployed and the client disconnects immediately.
Use `btrace <btrace options> -x <pid> <probe file>` to run in this mode. 
Upon disconnecting the probe ID is printed out, so it can be e.g. processed and stored.

## New possibilities
Having implemented the support for listing the detached probes and reconnecting to them as a form of command line switches 
opened doors to easy scripting when one can write a quick one-liner to attach to a named probe:
`./bin/btrace -r $(./bin/btrace -lp anagrams.jar | fgrep AllMethods1 | cut -f2 -d' ') anagrams.jar`


The unattended execution support was checked in and a development build binaries are [available](https://github.com/btraceio/btrace/actions/runs/394037357)