---
layout: post
title: "TIL: Data Dependency and Performance in Assembly"
---

I was playing with assembly today, trying to understand performance impact of breaking data dependencies.

Optimizing compilers tend to rewrite the human-obvious `sum` implementation to use multiple registers to keep track of parts of the total sum, adding them up at the end. For example here's an implementation of `sum` in Rust 1.75 with `-C opt-level=3` and the associated disassembly (shortened to only keep the relevant bits):

```rust
pub fn sum(arr: &[i32]) -> i32 {
    let mut result = 0;

    for n in arr.iter() {
        result += n
    }

    result
}
```

```nasm
sum:
        ; Set xmm0 to 0.
        pxor    xmm0, xmm0
        xor     eax, eax
        ; Set xmm1 to 0.
        pxor    xmm1, xmm1 ; Set xmm1 to 0
.loop:
        ; Move 4 i32 values from our array into xmm2
        movdqu  xmm2, xmmword ptr [rdi + 4*rax]
        ; Add the 4 i32 values in xmm2 to xmm0.
        paddd   xmm0, xmm2
        ; Move the next 4 i32 values from our array into xmm2.
        movdqu  xmm2, xmmword ptr [rdi + 4*rax + 16]
        ; Add the 4 i32 values in xmm2 to xmm1, not xmm0 this time
        paddd   xmm1, xmm2

        ; Move index by 8 and if we're not done jump back to loop.
        add     rax, 8
        cmp     r8, rax
        jne     .loop

        ; If we are done, add up what's in xmm0 and xmm1 and return.
        paddd   xmm1, xmm0
        pshufd  xmm0, xmm1, 238
        paddd   xmm0, xmm1
        pshufd  xmm1, xmm0, 85
        paddd   xmm1, xmm0
        movd    eax, xmm1
        ret
```

<br>

As expected the compiler splits the sum between two registers (xmm0, xmm1). My hypothesis, therefore, was that the following hand-written assembly:

<br>

```nasm
sum:
        lea     rcx, [rdi + 4*rsi]
        xor     eax, eax    
.loop:
        ; Use EAX for both additions
        add     eax, dword [rdi]    
        add     eax, dword [rdi + 4]
        lea     rdi, [rdi + 8] 
        cmp     rdi, rcx      
        jb      .loop        
        ret                         
```

Would be slower than this multi-register one.

```nasm
sum:
        lea     rcx, [rdi + 4*rsi]
        xor     eax, eax
        xor     ebx, ebx
.loop:
        ; Split the addition between EAX and EBX
        add     eax, dword [rdi]
        add     ebx, dword [rdi + 4]
        lea     rdi, [rdi + 8]
        cmp     rdi, rcx
        jb      .loop
        add     eax, ebx
        ret
```

Except... It's not? If anything the first version is slightly faster.

Is my measuring methodology horrible (hyperfine on shared vCPU isn't great, but I would expect it to be merely noisy, not biased)? Is this [register renaming](https://en.wikipedia.org/wiki/Register_renaming) making things efficient behind the scenes?

Curious!
