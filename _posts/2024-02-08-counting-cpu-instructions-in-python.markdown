---
layout: post
title:  "Counting CPU Instructions in Python"
---

Did you know it takes about 17,000 CPU instructions to `print("Hello")` in Python? And that it takes ~2 billion of them to import `seaborn`?

Today I was playing with [perf_event_open](https://man7.org/linux/man-pages/man2/perf_event_open.2.html), a linux syscall that lets you set up all kinds of performance monitoring. One way to interact with the system is through the `perf` CLI tool. The problem?

```
root@X:~# cat print.py
print("Hello")

root@X:~# perf stat -e instructions:u python3 ./print.py
Hello

 Performance counter stats for 'python3 ./print.py':

          45713378      instructions:u

       0.030247842 seconds time elapsed

       0.025787000 seconds user
       0.004282000 seconds sys

```

45,713,378 is how many instructions it takes to initialize Python, print Hello
and tear the whole thing down again. I want more precision!

Hence this little tool:

```python
from cirron import Collector

c = Collector()
c.start()

print("Hello")

print(c.end())
# Sample(instruction_count=17181, time_enabled_ns=92853)
```

Why would you want this? I was mostly just curious, but that's not to say
it can't be useful: let's say you really, really care about performance of your
`foo` and you want to have a test that fails if it gets slower than a threshold.
So you `time.time()` it, assert less than a threshold and... you realize your
CI box is running a ton of concurrent tests and the timing is therefore very
noisy.

Not (as much) instruction count!

![Cirron plot](/assets/cirron_plot.png)

This graph shows distribution of time measurements of the same piece of code
using `time.time()` and `time.perf_counter()` and the instruction count
measured with Cirron, scaled to fit with the time measurement[^0]. Note how
**impeccably tight** the instruction count distribution is (also note though
how it's *not* completely constant).

This is not to say instruction count is the be-all and end-all. Equal instruction
counts can have wildly different wall clock timings, etc. Still, it's a useful tool
if you're aware of its limitations.

Here's the [Github repo](https://github.com/s7nfo/Cirron), it's very janky and
only runs on Linux at the moment, thought I'll probably add Apple Arm support
soon. On the off-chance that there isn't a tool out there that does this way
better already and you'd find it useful if this was on PyPI or exposed more
perf events or worked with $LANGUAGE, etc., [let me
know](https://twitter.com/s7nfo).

[^0]: I'm sure comparing the two this way is completely incorrect, this is just me invoking the [Cunningham's Law](https://meta.wikimedia.org/wiki/Cunningham%27s_Law).
