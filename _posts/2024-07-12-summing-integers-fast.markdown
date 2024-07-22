---
layout: post
title:  "Summing ASCII Encoded Integers on Haswell at the Speed of memcpy"
---

<style>
  code {
    font-size: 13px;
    white-space: pre-wrap;
    word-wrap: break-word;
}
</style>

"Print the sum of 50 million ASCII-encoded integers uniformly sampled from [0, 2³¹−1], separated by a single new line and sent to standard input."

On the surface, a trivial problem. But what if you wanted to go as fast as possible?

I'm currently one of the top ranked competitors in [exactly that kind of challenge](https://highload.fun/tasks/1) and in this post I'll show you a sketch of my best performing solution. I'll leave out some of the µoptimizations and look-up table generation to keep this post short, easier to understand and to not completely obliterate the HighLoad leaderboard. Still, as far as I know nothing similar has been published yet, so I'm hoping you'll find it interesting.

On the target system, my program runs about 320x faster than the following naive C++ solution (and is about 1,000,000x more fragile):

```cpp
uint64_t sum = 0;
while (std::cin) {
    uint64_t v = 0;
    std::cin >> v;
    sum += v;
}

std::cout << sum << std::endl;
```

I'll write a companion post later on where I'll describe one of the techniques used here in more detail: what I think is a fairly novel, though certainly very insane, way of initializing sparse, very-wide, zero-overhead lookup tables. The whole story of how I made it work is a little long to fit into this post.

## Limitations

The program is over-fit to the input spec and the particular host it runs on (Intel Xeon E3-1271 v3 @ 3.60GHz, 512MB RAM, Ubuntu 20.04). Given the CPU, it only uses SIMD instructions up to AVX2, no AVX512. It assumes the input is exactly according to the spec and hence does zero error handling and even on such input will only produce correct results with probability < 1, though very close to 1, depending on the parameters you choose.

## The Algorithm

Here's the high-level overview: forget about parsing the input number-by-number and keeping a running sum! We'll instead iterate over 32 byte chunks of the input using SIMD, from back to front, keeping track of the sum of the digits in each decimal place. In other words, if the input was "123\n45\n678", we'll remember that we've seen a total of 1 + 6 = 7 in the "hundreds", 2 + 4 + 7 = 13 in the "tens" and 3 + 5 + 8 = 16 in the "ones" place. After we're done processing the whole input, we get the final sum by multiplying these decimal place sums with powers of ten: 7\*10² + 13\*10¹ + 16\*10⁰ = 846. Note that since the highest number we have to deal with is 2³¹−1, we have to track at most ⌈log₁₀(2³¹−1)⌉ = 10 decimal place sums.

How do we identify which byte of our input chunk is which decimal place? A look-up table. The mapping from the byte of an input chunk to its decimal place is determined by just two things: the location of newlines in the chunk and the length of the leftmost number in the previous chunk. In other words in a chunk like "???\n??\n???" that follows "???\n???\n???", the first byte is always the 3rd decimal place, then the 2nd, etc., and the last byte is always the 4th decimal place, because it follows a number with 3 digits in the previous chunk.

That's the high level, but of course the details of the implementation matter a lot too, so let's look at the source code. This is the meat of the post, I've commented the code extensively to explain how it works and why it works that way. It might be hard to read on a phone, in which case I recommend bookmarking it and reading it on desktop later.

```cpp
// First, the variables and constants we'll need:

// Pointer to the beginning of our input.
char* start = (...)

// Offset from `start` to the first byte of the current 32 byte chunk.
uint64_t offset = (...)

// SIMD vector full of ASCII '\n'.
const __m256i ascii_zero = _m256_set1_epi8(0x30);

// The size of the leftmost number in the previous chunk (remember, we iterate
// input back-to-front), which partially determines the decimal places of the
// rightmost number in the current chunk. Since we iterate from the end of the
// input, we can initialize this to 0.
last_number_size = 0;

// The decimal place sums we use to reconstruct the final sum as described in
// the overview.
uint64_t decimal_sums[10] = {0};

// For efficiency, we accumulate decimal place sums into this vector and dump
// them into the `decimal_sums` array every `BATCH_SIZE` iterations.
// The layout of this vector is below, where a number represents an exponent
// of the power of ten the byte represents:
// [5, 4, 3, 2, 1, 0, 9, 8, 7, 6, 5, 4, 3, 2, 1, 0 | 5, 4, 3, 2, 1, 0, 9, 8, 7, 6, 5, 4, 3, 2, 1, 0]
//                 ^ this byte accumulates 10^0 = "ones"      ^ this byte accumulates 10^2 = "hundreds"
// The somewhat unusual layout is motivated by the fact that AVX2 shuffle
// cannot move bytes across a lane boundary and because you expect to see
// more low decimal place digits.
__m256i sums_acc = _mm256_set1_epi8(0)

// The decimal place sums accumulator can only accumulate so many chunks
// before overflowing. Worst case scenario is a '9' hitting the same
// accumulator slot twice per iteration of the main loop. Therefore the
// maximum safe accumulation batch size is 255 / (2 * 9) = 14. In practice
// you can increase it to almost twice that number without lowering the
// probability of correct output too much (at least for HighLoad).
const BATCH_SIZE = 14;
int batch = BATCH_SIZE;

// The final sum!
uint64_t sum = 0;

// Next, we ensure our chunks are 64 byte aligned (`start` is already
// aligned) by processing input byte-by-byte with a simple
// scalar algorithm until we reach an aligned offset, keeping track
// of the running sum and the length of the last number encountered.
int exp = 1;
while ((offset & 0xFF) != 0) {
    uint8_t byte = *reinterpret_cast<uint8_t*>(start + offset);
    if (byte == 0x0A) {
        exp = 1;
        last_number_size = 0;
    } else {
        sum += exp * (byte - 0x30);
        exp *= 10;
        last_number_size++;
    }

    offset -= 1;
}

(...)

// Now for the performance critical section!
while (offset) {
    // Prefetch input for future iterations, 11 cache lines forward.
    // 11 chosen empirically.
    _mm_prefetch(reinterpret_cast<const char*>(start + offset - 11*64), _MM_HINT_T0);

    // Load a 32 byte chunk of input.
    __m256i input = _mm256_load_si256(reinterpret_cast<__m256i*>(start + offset));

    // Subtract value of ASCII '0' (0x30) from each byte of the chunk.
    // This accomplishes two things:
    //  1) Bytes that represent digits will now hold the digit value instead
    //     of the ASCII code for the digit. i.e. '0' (value 0x30) will now be
    //     0x00, '1' (value 0x31) will now be 0x01, etc.
    //  2) Bytes that represent newlines will have their top bit set, because
    //     newlines are 0x0A, 0x0A - 0x30 = 0b11011010. This will become
    //     relevant in the next step.
    input = _mm256_sub_epi8(input, ascii_zero);

    // Create a bitmask of newlines in the input chunk, i.e. given a chunk
    // "123\n456\n" return 0b00010001.
    // The bitmask is 32 bits, since the input is 32 bytes and we'll store
    // it in a 64 bit variable with the top half zeroed since we'll need
    // it in a 64 bit context later on when calculating our look-up table
    // location (this generates better assembly).
    // This is where 2) from the comment above comes into play. You might
    // be tempted to say we need `cmpeq(input, 0x0A)` before this `movemask`,
    // but `movemask` only looks at the top bit of each byte to
    // decide the value of the bit in the bitmask, so the fact that the `sub`
    // sets the top bit for each newline, but not for digits, is sufficient.
    uint64_t mask = (uint32_t)_mm256_movemask_epi8(input);

    // The location of the mappings from input bytes to decimal places is
    // determined by the mask and the leftmost number in the previous chunk.
    // There are two mappings for each (mask, last_number_size) pair, 32 bytes
    // each, for one cache line in total and there are 11 of them per newline
    // mask, because last_number_size can be 0 to 10.
    // Here you might be tempted to say we should modify our look-up table
    // structure to let us compute the location as
    // !16! * 64 * mask + 64 * last_number_size.
    // Since we'd only multiply by powers of two, this does result in nicer
    // assembly: `imul` (latency 3), `shl` (latency 1) turns into `shl`, `shl`,
    // but it's ultimately much slower due to cache associativity
    // issues (nice overview of the problem is for example here:
    // https://en.algorithmica.org/hpc/cpu-cache/associativity/)
    uint64_t lut_idx = 11 * 64 * mask + 64 * last_number_size

    // Now we dereference the location to get the two mappings.
    // This is a point where you should be a little confused:
    // 1) We're dereferencing the index by itself, not as an offset into an
    //    array base.
    // 2) The total range of the lut_index variable is very roughly 0 to 2^42,
    //   42 bits address space or some 4TB. That's more than the 500MB RAM we
    //   have available and more than you could create with a normal array
    //   literal.
    // On the positive side the lookup table is very sparse: thanks to the
    // specific input distribution, we only have O(1,000s) chunks of mappings
    // spread over the whole 42 bit address space.
    // This is the part that I'll explain in more detail in a follow up post,
    // but for now you can imagine as if we `mmap` and `memcpy` all the mappings
    // to the right addresses at the start of the program. Let me know if you
    // think you know how to do it ~zero-cost! :)
    // Before I came up with this approach I used a derived index based on the
    // size of the numbers in the chunk using chained `tzcnt`. This shrinks the
    // index space to (barely) fit in a normal look up table, but the index
    // computation was one long dependency chain with high latency instructions
    // and ended up being almost 50% of the runtime.
    __m256i shuffle_ctrl1 = _mm256_loadu_si256(reinterpret_cast<__m256i*>(lut_idx));
    __m256i shuffle_ctrl2 = _mm256_loadu_si256(reinterpret_cast<__m256i*>(lut_idx + 32));

    // When I talked about a mapping from input bytes to decimal places, it's
    // really a shuffle control mask that moves input bytes into the same
    // layout that the `sums_acc` vector has.
    // This is another one of those places where the specific input distribution
    // really matters:
    // There are 4 spots for "ones" in `sums_acc`. If our input chunk consisted
    // of only 1 digit numbers, "1\n2\n\3...", we'd need to perform the following
    // `shuffle`, `add` procedure up to 32 / 2 / 4 = 4 times.
    // Fortunately for us, this turns out to be very unlikely and we are almost
    // guaranteed to be able to completely accumulate the chunk within
    // 2 `shuffles`, so that is what we do in exchange for occasionally
    // producing incorrect results.
    // One of my solutions had a neat compression scheme here:
    // AVX2 `shuffle` only shuffles within each 16 byte lane of the full 32 byte
    // register. It therefore only uses the lower 4 bits of the shuffle control
    // mask so you can trivially pack two of them into one 32 byte vector.
    // Sort of -- it also uses the top bit to let you zero out a byte, which
    // we use a fair bit, (as you can imagine, by the second shuffle we've
    // accumulated most of the bytes of the input). Fortunately,
    // we are guaranteed to have at least one newline in each lane and since
    // we do not care about its value, we can zero it out at the start of each
    // iteration and point bytes that should to be zero to it, rather than
    // zeroing them out using the top bit.
    // I was very excited when I came up with this, but it turned out not
    // to do much performance-wise :) (Haswell can handle two loads per
    // cycle and the shuffle control maps are on a single cache line.)
    __m256i shuffled_input1 = _mm256_shuffle_epi8(input, shuffle_ctrl1);
    __m256i shuffled_input2 = _mm256_shuffle_epi8(input, shuffle_ctrl2);

    // Shuffled inputs 1 & 2 are now in the correct layout for us to add them
    // directly to the decimal place sums accumulator.
    sums_acc = _m256_add_epi8(sums_acc, shuffled_input1);
    sums_acc = _m256_add_epi8(sums_acc, shuffled_input2);

    // This stores the size of the leftmost number for the next iteration.
    // Note that on Haswell this will generate `xor B, B` in addition to
    // `tzcnt A, B`. This is meant as a fix for a false dependency bug on
    // bunch of BMI instructions on this µarch.
    // In our case the fix is counterproductive because we're not bottlenecked
    // on the latency of this instruction. I don't know of any way to bypass
    // that `xor` other than using __asm__ directly.
    // (More on the bug:
    // https://stackoverflow.com/questions/25078285/replacing-a-32-bit-loop-counter-with-64-bit-introduces-crazy-performance-deviati)
    last_number_size = _tzcnt_u32(mask);

    // Once we accumulate `BATCH_SIZE` chunks in `sums_acc`, dump them into
    // the sums array.
    batch--;
    if (!batch) {
        batch = BATCH_SIZE;
        // Extract all accumulated "ones"...
        decimal_sums[0] += _m256_extract_epi8(sums_acc, 5);
        decimal_sums[0] += _m256_extract_epi8(sums_acc, 15);
        decimal_sums[0] += _m256_extract_epi8(sums_acc, 21);
        decimal_sums[0] += _m256_extract_epi8(sums_acc, 31);
        (...)
        // ...and up to 10^9's.
        decimal_sums[9] += _m256_extract_epi8(sums_acc, 6);
        decimal_sums[9] += _m256_extract_epi8(sums_acc, 22);
    }

    // Move the offset to the next batch.
    offset -= 32;
}

(...)

// All that's left to do is to multiply our accumulated decimal place sums by
// the right power of ten and sum to get the final sum.
exp = 1;
for (int i = 0; i < 10; i++) {
    sum += decimal_sums[i] * exp;
    exp *= 10;
}

// And there it is!
print(sum)
```



# Fin
Let me know if you have any feedback and thank you to the HighLoad community on Telegram, especially gracefu and Jack Frigaard.