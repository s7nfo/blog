---
layout: post
title:  "Syscall Showdown: Python vs. Ruby"
---

We've released a new version of [Cirron](https://github.com/s7nfo/Cirron) that can now trace syscalls and record performance counters for individual lines of Ruby code, just like it already could do for Python (more [here](http://blog.mattstuchlik.com/2024/02/08/counting-cpu-instructions-in-python.html) and [here](http://blog.mattstuchlik.com/2024/02/16/counting-syscalls-in-python.html)). It makes it very easy to quickly inspect what's happening in any section of your code and even assert what should be happening in tests, for example.

To put it through its paces I've compared what syscalls each language uses for several common patterns: File IO, generating random numbers, telling time and even just printing a string.

### File IO

Let's start with something a little surprising right away. Here are the snippets under investigation, simply writing a string to a file (I'll be omitting the Cirron setup from the snippets later on):

```python
# python
with Cirron.Tracer() as t:
    with open("test.txt", "w") as f:
        f.write("Hello, File I/O!")
```

```ruby
# ruby
t = Cirron::tracer do
  File.open("test.txt", "w") do |f|
    f.write("Hello, File I/O!")
  end
end
```

And here then, are the system calls each language makes. I've highlighted calls that aren't made by the other language and added approximate time spent in each call on my system:

<style>
    .highlight { background-color: #ffffcc; }
    .duration {
        align-self: flex-end !important;
        font-size: 10px !important;
        color: #666 !important;
        padding: 1px 3px !important;
        border-radius: 2px !important;
        margin-top: 5px !important;
    }
</style>

<table>
    <tr>
        <th>Python</th>
        <th>Ruby</th>
    </tr>
    <tr>
        <td>
            <a href="https://linux.die.net/man/2/openat">openat</a>(AT_FDCWD, "test.txt", O_WRONLY|O_CREAT|O_TRUNC|O_CLOEXEC, 0666) = 3
            <span class="duration">~80µs</span>
        </td>
        <td>
            <a href="https://linux.die.net/man/2/openat">openat</a>(AT_FDCWD, "test.txt", O_WRONLY|O_CREAT|O_TRUNC|O_CLOEXEC, 0666) = 5
            <span class="duration">~80µs</span>
        </td>
    </tr>
    <tr>
        <td class="highlight">
            <a href="http://man.he.net/man2/newfstatat">newfstatat</a>(3, "", {st_mode=S_IFREG|0644, st_size=0, ...}, AT_EMPTY_PATH) = 0
            <span class="duration">~20µs</span>
        </td>
        <td></td>
    </tr>
    <tr>
        <td>
            <a href="https://linux.die.net/man/2/ioctl">ioctl</a>(3, TCGETS, 0x7ffe559ef8b0) = -1 ENOTTY
            <span class="duration">~20µs</span>
        </td>
        <td>
            <a href="https://linux.die.net/man/2/ioctl">ioctl</a>(5, TCGETS, 0x7ffe2a8d9e90) = -1 ENOTTY
            <span class="duration">~20µs</span>
        </td>
    </tr>
    <tr>
        <td class="highlight">
            <a href="https://linux.die.net/man/2/lseek">lseek</a>(3, 0, SEEK_CUR) = 0
            <span class="duration">~20µs</span>
        </td>
        <td></td>
    </tr>
    <tr>
        <td class="highlight">
            <a href="https://linux.die.net/man/2/lseek">lseek</a>(3, 0, SEEK_CUR) = 0
            <span class="duration">~20µs</span>
        </td>
        <td></td>
    </tr>
    <tr>
        <td>
            <a href="https://linux.die.net/man/2/write">write</a>(3, "Hello, File I/O!", 16) = 16
            <span class="duration">~30µs</span>
        </td>
        <td>
            <a href="https://linux.die.net/man/2/write">write</a>(5, "Hello, File I/O!", 16) = 16
            <span class="duration">~30µs</span>
        </td>
    </tr>
    <tr>
        <td>
            <a href="https://linux.die.net/man/2/close">close</a>(3) = 0
            <span class="duration">~80µs</span>
        </td>
        <td>
            <a href="https://linux.die.net/man/2/close">close</a>(5) = 0
            <span class="duration">~80µs</span>
        </td>
    </tr>
</table>

Both start off with <a href="https://linux.die.net/man/2/openat">openat</a> to open `test.txt` in the current directory.
Python then, unlike Ruby, uses [newfstat](http://man.he.net/man2/newfstatat) to figure out what buffer size to use by looking at `st_blksize`[^0].
Both languages then use <a href="https://linux.die.net/man/2/ioctl">ioctl</a> to figure out whether the file descriptor is a TTY.

So far as expected, but then... Python decides to [lseek](https://linux.die.net/man/2/lseek) to the start of the file twice. This doesn't seem strictly necessary and a quick look through the cPython codebase didn't reveal why this is happening. If you know, let me know!

Anyway: finally both languages use <a href="https://linux.die.net/man/2/write">write</a> to write our string and <a href="https://linux.die.net/man/2/close">close</a> to close the file handle.



Let's look at reading a file now:

```python
# python
with open("test.txt", "r") as f:
    content = f.read()
```

```ruby
# ruby
content = File.read("test.txt")
```

<table>
    <tr>
        <th>Python</th>
        <th>Ruby</th>
    </tr>
    <tr>
        <td>
            <a href="https://linux.die.net/man/2/openat">openat</a>(AT_FDCWD, "test.txt", O_RDONLY|O_CLOEXEC) = 3
            <span class="duration">~30µs</span>
        </td>
        <td>
            <a href="https://linux.die.net/man/2/openat">openat</a>(AT_FDCWD, "test.txt", O_RDONLY|O_CLOEXEC) = 5
            <span class="duration">~30µs</span>
        </td>
    </tr>
    <tr>
        <td class="highlight">
            <a href="http://man.he.net/man2/newfstatat">newfstatat</a>(3, "", {st_mode=S_IFREG|0644, st_size=16, ...}, AT_EMPTY_PATH) = 0
            <span class="duration">~20µs</span>
        </td>
        <td></td>
    </tr>
    <tr>
        <td>
            <a href="https://linux.die.net/man/2/ioctl">ioctl</a>(3, TCGETS, 0x7ffe559ef8b0) = -1 ENOTTY
            <span class="duration">~20µs</span>
        </td>
        <td>
            <a href="https://linux.die.net/man/2/ioctl">ioctl</a>(5, TCGETS, 0x7ffe2a8d9fd0) = -1 ENOTTY
            <span class="duration">~20µs</span>
        </td>
    </tr>
    <tr>
        <td class="highlight">
            <a href="https://linux.die.net/man/2/lseek">lseek</a>(3, 0, SEEK_CUR) = 0
            <span class="duration">~20µs</span>
        </td>
        <td></td>
    </tr>
    <tr>
        <td class="highlight">
            <a href="https://linux.die.net/man/2/lseek">lseek</a>(3, 0, SEEK_CUR) = 0
            <span class="duration">~20µs</span>
        </td>
        <td></td>
    </tr>
    <tr>
        <td>
            <a href="http://man.he.net/man2/newfstatat">newfstatat</a>(3, "", {st_mode=S_IFREG|0644, st_size=16, ...}, AT_EMPTY_PATH) = 0
            <span class="duration">~20µs</span>
        </td>
        <td>
            <a href="http://man.he.net/man2/newfstatat">newfstatat</a>(5, "", {st_mode=S_IFREG|0644, st_size=16, ...}, AT_EMPTY_PATH) = 0
            <span class="duration">~20µs</span>
        </td>
    </tr>
    <tr>
        <td></td>
        <td class="highlight">
            <a href="https://linux.die.net/man/2/lseek">lseek</a>(5, 0, SEEK_CUR) = 0
            <span class="duration">~20µs</span>
        </td>
    </tr>
    <tr>
        <td>
            <a href="https://linux.die.net/man/2/read">read</a>(3, "Hello, File I/O!", 17) = 16
            <span class="duration">~20µs</span>
        </td>
        <td>
            <a href="https://linux.die.net/man/2/read">read</a>(5, "Hello, File I/O!", 16) = 16
            <span class="duration">~20µs</span>
        </td>
    </tr>
    <tr>
        <td>
            <a href="https://linux.die.net/man/2/read">read</a>(3, "", 1) = 0
            <span class="duration">~20µs</span>
        </td>
        <td>
            <a href="https://linux.die.net/man/2/read">read</a>(5, "", 8192) = 0
            <span class="duration">~20µs</span>
        </td>
    </tr>
    <tr>
        <td>
            <a href="https://linux.die.net/man/2/close">close</a>(3) = 0
            <span class="duration">~20µs</span>
        </td>
        <td>
            <a href="https://linux.die.net/man/2/close">close</a>(5) = 0
            <span class="duration">~20µs</span>
        </td>
    </tr>
</table>

Nothing terribly surprising here. Python still doing its double <a href="https://linux.die.net/man/2/lseek">lseek</a> and Ruby, probably not wanting to be left behind, adds an <a href="https://linux.die.net/man/2/lseek">lseek</a> of its own (also not strictly necessary, as far as I can tell). Note how <a href="https://linux.die.net/man/2/openat">openat</a> here takes only ~30µs, but it was ~80µs when opening for with `O_WRONLY|O_CREAT|O_TRUNC`.

### Sneaky syscalls

Let's look at another interesting case—generating random numbers and telling time:

```python
# python
random_int = random.randint(1, 100)
```

```ruby
# ruby
random_int = rand(1..100)
```

```python
# python
current_time = time.time()
```

```ruby
# ruby
current_time = Time.now
```

You might reasonably assume you'd need a system call here too, but not so. Since these operations are frequently used and not privileged, they use [vDSO](https://man7.org/linux/man-pages/man7/vdso.7.html) and turn into normal function calls, avoiding paying the price of a syscall. This makes them invisible to `strace` but you'd see them pop up in `ltrace`.


### Printing

Here's one I thought would be obvious, but turned out a little surprising too:

```python
# python
print("Hello")
```

```ruby
# ruby
puts "Hello"
```

<table>
    <tr>
        <th>Python</th>
        <th>Ruby</th>
    </tr>
    <tr>
        <td>
            <a href="https://linux.die.net/man/2/write">write</a>(1, "Hello\n") = 6
            <span class="duration">~50-250µs</span>
        </td>
        <td>
            <a href="https://linux.die.net/man/2/writev">writev</a>(1, [{iov_base=\"Hello\", iov_len=5}, {iov_base=\"\\n\", iov_len=1}], 2) = 6
            <span class="duration">~50-250µs</span>
        </td>
    </tr>
</table>

I expected both languages to just use <a href="https://linux.die.net/man/2/write">write</a>, but apparently some 6 years ago Ruby switched to using <a href="https://linux.die.net/man/2/writev">writev</a> instead in [Feature #14042: IO#puts: use writev if available](https://bugs.ruby-lang.org/issues/14042). Before this, `puts` would use an extra <a href="https://linux.die.net/man/2/write">write</a> to output a newline and hence wasn't atomic.

### Fin
I hope you found this interesting! If you'd find it useful if Cirron supported another language too, let me know!

[^0]: [https://github.com/python/cpython/blob/main/Lib/_pyio.py#L247](https://github.com/python/cpython/blob/main/Lib/_pyio.py#L247)