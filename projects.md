---
layout: page
title: Projects
permalink: /projects/
---

<div class="projects">

<p class="projects-intro">Some things I've built outside work — mostly C++ pushed places it doesn't usually go: compile-time reflection, other languages' compilers, and algorithms that were supposed to stay theoretical. Everything lives on <a href="https://github.com/FranciscoThiesen">GitHub</a>.</p>

<h2>C++26 Reflection</h2>
<p class="section-lede">Making compile-time reflection do real work — bindings, serialization, hashing — before it ships in a standard compiler.</p>

<div class="project">
  <a class="thumb" href="https://github.com/FranciscoThiesen/mirror_bridge" aria-hidden="true" tabindex="-1"><img src="{{ site.baseurl }}/assets/images/projects/mirror_bridge.svg" alt="mirror_bridge — one C++ struct bridged to Python, Lua and JavaScript" width="480" height="360"></a>
  <div class="info">
    <h3><a href="https://github.com/FranciscoThiesen/mirror_bridge">mirror_bridge</a></h3>
    <p class="project-meta">c++26 · 2025– · <a href="https://github.com/FranciscoThiesen/mirror_bridge">code</a> <a href="{{ site.baseurl }}/Mirror-Bridge/">writeup</a> <a href="{{ site.baseurl }}/Mirror-Bridge-Open3D-71-Lines/">open3d port</a> <a href="{{ site.baseurl }}/Mirror-Bridge-Multi-Language/">benchmarks</a></p>
    <p class="desc">One <code>bind_class&lt;T&gt;()</code> call and C++26 reflection generates the entire Python, Lua, and JavaScript binding — no boilerplate, 3–5× faster dispatch than pybind11. Porting Open3D's point-cloud pipeline replaced 25,262 hand-written binding lines with 71.</p>
  </div>
</div>

<div class="project">
  <a class="thumb" href="https://github.com/FranciscoThiesen/simdjson_reflection_paper" aria-hidden="true" tabindex="-1"><img src="{{ site.baseurl }}/assets/images/projects/simdjson_reflection.svg" alt="Reflection-based JSON for simdjson at 7.8 GB/s" width="480" height="360"></a>
  <div class="info">
    <h3><a href="https://github.com/FranciscoThiesen/simdjson_reflection_paper">simdjson × reflection</a></h3>
    <p class="project-meta">c++26 · 2024–2026 · <a href="https://github.com/FranciscoThiesen/simdjson_reflection_paper">paper</a> <a href="{{ site.baseurl }}/Reflection-Based-Serialization/">writeup</a></p>
    <p class="desc">Compile-time reflection meets simdjson: JSON ⇄ native C++ structs at 3.5–7.8 GB/s with zero per-type code, 2–3× faster than yyjson and Rust's serde. Paper co-authored with Daniel Lemire.</p>
  </div>
</div>

<div class="project">
  <a class="thumb" href="https://github.com/FranciscoThiesen/mirror_hash" aria-hidden="true" tabindex="-1"><img src="{{ site.baseurl }}/assets/images/projects/mirror_hash.svg" alt="mirror_hash — a struct hashed into an avalanche of bits" width="480" height="360"></a>
  <div class="info">
    <h3><a href="https://github.com/FranciscoThiesen/mirror_hash">mirror_hash</a></h3>
    <p class="project-meta">c++26 · 2025 · <a href="https://github.com/FranciscoThiesen/mirror_hash">code</a> <a href="{{ site.baseurl }}/Mirror-Hash/">writeup</a></p>
    <p class="desc"><code>std::hash</code> specializations generated from reflection instead of written by hand: a trivially-copyable struct hashes in under 2 ns, and the whole thing passes SMHasher. The byte-hashing core uses ARM64 AES instructions to beat rapidhash by up to 147% on 64 B–8 KB keys.</p>
  </div>
</div>

<h2>Compilers &amp; Performance</h2>

<div class="project">
  <a class="thumb" href="https://github.com/FranciscoThiesen/goff" aria-hidden="true" tabindex="-1"><img src="{{ site.baseurl }}/assets/images/projects/goff.svg" alt="goff — Go SSA lowered to LLVM -O3, 3.94x on dot products" width="480" height="360"></a>
  <div class="info">
    <h3><a href="https://github.com/FranciscoThiesen/goff">goff</a></h3>
    <p class="project-meta">go + llvm · 2026 · <a href="https://github.com/FranciscoThiesen/goff">code</a></p>
    <p class="desc">A drop-in <code>go build</code> replacement that lifts Go's SSA into LLVM IR, runs the full -O3 pipeline, and links the result back through the standard toolchain — 2–4× on numeric kernels the stock compiler won't vectorize.</p>
  </div>
</div>

<div class="project">
  <a class="thumb" href="https://github.com/FranciscoThiesen/hotpath" aria-hidden="true" tabindex="-1"><img src="{{ site.baseurl }}/assets/images/projects/hotpath.svg" alt="hotpath — flame graph with the hot frame transpiled to C++" width="480" height="360"></a>
  <div class="info">
    <h3><a href="https://github.com/FranciscoThiesen/hotpath">hotpath</a></h3>
    <p class="project-meta">python → c++ · 2025 · <a href="https://github.com/FranciscoThiesen/hotpath">code</a></p>
    <p class="desc">Profiles your Python, scores each hot function for transpilability, then has an LLM rewrite the winners in C++ — with generated tests on both sides of the boundary to prove the rewrite is faithful.</p>
  </div>
</div>

<div class="project">
  <a class="thumb" href="https://github.com/FranciscoThiesen/simdex" aria-hidden="true" tabindex="-1"><img src="{{ site.baseurl }}/assets/images/projects/simdex.svg" alt="simdex — SIMD lanes scanning the predicted range of a sorted array" width="480" height="360"></a>
  <div class="info">
    <h3><a href="https://github.com/FranciscoThiesen/simdex">simdex</a></h3>
    <p class="project-meta">c++20 · 2026 · <a href="https://github.com/FranciscoThiesen/simdex">code</a></p>
    <p class="desc">The "last mile" of a learned index — the final 16–256-element search the model hands off — vectorized three ways: SIMD linear scan, k-ary search, and a cache-friendly Eytzinger layout, for both AVX2 and NEON.</p>
  </div>
</div>

<h2>Algorithm Engineering</h2>

<div class="project">
  <a class="thumb" href="https://github.com/FranciscoThiesen/karger-klein-tarjan" aria-hidden="true" tabindex="-1"><img src="{{ site.baseurl }}/assets/images/projects/kkt_mst.svg" alt="karger-klein-tarjan — minimum spanning tree glowing inside a random graph" width="480" height="360"></a>
  <div class="info">
    <h3><a href="https://github.com/FranciscoThiesen/karger-klein-tarjan">karger-klein-tarjan</a></h3>
    <p class="project-meta">c++17 · 2020–2022 · <a href="https://github.com/FranciscoThiesen/karger-klein-tarjan">code</a> <a href="{{ site.baseurl }}/Linear-Time-MST/">writeup</a></p>
    <p class="desc">A working implementation of the expected-linear-time randomized MST algorithm — including the linear-time verification step (Hagerup) that most people skip. Found a bug in a 13-year-old paper's implementation along the way.</p>
  </div>
</div>

<div class="project">
  <a class="thumb" href="https://github.com/FranciscoThiesen/dimacs_2026" aria-hidden="true" tabindex="-1"><img src="{{ site.baseurl }}/assets/images/projects/dimacs_maxflow.svg" alt="dimacs_2026 — flow network with the saturated min-cut edges highlighted" width="480" height="360"></a>
  <div class="info">
    <h3><a href="https://github.com/FranciscoThiesen/dimacs_2026">dimacs_2026 — max-flow</a></h3>
    <p class="project-meta">c++ · 2026 · <a href="https://github.com/FranciscoThiesen/dimacs_2026">code</a></p>
    <p class="desc">Five complementary max-flow solvers — push-relabel, pseudoflow, EIBFS, and two implicit-grid engines (one new to the literature) — built for the 13th DIMACS Implementation Challenge. 2.1× geometric-mean speedup over the reference solvers across 55 benchmark instances.</p>
  </div>
</div>

<h2>AI Experiments</h2>

<div class="project">
  <a class="thumb" href="https://github.com/FranciscoThiesen/llm_memory" aria-hidden="true" tabindex="-1"><img src="{{ site.baseurl }}/assets/images/projects/llm_memory.svg" alt="llm_memory — index/summary/details hierarchy and accuracy-vs-tokens chart" width="480" height="360"></a>
  <div class="info">
    <h3><a href="https://github.com/FranciscoThiesen/llm_memory">llm_memory</a></h3>
    <p class="project-meta">python · 2025–2026 · <a href="https://github.com/FranciscoThiesen/llm_memory">code</a></p>
    <p class="desc">A benchmark for LLM memory architectures: flat context vs. two- and three-level hierarchies, measured on accuracy, token cost, and latency across multi-hop, temporal, and contradiction-heavy scenarios.</p>
  </div>
</div>

<div class="project">
  <a class="thumb" href="https://github.com/FranciscoThiesen/am_i_wrong" aria-hidden="true" tabindex="-1"><img src="{{ site.baseurl }}/assets/images/projects/am_i_wrong.svg" alt="am_i_wrong — a news headline with the typo boxed in red" width="480" height="360"></a>
  <div class="info">
    <h3><a href="https://github.com/FranciscoThiesen/am_i_wrong">am_i_wrong</a></h3>
    <p class="project-meta">python · 2025 · <a href="https://github.com/FranciscoThiesen/am_i_wrong">code</a></p>
    <p class="desc">Points a local LLM (Ollama/vLLM) at live news headlines and proofreads them in context — proper nouns and slang excluded — rendering reports with every typo boxed in red.</p>
  </div>
</div>

<h2>Graphics, university era</h2>

<div class="project">
  <a class="thumb" href="https://github.com/FranciscoThiesen/global_illumination" aria-hidden="true" tabindex="-1"><img src="{{ site.baseurl }}/assets/images/projects/global_illumination.svg" alt="global_illumination — Cornell box with two spheres and bounced light" width="480" height="360"></a>
  <div class="info">
    <h3><a href="https://github.com/FranciscoThiesen/global_illumination">global_illumination</a></h3>
    <p class="project-meta">c++ · 2019 · <a href="https://github.com/FranciscoThiesen/global_illumination">code</a></p>
    <p class="desc">A Monte Carlo path tracer racing naive sampling against explicit light sampling on Cornell-box scenes — watching the same image converge from noise at 8 spp to smooth at 25,000.</p>
  </div>
</div>

<div class="project">
  <a class="thumb" href="https://github.com/FranciscoThiesen/VolumeRendering" aria-hidden="true" tabindex="-1"><img src="{{ site.baseurl }}/assets/images/projects/volume_rendering.svg" alt="VolumeRendering — scan rays sampling adaptively through a CT head" width="480" height="360"></a>
  <div class="info">
    <h3><a href="https://github.com/FranciscoThiesen/VolumeRendering">VolumeRendering</a></h3>
    <p class="project-meta">c++ · 2018 · <a href="https://github.com/FranciscoThiesen/VolumeRendering">code</a></p>
    <p class="desc">Volume renderer for 256³ CT scans: adaptive Simpson integration along each viewing ray, with transfer functions mapping density to color and opacity.</p>
  </div>
</div>

<p class="projects-outro">More experiments, contest code, and half-finished ideas: <a href="https://github.com/FranciscoThiesen?tab=repositories">github.com/FranciscoThiesen</a> →</p>

</div>
