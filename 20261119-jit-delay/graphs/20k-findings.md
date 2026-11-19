# 20k cache: JIT timeline counts and why `skip` collapses

Runs: `20k/skip-1` (94.11% on time), `20k/skip-2` (79.39%), `20k/warm-1` (99.90%), `20k/warm-2` (98.27%).
Graphs: `graphs/20k-<run>-jit-cpu.png`, `graphs/20k-<run>-queues.png`. Counts: `graphs/20k-<run>.json`, from `tools/timeline.py`, unchanged.
Times in the burst sections are seconds relative to the burst start (t = 0). The burst start is fitted (see "Alignment").

## 1. Counts (tools/timeline.py)

Idle = JVM start to traffic start (80.0 s for skip, 82.8 s for warm). Traffic = traffic start to the end of the burst (timeline marker). "Comp" means compilations and "meth" means distinct methods.

| run | idle C1 comp / meth | idle C2 comp / meth | traffic C1 comp / meth | traffic C2 comp / meth |
|---|---|---|---|---|
| skip-1 | 4789 / 4625 | 1423 / 1414 | 436 / 417 | 350 / 343 |
| skip-2 | 4817 / 4663 | 1427 / 1417 | 486 / 460 | 352 / 347 |
| warm-1 | 7206 / 6717 | 2435 / 2344 | 138 / 136 | 29 / 29 |
| warm-2 | 7199 / 6727 | 2408 / 2318 | 157 / 153 | 65 / 64 |

Traffic compilations by kind (compilations / methods):

| run | C1 new | C1 level_raise | C1 recompile | C2 new | C2 level_raise | C2 recompile |
|---|---|---|---|---|---|---|
| skip-1 | 382 / 381 | 41 / 40 | 13 / 13 | 128 / 128 | 214 / 214 | 8 / 8 |
| skip-2 | 380 / 379 | 92 / 91 | 14 / 14 | 126 / 126 | 220 / 220 | 6 / 5 |
| warm-1 | 110 / 110 | 19 / 19 | 9 / 8 | 5 / 5 | 20 / 20 | 4 / 4 |
| warm-2 | 123 / 123 | 25 / 25 | 9 / 7 | 7 / 7 | 53 / 53 | 5 / 5 |

`invalidated` (CompilerStatistics.invalidatedCount) is 0 at every marker in all four runs, so it gives no deopt signal here.
Timeline markers (s since JVM start), healthy / idle_end / burst_start: skip-1 10.7 / 80.0 / 174.4; skip-2 10.8 / 80.0 / 172.5; warm-1 72.8 / 82.8 / 176.4; warm-2 72.7 / 82.8 / 175.9.

The traffic window includes about 90 s of functional checks before the load test. Most skip C2 work in the burst itself is level_raise of methods sitting at C1 level 2 (section 3).

### Sanity checks of the graphs and the script

- All 8 PNGs show the healthy, end-of-idle and burst-start lines in the right places. Skip runs show the startup C2 spike (~1.0 CPU at 12-15 s), small C2 blips during the functional checks, and C2 at ~0.4-0.47 CPU for ~7-8 s after the burst start, with a C2 queue of ~100-115. Warm runs show warm-up (C2 ~0.35, other ~0.65 until ~61 s) and almost no JIT in the burst.
- JFR normalization is correct. Burst C2 CPU-seconds (10 s from the marker), JFR ThreadCPULoad vs host `threadcpu.txt` column 2: skip-1 3.09 vs 3.10, skip-2 3.31 vs 3.32, warm-1 0.12 vs 0.12, warm-2 0.54 vs 0.52. Per second they agree within ~0.03. 1.0 = one CPU.
- Startup artifact, a JFR issue and not a timeline.py bug: the "all other threads" curve reaches 1.5-1.6 CPU at ~5-10 s, which is impossible on 1 CPU. The cause is the `DestroyJavaVM` thread (the renamed main thread after `main()` returns), which reports a clamped 1.0 for 3 consecutive JFR periods in every run. Ignore "other" before ~11 s.
- Bucketing: timeline.py adds a ThreadCPULoad event at time t to bucket `int(t)`, but the event measures (t-1, t]. Curves are therefore shifted ~0.5 s late. Only the second containing the main-thread exit gets two ticks. Seconds with no JFR tick (fully idle) are skipped in the plot. These are cosmetic, so I left them unchanged.
- The `burst_start` marker is 0.1-0.45 s early. It is defined as the start of the first 1 s window with ≥0.3 CPU, so it is earlier than the fitted onset: skip-1 174.42 vs 174.51, skip-2 172.51 vs 172.88, warm-1 176.37 vs 176.54, warm-2 175.85 vs 176.03. This is fine for counts. For sub-second work I used the fitted start.
- Classification quirk, not a bug: 6 of skip's 13-14 "C1 recompile" entries are level 1 compiles of trivial methods that were at level 2/3 before (e.g. `HeapByteBuffer::isReadOnly`, `SocketChannel::validOps`). That is the tiered policy moving trivial methods to level 1, not a deopt. The other C1 recompiles are mostly level 3 after earlier level 4, which is a real deopt fallback (Jackson parse methods at -4.9 s, when the mixed-payload load starts).

### Alignment (how burst-relative times were obtained)

The checker has no absolute timestamps. I fitted a step model of the request schedule (100 req/s, then 2,000 req/s) to the host sampler's non-compiler CPU (0.5 s samples), using the onset edge. The end edge is unusable in collapsed runs because the server keeps working off abandoned requests after the load stops; in `10k/skip-2` that moved a full-burst fit by 1.5 s.

The fit is checked against JFR in the two passing skip runs. There the client-side stall at the first GC ends at -1.549 s (20k) and -1.186 s (40k), and JFR's first GC-notification compile starts at -1.550 s and -1.22 s. So alignment is good to about ±0.05 s.

## 2. Burst onset, measured

Full tables are in the appendix. Condensed:

**C2 backlog and CPU split (JFR per thread, 1 s).** In both skip runs the C2 queue goes from 0 to ~105-115 within the first second. Each skip burst starts 184-197 C2 compilations, about 7.1-7.6 s of compile wall time. Those include 159-167 compilations of 158-165 request-path methods (§3). The single C2 thread then takes 0.37-0.47 CPU until the queue empties at +7.0 s (skip-1) and +7.8 s (skip-2). Total API CPU is 0.93-1.01, so the container is saturated (cgroup `cpustat` also shows 1.00).

The non-compiler threads get ~0.55: the virtual-thread carrier `ForkJoinPool-1-worker-1` gets 0.31-0.35. It is the only carrier, because parallelism = 1 CPU. Warm runs do the same burst with carrier 0.29-0.33 and total 0.55-0.61. So the request path needs about 0.55 CPU at 2,000 req/s. In skip runs it gets about 0.55, with essentially no headroom.

At 0.5 s host resolution, every collapsed skip run (10k-1, 10k-2, 20k-2, 40k-2) is at 0.86-1.02 CPU within the first 0.5 s. The "0.44 CPU in the first second" figure comes from a 1 s window that straddles the onset.

**Latency (checker submissions, ms timestamps).**
- warm-1 / warm-2: p50 3-7 ms throughout. Short dips: warm-1 +3.5 s (16 misses); warm-2 +4.15-4.30 s (347 misses, a ~0.2 s stall of POST completions).
- skip-1: the first 50 ms of the burst already shows max 31-34 ms and 14 misses, even with no GC. Then p50 6-11 ms (on time) with C2 100% busy. A 0.53 s stall of POST completions at +3.43 s causes 1,199 misses (+3.25 to +4.0). It recovers by +4.25.
- skip-2: from the first 25 ms of the burst, 49 of 50 requests miss. Every request misses (p50 35.7 ms = client timeout) until +2.25, except a 163-request window at +1.5. From +2.25 it is on time (p50 7-11 ms), with small dips at +4.0 (57) and +5.0-5.25 (6).

**First GC (inferred from the first compile of the JMX GC-notification code).** The first compile of `GarbageCollectionNotificationInfo`/`GcInfo`/`MemoryUsage` code (~55 C1 compiles in 40 ms) marks the first time GC notifications run, which is inferred to be the JVM's first GC. JFR here has no GC events.

In passing skip runs the client sees a clean stop-the-world signature there. At 100 req/s, every request is released at the same instant, after a server stall of ~65 ms (20k skip-1, -1.616 to -1.549) and ~90 ms (40k skip-1). The release coincides with the notification compile.

Over all 6 skip runs, plus 4 warm runs (from 10k/40k, read-only):

| run | on time | first GC-notification compile vs burst start |
|---|---|---|
| 10k skip-1 | 4.1% | +0.15 s |
| 10k skip-2 | 4.0% | +0.06 s |
| 20k skip-2 | 79.4% | +0.12 s |
| 40k skip-2 | 3.7% | +0.07 s |
| 20k skip-1 | 94.1% | -1.54 s (100 req/s phase) |
| 40k skip-1 | 98.6% | -1.22 s (100 req/s phase) |
| warm (10k, 20k, 40k) | 98.3-99.99% | about -160 s (during warm-up) |

All 4 collapsed skip runs had their first GC within 0.15 s after the burst started. Both skip runs that passed had it 1.2-1.5 s before the burst, at 100 req/s, where a ~70-90 ms pause costs 3-5 requests.

Skip runs don't send traffic during idle, so eden first fills at about the same point of the checker's script, near the 100 → 2,000 req/s switch. Whether it fills just before the switch or just after is close to random.

**Connection churn (measured).** The checker closes its connection after every timeout (`exercise_harness.py` `send`), so after a miss the next request on that worker reconnects. Tomcat's Acceptor thread uses 0.05-0.09 CPU in every collapsed second (20k skip-2 +0.7..+2.8; 10k-1 and 40k-2 throughout; skip-1 at its +3.4 dip). It uses 0.00-0.01 in every healthy second. `SelectorImpl::register` gets its first-ever C1 compile at +0.013 s in skip-2.

**Server backlog (measured with the checker's GET lookups during the load).**
- 20k skip-2: lookups took 90, 55, 121, 38, 51, 77 ms during +0.09..+1.69, then 20, 12, 6 ms from +2.0. The backlog stayed bounded at ~40-120 ms and then drained.
- 40k skip-2: 25 → 152 → 189 → 279 → 2,597 → 1,353 → 5,026 ms.
- 10k skip-1: 133 → 219 → 407 → 950 → 1,602 ms, and still ~1.3 s at +10 s.
- Never-recovered runs also show 0.29-0.97 CPU-s of non-Java-thread CPU over the burst (host "other" minus JFR Java threads; with SerialGC on 1 CPU this is mostly the VM Thread doing GC). Every other run, including 20k skip-2, shows -0.02 to 0.02.
- 20k skip-1: one lookup took 422 ms at +3.46 (its dip). All other lookups were ≤18 ms.

## 3. Request-path methods not yet C2 at burst start

These are methods C2-compiled in [0, +12 s) in packages com.example.events, org.apache.tomcat/catalina/coyote, com.fasterxml.jackson, org.postgresql, com.zaxxer.hikari, java.util.concurrent, sun.nio.ch, java.nio, VirtualThread/Continuation, spring jdbc/web, jakarta.servlet, java.lang.invoke.

| | skip-1 | skip-2 | warm-1 | warm-2 |
|---|---|---|---|---|
| request-path C2 compilations (methods) | 159 (158) | 167 (165) | 9 (9) | 43 (43) |
| prior level: L2 / L3 / never compiled / L4 (recompile) | 143 / 7 / 4 / 5 | 145 / 13 / 5 / 4 | 4 / 1 / 1 / 3 | 34 / 4 / 0 / 5 |
| C2 installed by +1 / +2 / +3 / +4 / +5 / +6 / +7 / +8 s | 14 / 28 / 32 / 42 / 84 / 126 / 157 / 158 | 16 / 24 / 26 / 49 / 79 / 100 / 149 / 159 | 2 / 4 / 8 / 9 / 9 / 9 / 9 / 9 | 32 / 34 / 37 / 37 / 42 / 42 / 42 / 42 |
| request path fully C2 (last install) | +8.06 s | +10.64 s (95% by +8) | +3.24 s | +1.57 s (plus 1 at +11) |
| C2 queue first empty | +7.0 s | +7.8 s | never > 0 | +2.3 s |

By package, in both skip runs: Jackson 27-29, sun.nio.ch 21-24, coyote 22, catalina 18, tomcat 17, postgresql 14, java.nio 6-11, j.u.concurrent 9-11, VirtualThread 9, com.example.events 5, others ≤3.

In skip runs, ~90% of these methods were running C1 level-2 code at burst start (SkipTier2IfPossible), not interpreted code.

Compile wall times: median 9 ms, p90 86-95 ms, max ~1 s. The longest request-path compiles in both skip runs:
- `AbstractProtocol$ConnectionHandler::process`: 845 ms in skip-1 (+1.79..+2.64), 1,017 ms in skip-2 (+1.55..+2.56)
- `PgArrays::setTextArray`: 628 / 520 ms
- `QueryExecutorImpl::processResults`: 556 / 480 ms
- `Http11Processor::prepareResponse`: 373 / 444 ms
- `SocketWrapperBase::writeBlocking`: 258 / 333 ms
- `DelayScheduler::loop` OSR, ~180-210 ms each, compiled 2-3 times

skip-1 kept up from +0.25 to +3.25 with only 14-32 of 159 methods C2-compiled. Its carrier CPU (0.34) was within ~0.02 of warm's. So level-2 code is not dramatically more expensive; the problem is the CPU taken by C2.

**Recompiles during traffic (C2, "recompile").**
- skip-1: `AuthenticatorBase::invoke` (-74.6 s); `Nodes$FixedNodeBuilder::accept` (-0.6); Jackson `nextFieldName` 95 ms / `_checkDup` / `_parseName` (+1.05-1.15); `ForkJoinPool::tryTrim` (+4.4); `ArrayList$ArrayListSpliterator::forEachRemaining` (+5.8); `DelayScheduler::loop` OSR 206 ms (+7.8).
- skip-2: `AuthenticatorBase::invoke` (-71.9); `FixedNodeBuilder::accept` (+0.08); `nextFieldName` 127 ms (+0.43); `tryTrim` (+5.8); `DelayScheduler::loop` OSR 179 + 206 ms (+7.7, +8.2).
- warm-1: `AppendValidator::parseAppend` (-0.45, 84 ms), `nextFieldName` (+0.37), `ParserBase::close` (+1.06), `tryTrim` (+2.3).
- warm-2: `parseAppend` 633 ms (+0.03), `nextFieldName`/`_checkDup`/`_parseName` (+0.66-0.76), `ParserBase::close` (+1.45).

The Jackson/`parseAppend` recompiles follow deopts at ~-4.9 s, when the mixed-payload load phase starts, in all four runs. They are small (≤0.13 s of C2 in skip) and do not explain the collapse. warm-2 compiled `parseAppend` for 633 ms at the burst onset and lost only 3 requests.

## 4. Mechanism hypothesis

**Measured:**
1. In skip runs the single C2 thread competes with the single virtual-thread carrier for 1 CPU during the first ~7-8 s of the burst. CPU is 0.93-1.01 used. Request processing gets ~0.55 CPU, about what it needs (warm: 0.55-0.6 total, carrier ~0.32). Warm runs have ~45% CPU headroom in the burst; skip runs have about none.
2. The burst onset itself is a latency spike even without GC (skip-1: 31-34 ms max in the first 50 ms), because the client opens new connections and C2 starts.
3. In all 4 collapsed skip runs across 3 caches, the first GC lands 0.06-0.15 s into the burst. In both passing skip runs it lands 1.2-1.5 s earlier, at 100 req/s. That first GC stops the server for ~65-90 ms (measured at 100 req/s).
4. Once requests miss 35 ms, the client drops connections, and the Acceptor shows 0.05-0.09 CPU of reconnect work.
5. In runs that never recovered, the server backlog grows to seconds and non-Java (GC) CPU rises. In 20k skip-2 the backlog stayed ~40-120 ms and drained after ~2 s.

**Inferred (the hypothesis):** collapse is a metastable overload triggered by a pause during the low-headroom window, not by slow code alone.
- With C2 holding ~0.4 CPU, the server runs at utilization ≈ 1. A ~70-90 ms first-GC pause (plus the onset spike) queues ~200-300 requests, so every new request is >35 ms late.
- The backlog can only drain at (capacity - demand) ≈ 0, and clients' abandoned requests are still processed.
- Timeouts force reconnects (Acceptor work, connection setup on the carrier). A growing backlog holds more live objects, so GC costs more (0.3-1.0 CPU-s per burst in the runs that never recovered). Both push demand above capacity, so the overload feeds itself.
- 20k skip-2 sat at the tipping point: backlog ~40-120 ms, little extra GC, and it drained by +2.25 s. The data can't say why it tipped back down. Candidates are a smaller initial pause, request-path C2 code arriving (24 of 165 methods by +2 s, `ConnectionHandler::process` installed at +2.56), or less churn.
- 10k-1/2 and 40k-2 tipped up (backlog to 1.6-5 s, GC overhead 0.3-1.0 CPU-s) and could not recover even after the C2 queue emptied.

**skip-1's dip at +3.25..+4.0:** a 0.53 s stall of POST completions (+3.43 to ~+3.96). The client was on time (start delay ≤3.6 ms) and a GET still completed (422 ms). The container was at 1.00 CPU and Postgres at ~0.5, with no Postgres log entries. Its cause is not visible in this data. It is probably on the ingest-writer/DB commit path, since POSTs complete only on commit; a mid-burst GC is possible (Notification Thread activity in that second) but unconfirmed. warm-2 had the same kind of stall (~0.2 s at +4.17) and lost 347 requests. skip-1 lost 1,199, which fits the low-headroom amplification above. It did not tip into collapse.

**Comparison with warm:** warm runs had their first GCs during warm-up (~160 s earlier), have no C2 backlog in the burst (queue 0-27, ≤1.1 s of C2), and have ~45% CPU headroom. The same ~0.2 s stalls cost only a few hundred requests and clear within ~0.2 s.

**What would test it:** add `-Xlog:gc` / `jdk.GarbageCollection` and `jdk.SafepointBegin` to the JFR settings to confirm the first-GC pause timing and length. Force a GC (e.g. `System.gc()` or a short allocation burst) before the skip runs report healthy; collapses should then become rare. Alternatively, cap C2's CPU share during the burst (e.g. lower C2 thread priority), which should restore headroom.

## Appendix: burst-onset tables (t relative to fitted burst start)

Requests are binned by scheduled send time. "C2 busy %" is the share of the bin covered by C2 compilations. "other" in the CPU table means remaining Java threads (excludes the VM/GC threads, which JFR doesn't report).

#### skip-1 (burst start = 174.51 s after JVM start)

per 250 ms (t relative to burst start; requests by scheduled time; compiles by start time)
| t | sent | on time | missed | p50 ms | max ms | C2 started | C2 ms started | C2 busy % | C1 started |
|---|---|---|---|---|---|---|---|---|---|
| -1.00 | 25 | 25 | 0 | 2.7 | 7.1 | 1 | 15 | 6 | 0 |
| -0.75 | 25 | 25 | 0 | 2.5 | 3.7 | 1 | 2 | 1 | 0 |
| -0.50 | 25 | 25 | 0 | 2.5 | 6.0 | 0 | 0 | 0 | 0 |
| -0.25 | 25 | 25 | 0 | 2.8 | 6.5 | 1 | 1 | 0 | 0 |
| +0.00 | 500 | 486 | 14 | 9.0 | 37.5 | 10 | 253 | 68 | 6 |
| +0.25 | 500 | 500 | 0 | 8.0 | 20.1 | 5 | 722 | 100 | 1 |
| +0.50 | 500 | 500 | 0 | 6.2 | 15.5 | 0 | 0 | 100 | 5 |
| +0.75 | 500 | 500 | 0 | 10.7 | 22.6 | 0 | 0 | 100 | 9 |
| +1.00 | 500 | 500 | 0 | 8.2 | 21.5 | 4 | 232 | 100 | 0 |
| +1.25 | 500 | 500 | 0 | 7.0 | 17.9 | 1 | 258 | 100 | 0 |
| +1.50 | 500 | 500 | 0 | 7.3 | 19.0 | 7 | 218 | 99 | 0 |
| +1.75 | 500 | 500 | 0 | 8.8 | 23.6 | 2 | 868 | 100 | 1 |
| +2.00 | 500 | 500 | 0 | 6.2 | 16.5 | 0 | 0 | 100 | 1 |
| +2.25 | 500 | 500 | 0 | 6.1 | 15.3 | 0 | 0 | 100 | 1 |
| +2.50 | 500 | 500 | 0 | 8.1 | 29.9 | 1 | 116 | 100 | 2 |
| +2.75 | 500 | 500 | 0 | 7.2 | 21.2 | 3 | 801 | 100 | 3 |
| +3.00 | 500 | 500 | 0 | 10.9 | 27.6 | 0 | 0 | 100 | 1 |
| +3.25 | 500 | 357 | 143 | 7.7 | 38.2 | 0 | 0 | 100 | 4 |
| +3.50 | 500 | 0 | 500 | 36.0 | 41.2 | 7 | 223 | 99 | 2 |
| +3.75 | 500 | 18 | 482 | 35.7 | 37.2 | 5 | 232 | 100 | 1 |
| +4.00 | 500 | 426 | 74 | 22.0 | 65.5 | 10 | 292 | 99 | 2 |
| +4.25 | 500 | 500 | 0 | 7.9 | 34.7 | 14 | 200 | 98 | 0 |
| +4.50 | 500 | 500 | 0 | 5.9 | 17.1 | 18 | 245 | 99 | 1 |
| +4.75 | 500 | 500 | 0 | 6.9 | 16.3 | 4 | 438 | 100 | 1 |

per 1 s JFR ThreadCPULoad (event at t covers (t-1, t]) and C2 queue
| JFR t | C2 | C1 | VT carrier | ingest-writer | http-nio | other | total | C2 queue | C2 added/s | C2 removed/s |
|---|---|---|---|---|---|---|---|---|---|---|
| -1.97 | 0.01 | 0.00 | 0.05 | 0.03 | 0.02 | 0.01 | 0.11 | 0 | 3 | 3 |
| -0.97 | 0.11 | 0.01 | 0.04 | 0.02 | 0.01 | 0.01 | 0.21 | 0 | 8 | 8 |
| +0.03 | 0.02 | 0.00 | 0.05 | 0.02 | 0.02 | 0.01 | 0.12 | 0 | 4 | 4 |
| +1.03 | 0.37 | 0.01 | 0.34 | 0.06 | 0.08 | 0.05 | 0.93 | 105 | 117 | 12 |
| +2.03 | 0.46 | 0.00 | 0.34 | 0.06 | 0.08 | 0.06 | 1.01 | 96 | 4 | 13 |
| +3.04 | 0.43 | 0.01 | 0.34 | 0.07 | 0.09 | 0.06 | 0.98 | 107 | 14 | 3 |
| +4.04 | 0.40 | 0.01 | 0.33 | 0.03 | 0.12 | 0.04 | 0.94 | 115 | 20 | 12 |
| +5.04 | 0.44 | 0.00 | 0.33 | 0.06 | 0.10 | 0.06 | 0.99 | 74 | 3 | 44 |
| +6.04 | 0.43 | 0.00 | 0.31 | 0.06 | 0.09 | 0.06 | 0.95 | 31 | 3 | 46 |
| +7.05 | 0.40 | 0.00 | 0.31 | 0.06 | 0.09 | 0.06 | 0.92 | 0 | 6 | 37 |
| +8.05 | 0.14 | 0.00 | 0.33 | 0.06 | 0.11 | 0.07 | 0.71 | 0 | 7 | 7 |
| +9.05 | 0.00 | 0.00 | 0.33 | 0.06 | 0.12 | 0.07 | 0.59 | 0 | 0 | 0 |
| +10.05 | 0.00 | 0.00 | 0.32 | 0.06 | 0.11 | 0.07 | 0.57 | 0 | 0 | 0 |

#### skip-2 (burst start = 172.88 s after JVM start)

per 250 ms (t relative to burst start; requests by scheduled time; compiles by start time)
| t | sent | on time | missed | p50 ms | max ms | C2 started | C2 ms started | C2 busy % | C1 started |
|---|---|---|---|---|---|---|---|---|---|
| -1.00 | 25 | 25 | 0 | 2.5 | 6.8 | 1 | 31 | 13 | 0 |
| -0.75 | 25 | 25 | 0 | 2.5 | 5.0 | 0 | 0 | 0 | 0 |
| -0.50 | 25 | 25 | 0 | 2.6 | 3.6 | 0 | 0 | 0 | 0 |
| -0.25 | 25 | 25 | 0 | 2.4 | 4.2 | 1 | 1 | 0 | 0 |
| +0.00 | 500 | 4 | 496 | 35.7 | 38.4 | 12 | 163 | 65 | 56 |
| +0.25 | 500 | 0 | 500 | 35.7 | 39.9 | 5 | 252 | 100 | 2 |
| +0.50 | 500 | 0 | 500 | 35.7 | 36.9 | 3 | 473 | 100 | 1 |
| +0.75 | 500 | 0 | 500 | 35.7 | 36.5 | 1 | 154 | 100 | 18 |
| +1.00 | 500 | 0 | 500 | 35.7 | 39.4 | 2 | 240 | 100 | 48 |
| +1.25 | 500 | 0 | 500 | 35.7 | 36.3 | 5 | 137 | 100 | 1 |
| +1.50 | 500 | 163 | 337 | 35.6 | 37.9 | 2 | 1055 | 100 | 0 |
| +1.75 | 500 | 0 | 500 | 35.9 | 44.3 | 0 | 0 | 100 | 2 |
| +2.00 | 500 | 70 | 430 | 35.7 | 37.8 | 0 | 0 | 100 | 0 |
| +2.25 | 500 | 500 | 0 | 8.1 | 31.4 | 0 | 0 | 100 | 2 |
| +2.50 | 500 | 500 | 0 | 9.0 | 21.9 | 2 | 676 | 100 | 0 |
| +2.75 | 500 | 500 | 0 | 10.1 | 22.1 | 0 | 0 | 100 | 0 |
| +3.00 | 500 | 500 | 0 | 7.1 | 17.0 | 1 | 111 | 100 | 0 |
| +3.25 | 500 | 500 | 0 | 8.3 | 23.9 | 5 | 205 | 100 | 0 |
| +3.50 | 500 | 500 | 0 | 6.8 | 19.2 | 6 | 194 | 99 | 1 |
| +3.75 | 500 | 500 | 0 | 7.7 | 19.9 | 12 | 296 | 100 | 0 |
| +4.00 | 500 | 443 | 57 | 11.3 | 66.9 | 14 | 206 | 99 | 4 |
| +4.25 | 500 | 499 | 1 | 9.5 | 36.2 | 12 | 250 | 98 | 1 |
| +4.50 | 500 | 500 | 0 | 6.9 | 17.2 | 7 | 508 | 100 | 1 |
| +4.75 | 500 | 500 | 0 | 11.1 | 24.6 | 0 | 0 | 100 | 2 |

per 1 s JFR ThreadCPULoad (event at t covers (t-1, t]) and C2 queue
| JFR t | C2 | C1 | VT carrier | ingest-writer | http-nio | other | total | C2 queue | C2 added/s | C2 removed/s |
|---|---|---|---|---|---|---|---|---|---|---|
| -1.28 | 0.08 | 0.00 | 0.06 | 0.02 | 0.01 | 0.01 | 0.19 | 0 | 3 | 3 |
| -0.28 | 0.05 | 0.00 | 0.07 | 0.02 | 0.01 | 0.00 | 0.16 | 0 | 1 | 1 |
| +0.73 | 0.23 | 0.01 | 0.20 | 0.02 | 0.09 | 0.02 | 0.58 | 33 | 53 | 20 |
| +1.76 | 0.36 | 0.04 | 0.35 | 0.02 | 0.15 | 0.03 | 0.96 | 106 | 80 | 9 |
| +2.77 | 0.40 | 0.00 | 0.35 | 0.04 | 0.12 | 0.05 | 0.96 | 112 | 7 | 1 |
| +3.77 | 0.45 | 0.00 | 0.33 | 0.06 | 0.09 | 0.06 | 0.99 | 102 | 3 | 13 |
| +4.77 | 0.47 | 0.00 | 0.32 | 0.06 | 0.09 | 0.06 | 1.00 | 69 | 9 | 42 |
| +5.77 | 0.43 | 0.01 | 0.31 | 0.05 | 0.08 | 0.06 | 0.94 | 75 | 15 | 9 |
| +6.77 | 0.47 | 0.00 | 0.31 | 0.06 | 0.09 | 0.06 | 1.00 | 15 | 0 | 60 |
| +7.77 | 0.22 | 0.00 | 0.33 | 0.05 | 0.11 | 0.07 | 0.78 | 0 | 13 | 28 |
| +8.77 | 0.20 | 0.00 | 0.33 | 0.04 | 0.11 | 0.08 | 0.76 | 0 | 1 | 1 |
| +9.77 | 0.07 | 0.00 | 0.33 | 0.06 | 0.12 | 0.07 | 0.66 | 0 | 5 | 5 |
| +10.77 | 0.02 | 0.00 | 0.12 | 0.03 | 0.04 | 0.03 | 0.24 | 0 | 1 | 1 |

#### warm-1 (burst start = 176.54 s after JVM start)

per 250 ms (t relative to burst start; requests by scheduled time; compiles by start time)
| t | sent | on time | missed | p50 ms | max ms | C2 started | C2 ms started | C2 busy % | C1 started |
|---|---|---|---|---|---|---|---|---|---|
| -1.00 | 25 | 25 | 0 | 2.3 | 3.3 | 0 | 0 | 0 | 0 |
| -0.75 | 25 | 25 | 0 | 2.5 | 7.0 | 0 | 0 | 0 | 0 |
| -0.50 | 25 | 25 | 0 | 2.4 | 5.5 | 1 | 84 | 34 | 0 |
| -0.25 | 25 | 25 | 0 | 2.5 | 5.3 | 0 | 0 | 0 | 1 |
| +0.00 | 500 | 500 | 0 | 3.8 | 12.0 | 0 | 0 | 0 | 4 |
| +0.25 | 500 | 500 | 0 | 5.7 | 15.0 | 2 | 84 | 34 | 0 |
| +0.50 | 500 | 500 | 0 | 3.5 | 8.8 | 0 | 0 | 0 | 0 |
| +0.75 | 500 | 500 | 0 | 4.5 | 15.2 | 1 | 5 | 2 | 6 |
| +1.00 | 500 | 500 | 0 | 5.6 | 20.6 | 3 | 135 | 54 | 4 |
| +1.25 | 500 | 500 | 0 | 3.4 | 15.9 | 0 | 0 | 0 | 0 |
| +1.50 | 500 | 500 | 0 | 3.2 | 11.2 | 0 | 0 | 0 | 0 |
| +1.75 | 500 | 500 | 0 | 3.5 | 9.5 | 0 | 0 | 0 | 0 |
| +2.00 | 500 | 500 | 0 | 3.2 | 13.2 | 0 | 0 | 0 | 0 |
| +2.25 | 500 | 500 | 0 | 4.2 | 12.9 | 3 | 8 | 3 | 0 |
| +2.50 | 500 | 500 | 0 | 5.4 | 13.0 | 2 | 2 | 1 | 0 |
| +2.75 | 500 | 500 | 0 | 3.7 | 10.6 | 0 | 0 | 0 | 3 |
| +3.00 | 500 | 500 | 0 | 4.1 | 12.1 | 1 | 3 | 1 | 1 |
| +3.25 | 500 | 500 | 0 | 3.7 | 13.2 | 0 | 0 | 0 | 0 |
| +3.50 | 500 | 484 | 16 | 4.4 | 47.4 | 0 | 0 | 0 | 0 |
| +3.75 | 500 | 500 | 0 | 4.0 | 10.1 | 0 | 0 | 0 | 0 |
| +4.00 | 500 | 496 | 4 | 5.4 | 40.3 | 0 | 0 | 0 | 0 |
| +4.25 | 500 | 500 | 0 | 3.7 | 11.9 | 0 | 0 | 0 | 0 |
| +4.50 | 500 | 500 | 0 | 6.8 | 19.4 | 0 | 0 | 0 | 0 |
| +4.75 | 500 | 500 | 0 | 4.2 | 13.5 | 0 | 0 | 0 | 0 |

per 1 s JFR ThreadCPULoad (event at t covers (t-1, t]) and C2 queue
| JFR t | C2 | C1 | VT carrier | ingest-writer | http-nio | other | total | C2 queue | C2 added/s | C2 removed/s |
|---|---|---|---|---|---|---|---|---|---|---|
| -1.12 | 0.00 | 0.01 | 0.04 | 0.02 | 0.02 | 0.01 | 0.08 | 0 | 0 | 0 |
| -0.12 | 0.08 | 0.00 | 0.04 | 0.02 | 0.01 | 0.00 | 0.16 | 0 | 0 | 0 |
| +0.88 | 0.04 | 0.01 | 0.29 | 0.06 | 0.10 | 0.06 | 0.56 | 0 | 1 | 1 |
| +1.88 | 0.07 | 0.00 | 0.32 | 0.06 | 0.10 | 0.06 | 0.61 | 0 | 3 | 3 |
| +2.88 | 0.01 | 0.00 | 0.32 | 0.06 | 0.11 | 0.07 | 0.56 | 0 | 4 | 4 |
| +3.88 | 0.00 | 0.00 | 0.32 | 0.06 | 0.11 | 0.07 | 0.56 | 0 | 0 | 0 |
| +4.88 | 0.00 | 0.00 | 0.32 | 0.06 | 0.11 | 0.08 | 0.56 | 0 | 0 | 0 |
| +5.88 | 0.00 | 0.00 | 0.32 | 0.06 | 0.11 | 0.07 | 0.57 | 0 | 1 | 1 |
| +6.88 | 0.00 | 0.00 | 0.31 | 0.06 | 0.11 | 0.07 | 0.55 | 0 | 0 | 0 |
| +7.89 | 0.00 | 0.00 | 0.32 | 0.07 | 0.12 | 0.07 | 0.58 | 0 | 0 | 0 |
| +8.89 | 0.00 | 0.00 | 0.33 | 0.06 | 0.11 | 0.07 | 0.58 | 0 | 0 | 0 |
| +9.89 | 0.00 | 0.00 | 0.32 | 0.06 | 0.11 | 0.07 | 0.55 | 0 | 0 | 0 |
| +10.89 | 0.00 | 0.00 | 0.06 | 0.02 | 0.02 | 0.01 | 0.12 | 0 | 0 | 0 |

#### warm-2 (burst start = 176.03 s after JVM start)

per 250 ms (t relative to burst start; requests by scheduled time; compiles by start time)
| t | sent | on time | missed | p50 ms | max ms | C2 started | C2 ms started | C2 busy % | C1 started |
|---|---|---|---|---|---|---|---|---|---|
| -1.00 | 25 | 25 | 0 | 2.6 | 6.4 | 0 | 0 | 0 | 0 |
| -0.75 | 25 | 25 | 0 | 2.5 | 4.4 | 0 | 0 | 0 | 1 |
| -0.50 | 25 | 25 | 0 | 2.4 | 7.1 | 1 | 12 | 5 | 22 |
| -0.25 | 25 | 25 | 0 | 2.6 | 5.2 | 0 | 0 | 0 | 0 |
| +0.00 | 500 | 497 | 3 | 6.4 | 37.5 | 1 | 633 | 90 | 4 |
| +0.25 | 500 | 500 | 0 | 6.3 | 17.1 | 0 | 0 | 100 | 3 |
| +0.50 | 500 | 500 | 0 | 6.1 | 13.2 | 1 | 95 | 100 | 0 |
| +0.75 | 500 | 500 | 0 | 5.8 | 17.1 | 30 | 187 | 76 | 1 |
| +1.00 | 500 | 500 | 0 | 4.1 | 14.6 | 0 | 0 | 0 | 1 |
| +1.25 | 500 | 500 | 0 | 4.0 | 11.9 | 2 | 119 | 18 | 0 |
| +1.50 | 500 | 500 | 0 | 5.6 | 28.6 | 0 | 0 | 29 | 4 |
| +1.75 | 500 | 500 | 0 | 3.8 | 14.3 | 0 | 0 | 0 | 1 |
| +2.00 | 500 | 500 | 0 | 5.7 | 24.1 | 1 | 1 | 0 | 1 |
| +2.25 | 500 | 500 | 0 | 5.2 | 18.9 | 3 | 8 | 3 | 0 |
| +2.50 | 500 | 500 | 0 | 5.0 | 19.8 | 0 | 0 | 0 | 1 |
| +2.75 | 500 | 500 | 0 | 4.5 | 13.6 | 0 | 0 | 0 | 0 |
| +3.00 | 500 | 500 | 0 | 4.3 | 16.3 | 0 | 0 | 0 | 0 |
| +3.25 | 500 | 500 | 0 | 4.0 | 11.9 | 0 | 0 | 0 | 0 |
| +3.50 | 500 | 500 | 0 | 5.9 | 16.0 | 0 | 0 | 0 | 0 |
| +3.75 | 500 | 500 | 0 | 5.5 | 29.7 | 0 | 0 | 0 | 1 |
| +4.00 | 500 | 344 | 156 | 7.3 | 55.6 | 1 | 2 | 1 | 0 |
| +4.25 | 500 | 309 | 191 | 21.4 | 44.1 | 5 | 34 | 14 | 1 |
| +4.50 | 500 | 500 | 0 | 4.6 | 17.1 | 0 | 0 | 0 | 0 |
| +4.75 | 500 | 500 | 0 | 5.4 | 15.0 | 0 | 0 | 0 | 0 |

per 1 s JFR ThreadCPULoad (event at t covers (t-1, t]) and C2 queue
| JFR t | C2 | C1 | VT carrier | ingest-writer | http-nio | other | total | C2 queue | C2 added/s | C2 removed/s |
|---|---|---|---|---|---|---|---|---|---|---|
| -1.68 | 0.00 | 0.00 | 0.04 | 0.02 | 0.01 | 0.00 | 0.08 | 0 | 0 | 0 |
| -0.68 | 0.00 | 0.00 | 0.04 | 0.02 | 0.02 | 0.00 | 0.08 | 0 | 0 | 0 |
| +0.32 | 0.17 | 0.01 | 0.13 | 0.03 | 0.04 | 0.02 | 0.40 | 27 | 28 | 1 |
| +1.32 | 0.28 | 0.00 | 0.32 | 0.06 | 0.10 | 0.06 | 0.82 | 0 | 3 | 30 |
| +2.33 | 0.07 | 0.00 | 0.33 | 0.05 | 0.10 | 0.07 | 0.62 | 0 | 3 | 3 |
| +3.33 | 0.00 | 0.00 | 0.33 | 0.06 | 0.12 | 0.07 | 0.58 | 0 | 1 | 1 |
| +4.33 | 0.01 | 0.00 | 0.32 | 0.05 | 0.12 | 0.07 | 0.56 | 0 | 2 | 2 |
| +5.33 | 0.01 | 0.00 | 0.36 | 0.05 | 0.12 | 0.07 | 0.61 | 0 | 2 | 2 |
| +6.33 | 0.00 | 0.00 | 0.33 | 0.05 | 0.11 | 0.07 | 0.57 | 0 | 0 | 0 |
| +7.33 | 0.00 | 0.00 | 0.33 | 0.05 | 0.11 | 0.08 | 0.57 | 0 | 0 | 0 |
| +8.33 | 0.00 | 0.00 | 0.35 | 0.05 | 0.12 | 0.08 | 0.60 | 0 | 0 | 0 |
| +9.33 | 0.00 | 0.00 | 0.33 | 0.04 | 0.12 | 0.08 | 0.57 | 0 | 0 | 0 |
| +10.33 | 0.00 | 0.00 | 0.24 | 0.04 | 0.08 | 0.05 | 0.41 | 0 | 0 | 0 |

### Raw request-path C2 / recompile analysis

#### skip-1: C2 compilations started in [burst start, +12 s): 186 (185 methods); request-path packages: 159 compilations, 158 methods
- kinds (all C2 in window): {'level_raise': 170, 'recompile': 6, 'new': 10}; previous level of method (0 = never compiled): {0: 10, 2: 151, 3: 19, 4: 6}
- request-path: kinds {'level_raise': 150, 'recompile': 5, 'new': 4}; prev level {0: 4, 2: 143, 3: 7, 4: 5}; OSR 2
- C2 compile duration (wall, ms): median 9, p90 95, max 845, sum 7.1 s
- request-path C2 compile ends: first +0.02, 25% +3.80, 50% +4.75, 75% +5.79, last +8.06 s
- by package (compilations, last end s): com/fasterxml/jackson 29, org/apache/coyote 22, sun/nio/ch 21, org/apache/catalina 18, org/apache/tomcat 17, org/postgresql 14, java/lang/VirtualThread 9, java/util/concurrent 9, java/nio 6, com/example/events 5, org/springframework/web 3, jdk/internal/vm/Continuation 3, jakarta/servlet 1, com/zaxxer/hikari 1, org/springframework/jdbc 1
- C2 queue (1 s samples): max 115; first empty sample after onset at +7.05 s
- request-path C2 installs completed by: +0.5s 14, +1s 14, +2s 28, +3s 32, +4s 42, +5s 84, +6s 126, +7s 157, +8s 158, +10s 159
- longest request-path C2 compiles: `AbstractProtocol$ConnectionHandler::process` 845 ms (+1.79..+2.64); `PgArrays::setTextArray` 628 ms (+0.43..+1.05); `QueryExecutorImpl::processResults` 556 ms (+3.00..+3.55); `Http11Processor::prepareResponse` 373 ms (+4.82..+5.20); `SocketWrapperBase::writeBlocking` 258 ms (+1.29..+1.55); `DelayScheduler::loop` OSR 214 ms (+6.67..+6.88); `DelayScheduler::loop` OSR 206 ms (+7.85..+8.06); `CharacterEncodingFilter::doFilterInternal` 144 ms (+5.20..+5.34)
- recompiles during traffic (to end of burst): 63
  - -74.56 s L4 (earlier max L4) org/apache/catalina/authenticator/AuthenticatorBase::invoke (4.8 ms)
  - -4.92 s L3 (earlier max L4) com/fasterxml/jackson/core/sym/ByteQuadsCanonicalizer::findName (0.6 ms)
  - -4.92 s L3 (earlier max L4) com/fasterxml/jackson/core/json/UTF8StreamJsonParser::nextFieldName (3.2 ms)
  - -4.89 s L3 (earlier max L4) com/fasterxml/jackson/core/json/UTF8StreamJsonParser::_parseName (0.8 ms)
  - -4.73 s L3 (earlier max L4) com/fasterxml/jackson/core/json/JsonReadContext::_checkDup (0.9 ms)
  - -4.70 s L3 (earlier max L4) java/util/concurrent/ForkJoinPool::tryTrim (0.8 ms)
  - -3.97 s L1 (earlier max L3) java/util/ArrayList$ArrayListSpliterator::characteristics (0.2 ms)
  - -3.04 s L1 (earlier max L3) java/util/stream/Node::getChildCount (0.1 ms)
  - -3.00 s L3 (earlier max L3) OSR java/util/concurrent/DelayScheduler::loop (4.4 ms)
  - -1.89 s L1 (earlier max L3) java/nio/DirectByteBuffer::base (0.1 ms)
  - -1.73 s L1 (earlier max L3) java/nio/HeapByteBuffer::isReadOnly (0.1 ms)
  - -0.59 s L4 (earlier max L4) java/util/stream/Nodes$FixedNodeBuilder::accept (1.9 ms)
  - +0.54 s L1 (earlier max L3) java/nio/DirectByteBuffer::isReadOnly (0.1 ms)
  - +1.06 s L4 (earlier max L4) com/fasterxml/jackson/core/json/UTF8StreamJsonParser::nextFieldName (94.5 ms)
  - +1.15 s L4 (earlier max L4) com/fasterxml/jackson/core/json/JsonReadContext::_checkDup (11.2 ms)
  - +1.16 s L4 (earlier max L4) com/fasterxml/jackson/core/json/UTF8StreamJsonParser::_parseName (21.5 ms)
  - +3.76 s L1 (earlier max L3) java/nio/channels/SocketChannel::validOps (0.1 ms)
  - +4.44 s L4 (earlier max L4) java/util/concurrent/ForkJoinPool::tryTrim (6.5 ms)
  - +5.76 s L4 (earlier max L4) java/util/ArrayList$ArrayListSpliterator::forEachRemaining (7.8 ms)
  - +6.02 s L3 (earlier max L3) OSR org/apache/tomcat/util/net/NioEndpoint$Poller::run (1.5 ms)
  - +7.85 s L4 (earlier max L4) OSR java/util/concurrent/DelayScheduler::loop (206.4 ms)

#### skip-2: C2 compilations started in [burst start, +12 s): 199 (197 methods); request-path packages: 167 compilations, 165 methods
- kinds (all C2 in window): {'level_raise': 181, 'recompile': 5, 'new': 13}; previous level of method (0 = never compiled): {0: 13, 2: 158, 3: 23, 4: 5}
- request-path: kinds {'level_raise': 158, 'recompile': 4, 'new': 5}; prev level {0: 5, 2: 145, 3: 13, 4: 4}; OSR 3
- C2 compile duration (wall, ms): median 9, p90 86, max 1017, sum 7.7 s
- request-path C2 compile ends: first +0.03, 25% +3.89, 50% +5.74, 75% +6.22, last +10.64 s
- by package (compilations, last end s): com/fasterxml/jackson 27, sun/nio/ch 24, org/apache/coyote 22, org/apache/catalina 18, org/apache/tomcat 17, org/postgresql 14, java/nio 11, java/util/concurrent 11, java/lang/VirtualThread 9, com/example/events 5, org/springframework/web 3, jdk/internal/vm/Continuation 3, jakarta/servlet 1, com/zaxxer/hikari 1, org/springframework/jdbc 1
- C2 queue (1 s samples): max 112; first empty sample after onset at +7.77 s
- request-path C2 installs completed by: +0.5s 12, +1s 16, +2s 24, +3s 26, +4s 49, +5s 79, +6s 100, +7s 149, +8s 159, +10s 165
- longest request-path C2 compiles: `AbstractProtocol$ConnectionHandler::process` 1017 ms (+1.55..+2.56); `PgArrays::setTextArray` 520 ms (+2.72..+3.24); `QueryExecutorImpl::processResults` 480 ms (+5.02..+5.50); `Http11Processor::prepareResponse` 444 ms (+4.58..+5.02); `SocketWrapperBase::writeBlocking` 333 ms (+0.64..+0.98); `DelayScheduler::loop` OSR 206 ms (+8.31..+8.51); `DelayScheduler::loop` OSR 179 ms (+7.77..+7.95); `DelayScheduler::loop` OSR 179 ms (+7.26..+7.44)
- recompiles during traffic (to end of burst): 64
  - -71.83 s L4 (earlier max L4) org/apache/catalina/authenticator/AuthenticatorBase::invoke (6.1 ms)
  - -4.91 s L3 (earlier max L4) com/fasterxml/jackson/core/json/UTF8StreamJsonParser::nextFieldName (3.1 ms)
  - -4.91 s L3 (earlier max L4) com/fasterxml/jackson/core/sym/ByteQuadsCanonicalizer::findName (0.6 ms)
  - -4.88 s L3 (earlier max L4) com/fasterxml/jackson/core/json/UTF8StreamJsonParser::_parseName (0.9 ms)
  - -4.72 s L3 (earlier max L4) com/fasterxml/jackson/core/json/JsonReadContext::_checkDup (0.8 ms)
  - -4.67 s L3 (earlier max L4) java/util/concurrent/ForkJoinPool::tryTrim (0.8 ms)
  - -3.93 s L1 (earlier max L3) java/util/ArrayList$ArrayListSpliterator::characteristics (0.1 ms)
  - -2.98 s L1 (earlier max L3) java/util/stream/Node::getChildCount (0.1 ms)
  - -2.97 s L3 (earlier max L3) OSR java/util/concurrent/DelayScheduler::loop (3.7 ms)
  - -1.84 s L1 (earlier max L3) java/nio/DirectByteBuffer::base (0.1 ms)
  - -1.70 s L1 (earlier max L3) java/nio/HeapByteBuffer::isReadOnly (0.1 ms)
  - +0.09 s L2 (earlier max L4) sun/invoke/util/VerifyType::isNullConversion (0.6 ms)
  - +0.15 s L4 (earlier max L4) java/util/stream/Nodes$FixedNodeBuilder::accept (1.3 ms)
  - +0.40 s L1 (earlier max L3) java/nio/channels/SocketChannel::validOps (0.1 ms)
  - +0.50 s L4 (earlier max L4) com/fasterxml/jackson/core/json/UTF8StreamJsonParser::nextFieldName (127.0 ms)
  - +1.32 s L1 (earlier max L3) java/nio/DirectByteBuffer::isReadOnly (0.1 ms)
  - +5.85 s L4 (earlier max L4) java/util/concurrent/ForkJoinPool::tryTrim (8.1 ms)
  - +6.53 s L3 (earlier max L3) OSR org/apache/tomcat/util/net/NioEndpoint$Poller::run (6.0 ms)
  - +7.77 s L4 (earlier max L4) OSR java/util/concurrent/DelayScheduler::loop (179.4 ms)
  - +8.31 s L4 (earlier max L4) OSR java/util/concurrent/DelayScheduler::loop (205.7 ms)

#### warm-1: C2 compilations started in [burst start, +12 s): 14 (14 methods); request-path packages: 9 compilations, 9 methods
- kinds (all C2 in window): {'level_raise': 10, 'recompile': 3, 'new': 1}; previous level of method (0 = never compiled): {0: 1, 2: 6, 3: 4, 4: 3}
- request-path: kinds {'recompile': 3, 'level_raise': 5, 'new': 1}; prev level {0: 1, 2: 4, 3: 1, 4: 3}; OSR 0
- C2 compile duration (wall, ms): median 3, p90 83, max 109, sum 0.2 s
- request-path C2 compile ends: first +0.46, 25% +1.08, 50% +2.31, 75% +2.60, last +3.24 s
- by package (compilations, last end s): java/lang/VirtualThread 3, com/fasterxml/jackson 2, java/util/concurrent 2, org/springframework/jdbc 1, java/lang/invoke 1
- C2 queue (1 s samples): max 0; first empty sample after onset at +1.88 s
- request-path C2 installs completed by: +0.5s 1, +1s 2, +2s 4, +3s 8, +4s 9, +5s 9, +6s 9, +7s 9, +8s 9, +10s 9
- longest request-path C2 compiles: `JdbcTemplate$SimplePreparedStatementCreator::createPreparedStatement` 109 ms (+1.12..+1.23); `UTF8StreamJsonParser::nextFieldName` 83 ms (+0.37..+0.46); `ParserBase::close` 23 ms (+1.06..+1.08); `DelayScheduler$ScheduledForkJoinTask::cancel` 5 ms (+0.91..+0.92); `VirtualThread::cancelTimeoutTask` 4 ms (+2.44..+2.44); `DirectMethodHandle$Holder::newInvokeSpecial` 3 ms (+3.23..+3.24); `ForkJoinPool::tryTrim` 2 ms (+2.30..+2.31); `VirtualThread::unblock` 1 ms (+2.60..+2.60)
- recompiles during traffic (to end of burst): 53
  - -6.02 s L3 (earlier max L4) com/fasterxml/jackson/core/json/JsonReadContext::_checkDup (0.9 ms)
  - -5.01 s L2 (earlier max L4) com/fasterxml/jackson/core/json/UTF8StreamJsonParser::_skipColon (0.7 ms)
  - -5.01 s L3 (earlier max L4) com/example/events/service/AppendValidator::parseAppend (2.1 ms)
  - -4.95 s L3 (earlier max L4) com/fasterxml/jackson/core/json/UTF8StreamJsonParser::nextFieldName (2.6 ms)
  - -4.79 s L3 (earlier max L4) java/util/concurrent/ForkJoinPool::tryTrim (0.7 ms)
  - -4.76 s L3 (earlier max L4) com/fasterxml/jackson/core/json/UTF8StreamJsonParser::_parseName (0.8 ms)
  - -4.66 s L3 (earlier max L4) com/fasterxml/jackson/core/base/ParserBase::close (0.9 ms)
  - -3.18 s L3 (earlier max L4) com/fasterxml/jackson/core/json/UTF8StreamJsonParser::_skipColon (1.0 ms)
  - -0.45 s L4 (earlier max L4) com/example/events/service/AppendValidator::parseAppend (84.1 ms)
  - +0.37 s L4 (earlier max L4) com/fasterxml/jackson/core/json/UTF8StreamJsonParser::nextFieldName (83.3 ms)
  - +0.80 s L3 (earlier max L3) OSR com/example/events/service/IngestBatcher::run (1.0 ms)
  - +1.06 s L4 (earlier max L4) com/fasterxml/jackson/core/base/ParserBase::close (23.0 ms)
  - +2.30 s L4 (earlier max L4) java/util/concurrent/ForkJoinPool::tryTrim (2.4 ms)

#### warm-2: C2 compilations started in [burst start, +12 s): 46 (46 methods); request-path packages: 43 compilations, 43 methods
- kinds (all C2 in window): {'recompile': 5, 'level_raise': 41}; previous level of method (0 = never compiled): {2: 35, 3: 6, 4: 5}
- request-path: kinds {'recompile': 5, 'level_raise': 38}; prev level {2: 34, 3: 4, 4: 5}; OSR 0
- C2 compile duration (wall, ms): median 4, p90 26, max 633, sum 1.1 s
- request-path C2 compile ends: first +0.66, 25% +0.84, 50% +0.92, 75% +1.47, last +11.03 s
- by package (compilations, last end s): com/fasterxml/jackson 31, java/lang/VirtualThread 3, org/apache/tomcat 2, sun/nio/ch 2, java/nio 2, com/example/events 1, java/util/concurrent 1, org/springframework/jdbc 1
- C2 queue (1 s samples): max 27; first empty sample after onset at +2.33 s
- request-path C2 installs completed by: +0.5s 0, +1s 32, +2s 34, +3s 37, +4s 37, +5s 42, +6s 42, +7s 42, +8s 42, +10s 42
- longest request-path C2 compiles: `AppendValidator::parseAppend` 633 ms (+0.03..+0.66); `JdbcTemplate$SimplePreparedStatementCreator::createPreparedStatement` 106 ms (+1.47..+1.57); `UTF8StreamJsonParser::nextFieldName` 95 ms (+0.66..+0.75); `ByteSourceJsonBootstrapper::constructParser` 33 ms (+0.86..+0.89); `UTF8StreamJsonParser::_parseName` 26 ms (+0.76..+0.79); `SelectorImpl::register` 20 ms (+4.38..+4.40); `ParserBase::close` 14 ms (+1.45..+1.47); `JsonReadContext::_checkDup` 11 ms (+0.75..+0.76)
- recompiles during traffic (to end of burst): 65
  - -5.00 s L3 (earlier max L4) com/example/events/service/AppendValidator::parseAppend (1.4 ms)
  - -4.98 s L3 (earlier max L4) com/fasterxml/jackson/core/json/JsonReadContext::_checkDup (0.8 ms)
  - -4.96 s L3 (earlier max L4) com/fasterxml/jackson/core/json/UTF8StreamJsonParser::nextFieldName (2.5 ms)
  - -4.85 s L2 (earlier max L4) com/fasterxml/jackson/core/json/UTF8StreamJsonParser::_skipColon (0.8 ms)
  - -4.83 s L3 (earlier max L4) java/util/concurrent/ForkJoinPool::tryTrim (1.1 ms)
  - -4.71 s L3 (earlier max L4) com/fasterxml/jackson/core/json/UTF8StreamJsonParser::_parseName (0.9 ms)
  - -4.49 s L3 (earlier max L4) com/fasterxml/jackson/core/base/ParserBase::close (0.5 ms)
  - +0.03 s L4 (earlier max L4) com/example/events/service/AppendValidator::parseAppend (632.7 ms)
  - +0.05 s L3 (earlier max L4) com/fasterxml/jackson/core/json/UTF8StreamJsonParser::_skipColon (0.5 ms)
  - +0.66 s L4 (earlier max L4) com/fasterxml/jackson/core/json/UTF8StreamJsonParser::nextFieldName (94.7 ms)
  - +0.75 s L4 (earlier max L4) com/fasterxml/jackson/core/json/JsonReadContext::_checkDup (10.8 ms)
  - +0.76 s L4 (earlier max L4) com/fasterxml/jackson/core/json/UTF8StreamJsonParser::_parseName (25.8 ms)
  - +1.45 s L4 (earlier max L4) com/fasterxml/jackson/core/base/ParserBase::close (13.7 ms)
  - +2.21 s L2 (earlier max L4) java/util/concurrent/ForkJoinPool::tryTrim (0.5 ms)
