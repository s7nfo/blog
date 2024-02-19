---
layout: post
title:  "Tracing System Calls in Python"
---

Last time we [counted CPU
instructions](http://blog.mattstuchlik.com/2024/02/08/counting-cpu-instructions-in-python.html),
let's look at [syscalls](https://en.wikipedia.org/wiki/System_call) now!

I'll show you a little tiny tool I added to
[Cirron](https://github.com/s7nfo/Cirron) that lets you see exactly what
syscalls a piece of Python code is calling and how to analyze the trace more effectively.

Let's start with `print("Hello")` as before:
```python
from cirron import Tracer

with Tracer() as t
    print("Hello")

print(t.trace)
# write(1, "Hello\n", 6) = 6 <0.000150s>
```

You can see[^0] `print` uses only a single
[write](https://man7.org/linux/man-pages/man2/write.2.html) to write the string
`"Hello\n"` to stdout (that's what the `1` stands for) and asks it to write at
most `6` bytes. Write then returns `6`, meaning it managed to write all the
bytes we asked it to. You can also see it took 0.00015s or 150μs (that's just the
`write` call, not the whole `print` statement).

Pretty cool!

How does `Tracer` work? I initially wanted to use the
[ptrace](https://man7.org/linux/man-pages/man2/ptrace.2.html) syscall to
implement it, but that turned out to be a little more complicated that what I
wanted, so in the end I just used the `strace` tool, which also uses `ptrace`
but handles all the complexity. `Tracer` simply [starts tracing
itself](https://github.com/s7nfo/Cirron/blob/master/cirron/tracer.py#L138) with
it, redirecting output to a file, which is then parsed when it's asked to stop.

Let's trace `import seaborn` now:
```python
from cirron import Tracer

with Tracer() as t:
    import seaborn

print(len(t.trace))
# 20462
```

Turns out importing Seaborn takes ~20k syscalls! That's obviously too many to
just print out, so what's a better way to analyze what it's doing?

## Visualizing traces with Perfetto

[Perfetto Trace Viewer](https://ui.perfetto.dev) let's you visualize all kinds
of traces. It can't ingest `strace` output directly, but I've included a
function that converts Tracer output to [Trace Event
Format](https://docs.google.com/document/d/1CvAClvFfyA5R-PhYUmn5OOQtYMH4h6I0nSsKchNAySU/preview),
something Perfetto can load:

```python
from cirron import to_tef

(...)

open("/tmp/trace", "w").write(to_tef(t.trace))
```

<br>

This gets you a file you can open with Perfetto. I'm not going to describe all it can do; I uploaded a trace of `import seaborn` [here](https://gist.github.com/s7nfo/4cda90818a07d851fea79c8c17e8eab8), go [play with it](https://ui.perfetto.dev/)!

<br>

![Perfetto](/assets/perfetto.png)

<br>

I was surprised to find it uses 4 threads, which mostly spend time looking up files and reading them, but one of the threads seems to be very curious about your CPU details!

[^0]: If you try this yourself you'll see a couple more calls, but those are related to shutting down `strace` after we're done, not to the traced code itself.
