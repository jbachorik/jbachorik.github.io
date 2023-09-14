---
layout: post
title:  "BTrace Templated Variables"
date:   2020-12-02 12:00:00 +0200
categories: java btrace
---

1. TOC
{:toc}

# BTrace Templated Variables

Since version 1.3.11 it has been possible to use template variables in the probe declarations.
This allows delayed configuration at deployment time and makes the probe definition much more flexible.

A template variable has a value defined at deployment time and can be referenced using ant-like format - eg. `${refresh}`.
The variable value will be resolved as a plain string and will be the subject of any conversions necessary for the
corresponding place of use.

## Example
### Templated uptime probe
The following probe will run the uptime check each uptime_period_ms milliseconds.

{% highlight java %}
@BTrace
public class Uptime {
    @OnTimer(from  = "${uptime_period_ms}")
    public static void f() {
        println("uptime: " + Sys.VM.vmUptime());
    }
}
{% endhighlight %}

The uptime check period value must be provided to BTrace agent - either as a javaagent argument or an argument to 
btrace dynamic attach launcher.

### Attach on launch
{% highlight bash %}
java -jar application.jar -javaagent:btrace-agent.jar=script=Uptime.class,uptime_period_ms=3000
{% endhighlight %}

### Dynamic attach
{% highlight bash %}
btrace <pid> Uptime.java uptime_period_ms=3000
{% endhighlight %}

## Elements supporting templated variables
#### @OnMethod
Templating similar to what was shown in the previous example is available for all @OnMethod attributes - clazz, method, type and location whereas again the templated variables can be used in its clazz, method and type attributes in turn.

#### @OnTimer
The timer period can be defined via templated variable and is to be provided in the annotation's from attribute. The usage is shown in the previous example.
