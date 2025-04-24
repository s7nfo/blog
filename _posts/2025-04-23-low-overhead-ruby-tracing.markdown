---
layout: post
title:  "[WIP] Low Overhead Ruby Tracing Profiler"
---

> Note: this is a work-in-progress post, RSS betrayed me and let it out into the world before its time :)

![Profiler Overview](/assets/ruby-profiler-overview.png)

I’ve built what I think is a pretty neat Ruby tracing profiler. It captures four event streams: Ruby function calls, system calls, thread-state changes, and garbage-collection activity while adding less than 100 ns[^0] of overhead per Ruby function call, low enough for continuous use in large-scale production systems.

[^0]: Across 50 paired runs of a Fibonacci(20) benchmark (~22 k function calls per run) on an isolated CPU core of a recent Intel machine running Linux, Ruby with tracing enabled was 2 ms slower on average—roughly 100 ns per function call. However, a paired t‑test (p ~ 0.31) shows that difference isn’t statistically significant, so the per‑call overhead is effectively invisible at the macro level.

As far as I know, no other Ruby tracer offers this mix of signals at this cost, but if you’re aware of one, let me know and I’ll add a note here! If you aren’t and would like to try this profiler yourself, reach out!

## Why Trace in Production
In a word: outliers. Sampling profilers show where a program spends its time *on average*. However, problems such as high tail latency are by definition not well represented in an average view. This is where a tracing profiler comes in handy.

Let me illustrate this with a sophisticated SaaS simulator that handles a bunch of API calls and records their duration:

```
durations = []

100_000.times do
  start_time = Time.now

  api_handler

  durations << Time.now - start_time
end
```

Let’s say `api_handler` is a black box and it has a P99.9 SLO of < 10ms. Executing the code, we find:

```
P50 latency:     1.1 ms
P99.9 latency: 101.3 ms
```

Our median (P50) looks great, but the P99.9 breaches the SLO.

If you care about latency, you're probably already running a sampling profiler, so let's see what that shows:

![Sampling Profiler Example](/assets/ruby-profiler-sampling.png)

We see that two methods account for most of the runtime: `api_handler->Object#a` and `api_handler->Oject#b`, `a` using the vast majority of the time slice. From this, you might conclude that you should focus on optimizing `a`. In this case, that would be a waste of time!

To show this, let's try inspecting a trace from one of the slow `api_handler` executions with our new profiler. First we instrument the code:

```diff
+ require 'tracing'

 durations = []
 SLA = 0.01

 100_000.times do
   start_time = Time.now
+   Trace.reset

   api_handler

   duration = Time.now - start_time
+  if duration > SLA
+     Trace.save("api_call_trace.json")
+   end

   durations << duration
end
```

Now whenever the duration exceeds our SLA, we save a trace. Let's look at one of them:

![Tracing Profiler Example](/assets/ruby-profiler-tracing.png)


And now we see the exact opposite! `b` using the vast majority of the time slice. So who's right?

The `api_handler` implementation was this:


```
# api_handler always executes a, which sleeps for 1ms
# 0.1% of the time it also executes b, which sleeps for 1s
def api_handler
  a
  if rand >= 0.999
    b
  end
end

def a
  sleep(0.001)
end

def b
  sleep(0.1)
end
```

The sampling profiler correctly showed that *over the whole runtime*, `a` uses more time, but when trying to improve tail latency that is not what we care about. During  The tracing profiler output is therefore much more informative.

