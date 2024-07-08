---
layout: post
title:  "Syscall Showdown: Python vs. Ruby"
---

[Cirron](https://github.com/s7nfo/Cirron) recently gained the ability to trace syscalls and record performance counters for Ruby code, like it already could do for Python (more [here](http://blog.mattstuchlik.com/2024/02/08/counting-cpu-instructions-in-python.html) and [here](http://blog.mattstuchlik.com/2024/02/16/counting-syscalls-in-python.html)). To put it through its paces I've compared what syscalls each language uses for several common patterns.

### File IO

Let's start with something a little surprising right away. Here are the snippets under investigation, I'll be omitting the Cirron setup from the snippets later on.

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

And here then, are the system calls each language makes. I've highlighted the calls that aren't made by the other language and added approximate time spent in each on my system:

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

Both our competitors use <a href="https://linux.die.net/man/2/openat">openat</a> to open `test.txt` in current directory.
Python then, unlike Ruby, uses [newfstat](http://man.he.net/man2/newfstatat) to figure out what buffer size to use by looking at `st_blksize`[^0].
Both languages then use <a href="https://linux.die.net/man/2/ioctl">ioctl</a> to figure out whether the `fd` is a TTY.

So far as expected, but then... Python decides to [lseek](https://linux.die.net/man/2/lseek) to the start of the file twice. This doesn't seem strictly necessary, but a quick look through the cPython codebase didn't reveal why this is happening. If you know, let me know!

Anyway: finally both languages use <a href="https://linux.die.net/man/2/write">write</a> to write our string and <a href="https://linux.die.net/man/2/close">close</a> to close the file handle.



Now let's look at reading a file instead:

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

Nothing terribly surprising here. Python still doing its double lseek and Ruby, probably not wanting to be left behind, adds an lseek of it's own (also not strictly necessary, I think?). Do note how <a href="https://linux.die.net/man/2/openat">openat</a> here takes only ~30µs, but it was ~80µs when opening for writing with `O_WRONLY|O_CREAT|O_TRUNC`.

### Sneaky syscalls

Let's look at another somewhat interesting case -- generating random numbers and telling time:

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

You might reasonably assume you'd need a system call here too, but not so. Since these operations are frequently used and not privileged, they use [vDSO](https://man7.org/linux/man-pages/man7/vdso.7.html) and turn into a normal function calls, avoiding having to pay the price of a syscall. This makes them invisible to `strace` but you'd see them pop up in `ltrace`.


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

<a href="https://linux.die.net/man/2/write">write</a>

<a href="https://linux.die.net/man/2/writev">writev</a> is apparently atomic, so it should provide more reasonable output when multiple threads are writing stuff.