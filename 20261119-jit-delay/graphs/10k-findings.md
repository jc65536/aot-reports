# 10k cache: JIT timeline and why skip collapses

Runs: `10k/skip-1`, `10k/skip-2` (collapsed: 4.09% and 3.97% on time) and `10k/warm-1`, `10k/warm-2` (99.99% and 99.94%).
Graphs: `graphs/10k-<run>-jit-cpu.png`, `graphs/10k-<run>-queues.png`. Counts: `graphs/10k-<run>.json` (from `tools/timeline.py`, unchanged).

## 1. Counts (tools/timeline.py)

Markers are in seconds after JVM start. "idle" is JVM start to traffic start. For warm runs this includes the self-warm-up, which ends at healthy, about 67.7 s. "traffic" is traffic start to burst_end. C1 is tiers 1–3 and C2 is tier 4. Each cell is compilations / distinct methods.

| run | healthy | idle_end | burst_start | idle C1 | idle C2 | traffic C1 | traffic C2 |
|---|---|---|---|---|---|---|---|
| skip-1 | 10.8 | 80.0 | 170.5 | 4861 / 4683 | 1464 / 1448 | 549 / 529 | 353 / 342 |
| skip-2 | 10.7 | 80.0 | 172.4 | 4817 / 4641 | 1471 / 1457 | 549 / 531 | 354 / 341 |
| warm-1 | 67.7 | 77.8 | 170.9 | 7250 / 6782 | 2440 / 2353 | 186 / 183 | 37 / 36 |
| warm-2 | 67.7 | 77.8 | 170.4 | 7241 / 6804 | 2406 / 2322 | 167 / 163 | 34 / 34 |

Traffic compilations by kind, as compilations / methods:

| run | C1 new | C1 level_raise | C1 recompile | C2 new | C2 level_raise | C2 recompile |
|---|---|---|---|---|---|---|
| skip-1 | 420 / 418 | 114 / 114 | 15 / 15 | 119 / 119 | 225 / 225 | 9 / 8 |
| skip-2 | 423 / 421 | 112 / 112 | 14 / 14 | 120 / 120 | 224 / 224 | 10 / 9 |
| warm-1 | 144 / 144 | 28 / 28 | 14 / 11 | 4 / 4 | 28 / 28 | 5 / 5 |
| warm-2 | 130 / 130 | 20 / 20 | 17 / 13 | 3 / 3 | 24 / 24 | 7 / 7 |

`CompilerStatistics.invalidatedCount` is 0 at idle_end, burst_start and burst_end in all 4 runs.

### Sanity checks on the graphs and data

- **Graphs.** All 8 PNGs show the dashed lines at idle_end (green) and burst start (red), plus the dotted healthy line. The curves are plausible:
  - Skip runs: about 1.0 C2 CPU in the first ~14 s with a C2 queue near 1,000, then almost nothing until the burst. During the burst, C2 runs at about 0.4 CPU and the C2 queue jumps to about 100.
  - Warm runs: C2 at about 0.35 CPU for the whole warm-up, with the C2 queue near 1,800 and draining by about 60 s. The burst brings only a small blip.
- **ThreadCPULoad normalization is correct.** Over the burst window (10 s), the JFR values are on the 1-CPU scale. Totals are JFR vs the host sampler (threadcpu.txt):

  | run | C2, JFR | C2, host | other threads, JFR | other threads, host |
  |---|---|---|---|---|
  | skip-1 | 3.79 | 3.26 | 5.14 | 5.42 |
  | skip-2 | 3.59 | 3.21 | 5.16 | 5.42 |
  | warm-1 | 0.41 | 0.34 | 5.24 | 5.13 |
  | warm-2 | 0.28 | 0.16 | 5.55 | 5.11 |

  In the 0.5 s windows of the burst, JFR matches the host sampler within about 0.05. JFR C2 reads about 10–15% high in the skip runs, mostly in the last 2–3 s of the burst. Some caveats:
  - (a) JFR ThreadCPULoad only covers Java threads. VM Thread and GC CPU appear in the host "other" column, but not in the JFR "all other threads" line.
  - (b) The startup spike of 1.3–1.5 in "all other threads" (5–9 s) is not real, since the API runs on 1 CPU (cpuset). It comes from each thread's first JFR sample, which includes CPU used before that sample window. It only affects startup.
  - (c) Threads that used less than 1 ms in a period emit no event.
- **burst_start marker bias (not a bug, so not changed).** The marker is `first 1 s window with >= 0.3 other CPU` minus 1 s. That puts it 0.24–0.53 s before the real onset. Fitting the onset from the 0.5 s host samples gives:

  | run | real onset (s) | marker (s) |
  |---|---|---|
  | skip-1 | 170.99 | 170.46 |
  | skip-2 | 172.68 | 172.44 |
  | warm-1 | 171.30 | 170.86 |
  | warm-2 | 170.67 | 170.38 |

  This fit agrees with the phase-2 start 5 s earlier and with the phase-4 drop in warm-1. As a result, the "first burst second" of the 1 s bins is partly pre-burst, which is where the "~0.44 CPU in the first second" figure comes from. The analysis below uses the fitted onset t0. JFR and the checker have no shared clock, so the alignment uncertainty is about ±0.15 s.

## 2. Burst onset evidence

All times are relative to burst start (t0). Request times are the checker's `scheduledMs - 8000`.

### Requests per second

Each cell is sent / on-time 202 / deadline exceeded.

| s | skip-1 | skip-2 | warm-1 | warm-2 |
|---|---|---|---|---|
| -5 (100 rps) | 100/94/6 | 100/100/0 | 100/100/0 | 100/100/0 |
| -4 .. -2 | 100/100/0 each | 100/100/0 each | same | same |
| -1 | 100/100/0 | 100/97/3 | 100/100/0 | 100/100/0 |
| 0 (2000 rps) | 2000/0/2000 | 2000/4/1996 | 2000/2000/0 | 2000/2000/0 |
| 1 .. 8 | 2000/0/2000 each | 2000/0/2000 each | 2000/2000/0 (s4: 1997) | 2000/2000/0 (s4: 1988) |
| 9 | 1989/0/1989 (+11 not sent) | 1503/0/1503 (+497 not sent) | 2000/2000/0 | 2000/2000/0 |
| 10 (100 rps) | 100/0/100 | 100/0/100 | 100/100/0 | 100/100/0 |
| 11 | 100/61/39 | 100/30/70 | 100/100/0 | 100/100/0 |

At 250 ms resolution, every 250 ms bin from 0 to +10 s in both skip runs has 500 sent and 0 on time; skip-2's first bin has 4 on time. The median elapsed is 35.7–36.3 ms and the p90 is 35.8–36.5 ms in almost every bin. So the client got no response at all within 35 ms; these are not slow responses. The warm runs show p50 3.6–6.0 ms and p90 6–17 ms.

### The collapse is instantaneous (measured)

- **skip-1:** the 1st burst request (+0.0 ms) took 37.1 ms, and every request after it timed out.
- **skip-2:** requests at +0.0, +0.5, +1.0 and +1.5 ms got 202 in 4–6.6 ms. From the request at +2.0 ms on, everything timed out.
- **warm-1 / warm-2:** the first requests took 1.3–12 ms.

This is not a gradual capacity backlog: after 2 ms at 2,000 req/s only 4 requests have arrived. Something stopped responses for more than 35 ms right at the onset.

### The same stall shape already occurs at 100 req/s in skip, but never in warm (measured)

- **skip-1, at -4.78 s:** 6 consecutive requests 10 ms apart timed out. The next ones completed in 31.2, 21.4, 11.5 and 3.1 ms, so all held responses were released at the same instant, ±1 ms. That means nothing completed from about -4.783 s to -4.689 s, a stall of about 94 ms.
- **skip-2, at -0.077 s:** the same pattern. Nothing completed from about -77 ms to -8.5 ms (about 68 ms). The stall ended 8.5 ms before the burst, and the burst collapsed 2 ms after it started.
- **Warm runs:** no request in phases 1, 2 or 4 missed 35 ms.

### First GC-notification handling in the JVM's life lands at the burst onset in skip (inferred from JFR compiles; JFR has no GC events)

- **Skip runs:** the JMX GC-notification code was compiled for the first time at +0.07 to +0.15 s in both runs. Those methods are `com.sun.management.GcInfo::getDuration`, `getMemoryUsageAfterGc`, `GarbageCollectionNotificationInfo::getGcInfo/getGcName/...` and `MemoryUsageCompositeData::*`, which Micrometer's JvmGcMetrics listener uses. Before that, none of them had been compiled in either skip run.
- **Warm runs:** the same methods were first compiled at 9.7 s (warm-1) and 10.6 s (warm-2), early in the self-warm-up.
- **Interpretation (inference):** with an AOT cache and `-XX:+AOTCompileEagerly`, methods that have training data compile on their first invocation. So in the skip runs, the first GC notification was most likely handled about 0.1 s into the burst, and probably no GC happened in the 171 s before it.
  - The heap is fixed at 60% of 1 GiB. With 1 CPU, the JVM picks Serial GC, so collections are stop-the-world.
  - That first young collection has to copy every survivor allocated since startup, including the Spring context. Its pause is therefore likely to be long, but it is not measured here.
  - In warm runs, the first GC happens during the warm-up, so it is hidden.
- **Caveat:** this does not explain the skip-1 stall at -4.78 s at 100 rps. That stall has no GC-notification compile next to it, so its cause is unknown.

### Compiler activity

- **C2 thread is busy for the whole burst (measured).** C2 thread busy time is computed from JFR compile intervals per 250 ms.
  - Skip runs: about 0.8 in the first 250 ms, then 0.97–1.00 in every 250 ms bin from +0.25 s to +12 s.
  - Warm runs: the C2 thread is idle except for about 0.6 s around the onset. That is the `AppendValidator::parseAppend` recompile in warm-1 (632 ms, starting at -0.05 s) and about 0.3 s of compiles in warm-2.
- **Compilations started per 250 ms:**
  - skip-1: 11 C2 + 46 C1 in the first bin, then 1–11 C2 per bin, with C1 bursts of 21–33 at +1.0–1.75 s.
  - skip-2: 16 C2 + 47 C1 in the first bin, a similar pattern after.
  - warm-1: 0–7 C2 per bin, and only 7 C2 in the first 1 s.
  - warm-2: 0–3 C2 per bin.
- **C2 queue (1 s samples):**
  - skip-1: 3 at +0.3 s, 102 at +1.3 s, 99 at +2.3 s, 73 at +3.3 s, 51 at +4.4 s, 37 at +5.4 s, 6 at +6.4 s, then 7, 7, 0, 2, 4 as new requests keep arriving.
  - skip-2: 86 at +0.8 s, 90, 82, 50, 42, 26, 9, 7, 8, 9, 1, 0.
  - Warm runs: 0–1.
  - The C1 queue is 0 in all runs.
- **CPU split (measured):**
  - Container cgroup CPU (cpustat.txt): skip-1 is 0.69 in the 1 s window containing the onset, then 0.996–1.001 for every second of the burst, with 0 throttled periods because the container is pinned with cpuset. warm-1 runs 0.57–0.78.
  - Host thread sampler, 0.5 s windows, skip-1: first window after onset is C2 0.28 + other 0.56 + C1 0.02. After that, C2 0.31–0.42 + other 0.52–0.66, which saturates the CPU.
  - Host thread sampler, warm: C2 at about 0 after the first second; other at 0.54–0.60.
- **ThreadCPULoad per thread, steady burst seconds:**

  | thread | skip-1/2 | warm-1/2 |
  |---|---|---|
  | carrier `ForkJoinPool-1-worker-1` | 0.31–0.38 | 0.32–0.34 |
  | `http-nio-8080-Acceptor` | 0.06–0.09 | ≈ 0 |
  | `http-nio-8080-Poller` | 0.06–0.08 | 0.10–0.11 |
  | `ingest-writer` | 0.01–0.03 (same as at 100 rps) | 0.06–0.07 |
  | `delayScheduler` | 0.03 | 0.05–0.06 |
  | C2 | 0.36–0.60 | ~0 |

  So skip's carrier gets about the same CPU that warm's carrier needs to serve 2,000 req/s on time. The rest of the CPU goes to C2 and to accepting connections.
- **Connection churn (measured).** The harness closes a connection on every deadline miss and opens a new one for that worker's next request (`exercise_harness.py` `send()`: `connection.close()` when the deadline is exceeded).
  - In skip, where every request misses, this means one new TCP connection per request, about 2,000/s. It shows as the Acceptor thread at about 0.08 CPU, against about 0 in warm.
  - The client's `requestWrittenMs`, which includes connect, has p90 18–24 ms in the first 0.5 s. From +7 s it has p50 24–71 ms, so 3,662 requests in skip-1 missed the deadline before the request was even written.
  - The sender itself also lagged, with start delay p50 up to 232 ms in skip-2 at +9 s. That is why 497 requests in skip-2 were never sent.
- **Recovery after the burst (measured).** In phase 4 (100 rps), responses resume 1.40 s (skip-1) and 1.70 s (skip-2) after the burst ends. The first ones take 22.6 and 26.0 ms, then 7–9 ms. This is the signature of a FIFO backlog of abandoned requests draining.
  - Rough capacity estimate (inferred): if post-burst throughput is 2,000–3,000/s, the backlog at burst end was about 2,600–4,000 requests. That implies the server sustained only about 1,600–1,750 req/s during the burst, against 2,000 arriving.

### Request-path methods that were not C2-compiled at burst start

"Request path" here means these packages: com.example.events, org.apache.tomcat/catalina/coyote, com.fasterxml.jackson, org.postgresql, com.zaxxer.hikari, java.util.concurrent, org.springframework, sun.nio.ch / java.nio, java.lang.VirtualThread / jdk.internal.vm.Continuation. The count is methods with a standard (non-OSR) C2 compile starting in [t0, t0+12 s) and no C2 code before t0. It is a lower bound: methods that never reached the C2 threshold are not counted.

| run | methods | level at t0 | of these, C2 in warm before its burst | C2 installs done by 25% / 50% / 90% / last | C2 compile time (sum / max) |
|---|---|---|---|---|---|
| skip-1 | 165 (tomcat 80, nio 52, vthread 10, j.u.c 8, jackson 5, app 4, pgjdbc 3, spring 2, hikari 1) | L2: 156, never compiled: 9 | 108 (106 in both warm runs) | +3.0 / +5.5 / +9.5 / +11.4 s | 8.05 s / 0.58 s |
| skip-2 | 164 (tomcat 75, nio 56, vthread 10, j.u.c 10, ...) | L2: 155, never compiled: 9 | 110 (108) | +3.0 / +5.4 / +10.5 / +12.0 s | 8.45 s / 0.96 s |
| warm-1 | 9 | — | — | last +4.2 s | 0.03 s / 0.01 s |
| warm-2 | 6 | — | — | last +6.1 s | 0.03 s / 0.01 s |

Order in skip-1 (start..end relative to t0; skip-2 is nearly identical):

1. **The core request path, C2 in warm but only L2 (C1) in skip.**
   - `SocketChannelImpl::configureSocketNonBlockingIfVirtualThread` +0.06..0.11
   - `OncePerRequestFilter::doFilter` +0.12..0.16
   - `VirtualThread::runContinuation` +0.22..0.26
   - `IOUtil::read` +0.32..0.40
   - `NioSocketWrapper::fillReadBuffer` +0.51..0.58
   - `Http11InputBuffer::parseRequestLine` +0.71..0.83
   - `Http11Processor::service` +0.83..1.23 (392 ms)
   - `ApplicationFilterFactory::createFilterChain` +1.31..1.42
   - `PgArrays::setTextArray` +1.56..1.72
   - `RequestBody::readExactly` +1.76..1.84
   - `CompletableFuture::timedGet` +1.99..2.05
   - `Http11InputBuffer::fill` +2.16..2.32
   - `StandardWrapperValve::invoke` +2.48..2.56
   - `ConnectionHandler::process` +2.56..2.65
   - `EventsServlet::doPost` +3.19..3.25
   - `CoyoteAdapter::service` +3.33..3.63 (308 ms)
   - `EventService::accept` +3.70..3.71
   - `ByteChunk::toStringInternal` +4.88..5.06
   - `HikariProxyConnection/PgConnection::createArrayOf` +5.81..6.28
2. **From +6.4 s, the connection open/close path.** These methods were not C2 in either warm run, because there is no churn in warm:
   - `AbstractInterruptibleChannel::close` +6.41..6.52
   - `NioSocketWrapper::doClose`
   - `NioEndpoint::serverSocketAccept` +6.68..6.81
   - `ServerSocketChannelImpl::accept` +6.83..6.94
   - `NioEndpoint::setSocketOptions` (two overloads, 429 + 472 ms, +7.79..8.69)
   - `NioSocketWrapper::<init>`, `SocketWrapperBase::<init>`
   - `Http11Processor::<init>` +9.55..10.12 (577 ms)
   - `ConnectionHandler::register` +10.27..10.49
   - `Request::<init>` +10.77..11.10 (327 ms)
   - `Connector::createRequest/createResponse` until +11.4 s

   So the request path was not fully C2 until about +11.4 s (skip-1) and +12.0 s (skip-2), after the burst had ended. About 50 of the methods C2-compiled during the burst exist only because of the collapse, through connection churn.

In the warm runs, 2,353 and 2,322 methods had C2 code before traffic, against 1,448 and 1,457 in skip. Warm C2 activity in the burst is a handful of re-tier-ups of Jackson parser methods plus `parseAppend` (see below).

### Recompiles during traffic

These follow timeline.py's rule (level ≤ an earlier level of the same method), with times relative to t0.

- **skip-1:**
  - Long before the burst:
    - -74 s: `TypeFactory::_findWellKnownSimple` L3
    - -72 s: `ForkJoinPool::tryTrim` L3
    - -71 s: `ForkJoinPool::execute` L3
    - -45 s: `VirtualThread::submitRunContinuation` L4
    - -33 s: `DefaultSerializerProvider$Impl::createInstance` L4
  - At the phase-2 start, -4.9 to -4.3 s:
    - Jackson `JsonReadContext::_checkDup`, `UTF8StreamJsonParser::nextFieldName` and `_parseName` back to L3 after L4
    - `TypeInfoCache::getTypeForAlias` L4 again
    - `JdbcTemplate::execute` L4
  - During the burst:
    - +0.16 s: `Nodes$FixedNodeBuilder::accept` L4
    - +0.40 s: `Http11InputBuffer::parseRequestLine` OSR L4
    - +0.41 s: `VirtualThread::submitRunContinuation` L3, then L4 at +1.84 s (its 4th compile)
    - +0.58 s: `nextFieldName` L4
    - +1.72 s: `ForkJoinPool::execute` L4
    - plus a few trivial L1 getters
- **skip-2:** the same set at nearly the same times. Its extras are `Http11InputBuffer::parseHeader` OSR L4 at -2.45 s and `PgArrays::setTextArray` L4 at +5.78 s.
- **Warm runs:** the same kind of thing. At the phase-2 start (about -5 s), `parseAppend` and the Jackson parser methods go back to L3 after L4 and then return to L4. In warm-1 that is `parseAppend` L4 at -0.05 s, taking 632 ms. During the burst: `nextFieldName`/`_parseName` L4 at +0.6 s, `VirtualThread::schedule`, `ParserBase::close`, and in warm-2 `NioSocketWrapper::read`/`Http11InputBuffer::fill` L4 at +0.9 s.
- So deopt→recompile cycles at the onset of real traffic happen in both variants. They do not distinguish collapse from success.

## 3. Mechanism hypothesis

**Measured:**
1. At burst start in skip, the request path runs C1 (tier-2) code. 165 hot request-path methods, including `Http11Processor::service`, `CoyoteAdapter::service`, `EventsServlet::doPost`, `EventService::accept` and `VirtualThread::runContinuation`, had no C2 code. In warm, 108 of them already had C2 code.
2. The burst queues about 100 C2 compilations at once. The single C2 thread is then 100% busy from +0.25 s to +12 s and takes 0.31–0.42 of the API's single CPU. The CPU is saturated (cgroup 1.00) from the first second.
3. Responses stop within 0–2 ms of the onset. After that, not one request gets a response within 35 ms for 10 s.
4. The client then reconnects for every request. That adds about 2,000 accept/close cycles per second, with the Acceptor thread at about 0.08 CPU and the connection-setup code cold until +6.4 to +11.4 s. Meanwhile the server keeps processing requests whose clients are gone, in FIFO order.
5. The backlog drains only once arrivals drop to 100 req/s, 1.4–1.7 s after the burst.
6. Skip runs also show complete response stalls of 68–94 ms at only 100 req/s. Warm runs show none.

**Inferred:**
- **The collapse is metastable.** It needs a trigger plus a system with no spare capacity.
- **The trigger** is a stall of tens of ms at the onset, long enough for the first ~100+ requests to miss 35 ms. The best candidate is the JVM's first garbage collection: GC-notification code is compiled for the first time 0.07–0.15 s into the burst in both skip runs, and about 160 s earlier in warm. It would be a Serial, stop-the-world collection that copies everything that survived since startup. Another candidate is the same unexplained stall type seen at 100 rps.
- **Once requests miss the deadline, the late state sustains itself:**
  - Each request now also costs a TCP accept/setup/close on cold code.
  - Work is wasted on abandoned requests.
  - C2 takes about 40% of the CPU for the whole 10 s, compiling the request path (done by about +6 s) and then the connection-churn path (done by about +11.4 s).
  - With about 0.6 CPU left for the app, running C1 code, throughput stays below 2,000/s (est. 1,600–1,750/s). The queue therefore never drops back under 35 ms.
- **Why warm stays healthy:** it has the request path fully C2, no C2 contention, and its first GCs happened during the warm-up. It handles the same load with 0.58 CPU and 3–6 ms latency.
- **Why skip sometimes passes on other caches:** this fits runs where there was no onset stall, or where it was short enough not to tip the first requests over 35 ms. Without the first wave of timeouts there is no churn and no backlog, and C2 can catch up while latency stays under the deadline. The system is bistable, so the outcome depends on a few tens of ms at the onset. I did not analyze the 20k/40k runs to test this.

**What would confirm it.** Next run, add `jdk.GarbageCollection`, `jdk.GCPhasePause`, `jdk.SafepointBegin`/`jdk.SafepointEnd` and `jdk.Deoptimization` to jit.jfc, or use `-Xlog:gc,safepoint`. That would show whether a GC or safepoint pause of 35 ms or more lines up with the onset and with the 100-rps stalls. It would also help to count server-side connections accepted per second, and to test a forced `System.gc()` (or a short burst) before the skip runs' idle period ends.
