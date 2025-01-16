+++
title = 'Profiling Waveform Rendering in KDenlive'
date = 2025-01-16T09:58:52+01:00
draft = true
+++

One of the ways to identify the root cause of the performance problems is using a [profiler](https://en.wikipedia.org/wiki/Profiling_(computer_programming)). A profiler instruments the program and measures statistics like the amount of function calls. What we're after here is to get a report on how much time is spent in which functions.

Profiling for performance in a complex program like Kdenlive is a bit tricky: to obtain representative results, one has to be careful on the profiling approach. It would be infeasible to use traditional event-based profilers, as they would slow down the execution too much. Since we suspect that a lot of time is spent doing i/o, we would skew the results if we altered the program's execution speed too much.

So, we turn ourselves to sampling profilers. Instead of instrumenting every single function call in a target program, these profilers take "snapshots" of their call stack at regular intervals. I will be using [perf](https://perfwiki.github.io/main/) for profiling and [hotspot](https://github.com/KDAB/hotspot) for visualisation, but many other tools are available.

### Profiling the audio summary generation in Kdenlive

- Insert start and stop collecting signals in source
- Build with optimizations and debug symbols
- perf record, use high frequency
- run kdenlive and load a preferably long audio file
- use hotspot
- see that most of the time is spent doing sth other than decoding or computing peaks :@
