# AOT cache + SkipTier2IfPossible vs startup warm-up: JIT and GC reports

Experiments on a Spring Boot API (JDK 25, AOT cache, 1 CPU, SerialGC) under a 2,000 req/s burst, comparing
`-XX:+SkipTier2IfPossible` with an idle period against a startup warm-up.

1. [`20261119-jit-delay/REPORT.md`](20261119-jit-delay/REPORT.md): 18 runs. JIT timelines and compile counts,
   and why skip sometimes collapses (inferred: the first GC at the burst onset).
2. [`20261119-jit-5s/REPORT.md`](20261119-jit-5s/REPORT.md): 52 runs with JFR GC and deopt events. Skip's first
   two GCs (37–82 ms each) land in the burst; warm-up passes 23 of 26 runs, skip 3 of 26.

Each report's `graphs/` folder holds its images. Raw run data and the experiment scripts referenced in the
reports' "Files" sections are not included in this repository.
