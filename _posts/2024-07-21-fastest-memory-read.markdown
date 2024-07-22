---
layout: post
title:  "Counting Bytes Faster Than You'd Think Possible"
---

<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 
<blockquote class="twitter-tweet">
  <a href="https://twitter.com/s7nfo/status/1814337750109237399"></a> 
</blockquote>

[Summing ASCII Encoded Integers on Haswell at the Speed of memcpy](https://blog.mattstuchlik.com/2024/07/12/summing-integers-fast.html) turned out more popular than I expected, which inspired me to take on another challenge on HighLoad: [Counting uint8s](https://highload.fun/tasks/5). I'm currently only #13 on the leaderboard, ~7% behind #1, but already learned some interesting things. In this post I'll describe my complete solution including a very suprising memory read pattern that achieves up to ~30% higher transfer rates on fully memory bound workloads compared to naive sequential access (It's free real estate!!).

As before, the program is tuned for the HighLoad system: Intel Xeon E3-1271 v3 @ 3.60GHz, 512MB RAM, Ubuntu 20.04. It only uses AVX2, no AVX512. As presented, it produces correct result with probability < 1, though very close to 1 and can be tuned to get to 1 in exchange for a very minor performance hit.

## The Challenge

"Print the number of bytes whose value equals 127 in a 250MB stream sent to standard input."

Nothing to it! The presented solution is 550x+ faster than the following naive program.

```cpp
uint64_t count = 0;
while (std::cin) {
    uint8_t v = 0;
    std::cin >> v;
    if (v == 127)
        ++count;
}

std::cout << count << std::endl;
return 0;
```

## The Kernel

You'll find the full source code of the solution at the end of the post. But first I'll build up to how it works.

The kernel of the solution is fairly simple, so I went straight to an `__asm__` block (sorry!). The basic building block is just a couple instructions long:

```nasm
; rax is the base of the input
; rsi is an  offset to current chunk
vmovntdqa    (%rax, %rsi, 1), %ymm4
; ymm2 is a vector full of 127
vpcmpeqb     %ymm4, %ymm2, %ymm4
; ymm6 is an accumulator, whose each
; byte represents a running count
; of 127s at that position in the
; input chunk
vpsubb       %ymm4, %ymm6, %ymm6
```

With this, we iterate over 32-byte chunks of the input and:
* Load the chunk with `vmovntdqa` (it's a non-temporal move just for style points, it doesn't make a difference to runtime).
* Compare each byte in the chunk with `127` using `vpcmpeqb`, giving us back `0xFF` (aka `-1`) where the byte is equal to `127` and `0x00` elsewhere. For example `[125, 126, 127, 128]` becomes `[0, 0, -1, 0]`.
* Substract the result of the comparison from an accumulator. Continuing the example above and assuming a zeroed accumulator, we'd get `[0, 0, 1, 0]`.

Then, every once in a while, to avoid this narrow accumulator overflowing, we dump it into a wider one with the following:

```nasm
; ymm1 is a zero vector
; ymm6 is the narrow accumulator
vpsadbw      %ymm1,%ymm6,%ymm6
; ymm3 is a wide accumulator
vpaddq       %ymm3,%ymm6,%ymm3
```

`vpsadbw` sums each eight bytes in the accumulator together into four 64-bit numbers, then `vpadddq` sums it with a wider accumulator, that we now won't overflow and that we extract at the end to arrive at the final count.

So far nothing revolutionary. In fact you can find this kind of approach on StackOverflow: [How to count character occurrences using SIMD](https://stackoverflow.com/questions/54541129/how-to-count-character-occurrences-using-simd).

## The Magic Sauce

The thing with this challenge is, we do so little computation it's almost entirely memory bound. Further, we're really bound by the number of load slots we can fill.

A first came up on the following idea when reading through the typo-ridden Intel Optimization Manual. On page 788 it describes the 4 hardware prefetches, which mostly seem to help with sequential access pattern, except for the "Streamer" prefetcher, which:

"Detects and maintains up to 32 streams of data accesses. For each 4K byte page, you can maintain one forward and one backward stream can be maintained."

"For each 4K byte page". Can you see where this is going? Instead of processing the whole input sequentially, we'll interleave processing 4K pages. We also unroll a bit and process a whole cache line in each block.

```cpp
#define BLOCK(offset) \
    "vmovntdqa    " #offset " * 4096 (%6, %2, 1), %4\n\t" \
    "vpcmpeqb     %4, %7, %4\n\t" \
    "vmovntdqa    " #offset " * 4096 + 0x20 (%6, %2, 1), %3\n\t" \
    "vpcmpeqb     %3, %7, %3\n\t" \
    "vpsubb       %4, %0, %0\n\t" \
    "vpsubb       %3, %1, %1\n\t" \
```

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 400 200">
  <style>
    @keyframes sequential {
      0% { transform: translateX(0); }
      100% { transform: translateX(310px); }
    }
    @keyframes interleaved {
      0% { transform: translateX(0); }
      100% { transform: translateX(75px); }
    }
    .sequential { animation: sequential 4s linear infinite; }
    .interleaved { animation: interleaved 4s linear infinite; }
  </style>
  
  <!-- Sequential access -->
  <rect x="40" y="20" width="320" height="40" fill="#e0e0e0" stroke="#000" />
  <rect class="sequential" x="40" y="20" width="10" height="40" fill="#ff6b6b" />
  <text x="200" y="75" text-anchor="middle">Sequential Access</text>
  
  <!-- Interleaved access -->
  <g transform="translate(40, 100)">
    <rect x="0" y="0" width="80" height="40" fill="#e0e0e0" stroke="#000" />
    <rect x="80" y="0" width="80" height="40" fill="#e0e0e0" stroke="#000" />
    <rect x="160" y="0" width="80" height="40" fill="#e0e0e0" stroke="#000" />
    <rect x="240" y="0" width="80" height="40" fill="#e0e0e0" stroke="#000" />
    <rect class="interleaved" x="0" y="0" width="5" height="40" fill="#4ecdc4" />
    <rect class="interleaved" x="80" y="0" width="5" height="40" fill="#4ecdc4" />
    <rect class="interleaved" x="160" y="0" width="5" height="40" fill="#4ecdc4" />
    <rect class="interleaved" x="240" y="0" width="5" height="40" fill="#4ecdc4" />
  </g>
  <text x="200" y="155" text-anchor="middle">Interleaved Access (4 pages)</text>
</svg>

One additional thing we'll add is a prefetch:

```cpp
#define BLOCK(offset) \
    "vmovntdqa    " #offset " * 4096 (%6, %2, 1), %4\n\t" \
    "vpcmpeqb     %4, %7, %4\n\t" \
    "vmovntdqa    " #offset " * 4096 + 0x20 (%6, %2, 1), %3\n\t" \
    "vpcmpeqb     %3, %7, %3\n\t" \
    "vpsubb       %4, %0, %0\n\t" \
    "vpsubb       %3, %1, %1\n\t" \
    "prefetcht0   " #offset " * 4096 + 4 * 64 (%6, %2, 1)\n\t"
```

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 400 240">
  <style>
    @keyframes interleaved {
      0% { transform: translateX(0); }
      100% { transform: translateX(75px); }
    }
    .interleaved { animation: interleaved 4s linear infinite; }
    .prefetch { animation: interleaved 4s linear infinite; }
    text { font-family: Arial, sans-serif; font-size: 14px; }
  </style>
  
  <!-- Interleaved access without prefetch -->
  <g transform="translate(40, 20)">
    <rect x="0" y="0" width="80" height="40" fill="#e0e0e0" stroke="#000" />
    <rect x="80" y="0" width="80" height="40" fill="#e0e0e0" stroke="#000" />
    <rect x="160" y="0" width="80" height="40" fill="#e0e0e0" stroke="#000" />
    <rect x="240" y="0" width="80" height="40" fill="#e0e0e0" stroke="#000" />
    <rect class="interleaved" x="0" y="0" width="5" height="40" fill="#4ecdc4" />
    <rect class="interleaved" x="80" y="0" width="5" height="40" fill="#4ecdc4" />
    <rect class="interleaved" x="160" y="0" width="5" height="40" fill="#4ecdc4" />
    <rect class="interleaved" x="240" y="0" width="5" height="40" fill="#4ecdc4" />
  </g>
  <text x="200" y="75" text-anchor="middle">Interleaved Access without Prefetch</text>
  
  <!-- Interleaved access with prefetch -->
  <g transform="translate(40, 120)">
    <rect x="0" y="0" width="80" height="40" fill="#e0e0e0" stroke="#000" />
    <rect x="80" y="0" width="80" height="40" fill="#e0e0e0" stroke="#000" />
    <rect x="160" y="0" width="80" height="40" fill="#e0e0e0" stroke="#000" />
    <rect x="240" y="0" width="80" height="40" fill="#e0e0e0" stroke="#000" />
    <rect class="prefetch" x="10" y="0" width="5" height="40" fill="#a9a9a9" opacity="0.5" />
    <rect class="prefetch" x="90" y="0" width="5" height="40" fill="#a9a9a9" opacity="0.5" />
    <rect class="prefetch" x="170" y="0" width="5" height="40" fill="#a9a9a9" opacity="0.5" />
    <svg x="240" y="0" width="80" height="40" overflow="hidden">
      <rect class="prefetch" x="10" y="0" width="5" height="40" fill="#a9a9a9" opacity="0.5" />
    </svg>
    <rect class="interleaved" x="0" y="0" width="5" height="40" fill="#4ecdc4" />
    <rect class="interleaved" x="80" y="0" width="5" height="40" fill="#4ecdc4" />
    <rect class="interleaved" x="160" y="0" width="5" height="40" fill="#4ecdc4" />
    <rect class="interleaved" x="240" y="0" width="5" height="40" fill="#4ecdc4" />
  </g>
  <text x="200" y="175" text-anchor="middle">Interleaved Access with Prefetch</text>
</svg>






Mention how much fater it is when fully memory bounds vs. how much in HighLoad.

Test interleaving 8 pages across rank.

Current: 15,748
Serial: 18,182

## The Source
```
#define BLOCK_COUNT 8

#define PAGE_SIZE 4096

#define BLOCKS_8 \
    BLOCK(0)  BLOCK(1)  BLOCK(2)  BLOCK(3) \
    BLOCK(4)  BLOCK(5)  BLOCK(6)  BLOCK(7)

#define BLOCK(offset) \
    "vmovntdqa    " #offset "*4096(%6,%2,1),%4\n\t" \
    "vpcmpeqb     %4,%7,%4\n\t" \
    "vmovntdqa    " #offset "*4096+0x20(%6,%2,1),%3\n\t" \
    "vpcmpeqb     %3,%7,%3\n\t" \
    "vpsubb       %4,%0,%0\n\t" \
    "vpsubb       %3,%1,%1\n\t" \
    "prefetcht0   " #offset "*4096+4*64(%6,%2,1)\n\t"

static inline
__m256i hsum_epu8_epu64(__m256i v) {
    return _mm256_sad_epu8(v, _mm256_setzero_si256());
}


int main() {
    struct stat sb;
    fstat(STDIN_FILENO, &sb);
    size_t length = sb.st_size;
    char* start = static_cast<char*>(mmap(nullptr, length, PROT_READ, MAP_PRIVATE | MAP_POPULATE, STDIN_FILENO, 0));

    if (start == MAP_FAILED) {
        return 1;
    }

    uint64_t count = 0;
    __m256i sum64 = _mm256_setzero_si256();
    size_t i = 0;

    __m256i compare_value = _mm256_set1_epi8(127);
    __m256i acc1 = _mm256_set1_epi8(0);
    __m256i acc2 = _mm256_set1_epi8(0);
    __m256i temp1 = _mm256_set1_epi8(0);
    __m256i temp2 = _mm256_set1_epi8(0);

    // Align to page boundary
    while (reinterpret_cast<uintptr_t>(start + i) % (1 * PAGE_SIZE) != 0) {
        if (start[i] == 127) ++count;
        ++i;
    }

    for (; i + BLOCK_COUNT*PAGE_SIZE <= length;) {
        int batch = PAGE_SIZE / 64;
        asm volatile(
            ".align 16\n\t"
            "0:\n\t"

            BLOCKS_8

            "add          $0x40, %2\n\t"
            "dec          %5\n\t"
            "jg           0b"
            : "+x" (acc1), "+x" (acc2), "+r" (i), "+x" (temp1), "+x" (temp2), "+r" (batch)
            : "r" (start), "x" (compare_value)
            : "cc", "memory"
        );

        i += (BLOCK_COUNT - 1)*PAGE_SIZE;

        sum64 = _mm256_add_epi64(sum64, hsum_epu8_epu64(acc1));
        sum64 = _mm256_add_epi64(sum64, hsum_epu8_epu64(acc2));

        acc1 = _mm256_set1_epi8(0);
        acc2 = _mm256_set1_epi8(0);
    }

    sum64 = _mm256_add_epi64(sum64, hsum_epu8_epu64(acc1));
    sum64 = _mm256_add_epi64(sum64, hsum_epu8_epu64(acc2));

    count += _mm256_extract_epi64(sum64, 0);
    count += _mm256_extract_epi64(sum64, 1);
    count += _mm256_extract_epi64(sum64, 2);
    count += _mm256_extract_epi64(sum64, 3);

    for (; i < length; ++i) {
        if (start[i] == 127) {
            ++count;
        }
    }

    std::cout << count << std::endl;
    return 0;
}
```