---
layout: post
title: mirror_hash - Automatic hashing for any class
tags: Modern C++, Hashing, Reflection, C++26, Performance
---

# "I Love to Specialize Hash Functions for My C++ Classes" (No One, Ever)

Writing `std::hash` specializations is one of those C++ rituals that (almost) nobody enjoys. You've probably done something like:

```cpp
struct Point {
    int x, y, z;
    bool operator==(const Point&) const = default;
};

// Now you have to write this boilerplate:
template<>
struct std::hash<Point> {
    std::size_t operator()(const Point& p) const noexcept {
        std::size_t seed = 0;
        // boost::hash_combine pattern
        seed ^= std::hash<int>{}(p.x) + 0x9e3779b9 + (seed << 6) + (seed >> 2);
        seed ^= std::hash<int>{}(p.y) + 0x9e3779b9 + (seed << 6) + (seed >> 2);
        seed ^= std::hash<int>{}(p.z) + 0x9e3779b9 + (seed << 6) + (seed >> 2);
        return seed;
    }
};
```

This pattern is based on [boost::hash_combine](https://www.boost.org/doc/libs/1_86_0/libs/container_hash/doc/html/hash.html#combine).

For *every single type* you want to use in an ~~unordered_map~~ _insert your favorite hash table here_. And when your struct changes? Update the hash function. Forget a member? Silent bugs. Add a pointer to a vector? Now you need to think about whether to hash the pointer or the contents.

**What if I told you C++26 can help us do this automatically?**

*"To a man with a hammer, every problem looks like a nail."*

I first heard of this concept from Charlie Munger's excellent talk [The Psychology of Human Misjudgment](https://www.youtube.com/watch?v=AKxE4RlCgjY), one of my favorite talks ever. It's also known as [Law of the instrument](https://en.wikipedia.org/wiki/Law_of_the_instrument).

Automatic hashing is actually one of the motivating examples in [P2996 itself](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p2996r13.html), listed as "member-wise hash_append." The proposal authors clearly saw this use case coming. Other applications include [mirror_bridge](https://chico.dev/Mirror-Bridge) for automatic binding generation and [simdjson](https://github.com/simdjson/simdjson) for [reflection-based JSON parsing](https://github.com/simdjson/simdjson/blob/master/doc/basics.md#c26-static-reflection). This hammer is pretty versatile (:

## Enter C++26 Reflection

C++26's reflection proposal (P2996) gives us the ability to introspect types at compile time. We can iterate over struct members, get their names, types, and values, all without macros or code generation.

Here's the core idea behind [mirror_hash](https://github.com/FranciscoThiesen/mirror_hash):

```cpp
#include "mirror_hash/mirror_hash.hpp"

struct Point {
    int x, y, z;
    bool operator==(const Point&) const = default;
};

int main() {
    Point p{10, 20, 30};

    // Just works. No specialization needed.
    auto h = mirror_hash::hash(p);

    // Works with standard containers too
    std::unordered_set<Point, mirror_hash::hasher<>> points;
    points.insert({1, 2, 3});
}
```

No boilerplate. No manual member enumeration. It just works out of the box with any struct/class/type you throw at it.

## How It Works: Reflection at Compile Time

The magic happens with two new C++26 features working together:

1. **`nonstatic_data_members_of(^^T)`**: Returns reflections of all data members
2. **`template for`**: Iterates over reflections at compile time (P1306 expansion statements)

```cpp
template<typename Policy, Reflectable T>
std::size_t hash_impl(const T& value) noexcept {
    std::size_t result = 0;

    // template for: compile-time iteration over all members
    template for (constexpr auto member :
        std::meta::nonstatic_data_members_of(^^T, std::meta::access_context::unchecked()))
    {
        // value.[:member:] splices the reflection to access the actual member
        result = Policy::combine(result, hash_impl<Policy>(value.[:member:]));
    }

    return result;
}
```

Let's break down the syntax:

- **`^^T`**: The *reflection operator*. Returns a compile-time handle to type `T`'s metadata.
- **`nonstatic_data_members_of(...)`**: Returns a range of reflections, one per data member.
- **`template for`**: Like a regular `for`, but unrolls at compile time. Each iteration is a separate instantiation.
- **`value.[:member:]`**: The *splice operator*. Converts a reflection back into code (in this case, member access).

## The Key Insight: Hash Only What You Use

One concern with automatic hashing: "Won't this generate code for every type in my program?"

**No.** C++ templates are lazy. Hash functions are only instantiated when you actually use them:

```cpp
struct NeverHashed {
    std::string name;
    std::vector<int> data;
};

struct ActuallyHashed {
    int id;
    double value;
};

int main() {
    // This instantiates hash code for ActuallyHashed only
    std::unordered_set<ActuallyHashed, mirror_hash::hasher<>> set;

    // NeverHashed gets no hash code generated
    NeverHashed unused;
}
```

The compiler only generates `hash_impl<Policy, ActuallyHashed>`. The `NeverHashed` struct incurs zero overhead because we never hash it.

This is the beauty of C++ templates: you don't pay for what you don't use. Reflection extends this principle: metadata is queried at compile time, and only for types that actually need it. See [`test_lazy_instantiation`](https://github.com/FranciscoThiesen/mirror_hash/blob/main/tests/test_mirror_hash.cpp#L658) for the test that verifies this behavior.

## Handling Non-Trivially Copyable Types

A critical design decision in any hashing library: how do you handle types like `std::string`, `std::vector`, or structs containing them?

mirror_hash handles this through **concept-based dispatch**:

```cpp
// Non-trivially copyable types get per-member hashing
template<typename T>
concept Reflectable = std::is_class_v<T> &&
    !StringLike<T> && !Container<T> && !Optional<T> && !Variant<T>;

// Strings hash their contents, not their internal pointers
template<typename Policy, StringLike T>
std::size_t hash_impl(const T& value) noexcept {
    return std::hash<T>{}(value);
}

// Containers iterate over elements
template<typename Policy, Container T>
std::size_t hash_impl(const T& value) noexcept {
    std::size_t result = hash_impl<Policy>(value.size());
    for (const auto& elem : value) {
        result = Policy::combine(result, hash_impl<Policy>(elem));
    }
    return result;
}
```

When you hash a struct containing a `std::string`:

```cpp
struct Person {
    std::string name;  // Non-trivially copyable!
    int age;
};
```

mirror_hash recursively hashes each member. For `name`, it hashes the string *contents*. For `age`, it hashes the integer directly. This is safe and correct for complex types.

## The Fast Path: Trivially Copyable Types

For simple structs like `Point`, we can do much better. If all members are trivially copyable (ints, floats, no pointers), we can hash the entire struct as raw bytes:

```cpp
// Check if all members are trivially copyable using template for
template<typename T>
consteval bool all_members_trivially_copyable() {
    template for (constexpr auto member :
        std::meta::nonstatic_data_members_of(^^T, std::meta::access_context::unchecked()))
    {
        // [:type_of(member):] splices the member's type
        if (!std::is_trivially_copyable_v<[:std::meta::type_of(member):]>) {
            return false;
        }
    }
    return true;
}

template<typename Policy, Reflectable T>
std::size_t hash_impl(const T& value) noexcept {
    // Fast path: if all members are trivially copyable, hash raw bytes
    if constexpr (std::is_trivially_copyable_v<T> && all_members_trivially_copyable<T>()) {
        return hash_bytes_fixed<Policy, sizeof(T)>(&value);
    }

    // Slow path: iterate over members with template for
    std::size_t result = 0;
    template for (constexpr auto member : std::meta::nonstatic_data_members_of(^^T)) {
        result = Policy::combine(result, hash_impl<Policy>(value.[:member:]));
    }
    return result;
}
```

This compile-time optimization means a `struct { int x, y, z; }` gets hashed as a single 12-byte block, not three separate hash operations combined together.

## Great, This is Convenient. But Can We Make It Fast?

<p align="center">
<img src="https://i.imgflip.com/51mqjx.jpg" alt="I have no idea what I'm doing" />
<br>
<em>"I have no idea what I'm doing"</em>
</p>

Let me be upfront: **I'm not a cryptography expert.** As [Schneier's Law](https://www.schneier.com/blog/archives/2011/04/schneiers_law.html) reminds us: *"Anyone can design a security system that he himself cannot break. This doesn't mean it's secure."* The same applies to hash functions. It's easy to write something that *looks* like it works, but designing one with good statistical properties is hard.

That's why the core mixing primitives in mirror_hash come from people who actually know what they're doing:

- **[wyhash](https://github.com/wangyi-fudan/wyhash)** by Wang Yi: The default hash used by Go, Zig, and Nim
- **[rapidhash](https://github.com/Nicoshev/rapidhash)** by Nicolas De Carli: A faster variant of wyhash
- **[xxHash](https://github.com/Cyan4973/xxHash)** by Yann Collet: Battle-tested for a decade

What I contributed is the *integration layer*: using C++26 reflection to automatically feed your struct's bytes into these proven algorithms. For the `mirror_hash::fast` namespace, I did venture into optimization territory, but the core mixing function is straight from rapidhash.

**Important disclaimer:** I make no cryptographic claims about `mirror_hash::fast`. While it passes SMHasher's quality tests, it hasn't been analyzed in depth by cryptography experts. For hash tables and checksums, it should be fine. For anything security-sensitive, use a properly vetted cryptographic hash.

### The `fast` Namespace: Benchmark-Driven Optimization

The `mirror_hash::fast::hash()` function started as a rapidhash port, but I kept benchmarking and tweaking until the numbers stopped improving. The result is a collection of size-specific strategies. Not because "different sizes need different algorithms," but because **I ran benchmarks and these are what worked best**.

**The Core Mixing Primitive:**

This is the one thing I didn't touch. It's the heart of rapidhash/wyhash:

```cpp
inline void mum(uint64_t* A, uint64_t* B) noexcept {
    __uint128_t r = static_cast<__uint128_t>(*A) * (*B);
    *A = static_cast<uint64_t>(r);       // Low 64 bits
    *B = static_cast<uint64_t>(r >> 64); // High 64 bits
}

inline uint64_t mix(uint64_t A, uint64_t B) noexcept {
    mum(&A, &B);
    return A ^ B;
}
```

A 64×64→128 bit multiply has excellent avalanche properties. The XOR of both halves ensures information from both inputs propagates throughout. This is well-studied math. I just use it.

**Size-Specific Optimizations:**

Here's where the benchmarking comes in. I partitioned inputs by size and optimized each range separately:

```
0-3 bytes:    Branchless byte packing
4-8 bytes:    Overlapping 32-bit reads
9-16 bytes:   Overlapping 64-bit reads
17-48 bytes:  Progressive mixing
49-96 bytes:  4-way parallel accumulators
97B-128KB:    6-way parallel mixing with unrolled loops
>128KB:       Weak accumulation + strong finalization
```

### Overlapping Reads

For inputs like 5 bytes, we read bytes 0-3 and bytes 1-4. They overlap, but that's fine: we're hashing, not parsing.

```cpp
// 4-8 bytes: Read from start and end, they might overlap
template<std::size_t N>
inline uint64_t hash_4_8(const void* data) noexcept {
    const auto* p = static_cast<const uint8_t*>(data);
    uint64_t a = read32(p);
    uint64_t b = read32(p + N - 4);  // Overlaps if N < 8
    return mix(a ^ S[1], b ^ (BASE_SEED ^ N));
}
```

Note that in this template, **N is a compile-time constant**. There are no runtime branches here. The compiler generates specialized code for each size. The overlapping reads technique is primarily about **code simplicity**: one template handles the entire 4-8 byte range elegantly.

This pattern matters more in the runtime hash function (`hash(const void* key, std::size_t len, ...)`) where `len` is unknown at compile time. There, overlapping reads avoid branching on the exact size within a range.

### Parallel Accumulators: Exploiting Instruction-Level Parallelism

Modern CPUs can perform multiple multiplications at the same time, but only if there are no data dependencies between them. A single chain of `mix()` calls forces the CPU to wait:

```cpp
// Each mix() waits for the previous one - serial execution
h = mix(h, read64(p));
h = mix(h, read64(p + 8));   // Must wait for previous mix
h = mix(h, read64(p + 16));  // Must wait again
```

With independent accumulators, the CPU can issue all multiplications simultaneously:

```cpp
// 4 independent multiplication chains - CPU executes in parallel
uint64_t a = mix(read64(p), read64(p + 32));
uint64_t b = mix(read64(p + 8), read64(p + 40));
uint64_t c = mix(read64(p + 16), read64(p + 48));
uint64_t d = mix(read64(p + 24), read64(p + 56));

// Combine at the end
return mix(mix(a, c), mix(b, d));
```

I tried 2-way, 4-way, 6-way, and 8-way parallelism. 4-way won for medium sizes; 6-way won for larger inputs. The numbers came from benchmarks, not theory.

### The 50% Bulk Improvement: Weak Accumulation + Strong Finalization

This is where the biggest gain comes from. For inputs larger than 128KB, even parallel `mix()` calls become a bottleneck. Here's why:

The `mix()` function performs a **128-bit multiply** (64×64→128 bits), then XORs the two halves. On modern CPUs, this takes about **7 cycles**. For bulk data, we're calling it millions of times.

The insight: **what if we used a cheaper operation during the main loop, then compensated with a thorough finalization?**

```cpp
// The "cheap" accumulator: ~3-4 cycles per block
inline uint64_t cheap_acc(uint64_t acc, uint64_t a, uint64_t b) noexcept {
    return acc * 0x9e3779b97f4a7c15ULL + (a ^ b);
}
```

This is just a 64-bit multiply-add, half the work of `mix()`. The constant `0x9e3779b97f4a7c15` is derived from the [golden ratio](https://probablydance.com/2018/06/16/fibonacci-hashing-the-optimization-that-the-world-forgot-or-a-better-alternative-to-integer-modulo/). Specifically, it's `floor(2^64 / φ)` where φ is the golden ratio. This constant is also used in the [Linux kernel's hash functions](https://github.com/torvalds/linux/blob/master/include/linux/hash.h) and provides excellent bit dispersion.

The trick is that we run **8 independent accumulators** in parallel, processing 128 bytes per iteration:

```cpp
while (i >= 128) {
    __builtin_prefetch(p + 256, 0, 3);  // Hide memory latency
    acc0 = cheap_acc(acc0, read64(p), read64(p + 8));
    acc1 = cheap_acc(acc1, read64(p + 16), read64(p + 24));
    acc2 = cheap_acc(acc2, read64(p + 32), read64(p + 40));
    acc3 = cheap_acc(acc3, read64(p + 48), read64(p + 56));
    acc4 = cheap_acc(acc4, read64(p + 64), read64(p + 72));
    acc5 = cheap_acc(acc5, read64(p + 80), read64(p + 88));
    acc6 = cheap_acc(acc6, read64(p + 96), read64(p + 104));
    acc7 = cheap_acc(acc7, read64(p + 112), read64(p + 120));
    p += 128;
    i -= 128;
}
```

Why 8-way? The CPU can execute all 8 multiplications simultaneously because there are no dependencies between `acc0`-`acc7`. We're processing 128 bytes per ~4 cycles instead of ~56 cycles (8 × 7 cycles for sequential `mix()`).

The `__builtin_prefetch` tells the CPU to start fetching the next cache line before we need it, hiding memory latency.

**But doesn't this weaken the hash quality?** Yes, the per-block mixing is weaker. Let me explain these terms:

- **Per-block mixing** is the operation applied to each chunk of input data as you iterate through it. Strong per-block mixing ensures that even small changes in one block affect the accumulator significantly. Our `cheap_acc` uses a simple multiply-add, which is fast but doesn't fully scramble the bits.

- **Finalization** is the final step after processing all input blocks. It takes the accumulated state and produces the final hash value. Strong finalization ensures good [avalanche](https://en.wikipedia.org/wiki/Avalanche_effect), where changing a single input bit flips approximately 50% of output bits.

The insight is that you can trade off between these two: use cheap per-block mixing to maximize throughput, then apply expensive finalization once at the end. That's exactly what we do with a **triple-mix finalization**:

```cpp
// Combine all 8 accumulators
uint64_t h = acc0 ^ acc1 ^ acc2 ^ acc3 ^ acc4 ^ acc5 ^ acc6 ^ acc7;
h ^= len;

// Strong finalization: three full mix() calls
h = mix(h ^ S[0], S[1]);
h = mix(h ^ S[2], S[3]);
h = mix(h ^ S[4], S[5] ^ len);
```

Three `mix()` calls provide excellent avalanche: every input bit affects every output bit. This is only 21 cycles of overhead, amortized over potentially gigabytes of data.

**The result:** ~50% faster than rapidhash for inputs >128KB, while still passing all SMHasher quality tests.

## Quality Validation: SMHasher

Quality matters as much as speed. A fast hash that produces collisions is useless.

We validate using the **official [SMHasher](https://github.com/rurban/smhasher) test suite**, the canonical benchmark for non-cryptographic hash functions. It runs dozens of statistical tests:

| Test | What It Checks |
|------|----------------|
| **Avalanche** | Each input bit affects ~50% of output bits |
| **Sparse** | Inputs with few set bits don't collide |
| **Permutation** | Reordered bytes produce different hashes |
| **Cyclic** | Rotated inputs don't correlate |
| **TwoBytes** | Two-byte variations distribute uniformly |
| **Text** | ASCII patterns don't collide |
| **MomentChi2** | Statistical distribution is uniform |

**mirror_hash passes all SMHasher tests:**

| Test | Result |
|------|--------|
| Sanity | PASS |
| Avalanche | PASS (worst bias: 0.77%) |
| Keyset Sparse | PASS |
| Keyset Permutation | PASS |
| Keyset Window | PASS |
| Keyset Cyclic | PASS |
| Keyset TwoBytes | PASS |
| Keyset Text | PASS |
| MomentChi2 | Great |
| Prng | PASS |
| BadSeeds | PASS |

To run SMHasher yourself, see the `smhasher/` directory in the repository for integration instructions.

## Benchmarks

### Trivially Copyable Structs (Fast Path)

These benchmarks focus on **trivially copyable structs**: types where all members are primitive types (int, float, etc.) with no pointers, strings, or containers. For these types, mirror_hash uses the fast path: hashing the struct as a contiguous block of bytes.

Tested on ARM64 (Apple Silicon M3 Max), Clang 21, `-O3 -march=native`:

| Size | mirror_hash::fast vs rapidhash |
|------|--------------------------------|
| 4-16B | +6% faster |
| 128B | +16% faster |
| 512B-64KB | +3-4% faster |
| 128KB+ | **+50% faster** |

**Throughput:** ~58 GB/s bulk, ~10.7 cycles/hash for small keys.

### What is "Trivially Copyable"?

A type is **trivially copyable** if it can be safely copied by just copying its raw bytes (like with `memcpy`). The C++ standard defines this precisely, but intuitively:

**Trivially copyable:**
- Primitive types: `int`, `double`, `char`, `bool`, pointers
- C-style arrays of trivially copyable types
- Structs/classes with only trivially copyable members and no custom copy/move constructors

**NOT trivially copyable:**
- `std::string` (has internal pointer to heap-allocated buffer)
- `std::vector` (has pointer to dynamic array)
- `std::unique_ptr`, `std::shared_ptr` (manage heap resources)
- Any class with virtual functions or custom copy semantics

You can check at compile time with `std::is_trivially_copyable_v<T>`.

### Non-Trivially Copyable Types

For structs containing `std::string`, `std::vector`, smart pointers, or other non-trivially copyable types, mirror_hash uses **member-by-member hashing**: iterating over each member using `template for` and hashing them recursively.

This is the only correct approach:

```cpp
struct Person {
    std::string name;   // 32 bytes (SSO buffer + pointer + size + capacity)
    int age;            // 4 bytes
};
// You CAN'T hash raw bytes here - the string contains pointers!
// Two identical strings at different addresses would hash differently.
```

**What does mirror_hash generate?** Essentially what you'd write by hand:

```cpp
// What mirror_hash generates (conceptually):
size_t hash(const Person& p) {
    size_t h = std::hash<std::string>{}(p.name);  // Hash string contents
    h = combine(h, std::hash<int>{}(p.age));       // Combine with age
    return h;
}
```

**Measured times** (ARM64, Clang 21, `-O3`):

| Struct Type | Time | Notes |
|-------------|------|-------|
| `Point{int, int}` | ~0.3 ns | Trivially copyable, raw bytes |
| `Person{string(5), int}` | ~2.5 ns | String + int |
| `Person{string(32), int}` | ~1.0 ns | Longer string (still fast) |
| `Data{vector(5), string}` | ~15 ns | Must iterate 5 elements |
| `Data{vector(100), string}` | ~45 ns | Must iterate 100 elements |

The time scales with the actual work required:
- **String hashing** depends on string length (though [SSO, or Small String Optimization](https://devblogs.microsoft.com/oldnewthing/20240510-00/?p=109742), helps for short strings by storing them inline)
- **Container hashing** iterates over all elements. There's no shortcut
- **Member count** adds overhead for the combine operations

This isn't "overhead" in the sense of wasted work. It's the inherent cost of correctly hashing dynamic data. A hand-written hash function would have the same cost. The value of mirror_hash is that you don't have to write it yourself.

Run the benchmark yourself:
```bash
./build/trivial_vs_nontrivial
```

## What About Existing Solutions?

Fair question: do we really need C++26 reflection for this? There are existing approaches:

**[Boost.Describe](https://www.boost.org/doc/libs/1_87_0/libs/describe/doc/html/describe.html):** Requires annotating each struct with a macro:
```cpp
struct Point { int x, y; };
BOOST_DESCRIBE_STRUCT(Point, (), (x, y))  // Manual member list
```
Works well, but you must remember to update the macro when adding members. Miss one, and you get silent bugs.

**[Cista](https://github.com/felixguendling/cista):** Requires a `cista_members()` function in each type:
```cpp
struct Point {
    int x, y;
    auto cista_members() { return std::tie(x, y); }  // Manual
};
```
Same issue: manual enumeration that can drift out of sync.

**[Abseil Hash](https://abseil.io/docs/cpp/guides/hash):** Requires a friend function:
```cpp
struct Point {
    int x, y;
    template<typename H>
    friend H AbslHashValue(H h, const Point& p) {
        return H::combine(std::move(h), p.x, p.y);  // Manual
    }
};
```

The pattern is clear: all existing solutions require **manual member enumeration**. You're still writing boilerplate, just in a different form. Add a member, forget to update the hash, get subtle bugs.

**mirror_hash is different.** With C++26 reflection, the compiler provides the member list. There's nothing to keep in sync:

```cpp
struct Point { int x, y, z; };  // Add z, hash automatically includes it
auto h = mirror_hash::hash(Point{1, 2, 3});  // Just works
```

This is the fundamental advantage: **true automatic generation** with zero annotation overhead.

## The Catch: You Need a Special Compiler (for now)

C++26 reflection isn't in any released compiler yet. You need Bloomberg's [clang-p2996](https://github.com/bloomberg/clang-p2996) branch:

```bash
# Compile flags
clang++ -std=c++2c -freflection -freflection-latest mycode.cpp
```

It's experimental. It might break. But it's a glimpse of C++'s future, and that future involves a lot less boilerplate.

## Try It Out

```bash
git clone https://github.com/FranciscoThiesen/mirror_hash
cd mirror_hash
./start_dev_container.sh  # Uses Docker with clang-p2996
./build.sh
./run_tests.sh
```

## Conclusion

The days of manually writing `std::hash` specializations are numbered. C++26 reflection gives us the tools to generate them automatically, with zero runtime overhead and optimal performance.

For years, we've written code that the compiler could infer. We've manually enumerated members that the compiler already knows about. P2996 finally bridges that gap.

The future of C++ is less boilerplate and more automation. And I, for one, can't wait.

---

*mirror_hash is MIT licensed. The underlying hash algorithms (wyhash, rapidhash, xxHash) have their own licenses. Please check their respective repositories.*

## Acknowledgments

Thanks to Wyatt Childers, Peter Dimov, Dan Katz, Barry Revzin, Andrew Sutton, Faisal Vali, and Daveed Vandevoorde for authoring and championing P2996, the reflection proposal that makes this all possible.

Special thanks to Dan Katz for maintaining the clang-p2996 branch. Without a working compiler, this would all be theoretical.

On the hashing side: thanks to Wang Yi for creating wyhash, which has become the foundation of modern fast hashing. Thanks to Nicolas De Carli for rapidhash. In our benchmarks, it was the fastest hash function we tested, which is why we use it as our primary comparison point. And thanks to Yann Collet for xxHash, which has been battle-tested for over a decade and set the standard for what a high-quality hash function should be.

## References

- [P2996R13 - Reflection for C++26](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p2996r13.html): The main reflection proposal
- [P1306R5 - Expansion Statements](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p1306r5.html): The `template for` feature
- [Bloomberg clang-p2996](https://github.com/bloomberg/clang-p2996): Experimental compiler with reflection support
- [Trip Report: June 2025 ISO C++ Meeting](https://herbsutter.com/2025/06/21/trip-report-june-2025-iso-c-standards-meeting-sofia-bulgaria/): When reflection was voted into C++26
- [SMHasher](https://github.com/rurban/smhasher): The canonical hash quality test suite
- [rapidhash](https://github.com/Nicoshev/rapidhash): The hash algorithm that mirror_hash::fast builds upon
- [wyhash](https://github.com/wangyi-fudan/wyhash): The foundation of modern fast hashing
