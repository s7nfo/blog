---
layout: post
title:  "Injecting syscall faults in Python and Ruby"
---

Since syscalls are near the very bottom of any software stack, their misbehavior can be particularly hard to test for. Stuff like running out of disk space, network connections timing out or bumping into system limits all ultimately manifest as a syscall failing somewhere. If you want your code to be resilient to these kinds of failures, it sure would be nice if you could simulate these situations easily.

Now, you might already know `strace` lets you [trace system calls](https://blog.mattstuchlik.com/2024/02/16/counting-syscalls-in-python.html), but did you know it can also change their behavior? You can modify their input and output, inject errors and add time delays (though be aware of the [limitations](#limitations)).

To demonstrate how it could be useful, I've added the ability to easily use this functionality from Python and Ruby to [Cirron](https://github.com/s7nfo/Cirron) (my grab-bag of a project). Now you can now do things like:

Test how code handles insufficient space.

```python
from cirron import Injector

injector = Injector()
# Make the "openat" syscall return the ENOSPC error.
injector.inject("openat", "error", "ENOSPC")

# All "openat" calls will return ENOSPC within this context.
with injector:
    # Fails with "No space left on device".
    f = open("test.txt", "w")

# From here on "openat" behaves normally again.
```

Inject occasional errors and delays to network operations.

```python
(...)
# Make every other "connect" syscall return the ETIMEDOUT error.
# when="2+2" means to perform the injection for the second syscall
# invocation and then again every two invocations.
injector.inject("connect", "error", "ETIMEDOUT", when="2+2")

# Also add 1s of latency to "send".
injector.inject("send", "delay_exit", "1s")

with injector:
    (...)
```

Simulate signals being sent before particular syscalls.
```python
(...)
# Simulate user pressing Ctrl+C before the first "read".
injector.inject("read", "signal", "SIGINT", "when=1") 

with injector:
    (...)
```

And more! In addition to the `error`, `delay_exit` and `signal` actions demonstrated above, it also supports `retval` for changing a return value without making it an error, `delay_enter` for delaying entry into a syscall rather than an exit and `poke_enter` and `poke_exit` for modifying the process memory on syscall entry or exit. See the ["Tampering" section of strace's man page](https://man7.org/linux/man-pages/man1/strace.1.html) for details on all these, including a description of the format of the `when` argument.

If you want to try this but are unsure what syscalls your code uses you can get a list of them easily with Cirron too:

```python
from cirron import Tracer

# Tracer records all syscalls made within the context.
with Tracer() as t:
    print("Hello!")

print(t)
# [Syscall(name='write', args='1, "Hello!\\n", 7', retval='7', duration='0.000197', timestamp='1725900869.238673', pid='438862')]
```

<br>

# How?

Cirron implements the `Injector` in the simplest way possible: it executes `strace` with the appropriate inject options and points it to the current process. After leaving the injection context the strace process is killed. If you were to make heavy use of this, it's probably worth implementing the functionality directly, rather than executing strace every time (let me know if you'd find it useful if Cirron did this more efficiently).

So how does `strace` do this? [Ptrace](https://en.wikipedia.org/wiki/Ptrace)! It [attaches](https://github.com/strace/strace/blob/0f9f46096fa8da84e2e6a6646cd1e326bf7e83c7/src/strace.c#L1388) to (or [seizes](https://github.com/strace/strace/blob/0f9f46096fa8da84e2e6a6646cd1e326bf7e83c7/src/strace.c#L569); dramatic!) a process with `ptrace(PTRACE_ATTACH, ...)`. This causes the traced process to stop on (among other things) entry and exit from syscalls. Strace can then inspect and modify the program before letting it continue.

To inject a [delay](https://github.com/strace/strace/blob/0f9f46096fa8da84e2e6a6646cd1e326bf7e83c7/src/delay.c), as with `delay_enter` or `delay_exit`, strace simply waits before continuing the process.

To modify the syscall's inputs or output it uses either `PTRACE_POKEDATA` (or [process_vm_writev](https://linux.die.net/man/2/process_vm_writev)) to mess with the traced process's memory (`poke_enter`, `poke_exit`) or `PTRACE_POKEUSER` to modify the USER area, containing the process's registers state, which lets you, for example, change the return value (`error`, `retval`).

# Limitations

There's the obvious performance impact, particularly if simply using `strace` instead of using `ptrace` directly.

Also consider that making a syscall fail this way does not remove the side effects it might have: making a "write" call return an error will still (possibly) perform the "write", it will just appear to have failed to the application. Along the same lines the delay injections delay before syscall entry or after exit, which may have very different impact compared to delaying something in the middle of the call.