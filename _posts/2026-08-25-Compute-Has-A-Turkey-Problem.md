---
layout: post
title: "Compute has a turkey problem"
tags: [Modern C++, Python, Bindings, Reflection, C++26, Performance, Cloud]
---

*Written with LLM assistance. Details at the end.*

![Reported compute price increases in 2026: Hetzner +40%, Intel CPUs +10%, TSMC wafers +10%, OVHcloud +8%](/assets/images/cpu_thanksgiving/p8_prices.png)

This chart is something we are not used to seeing. The price of
compute has always trended lower, while the hardware kept getting
more efficient at the same time. Whole careers, languages and
architectural philosophies were built on the assumption that CPUs
are abundant and will cost less next year.

The evidence that the assumption no longer holds, at least for a
while:
[Hetzner citing component costs][hetzner],
[Intel raising CPU prices three years after launch][intel-hike],
[TSMC lifting every leading node][tsmc-hike], and
[OVHcloud telling customers more is coming][ovh]. Behind it, the
hyperscalers are pouring [hundreds of billions a year][capex-2026]
into new datacenters. Every one of those machines bids for the same wafers,
power, and rack space your next CPU instance comes from.

Things were not always this way. The computer that landed
people on the Moon had about 4 KB of RAM and executed roughly 85,000
instructions per second. Its source code, printed out, made a stack
of paper as tall as Margaret Hamilton, who led the team that wrote
it. Cycles weren't a budget line, they were the design constraint.
During Apollo 11's final descent, a rendezvous radar left in the
wrong switch position started flooding the guidance
computer with counter interrupts, silently stealing roughly 15% of
its cycles, and with Armstrong and Aldrin minutes from the surface
the machine ran out of headroom. It did not crash. The famous 1201
and 1202 alarms it threw were not the failure. They were the
software's designed response to overload firing exactly as built:
restart, shed every low-priority job, keep the landing-critical ones
running on the cycles that remained. Mission control called "go" on
the alarms and Eagle kept flying. Don Eyles, who wrote the
lunar-landing software in Hamilton's group at the MIT
Instrumentation Laboratory, tells the story firsthand in
[Tales from the Lunar Module Guidance Computer][eyles]. That is
what engineering looks like when every cycle is valuable.

![Left: Margaret Hamilton beside the printed Apollo Guidance Computer listings, 1969. Right: the LUMINARY source lines that raise the 1201 and 1202 alarms, and the Apollo 11 transcript of the moment they fired](/assets/images/cpu_thanksgiving/apollo_pair.png)
*Left: Hamilton beside the listings of the software her team wrote.
Right: [the lines in the lunar module's flight software][luminary]
that raise the 1201 and 1202 alarms, and [the transcript][alsj] of
the moment they fired.*

At some point, cycles became abundant and we built an entire
engineering culture around that abundance. Niklaus Wirth was
already complaining about it in 1995 in [A Plea for Lean Software][wirth], and "software is
getting slower more rapidly than hardware becomes faster" now has a
name, Wirth's law. The modern version of the rant is a genre of its
own ([Muratori's is the classic][muratori]). We stopped asking what a
cycle costs because the answer almost never mattered.

Interpreted languages in the hot path, a hundred-fold overhead,
whatever: the hardware will absorb it. It was even the rational
call for some scenarios. Engineer time was the scarce input. CPU
time was not.

## CPU Thanksgiving

In The Black Swan, [Taleb tells the story of a turkey][taleb] that
is fed every day for a thousand days. Plot its well-being and you
get a smooth rising line, and every new day of data makes the
extrapolation look safer. Then, a few days before Thanksgiving, the
process generating the data changes, and the line does something
the previous thousand points gave no warning of. That is the turkey
problem. It is about extrapolation: a long, consistent trend
describes the past of a process, not whether the process will keep
running. Twenty years of always-cheaper compute is a long,
consistent trend. It was never a law of nature, just the output of
fab economics and demand sitting comfortably below supply, and the
demand side just changed.

![The chart from The Black Swan: 1000 and 1 Days in the Life of a Thanksgiving Turkey, well-being rising for a thousand days then collapsing at day 1,001](/assets/images/cpu_thanksgiving/taleb_turkey.png)
*The turkey's data, from Nassim Taleb's The Black Swan. A thousand
clean data points, one broken process.*

Then, in August 2026, the argument staged itself on X in a single
exchange. Chamath posted the cost-of-computation curve, 18 orders of
magnitude in 80 years, calling it [the one chart that sits upstream
of everything our species has accomplished][chamath-post], as long
as we keep it going. Musk's reply was a cartoon turkey presenting
its own weight chart, captioned "I see no reason why excellent
growth shouldn't continue," with three words of commentary:
["Nothing is guaranteed"][musk-post].

![The exchange: Chamath's cost-of-computation chart next to the turkey-weight cartoon Musk replied with, captioned "Nothing is guaranteed"](/assets/images/cpu_thanksgiving/turkey_pair.png)

## Day 1,001

Behind the prices, the physical constraints are blunt: North
American datacenter vacancy has sat at **~1%** for three years
([CBRE][cbre]), grid transformers quote 128-week lead times, and
Microsoft's CFO said it plainly: *"We have been short of power and
space."* The capacity being built through 2028 is already spoken
for. Whatever this turns out to be, it does not resolve next
quarter.

## The tax

How much CPU is actually on the table? For compute-heavy loops, C++
is routinely [one to two orders of magnitude faster than
Python][pereira]. Even I/O-heavy services, where the interpreter
spends most of its time waiting on something else, [tend to land
somewhere between 2x and 5x][greencoding]. Either way it is a cost
story: save the CPU-seconds and the dollars follow.

And CPU-seconds are servers. Meta's fleet profiler found a
**one-character** C++ fix (a copy that should have been a reference)
worth an estimated **15,000 servers a year** of capacity
([Strobelight, in the section "The Biggest Ampersand"][strobelight]).

So here is a practical question: when did you last profile your
compute-heaviest workloads? An afternoon with a profiler usually
turns up a shortlist of Python loops responsible for a surprising
share of the bill, and those are the natural candidates for a C++
port. What kept shortlists like that unacted on for years was not
the measurement but what came after it. The rewrite carried the
cost of the bridge: hand-written pybind11 glue, a second build
system, stubs that drift, a maintenance surface *nobody wants to
own*. The cycles were cheaper than the glue.

That trade deserves a rerun, because the bridge side of it quietly
collapsed. C++26 reflection lets a binding layer *derive* the glue
from the class itself: [mirror_bridge][repo] turns a C++ struct
into a Python module with one command and zero hand-written binding
code. The bridge is now almost free!

## Try it

Here is the whole workflow on a header the binder has never seen,
with the timing of the real run:

![Animated terminal: cat a plain C++ header, run one mirror_bridge command, then import the module from Python and call it](/assets/images/cpu_thanksgiving/mb_terminal.svg)

The command is one line:

```bash
mirror_bridge generate include/ --module vec3 --lang python --release-gil --stubs
```

A quick tour of what that line does. `generate include/` scans the
directory and discovers every class in it, so there is no list of
things to bind. `--module vec3` names the Python module you will
import, and `--lang python` picks the target (Lua and JavaScript are
the other options). `--release-gil` makes the generated methods drop
Python's global interpreter lock while the C++ body runs, so a long
native call no longer blocks your other Python threads. `--stubs`
writes a `vec3.pyi` next to the module with the exact signatures and
parameter names taken from the C++ class, which is what gives you
autocomplete and type checking in the editor.

The maintenance story is the part that usually kills these tools,
so it deserves a plain answer. The C++ you own is an ordinary
header. No macros, no annotations, no wrapper classes, no second
build system. When the class changes, you rerun the same command
and the module and its type stubs are regenerated from the class
itself, so there is nothing to keep in sync by hand.

How much you get back depends on the shape of the hot path, and I
have written up real cases at both ends: porting Open3D
([25,262 hand-written binding lines to 71][open3d]) and racing the
generated bindings against [V8, PyPy, and LuaJIT][multilang].

## The point

None of this is an argument to rewrite your service in C++. It's an
argument that the old equilibrium, *leave it in Python, the rewrite
isn't worth the bridge*, was priced against hardware that got cheaper
every year and glue that cost weeks. Both inputs changed. The bridge
is now one command on stock GCC 16.1 with `-std=c++26 -freflection`
(reflection is in C++26, this is standard C++). Move the
hundred-odd lines that burn 80% of your cycles and leave everything
else where your team is productive.

## Disclaimer

This post was written with LLM assistance. The opinions and the
mistakes are mine.

[wirth]: https://cr.yp.to/bib/1995/wirth.pdf
[eyles]: https://www.doneyles.com/LM/Tales.html
[luminary]: https://github.com/chrislgarry/Apollo-11/blob/master/Luminary099/EXECUTIVE.agc
[alsj]: https://www.nasa.gov/wp-content/uploads/static/history/alsj/a11/a11.1201-pa.html
[taleb]: https://en.wikipedia.org/wiki/The_Black_Swan:_The_Impact_of_the_Highly_Improbable
[chamath-post]: https://x.com/chamath/status/2090076460866244855
[musk-post]: https://x.com/elonmusk/status/2090138513421512999
[capex-2026]: https://techblog.comsoc.org/2025/12/22/hyperscaler-capex-600-bn-in-2026-a-36-increase-over-2025-while-global-spending-on-cloud-infrastructure-services-skyrockets/
[muratori]: https://www.computerenhance.com/p/clean-code-horrible-performance
[cbre]: https://www.cbre.com/insights/reports/global-data-center-trends-2026
[hetzner]: https://www.datacenterdynamics.com/en/news/german-cloud-firm-hetzner-hikes-prices-by-up-to-50-percent-due-to-drastic-component-price-increases/
[intel-hike]: https://www.tomshardware.com/pc-components/cpus/intel-reportedly-raising-prices-on-ever-popular-raptor-lake-chips-outdated-cpus-to-get-over-10-percent-price-hike-due-to-disinterest-in-ai-processors
[tsmc-hike]: https://www.tomshardware.com/tech-industry/semiconductors/tsmc-is-reportedly-hiking-prices-for-all-advanced-nodes-accounting-for-74-percent-of-the-companys-wafer-business-nvidia-amd-apple-qualcomm-and-others-will-face-higher-wafer-costs
[ovh]: https://www.theregister.com/2025/11/24/ovh_cloud_price_rise_prediction/
[pereira]: https://www.researchgate.net/publication/348523926_Ranking_programming_languages_by_energy_efficiency
[greencoding]: https://www.green-coding.io/case-studies/energy-efficiency-python/
[strobelight]: https://engineering.fb.com/2025/01/21/production-engineering/strobelight-a-profiling-service-built-on-open-source-technology/#:~:text=The%20Biggest%20Ampersand
[open3d]: https://chico.dev/Mirror-Bridge-Open3D-71-Lines/
[multilang]: https://chico.dev/Mirror-Bridge-Multi-Language/
[repo]: https://github.com/FranciscoThiesen/mirror_bridge
