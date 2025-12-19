---
layout: post
title: "Mirror Bridge: Outrunning V8, PyPy, and LuaJIT with C++26 Reflection"
tags: Modern C++, Python, Lua, JavaScript, Bindings, Reflection, C++26, Performance
---

*We know Mirror Bridge generated bindings beat the standard Python interpreter. But how do they fare against the big guns?*

---

![Mandelbrot Zoom Animation](/images/mandelbrot_zoom.gif)
*Seahorse Valley (-0.75, 0.1), rendered identically by Python, Lua, and JavaScript calling the same C++ engine*

---

## The Benchmark

PyPy. LuaJIT. V8. These are serious JIT compilers with years of optimization work behind them. Let's see how they compare.

800×600 Mandelbrot, 256 max iterations. Lower is better.

```
PYTHON
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Pure CPython:    2.54s  ████████████████████████████████████████
Pure PyPy:       0.14s  ██▎
C++ Binding:     0.06s  █

LUA
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Pure Lua 5.4:    0.76s  ████████████
Pure LuaJIT:     0.24s  ████
C++ Binding:     0.07s  █▏

JAVASCRIPT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Pure V8:         0.09s  █▌
C++ Binding:     0.06s  █
```

| Runtime | Time | vs C++ Binding |
|---------|------|----------------|
| CPython | 2.54s | 42× slower |
| PyPy | 0.14s | 2.3× slower |
| Lua 5.4 | 0.76s | 11× slower |
| LuaJIT | 0.24s | 3.4× slower |
| V8 | 0.09s | 1.5× slower |
| **C++ Binding** | **0.06s** | — |

V8 is a serious piece of engineering: speculative optimization, inline caching, hidden classes. And we can still beat it.

---

## How It Works

If you're not familiar with C++26 reflection, check out my [previous post on Mirror Bridge](/Mirror-Bridge) for the fundamentals.

The short version: C++26 reflection lets you iterate over struct members at compile time:

```cpp
template for (constexpr auto member : std::meta::nonstatic_data_members_of(^^T)) {
    auto& value = obj.[:member:];  // splice member directly into code
    // generate binding for this member...
}
```

The `[:member:]` splice compiles to a direct memory offset, exactly like hand-written `obj.width`. No string lookup, no virtual dispatch.

**The entire binding for Mandelbrot:**

```cpp
MIRROR_BRIDGE_MODULE(mandelbrot,
    mirror_bridge::bind_class<Mandelbrot>(m, "Mandelbrot");
)
```

One line. Every constructor, method, and property discovered and bound automatically.

---

## Why Was C++ Losing to V8?

Early benchmarks showed something unexpected: the C++ binding was running at 0.7× V8's speed. What was going on?

The culprit wasn't the computation. It was the data transfer.

`render_rgb()` returns 1.44 million bytes (800×600×3). A naive binding converts element-by-element:

```cpp
// The bottleneck: 1.44 million calls to napi_set_element
for (size_t i = 0; i < data.size(); i++) {
    napi_set_element(env, array, i, to_js(data[i]));
}
```

The fix: detect byte containers and use bulk transfer:

```cpp
template<ByteContainer T>
napi_value to_javascript(napi_env env, const T& container) {
    void* buffer_data;
    napi_create_arraybuffer(env, container.size(), &buffer_data, &buffer);
    std::memcpy(buffer_data, container.data(), container.size());  // Single copy
    return typed_array;  // Returns Uint8Array
}
```

This single change took the JavaScript binding from 0.7× to 1.5× V8's speed. The C++ computation was always winning; naive data transfer was losing it all back.

Mirror Bridge now detects `ByteContainer` types (contiguous ranges of bytes) and uses bulk transfers across all three language bindings.

---

## vs. Existing Solutions

| | pybind11 | sol2 | N-API | Mirror Bridge |
|---|----------|------|-------|---------------|
| Lines per method | 3-5 | 2-4 | 5-10 | 0 |
| New method added | Update binding | Update binding | Update binding | Recompile |
| Multi-language | No | No | No | Yes |
| Type conversions | Manual registration | Automatic | Manual | Automatic |

**Why not just use pybind11?**

You absolutely can. pybind11 is mature, well-documented, and battle-tested.

But if you're binding the same types to multiple languages, or you're tired of keeping binding declarations in sync with your headers, reflection gives you a third option: declare nothing, discover everything.

---

## What About C++26?

> "C++26 isn't in compilers yet!"

True. P2996 was voted into C++26 at the Sofia meeting in June 2025, so the standard is locked. But mainstream compiler support is still catching up.

Mirror Bridge uses [Bloomberg's experimental Clang fork](https://github.com/bloomberg/clang-p2996). The dev container includes everything pre-configured:

```bash
./start_dev_container.sh  # Reflection-enabled Clang, ready to go
```

This is what idiomatic C++ binding code will look like once compilers catch up.

---

## Try It

```bash
git clone https://github.com/FranciscoThiesen/mirror_bridge
cd mirror_bridge
./start_dev_container.sh

cd examples/mandelbrot-demo
bash build_mandelbrot_full.sh
```

You'll see:
- Benchmark comparisons across all runtimes
- Generated animation frames (the GIF above)
- The full binding code (it really is one line per language)

---

## The Code

**C++ (the only implementation):**

```cpp
struct Mandelbrot {
    int width, height, max_iter;
    double x_min, x_max, y_min, y_max;

    void set_viewport(double xmin, double xmax, double ymin, double ymax) {
        x_min = xmin; x_max = xmax; y_min = ymin; y_max = ymax;
    }

    std::vector<uint8_t> render_rgb() const {
        std::vector<uint8_t> pixels(width * height * 3);
        for (int py = 0; py < height; py++) {
            for (int px = 0; px < width; px++) {
                double x = x_min + (x_max - x_min) * px / width;
                double y = y_min + (y_max - y_min) * py / height;
                double t = escape_time(x, y);
                // ... color mapping ...
            }
        }
        return pixels;
    }

private:
    double escape_time(double cr, double ci) const {
        double zr = 0, zi = 0;
        int iter = 0;
        while (zr*zr + zi*zi <= 4.0 && iter < max_iter) {
            double temp = zr*zr - zi*zi + cr;
            zi = 2*zr*zi + ci;
            zr = temp;
            iter++;
        }
        return iter;  // + smooth coloring in real code
    }
};
```

**Python:**
```python
m = mandelbrot.Mandelbrot(800, 600, 256)
m.set_viewport(-0.75, -0.73, 0.1, 0.12)
pixels = m.render_rgb()  # bytes, 42× faster than pure Python
```

**Lua:**
```lua
local m = mandelbrot.Mandelbrot(800, 600, 256)
m:set_viewport(-0.75, -0.73, 0.1, 0.12)
local pixels = m:render_rgb()  -- 11× faster than pure Lua
```

**JavaScript:**
```javascript
const m = new mandelbrot.Mandelbrot(800, 600, 256);
m.set_viewport(-0.75, -0.73, 0.1, 0.12);
const pixels = m.render_rgb();  // Uint8Array, 1.5× faster than V8
```

Same computation. Same pixels. Three languages calling one C++ implementation.

---

## What's Next

The same reflection infrastructure that generates language bindings can power:

- **Serialization**: JSON, MessagePack, Protobuf from struct definitions
- **RPC**: gRPC service stubs generated from C++ classes
- **GUI**: Automatic property editors for your types
- **ORM**: Database schemas derived from struct layouts

One new C++ language feature, many interesting applications!

---

<small>

**Benchmark methodology:** Each implementation renders the same viewport 10 times; reported time is the median. "Pure" implementations use idiomatic code for each language (numpy-style vectorization for Python, standard loops for Lua/JS). Hardware: Apple M3 Max, single-threaded. Full benchmark code in the repo.

</small>

---

*[Mirror Bridge on GitHub](https://github.com/FranciscoThiesen/mirror_bridge)*
