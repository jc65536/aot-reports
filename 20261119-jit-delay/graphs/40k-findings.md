# 40k cache: why skip-2 collapsed and skip-1 did not

Runs: `40k/skip-1` (passed, 98.6 %), `40k/skip-2` (collapsed, 3.7 %), `40k/warm-1` and `40k/warm-2` (99.98 % and 99.95 %).
Graphs: `graphs/40k-<run>-jit-cpu.png` and `-queues.png`. Counts: `graphs/40k-<run>.json`, made by `tools/timeline.py` (unchanged).
Scratch scripts for the burst-onset analysis are in the session scratchpad. They are not part of the repo.

## 1. Counts (timeline.py)

idle = JVM start to traffic start; traffic = traffic start to burst_end. C1 = tiers 1-3, C2 = tier 4.

| run | idle C1 (methods) | idle C2 (methods) | traffic C1 (methods) | traffic C2 (methods) |
|---|---|---|---|---|
| skip-1 | 4981 (4807) | 1443 (1434) | 405 (394) | 355 (341) |
| skip-2 | 4979 (4809) | 1440 (1430) | 494 (474) | 320 (308) |
| warm-1 | 7333 (6920) | 2430 (2350) | 70 (65) | 40 (39) |
| warm-2 | 7330 (6948) | 2463 (2382) | 69 (65) | 42 (40) |

Traffic compilations by kind, shown as compilations / methods:

| run | C1 new | C1 level_raise | C1 recompile | C2 new | C2 level_raise | C2 recompile |
|---|---|---|---|---|---|---|
| skip-1 | 354 / 353 | 30 / 30 | 21 / 20 | 111 / 111 | 231 / 230 | 13 / 13 |
| skip-2 | 360 / 357 | 111 / 111 | 23 / 22 | 109 / 109 | 200 / 200 | 11 / 11 |
| warm-1 | 35 / 35 | 20 / 20 | 15 / 11 | 6 / 6 | 29 / 29 | 5 / 5 |
| warm-2 | 30 / 30 | 25 / 25 | 14 / 11 | 5 / 5 | 33 / 33 | 4 / 4 |

Markers from timeline.py (seconds after JVM start): skip-1 idle_end 80.0, burst_start 172.9; skip-2 80.0, 171.9; warm-1 77.9, 170.6; warm-2 77.8, 170.3.
`invalidated` (CompilerStatistics.invalidatedCount) is 0 at every marker in all 4 runs, even though there are recompiles. The counter gives no information here.

### Sanity checks of the graphs and script

- All 8 PNGs have the expected lines: healthy, end of idle and burst start. The curves look plausible. Skip runs show C2 at about 0.4-0.45 CPU for the whole burst, with a C2 queue of 60-100 that drains by about +8 s. Warm runs show one C2 blip of about 0.3 at the burst start, and the C2 queue was empty after about 60 s.
- **C2 CPU from JFR matches the host sampler.** Over the script's burst window, JFR ThreadCPULoad gives skip-1 4.16 s vs threadcpu.txt 4.09 s, skip-2 3.18 vs 3.14, warm-1 0.36 vs 0.36, and warm-2 0.44 vs 0.43. The JFR normalization is correct for compiler threads.
- **The JFR "other" curve leaves out non-Java threads** (GC, VM Thread). For skip-2 in the burst, JFR other = 4.54 s, host = 6.08 s and cgroup total = 9.65 s. For skip-1 it is 5.32 vs 5.13, which fits. About 1.5 CPU-s in skip-2 went to threads that JFR does not see. This is evidence for extra GC/VM work in the collapsed run (inferred).
- **The "other" curve goes above 1.0 at startup** (up to 1.6-1.9 at 5-10 s) on a 1-CPU cpuset. This is a JFR artifact: the `DestroyJavaVM` thread (main after `main()` returns) reports a load of 1.0 for 2-3 samples. It affects only the startup part of the "other" curve.
- **burst_start is 0.3-0.55 s early.** It is defined as the start of the first 1 s threadcpu window with >= 0.3 non-JIT CPU. If the onset is interpolated from the 0.5 s threadcpu samples instead, it is skip-1 173.31, skip-2 172.44, warm-1 171.05 and warm-2 170.61 (vs 172.9 / 171.9 / 170.6 / 170.3). This is by definition, not a bug, so I did not change it. But any "first second of the burst" figure taken from these markers includes 0.3-0.5 s of pre-burst idle. That would explain a "first second not saturated (~0.44 CPU)" reading. When aligned to the onset, 40k-skip-2's first second is saturated: host threads 0.95, cgroup 0.79-1.0. The CPU curves also have about 0.5-1 s of extra lag, because a ThreadCPULoad sample at t covers (t-1, t] and is plotted at int(t).
- The JFR clock and the checker clock are aligned only through the onset estimate, to about ±0.1 s. Below, "+x s" means seconds after the onset T, and checker times are `scheduledMs - 8000`.

## 2. Burst-onset evidence

### 2.1 Per second around the onset (k = seconds after onset)

Columns: requests by checker schedule (on time = 202 within 35 ms); p50 client elapsed time; C2/C1 compilations started in that second (with the summed C2 compile wall time); C2 queue size (1 s samples); host threadcpu C2 / non-JIT CPU; JFR ThreadCPULoad for the virtual-thread carrier (`ForkJoinPool-1-worker-1`, the only carrier, which runs all request code) / Tomcat Acceptor / Poller / `ingest-writer`; cgroup CPU and throttled ms per second.

**skip-1 (pass)**

| k | sent | on time | late | p50 ms | C2 started (dur s) | C1 started | C2 q | C2 / non-JIT | carrier / acceptor / poller / writer | cgroup | thr ms |
|---|---|---|---|---|---|---|---|---|---|---|---|
| -2 | 100 | 95 | 5 | 2.8 | 12 (0.12) | 61 | 0 | 0.05 / 0.13 | 0.04 / 0 / 0.01 / 0.02 | 0.20 | 0 |
| -1 | 100 | 100 | 0 | 2.8 | 6 (0.03) | 7 | 0 | 0.05 / 0.17 | 0.06 / 0 / 0.02 / 0.03 | 0.33 | 0 |
| +0 | 2000 | 2000 | 0 | 7.0 | 18 (0.97) | 13 | 2 | 0.40 / 0.49 | 0.34 / 0 / 0.08 / 0.07 | 0.85 | 0 |
| +1 | 2000 | 1998 | 2 | 8.1 | 16 (0.97) | 2 | 99 | 0.43 / 0.53 | 0.33 / 0 / 0.08 / 0.06 | 1.00 | 0 |
| +2 | 2000 | 2000 | 0 | 7.4 | 20 (1.02) | 7 | 84 | 0.46 / 0.53 | 0.32 / 0 / 0.09 / 0.07 | 1.00 | 1 |
| +3 | 2000 | 1761 | 239 | 9.6 | 28 (1.07) | 9 | 82 | 0.44 / 0.51 | 0.29 / 0.01 / 0.08 / 0.05 | 1.00 | 0 |
| +4 | 2000 | 1948 | 52 | 8.4 | 30 (0.91) | 1 | 84 | 0.46 / 0.53 | 0.31 / 0 / 0.09 / 0.06 | 1.00 | 0 |
| +5 | 2000 | 2000 | 0 | 6.9 | 26 (1.06) | 1 | 53 | 0.48 / 0.49 | 0.31 / 0 / 0.09 / 0.06 | 1.00 | 1 |
| +6..+9 | 2000/s | 100 % | 0 | 7-8 | 49 total | 7 | 30→0 | 0.24-0.45 / ~0.5 | ~0.31 / 0 / 0.09 / 0.05 | 0.7-1.0 | 0 |

**skip-2 (collapse)**

| k | sent | on time | late | p50 ms | C2 started (dur s) | C1 started | C2 q | C2 / non-JIT | carrier / acceptor / poller / writer | cgroup | thr ms |
|---|---|---|---|---|---|---|---|---|---|---|---|
| -2 | 100 | 100 | 0 | 2.9 | 6 (0.14) | 9 | 0 | 0.14 / 0.09 | 0.00 / 0 / 0.02 / 0.03 | 0.20 | 0 |
| -1 | 100 | 100 | 0 | 2.6 | 4 (0.06) | 2 | 0 | 0.00 / 0.09 | 0.10 / 0 / 0.01 / 0.02 | 0.37 | 0 |
| +0 | 2000 | **0** | 2000 | 35.7 | 40 (0.97) | **95** | 6 | 0.37 / 0.58 | **0.24 / 0.08** / 0.06 / **0.01** | 0.79 | 0 |
| +1 | 2000 | 1 | 1999 | 35.7 | 19 (0.96) | 56 | 63 | 0.36 / 0.57 | 0.36 / 0.09 / 0.07 / 0.02 | 1.00 | 0 |
| +2 | 2000 | 0 | 2000 | 35.7 | 11 (1.77) | 5 | 76 | 0.38 / 0.62 | 0.33 / 0.08 / 0.06 / 0.02 | 0.95 | 54 |
| +3 | 2000 | 0 | 2000 | 35.7 | 5 (0.23) | 8 | 77 | 0.30 / 0.55 | 0.35 / 0.08 / 0.06 / 0.02 | 0.95 | 47 |
| +4 | 2000 | 0 | 2000 | 35.8 | 32 (1.01) | 6 | 78 | 0.34 / 0.67 | 0.33 / 0.08 / 0.06 / 0.02 | 1.00 | 0 |
| +5..+9 | 1609-2000 | 0 | all | 36-75 | 65 total | 13 | 51→8 | 0.23-0.35 / 0.61-0.74 | ~0.3 / 0.05-0.10 / 0.05 / 0.01-0.03 | 1.00 | 0 |
| +10, +11 | 100 | 0 | 100 | 35.7 | 40 | 13 | 6-8 | 0.27-0.38 / 0.6-0.68 | 0.43-0.47 / ... | 0.92-0.99 | 0 |
| +12 | 100 | 61 | 39 | 15.8 | 15 | 6 | 0 | 0.13 / 0.33 | 0.04 | 0.58 | 0 |

**warm-1 / warm-2:** 2000/2000 on time in every second except +4 (5 and 8 late), with p50 3.5-6 ms. Total CPU is 0.55-0.6 (C2 only about 0.3 in the first second, then about 0), the Acceptor is 0, and there is no throttling. Carrier 0.33, poller 0.11, writer 0.06, which is the same request work as skip-1.

Measured points:
- At 250 ms resolution, skip-2 already had 0/500 on time in the first bin (0-250 ms). The very first burst request (sent +0.18 ms after its schedule) timed out. The previous request, 10 ms earlier, took 2.2 ms. No 202 arrived from -0.008 s until +1.979 s (the longest response gap). In skip-1, the first burst requests took about 20 ms (vs 2.8 ms before the burst), and then p50 stayed at 7 ms.
- The checker's GET lookups during the load are not cut off at 35 ms, so they measure the server-side backlog in skip-2: 25 ms at +0.08 s, 1 ms at +0.36, 18 ms at +0.61, **152 ms at +0.88, 189 ms at +1.28, 279 ms at +1.72, 2597 ms at +2.25**, 1353 ms at +5.1, and more than 5 s (transport timeout) at +6.7. In skip-1 the lookups stayed at 2-31 ms. In skip-2 the API stayed at 1.0 CPU for about 2.6 s after the burst ended, and phase-4 requests (100 req/s) were late for about 2.4 s.
- CPU is fully used in both skip runs (host threads 0.95-1.0, cgroup 1.0). In skip-1 the 1 CPU splits into C2 at about 0.45 and request threads at about 0.5. The carrier gets 0.33-0.34, and that is just enough for 2000 req/s (p50 7 ms vs 4-5 ms warm). In skip-2's first second the carrier got only **0.24**. CPU went to the Acceptor (0.08, which is 0 in skip-1 and warm), C1 (95 compiles), and non-Java threads (host non-JIT 0.58 vs JFR-visible 0.40). Postgres CPU was lower in skip-2's first 2 s (0.32-0.36 vs 0.55-0.61), and the writer's CPU fell to 0.01-0.02, so less work reached the database. The bottleneck is the API's request path, not Postgres.
- cgroup throttling is negligible in all runs: 0-1 ms/s, apart from 54 and 47 ms in skip-2 at +2 and +3.

### 2.2 The trigger: a stall of about 90 ms at the first GC, which happened to land on the burst onset

Measured:
- Every run has exactly one burst of about 60 C1 compilations of JMX GC-notification code (`GarbageCollectionNotificationInfo`, `GcInfo`, `MemoryUsageCompositeData`, `TabularDataSupport`, ...). It is the same method list in all runs, so it marks the first time the JVM delivers a GC notification, which means the first GC. Timing relative to onset: **skip-1 -1.25 s, skip-2 +0.04 s**, warm-1 -160 s and warm-2 -160 s (during warm-up).
- In skip-1, the client saw a full server stall at that moment. No responses came from -1.275 to -1.186 s, a gap of 89 ms. Five phase-2 requests timed out and the next four were released together at -1.186. This is the only client-visible stall before the burst. In skip-2 there was no stall before the burst. The stall fell on the onset instead. In the same 60 ms window, a trivial C2 compile (`configureSocketNonBlockingIfVirtualThread`) took 61 ms of wall time in skip-2 vs 2 ms in skip-1. The GC-notification code then ran at +0.04 to +0.07 s.
- Other caches (read-only check of their runs) show the same split. First-GC cluster relative to onset, and the result:

| run | first GC | stall seen by client | result |
|---|---|---|---|
| 10k skip-1 | +0.11 s | (in burst) | collapsed |
| 10k skip-2 | +0.10 s | -0.075..-0.008 s, about 70 ms | collapsed |
| 20k skip-1 | -1.58 s | -1.617..-1.548, about 70 ms | passed 94 % |
| 20k skip-2 | -0.08 s | at onset | 2 s collapse, then recovered |
| 40k skip-1 | -1.23 s | -1.275..-1.186, 89 ms | passed |
| 40k skip-2 | +0.04 s | at onset | collapsed |

The 6 skip runs split exactly by whether the first GC fell within about 0.1 s of the burst onset (4 runs, all collapsed or degraded) or 1.2-1.6 s before it, during the 100 req/s phase (2 runs, both passed). Warm runs had their first GC about 160 s before the burst.

Inferred: the stall is a stop-the-world young GC. The cpuset has 1 CPU and there is no GC flag, so JVM ergonomics should pick SerialGC. There are no GC events in this JFR and no GC log, so this is not verified. The first young collection has to copy whatever the startup and functional checks left live in eden, so it is long (70-90 ms). In skip runs the idle period allocates almost nothing, so eden fills during the checker traffic. The burst's 20x allocation rate makes the first GC land either in phase 2 (harmless: 5-9 late requests) or in the first ~0.1 s of the burst. That is a matter of a few MB of allocation difference, which explains why the result varies between otherwise identical runs.

Later GC pauses during the burst (inferred from the Notification Thread waking and from client-side response gaps):
- warm runs: one gap of about 27 ms at +4.2 s (5 and 8 late).
- skip-1: a gap of about 125 ms at +3.60 to +3.72 s (72 + 52 ms gaps; 233 + 57 late requests). It recovered within about 0.3 s.

GC pauses during the C2 storm are about 4x longer than in warm runs. This is probably because the C2 thread keeps running during the safepoint and shares the single CPU with the GC (inferred).

### 2.3 Amplification: why the onset stall never recovers in skip-2

Measured:
- **Connection churn starts right away and only in skip-2.** On a 35 ms timeout the checker closes the socket, and its next request on that worker opens a new connection. The Acceptor thread runs at 0.08 CPU/s for the whole burst in skip-2. In skip-1 and warm runs it uses about 0.00.
- **Tomcat's connect, close and error paths are cold.** AOT training never exercised them. In skip-2 they hit C1 tier-3 thresholds at +0.96 to +1.47 s (`NioEndpoint.serverSocketAccept`, `setSocketOptions`, `SocketChannelImpl` accept/close, `NioSocketWrapper.doClose`, `EPollSelectorImpl.implDereg`, `Http11Processor.<init>`, `connector.Request.<init>`, `ConnectionHandler.register`, `AbstractProcessor.setErrorState`, `Throwable.<init>`). They reached C2 only at +7.5 to +12 s, after the C2 queue drained. In skip-1 none of these was ever compiled, even after its +3.6 s stall. In skip-2 `Throwable.fillInStackTrace` was C2-compiled at +5.5 s. Exceptions are on the hot path there, from writing responses to sockets the client already closed.
- **More GC and VM work.** The Notification Thread ran at +0, 1, 3, 5, 6, 8, 10 and 11 s in skip-2, vs once (+3.6 s) in skip-1 and once (about +4 s) in warm. Over the burst, about 1.5 CPU-s ran in threads JFR does not see (GC/VM).
- **Different C2 order in the first 2 s.** skip-2 started 62 C2 compiles (2.0 s of compile time), including JMX/TreeMap code for the GC notification and `ConcurrentHashMap.transfer` OSR. skip-1 started 39 (1.97 s). Hot methods by the time their first C2 compile started (both runs): `OncePerRequestFilter.doFilter`, `VirtualThread.runContinuation`, `Http11InputBuffer.parseRequestLine` (+0.18 / +0.22), `UTF8StreamJsonParser.nextFieldName` (+0.92 / +0.52), `SocketWrapperBase.writeBlocking` (+1.34 / +0.73), `ApplicationFilterFactory.createFilterChain` (+1.23 / +0.93), `Http11Processor.writeHeaders` (+1.46 / +1.13), `RequestBody.readExactly` (+1.72 / +1.30). The core path was C2-compiled no later in skip-2 than in skip-1. The exception is the ingest writer's per-row encoder, `PgArrays.setTextArray`. skip-1 C2-compiled it at +0.37 to +0.92 s (551 ms compile). skip-2 compiled it only as OSR at +2.91 to +3.78 s, then standard at +5.77 s, then recompiled at +8.01 s (1513 ms). `ArrayEncoding$13.toBinaryRepresentation` was compiled at +1.65 vs +3.78 s. With the backlog, batches are large: few calls, long loops. So the standard-compile trigger came late.
- **Request-path methods that were not C2 at the onset** (packages com.example.events, tomcat/catalina/coyote, jackson, postgresql, hikari, java.util.concurrent, nio, VirtualThread, spring):

| | skip-1 | skip-2 | warm-1 | warm-2 |
|---|---|---|---|---|
| first C2 compiled in [T, T+12 s] | 146 | 162 | 10 | 14 |
| ... of those in the first 1 s / first 2 s | 15 / 27 | 27 / 43 | 5 / 5 | 6 / 6 |
| completion times, 25 / 50 / 75 / 90 % | 2.4 / 4.3 / 5.8 / 7.3 s | 1.8 / 5.0 / 6.9 / 10.8 s | 0.6 / 0.8 / 2.3 / 3.6 s | 0.7 / 2.2 / 2.8 / 7.1 s |
| last one done ("request path fully C2") | +9.8 s | +12.2 s | +6.5 s | +9.2 s |
| C2 compile time spent on them | 6.7 s | 8.0 s | 0.07 s | 0.08 s |

Largest packages in skip: sun.nio.ch (20 / 24), postgresql.jdbc (18 / 10), coyote.http11 (13 / 16), tomcat.util.net (10 / 19), catalina.connector (10 / 10). Before the onset, skip-1 had C2 for 1593 methods and skip-2 for 1574. Hot-package differences: 14 methods only in skip-1, 7 only in skip-2.

The clearest difference is specific to 40k-skip-2. At 97.7 s, skip-1 C2-compiled the Tomcat Poller loop together with its callees (`Poller.run/processKey/timeout/unreg`, `SocketProcessor.doRun`, `Continuation.enter`, `SelectionKeyImpl.interestOps/readyOps`). skip-2 did not. It C2-compiled `processKey` at +0.69 s, `interestOps` at +1.04, `doRun` at +4.37, `Continuation.enter` at +5.21, `unreg` at +5.70, and `Poller.run` only at +17.7 s. 10k-skip-1/2 and 20k-skip-1/2 all had this cluster compiled before the burst, so it is not what separates pass from collapse. It is an extra handicap in 40k-skip-2.

- **Recompiles in traffic.** Both skip runs had the same pattern: 34 events each, 13 and 11 of them at C2. When the functional checks started (-75 s), and again at -47, -16 and -5 s, code compiled at startup was dropped and recompiled at tier 3. This applied to `ForkJoinPool.signalWork/poolSubmit/execute/tryTrim`, `VirtualThread.submitRunContinuation/unpark/unblock`, `UTF8StreamJsonParser.nextFieldName/_parseName` and `TextBuffer.releaseBuffers`. During the burst these were compiled at C2 again (skip-1 at +0.9 to +2.3 s and +7.0 s; skip-2 at +0.5 to +2.9 s). Extra in skip-2: `CoyoteAdapter.checkRecycled` (+0.17 s) and `PgArrays.setTextArray` (+8.01 s, its earlier C2 code was dropped). Extra in skip-1: `JdbcTemplate.execute` (+8.73 s, 1.02 s compile). Warm runs had 18-20 recompiles; their C2 recompiles are `AppendValidator.parseAppend` (+0.0 s, 470-512 ms), `nextFieldName`, `ParserBase.close` and Tomcat `read/fillReadBuffer`. No recompile of a request-path method lines up with skip-2's collapse.

## 3. Mechanism hypothesis

1. **Precondition (measured): no CPU headroom during a skip burst.** With SkipTier2 and no warm-up, the burst sets off a C2 storm: queue 60-100, about 0.35-0.45 CPU for about 8 s, and 146-162 request-path methods still waiting for their first C2. The other about 0.55 CPU is just enough to serve 2000 req/s when nothing goes wrong (skip-1: p50 7 ms, carrier 0.33 CPU). Warm runs use 0.6 CPU in total.
2. **Trigger (measured timing, cause inferred).** The JVM's first GC, a stop-the-world stall of about 70-90 ms that lasts longer while C2 competes for the CPU, falls within about 0.1 s of the burst onset. 6 of 6 skip runs across the three caches match this. When it lands in the 100 req/s phase instead, it costs 5 requests (skip-1).
3. **Amplification (measured symptoms, causal chain inferred).** The 170-200 requests queued during the pause all pass 35 ms. Each timeout makes the client drop its connection and reconnect. The server still processes every abandoned request and then fails writing the response. That adds work on Acceptor and connect, on close and exceptions (`Throwable`), on creating `Http11Processor`/`Request` objects, and on extra allocation and GC. All of these paths are cold: interpreted, then C1 tier 3 from about +1 s, and C2 only after the queue drains at about +8 to +12 s. This work takes about 0.1 (Acceptor) + 0.1-0.2 (GC/VM) CPU away from the single carrier thread (0.24 vs 0.34 in the first second). Request capacity drops below 2000/s, the backlog grows instead of draining (GET latency 25 ms, then 150 ms after 0.9 s, then 2.6 s after 2.25 s), and every later request times out, which feeds more churn. This is a metastable failure. It ends only when the load drops (the API stays busy 2.6 s after the burst).
4. **Why low load hides it.** At 100 req/s the API needs about 0.1 CPU, so a 90 ms stall drains within milliseconds (2.8 ms median outside the stall).
5. **Weak points (honest limits).** skip-1 survived a stall of about 125 ms at +3.6 s, mid-storm (233 + 57 late, recovered in about 0.3 s). So a pause alone is not enough. What matters is a pause at the onset, when nothing in the burst-specific state is warm yet: carrier-only capacity just equal to the load, connection pool growth, empty processor cache, and no prior churn-path compiles. I cannot measure server capacity directly (there are no per-request server timings, GC events or deopt events in this JFR). The capacity numbers above come from thread CPU, lookup latency and Postgres CPU. Clock alignment between JFR and checker is about ±0.1 s.

**Tests that would confirm or refute this:**
- Rerun skip with `-Xlog:gc*` and JFR `jdk.GarbageCollection`/`jdk.SafepointBegin`.
- Force a GC at the end of idle (e.g. `jcmd GC.run` at 79 s), or use a much larger young gen. The prediction is that skip then passes consistently.
- Conversely, trigger a `GC.run` exactly at the burst onset in a warm run. The prediction is that warm survives, thanks to its CPU headroom.
- Lower the C2 storm's CPU share at onset (e.g. CICompilerCount / compile-thread priority). The prediction is that this restores the headroom and removes the collapse.
