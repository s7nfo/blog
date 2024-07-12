---
layout: post
title:  "Probably the fastest way to sum integers you've ever seen"
---

"Print the sum of 50 million ASCII-encoded integers uniformly sampled from [0, 2³¹−1], separated by a single new line and sent to standard input."

On the surface, a trivial problem. But what if you wanted to go as fast as possible?

I'm currently one of the top ranked competitors in [exactly that kind of challenge](https://highload.fun/tasks/1) and in this post I'll show you a sketch of my best performing solution. I'll leave out a some of the µoptimizations and look-up table generation to keep this post short, easier to understand and to not completely obliterate the HighLoad leaderboard. Regardless, as far as I know nothing similar has been published yet, so I'm hoping you'll find it valuable.

I'll write a companion post later on where I'll describe one of the optimizations I'm leaving out: what I think is a fairly novel, though certainly very insane, way of initializing ultra-wide, zero-overhead lookup tables. The whole story of how I made it works is somewhat complex.

On the target hardware my program runs about 320x faster than the following naive C++ solution (and is about 1,000,000x more fragile):

```cpp
uint64_t sum = 0;
while (std::cin) {
    uint64_t v = 0;
    std::cin >> v;
    sum += v;
}

std::cout << sum << std::endl;
```

## Limitations

The program is over-fit to the specifics of the input on HighLoad and the particular host it runs on (Intel Xeon E3-1271 v3 @ 3.60GHz, 512MB RAM, Ubuntu 20.04). Given the CPU, it only uses SIMD instructions up to AVX2, no AVX512. It assumes the input is exactly according to the spec and hence does zero error handling and even on such input will produce correct results with probability < 1, though very close to 1, depending on the parameters you choose.

## The Algorithm

Here's the high-level overview: forget about parsing the input number-by-number and keeping a running sum! We'll instead iterate over 32 byte chunks of the input using SIMD, from back to front, keeping track of the sum of the digits in each decimal place. In other words, if the input was "123\n45\n678", we'll remember that we've seen a total of 7 in the "hundreds", 13 in the "tens" and 11 in the "ones" place. After we're done processing the whole input, we get the final sum by multiplying these decimal place sums with powers of ten: 7\*10² + 13\*10¹ + 11\*10⁰ = 846. Note that since the highest number we have to deal with is 2³¹−1, we have to track at most ⌈log₁₀(2³¹−1)⌉ = 10 decimal place sums.

How do we identify which byte of our input chunk is which decimal place? A look-up table. The mapping from byte of an input chunk to its decimal place is determined by just two things: the location of newlines in the chunk and the length of the left-most number in the previous chunk. In other words in a chunk like "???\n??\n???" that follows "???\n???\n???", the first byte is always the 3rd decimal place, then the 2nd, etc., and the last byte is always the 4th decimal place, because it follows a number with 3 digits in the previous chunk.

That's the high level, but of course the details of the implementation matter a lot too, so let's look at the source code.

```cpp
// First, the variables and constants we'll need:

// Pointer to the beginning of our input.
char* start = (...)

// Offset from `start` to the first byte of current 32 byte chunk.
// TODO: long long??
long long offset = (...)

// SIMD vector full of ASCII '\n'.
const __m256i ascii_zero = _m256_set1_epi8(0x30);

// The size of the left-most number in the previous chunk (remember, we iterate
// input back-to-front), which partially determines the decimal places of the
// right-most number in the current chunk. Since we iterate from the end of the
// input, we can initialize this to 0.
last_number_size = 0;

// The decimal place sums we use to reconstruct the final sum as described in
// the overview.
uint64_t decimal_sums[10] = {0};

// For efficiency, we accumulate decimal place sums into this vector and dump
// it into the array above every `BATCH_SIZE` iterations.
// The layout of this vector is below, where a number represents an exponent
// of the power of ten the byte represents:
// [5, 4, 3, 2, 1, 0, 9, 8, 7, 6, 5, 4, 3, 2, 1, 0 | 5, 4, 3, 2, 1, 0, 9, 8, 7, 6, 5, 4, 3, 2, 1, 0]
//                 ^ this byte accumulates 10^0 = "ones"      ^ this byte accumulates 10^2 = "hundreds"
// The somewhat unusual layout is motivated by the fact that AVX2 shuffle
// cannot move bytes accross a lane boundary and because you expect to see
// more low exponent digits.
__m256i digits = _mm256_set1_epi8(0)

// The digits accumulator can only accumulate a limited number of chunks.
// Worst case scenario is a '9' hitting the same decimal slot in the
// accumulator twice per iteration of the main loop. Therefore the
// maximum safe accumulation batch size is 255 / (2 * 9) = 14. In practice
// you can increase it to almost twice that number without lowering the
// probability of correct output too much.
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

// Now for the performance critical section.
while (offset) {
    // Prefetch input for future iterations, 11 cache lines forward.
    // 11 was chosen empirically.
    _mm_prefetch(reinterpret_cast<const char*>(start + offset - 11*64), _MM_HINT_T0);

    // Load 32 byte chunk of input.
    __m256i input = _mm256_load_si256(reinterpret_cast<__m256i*>(start + offset));

    // Substract value of ASCII '0' (0x30) from each byte of the chunk.
    // This accomplishes two things:
    //  1) Bytes that represent digits will now hold the digit value instead
    //     of the ASCII code for the digit. ie. '0' (value 0x30) will now be
    //     0x00, '1' (value 0x31) will now be 0x01, etc.
    //  2) Bytes that represent newlines will have their top bit set, because
    //     newlines are 0x0A, 0x0A - 0x30 = 0b11011010. This will become
    //     relevant in the next step.
    input = _mm256_sub_epi8(input, ascii_zero);

    // Create a bitmask of newlines in the input chunk, ie. given a chunk
    // "123\n456\n" return 0b00010001.
    // The bitmask is 32 bits, since the input is 32 bytes and we'll store
    // it in a 64 bit variable with the top half zeroed since we'll need
    // it in a 64 bit context later on when calculating our look-up table
    // index (this generates better assembly).
    // This is where 2) from the comment above comes into play. You might
    // be tempted to say we need `cmpeq(input, 0x0A)` before this `movemask`,
    // but `movemask` only looks at the top bit of each byte to
    // decide the value of the bit in the bitmask, so the fact that the `sub`
    // sets the top bit for each newline, but not for digits, is precisely
    // what we need here.
    uint64_t mask = (uint32_t)_mm256_movemask_epi8(input_next);

    // The location of the mappings from input bytes to decimal places is
    // determined by the mask and the left-most number in the previous chunk.
    // There are two mappings for each (mask, last_number_size) pair, 32 bytes
    // each, for 64 bytes in total and there are 11 of them per newline mask,
    // because last_number_size can be 0 to 10.
    // Here you might be tempted to say we should slightly modify our look-up
    // table structure to let us compute the index as
    // !16! * 64 * mask + 64 * last_number_size.
    // Since we only multiply by powers of two, this does result in nicer
    // assembly: `imul` (latency 3), `shl` (latency 1) turns into
    // `shl`, `shl`, but is ultimately much slower due to cache associativity
    // issues (nice overview of the problem is for example here:
    // https://en.algorithmica.org/hpc/cpu-cache/associativity/)
    lut_idx = 11 * 64 * mask + 64 * last_number_size

    // Now we dereference the location to get the two mappings.
    // This is also a point where you should be a little confused:
    // 1) We're dereferencing the index by itself, not as an offset into an
    //    array base.
    // 2) The total range of the lut_index variable is very roughly 0 to 2^42,
    //   42 bits address space or some 4TB.  That's a little more than the 500MB we have available.
    // Fortunately, the lookup table is very sparse: we only have O(1000s) chunks of entries spread over whole 42 bit address space.
    // For now you can imagine as if we `memmap` and `memcpy` all those chunks into the right addresses at the start of the program.
    // This is not what the actual solution does and as I mentioned at the start, I'll be publishing a companion post on how to do this
    // nearly zero-cost since it's a little too complex for a single post.
    // Before I came up with this approach I used a derived index based on the size of the numbers in our chunk using chained `tzcnt`. This shrinks the index space to (barely)
    // fit in a normal look up table, but the index computation was one long dendency chain with high latency instructions and ended up being almost 50% of the runtime.
    __m256i shuffle_ctrl1 = _mm256_loadu_si256(reinterpret_cast<__m256i*>(lut_idx));
    __m256i shuffle_ctrl2 = _mm256_loadu_si256(reinterpret_cast<__m256i*>(lut_idx + 32));

    // When I talked about mapping from input bytes to decimal place it was really a shuffle control register that move input bytes into the same layout as the `digits` accumulator vector has.
    // This is one of those places where the specifics of the input distribution really matter:
    //   There are 4 spots for "ones" in the digits accumulator / shuffled input vectors. If our input chunk consisted only of 1 digit numbers,
    // "1\n2\n\3...", then we'd need to perform this `shuffle` + `add` procedure up to 32 / 2 / 4 = 4 times.
    // Fortunately for us, this turns out to be overwhelmingly unlikely and we are almost guaranteed to be able to completely accumulate the input
    // within 2 `shuffles`, so that is what we do in exchange for accepting occasionally producing incorrect results.
    // We use a `shuffle` to map the abitrary position in the input chunk to the positions in the digits accumulator I've described
    // earlier.
    // One of my solutions had a neat compression scheme here:
    //
    __m256i shuffled_input1 = _mm256_shuffle_epi8(input, shuffle_ctrl1);
    __m256i shuffled_input2 = _mm256_shuffle_epi8(input, shuffle_ctrl2);

    // shuffled input 1 & 2 are now in the correct layout for us to add them directly to the digits accumulator.
    digits = _m256_add_epi8(digits, shuffled_input1)
    digits = _m256_add_epi8(digits, shuffled_input2)

    // Why am I counting RIGHT zeros, lmao??
    // 
    last_number_size = std::countr_zero(mask)
    // Talk about bad asm generation.

    batch--;
    if (!batch) {
        batch = BATCH_SIZE;
        // TODO: Fix up the locations
        sums[0] = _m256_extract_epi8(digits, 0)
        sums[0] = _m256_extract_epi8(digits, 6)
        (...)
        sums[9] = _m256_extract_epi8(digits, 1)

    }

    // Move the offset to the next batch.
    offset -= 32;
}

(...)

// All that's left to do is to multiply our accumulated decimal place sums by the right power of ten and sum to get the final sum.
exp = 1;
for (int i =0; i < 10, i++) {
    sum += sums[i] * exp;
    exp *= 10;
}

// And there it is!
print(sum)
```

# Acknowledgements
Thanks to the HighLoad community on Signal and especially Grace Fu and Jack Frigaard.