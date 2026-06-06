---
layout: page
title: Projects
permalink: /projects/
---

<div class="projects">

<p class="projects-intro">Some things I've built outside work — mostly C++ pushed places it doesn't usually go: compile-time reflection, performance tooling, and algorithms that were supposed to stay theoretical. Everything lives on <a href="https://github.com/FranciscoThiesen">GitHub</a>.</p>

<h2>C++26 Reflection</h2>
<p class="section-lede">Making compile-time reflection do real work — bindings, serialization, hashing — before it ships in a standard compiler.</p>

<div class="project">
  <a class="thumb" href="https://github.com/FranciscoThiesen/mirror_bridge" aria-hidden="true" tabindex="-1"><img src="{{ site.baseurl }}/assets/images/projects/mirror_bridge.svg" alt="mirror_bridge — a suspension bridge carrying C++ across to Python, Lua and JavaScript" width="480" height="360"></a>
  <p class="desc"><a class="pname" href="https://github.com/FranciscoThiesen/mirror_bridge">mirror_bridge</a>: One <code>bind_class&lt;T&gt;()</code> call and C++26 reflection generates the entire Python, Lua, and JavaScript binding — no boilerplate, 3–5× faster dispatch than pybind11. Porting Open3D's point-cloud pipeline replaced 25,262 hand-written binding lines with 71. <a class="plink" href="{{ site.baseurl }}/Mirror-Bridge/">writeup</a> <a class="plink" href="{{ site.baseurl }}/Mirror-Bridge-Open3D-71-Lines/">open3d port</a> <a class="plink" href="{{ site.baseurl }}/Mirror-Bridge-Multi-Language/">benchmarks</a></p>
</div>

<div class="project">
  <a class="thumb" href="https://github.com/FranciscoThiesen/simdjson_reflection_paper" aria-hidden="true" tabindex="-1"><img src="{{ site.baseurl }}/assets/images/projects/simdjson_reflection.svg" alt="simdjson reflection — JSON braces split by a lightning bolt" width="480" height="360"></a>
  <p class="desc"><a class="pname" href="https://github.com/FranciscoThiesen/simdjson_reflection_paper">simdjson × reflection</a>: Compile-time reflection meets simdjson: JSON ⇄ native C++ structs at 3.5–7.8 GB/s with zero per-type code, 2–3× faster than yyjson and Rust's serde. Paper co-authored with Daniel Lemire. <a class="plink" href="{{ site.baseurl }}/Reflection-Based-Serialization/">writeup</a></p>
</div>

<div class="project">
  <a class="thumb" href="https://github.com/FranciscoThiesen/mirror_hash" aria-hidden="true" tabindex="-1"><img src="{{ site.baseurl }}/assets/images/projects/mirror_hash.svg" alt="mirror_hash — a fingerprint dissolving into hash bits" width="480" height="360"></a>
  <p class="desc"><a class="pname" href="https://github.com/FranciscoThiesen/mirror_hash">mirror_hash</a>: <code>std::hash</code> specializations generated from reflection instead of written by hand — a trivially-copyable struct hashes in under 2 ns, and the whole thing passes SMHasher. The byte-hashing core uses ARM64 AES instructions to beat rapidhash by up to 147% on 64 B–8 KB keys. <a class="plink" href="{{ site.baseurl }}/Mirror-Hash/">writeup</a></p>
</div>

<h2>Algorithms &amp; Performance</h2>

<div class="project">
  <a class="thumb" href="https://github.com/FranciscoThiesen/karger-klein-tarjan" aria-hidden="true" tabindex="-1"><img src="{{ site.baseurl }}/assets/images/projects/kkt_mst.svg" alt="karger-klein-tarjan — minimum spanning tree glowing inside a random graph" width="480" height="360"></a>
  <p class="desc"><a class="pname" href="https://github.com/FranciscoThiesen/karger-klein-tarjan">karger-klein-tarjan</a>: A working implementation of the expected-linear-time randomized MST algorithm — including the linear-time verification step (Hagerup) that most people skip. Found a bug in a 13-year-old paper's implementation along the way. <a class="plink" href="{{ site.baseurl }}/Linear-Time-MST/">writeup</a></p>
</div>

<div class="project">
  <a class="thumb" href="https://github.com/FranciscoThiesen/dimacs_2026" aria-hidden="true" tabindex="-1"><img src="{{ site.baseurl }}/assets/images/projects/dimacs_maxflow.svg" alt="dimacs_2026 — flow network as water pipes, saturated pipes running full" width="480" height="360"></a>
  <p class="desc"><a class="pname" href="https://github.com/FranciscoThiesen/dimacs_2026">dimacs_2026 — max-flow</a>: Five complementary max-flow solvers — push-relabel, pseudoflow, EIBFS, and two implicit-grid engines (one new to the literature) — built for the 13th DIMACS Implementation Challenge. 2.1× geometric-mean speedup over the reference solvers across 55 benchmark instances.</p>
</div>

<div class="project">
  <a class="thumb" href="https://github.com/FranciscoThiesen/hotpath" aria-hidden="true" tabindex="-1"><img src="{{ site.baseurl }}/assets/images/projects/hotpath.svg" alt="hotpath — flame graph with the hottest frame extracted into a C++ block" width="480" height="360"></a>
  <p class="desc"><a class="pname" href="https://github.com/FranciscoThiesen/hotpath">hotpath</a>: Profiles your Python, scores each hot function for transpilability, then has an LLM rewrite the winners in C++ — with generated tests on both sides of the boundary to prove the rewrite is faithful.</p>
</div>

<h2>Graphics, university era</h2>

<div class="project">
  <a class="thumb" href="https://github.com/FranciscoThiesen/global_illumination" aria-hidden="true" tabindex="-1"><img src="{{ site.baseurl }}/assets/images/projects/global_illumination.svg" alt="global_illumination — Cornell box with two spheres and bounced light" width="480" height="360"></a>
  <p class="desc"><a class="pname" href="https://github.com/FranciscoThiesen/global_illumination">global_illumination</a>: A Monte Carlo path tracer racing naive sampling against explicit light sampling on Cornell-box scenes — watching the same image converge from noise at 8 spp to smooth at 25,000.</p>
</div>

<div class="project">
  <a class="thumb" href="https://github.com/FranciscoThiesen/VolumeRendering" aria-hidden="true" tabindex="-1"><img src="{{ site.baseurl }}/assets/images/projects/volume_rendering.svg" alt="VolumeRendering — real skull render produced by the project from CT data" width="480" height="360"></a>
  <p class="desc"><a class="pname" href="https://github.com/FranciscoThiesen/VolumeRendering">VolumeRendering</a>: Volume renderer for 256³ CT scans — adaptive Simpson integration along each viewing ray, with transfer functions mapping density to color and opacity.</p>
</div>

<p class="projects-outro">More experiments, contest code, and half-finished ideas: <a href="https://github.com/FranciscoThiesen?tab=repositories">github.com/FranciscoThiesen</a> →</p>

</div>
