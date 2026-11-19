# Why SkipTier2IfPossible loses to warm-up: methods at C2 when traffic starts

Date: 2026-11-19. This report summarizes and extends the earlier ones: [jit-delay](../20261119-jit-delay/REPORT.md)
and [jit-5s](../20261119-jit-5s/REPORT.md). It adds a source-level investigation of HotSpot's training replay
and a follow-up experiment (fix A plus an idle-period GC).

**Naming.** `warm` and `skip` in monospace are the two test branches described in section 1. Plain-text "the
warm-up" or "the startup warm-up" means the mechanism: the self-traffic the API sends itself at startup.

## Summary

- **`warm` starts traffic with about 900 more methods at C2 than `skip`: 2,339 vs 1,436 per run.**
  - About 300 of the difference is the warm-up's own HTTP client and generated code. Serving doesn't need them.
  - **488 are server methods on the request path,** such as Tomcat, NIO, Jackson, PgJDBC, Hikari and our
    servlet. `skip` has to compile those with C2 after traffic arrives, about 300–370 C2 compiles in all. That
    keeps C2 at about 0.4 of the API's single CPU for the first 8 s of the burst.
  - This is the root of `skip`'s failures: it passed 3 of 26 runs, against 23 of 26 for `warm`.
- **The AOT cache knows about all of these methods.** All but one have training data, with stored profiles.
  Eager compilation still doesn't C2-compile them before traffic, for reasons visible in the HotSpot source
  (JDK 25):
  - **A. Dependencies only the training run initialized (108 methods).** A replayed C2 compile waits until every
    class the training compile looked at is initialized. Some of those classes belonged to the warm-up's own
    HTTP client and are never initialized in `skip`, which runs no warm-up. This one we could fix in our setup,
    and it's now done.
  - **B. A dropped upgrade (78 methods).** Each method's last dependency was met while its eager level-2 compile
    was still queued, so the C2 upgrade was skipped and never retried.
  - **C. No top-level C2 compile in training (170 methods).** Training compiled them on their own only at C1. 134
    reached C2 only inlined into callers. Eager compilation never takes these past level 2.
  - **D. Dependencies initialized by the first real requests (103 methods).** These work as designed, but only
    once traffic has started.
- **Preloaded is not initialized.**
  - The AOT cache loads and links every cached class at startup, including the request-path classes.
  - JIT dependencies wait for class initialization, which runs a class's static initializer at its first use.
    Right after preloading, 1,136 of 1,137 application and library classes in the training data were only
    linked.
  - JDK 25 skips static initializers ("AOT-initialized") only for classes defined entirely by the JDK.
- **The general hurdle is D.** Request-path code depends on classes that are initialized by the first request.
  JDK 25's eager compilation therefore can't put a service's request path at C2 before the first request, however
  much it was trained. For a service with many endpoints, the C2 backlog at first traffic grows with the number
  of endpoints; the projection puts it at 600–1,900 methods for 10 endpoints.
- **The end goal is having the request path's code ready before the first request, at little CPU cost.**
  - JDK 25 can't do this without synthetic traffic.
  - `warm` does it, at about 60 s of startup CPU.
  - JDK 28's AOT code compilation (JEP 544, in mainline) is designed for it: "preload" C2 code with
    class-initialization checks, which can run before dependencies are met.

## 1. The two test branches

Both branches use the same setup:
- **The service:** the same build of a Spring Boot API on JDK 25 (Temurin 25.0.5).
- **Its container:** 1 CPU, SerialGC, heap at 60% of container memory.
- **Around it:** Postgres and four delivery workers.
- **The AOT cache:** JEP 483/515, recorded by `scripts/train.sh` on a training load of 10k, 20k or 40k calls. It
  is used with `-XX:+AOTCompileEagerly`, which the entrypoint sets whenever a cache is present.
- **Each run:** a fresh API container, then the same checker.
  - First, the functional checks.
  - Then the load test: 1 / 100 / 2,000 / 100 / 1 req/s, where the 2,000 req/s burst lasts 10 s.
  - To pass, more than 95% of the load test's requests must be acknowledged within 35 ms.

The branches differ only in how the API gets ready for traffic:

| | `warm` | `skip` |
|---|---|---|
| Idea | Run real request code in the serving JVM before admitting traffic | Use the AOT cache's training profiles to compile hot methods early, with no synthetic traffic |
| API startup warm-up | On (`API_WARMUP_ENABLED=true`) | Off |
| Extra JVM flags | none | `-XX:+UnlockDiagnosticVMOptions -XX:+SkipTier2IfPossible` |
| Before traffic | The API sends itself representative traffic over loopback until the JIT settles: 16 concurrent `java.net.http` requests; appends, reordered retries, lookups, 400 and 404 paths. It takes 60–65 s, at most 90 s. Outside requests get 503 and the health check reports out of service until it finishes; then the warm-up rows are deleted and the API reports healthy. | The API reports healthy at about 10 s and then sits idle until 80 s after container start, so eager compilation can finish. |
| Checker starts | 10 s after healthy | 80 s after container start |
| In the follow-up (section 5) | unchanged | plus one `System.gc()` 60 s after startup (`api.warmup.gc-after=60s`) |

`SkipTier2IfPossible` is a diagnostic HotSpot flag. A method that has a usable C2 compile in the training data
skips C1's limited-profile tier (level 2) and goes straight to C2 (level 4). The hypothesis behind `skip` was
that eager compilation would then put the hot methods at C2 during the idle period, so no warm-up traffic would
be needed.

## 2. Background: why more C2 at startup, and the ideal

HotSpot starts every method in the interpreter, then compiles it with C1, which is quick but only moderately
fast code, and then with C2 once it's hot enough and profiled. C2 code is typically several times faster than
C1 code, but each C2 compile costs milliseconds to about a second of CPU. In this service, about 300 burst C2
compiles cost about 3.1 CPU-seconds.

That compile work has to happen somewhere. If it happens after traffic starts, it competes with requests for
CPU exactly when the service is least warm.
- **`skip`:** C2 used 0.37–0.41 CPU for the first 8 s of the burst, leaving almost no headroom. Any pause, such
  as a 40–80 ms GC or a 45 ms hiccup in the checker, then turned into hundreds or thousands of late requests,
  and sometimes a collapse into overload that never recovered.
- **`warm`:** it had done that work before reporting healthy. Its C2 used 0.04–0.05 CPU in the burst, the same
  stalls cost a few dozen requests, and it passed 23 of 26 runs.

**The ideal end state** is that, before the instance reports healthy:
1. Every method that will be hot in steady state is already at C2, so no JIT work competes with requests. In
   this service that's about 2,000 methods. `warm` gets close, with about 45 C2 compiles left for traffic.
2. The profiles behind that code match production, so there's no wave of deoptimizations and recompiles at
   first traffic.
3. The heap has its long-lived startup objects already promoted, so the first young GCs under load are cheap.
   This was `skip`'s second problem, fixed separately (section 5).
4. Ideally, none of this costs much CPU at startup either: compiled code is loaded rather than compiled, and
   no synthetic traffic is needed. `warm` meets 1–3, but costs about 60 s of startup CPU and needs
   side-effect-free self-traffic, which here means a separate warm-up table.

`skip` was meant to get to 1 using the AOT cache's training data instead of traffic. The rest of this report
explains why it doesn't.

## 3. Measured gap

Runs: 26 per branch (jit-5s).

![Methods at C2 when traffic starts](graphs/c2-before-traffic.png)

Counting methods C2-compiled before traffic in at least 80% of a branch's runs:

| | Methods |
|---|---|
| C2 in both branches | 977 |
| C2 in `warm`, and in 20–80% of `skip` runs (varies run to run) | 313 |
| C2 in `warm` only: server code | **488** |
| C2 in `warm` only: the warm-up's own `java.net.http` client and its `CompletableFuture` chains | 261 |
| C2 in `warm` only: generated code (lambda forms) | 37 |
| C2 in `skip` only | 101 |

The 488 server methods are the gap that matters: JDK and Jakarta 242, Tomcat 85, Jackson 54, PgJDBC 46,
Spring 18, the application 17, Hikari 16 and others 10. During traffic, `skip` did about 300–370 C2 compiles,
against about 45 for `warm`. About 230 of `skip`'s were level-2 → C2 upgrades of methods eager compilation had
left at C1.

## 4. Why eager compilation leaves them at C1 (JDK 25 source)

The relevant code is in `src/hotspot/share/compiler/compilationPolicy.cpp` and `oops/trainingData.cpp`, at tag
`jdk-25-ga`. The runtime is Temurin 25.0.5.

**The rule.** `trained_transition_from_none` sends a method straight to level 4 only if all three hold:
1. its training data has a top-level C2 compile;
2. a profile was stored;
3. `SkipTier2IfPossible && ctd->init_deps_left() == 0`.

Otherwise it gets level 2. A method waiting at level 2 is upgraded when its dependency count reaches 0, via
`replay_training_at_init` → `maybe_compile_early` → `trained_transition_from_limited_profile`.

**What counts as a dependency.** In `CompileTrainingData::notice_jit_observation`, every class that a training
C2 compile looked at, and that was initialized at the time, becomes an init dependency. The code comment
describes this as "conservatively" assuming the JIT depends on its initialization. For example,
`HikariDataSource::getConnection` has 52 dependencies, including Micrometer timer classes.

### Preloaded is not initialized

The AOT cache in JDK 25 does preload every cached class, request-path classes included, so it's natural to
expect the dependencies to be met at startup. They aren't, because loading and initialization are separate
steps:

1. **Loading and linking.** The AOT cache does this for every cached class during startup, before `main` runs.
2. **Initialization.** This runs the class's static initializer (`<clinit>`) and sets up its static fields. The
   Java Language Specification (§12.4) ties it to first active use: the first `new`, static method call or
   static field access. The JVM can't initialize classes early, because a static initializer can have side
   effects, and the order of those effects is visible to the program.

**What the dependencies wait for.** Initialization, not loading. A dependency is counted down by
`replay_training_at_init`, which `InstanceKlass::initialize_impl` calls once the class reaches
`fully_initialized`. This matters: C2 code compiled in training may assume a class's static state is set up, for
example by folding a `static final` value or leaving out an initialization check.

**Measured.** `-XX:+AOTPrintTrainingInfo` prints its dump "after all classes preloaded", with each class's state
at that moment:

| Classes in the training data | Loaded and linked, not initialized | Initialized |
|---|---|---|
| Application and libraries (Spring, Tomcat, Jackson, Hikari, Caffeine, …) | 1,136 | 1 |
| JDK | 1,031 | 308 |

So request-path classes such as Tomcat's `OutputBuffer`, `PgPreparedStatement` and Jackson's
`UTF8StreamJsonParser` are loaded early but stay merely linked until a request first uses them.

**The narrow exception.** JDK 25 can store some classes already initialized ("AOT-initialized"), so production
skips their static initializer. That's where the 308 initialized JDK classes come from. It applies only to
classes defined entirely by the JDK, which are known to have no side effects. `aotClassInitializer.cpp` says:

> the optimization (of skipping `<clinit>` for aot-init classes) is only applied to classes fully defined by the
> JDK. (A day may come when we figure out how to gracefully extend this optimization to untrusted third
> parties, but it is not this day.)

### The four causes

How we measured them:
- **The cache's training data:** `-XX:+AOTPrintTrainingInfo` dumps every compile record and its dependencies.
- **When each class was counted down:** `-Xlog:training=debug`.
- **Every eager decision and the method's dependency counter:** `-XX:+PrintTieredEvents`.
- **Which methods got compiled when:** JFR.

The data is from one instrumented `skip` run and one idle-only run with `skip`'s settings, both on the 40k cache.

![Causes and when the skip branch compiles them](graphs/gap-causes.png)

| Cause | Methods | C2 during the burst | Never C2 in the run |
|---|---|---|---|
| A. A dependency class initialized only by the training run's warm-up client (`MinimalFuture`, `CompletableFuture$Uni*`, `Http1Exchange`, `HeapByteBufferR`, …) | 108 | 35 | 64 |
| B. Last dependency met while the eager level-2 compile was still queued; `maybe_compile_early` then skips the C2 upgrade (`!compilation_is_in_queue(m)`) and nothing re-checks until traffic | 78 | 30 | 40 |
| C. No top-level C2 compile in training (C1 only; 134 were C2 only as inlinees), so the trained path stops at level 2 and C2 follows only from this run's counters | 170 | 51 | 115 |
| D. Dependency classes initialized only by the first real requests (Jackson parser, `PgPreparedStatement`, …); became eligible during the functional checks | 103 | 3 | 3 |
| Other: no training data (1), or dependencies met with no C2 and no clear cause | 29 | 7 | 13 |

In D, 92 of the 103 methods reached C2 during the functional checks.

Evidence for each cause:
- **A** was possible because the training JVM ran the startup warm-up, as `warm` does. The C2 compiles of shared
  code (`CompletableFuture`, NIO buffers, Tomcat output paths) therefore saw the warm-up client's classes, and
  `skip` never initializes them.
- **B in the idle-only run:**
  - **Last dependency met while the level-2 compile was queued:** 104 of 124 methods never got C2 during idle.
  - **Met after level 2 was installed:** 142 of 185 did.
  - **Example:** `getConnection` was queued at level 2 at 10.2799 s with `deps=9`. Its remaining dependencies
    were counted down at 10.280 s while 894 compiles waited in the C2 queue.
- **B in current JDK `master`:** still present; the queue check is unchanged.
- **D is not a bug.** It is the preloaded-but-not-initialized gap described above. During an idle period with no
  traffic, nothing initializes the request-path classes, so code that assumes they're initialized can't be
  installed.

## 5. Follow-up: fixing A, plus a GC during the idle period

Two changes, then 4 runs of each branch on a 40k cache retrained with them (experiment depfix-gc):
- **Fix A:** record the AOT cache with the startup warm-up off. Compile records depending on HttpClient classes
  went from 20,740 of 109,210 to 0 of 78,669. Both branches use the new cache.
- **Idle-period GC, `skip` only:** one `System.gc()` 60 s after startup. `skip`'s first two young collections had
  cost 37–82 ms each in the burst, because they copied the objects left live by startup.

| | `skip` before (10 runs) | `skip` after (4 runs) |
|---|---|---|
| C2 compiles during traffic | 369 | 304 |
| C2 level raises during traffic | 244 | 192 |
| C2 CPU, first 8 s of the burst | 0.41 | 0.39 |
| First GC pause in the burst | 46–56 ms | 5–11 ms (52 ms in the collapsed run) |
| On time | 2 of 10 passed | 1 of 4 passed (86.3%, 95.8%, 9.6%, 93.9%) |

`warm` passed 3 of 4 (99.4%, 97.9%, 60.3%, 99.97%).

The GC problem is gone and the compile backlog shrank by about 18%, but C2 still takes about 0.4 CPU for 8 s:
- **B, C and D are untouched.**
- **Losses now come from other small stalls.** Most came from a 30–48 ms stall in the load generator's own
  sender at the same point in every run of both branches. `warm` absorbs it in about 0.2 s; `skip` needs about
  0.6 s.
- **One `skip` run collapsed with no trigger at all.**
- **The failed `warm` run (60%) failed for an unrelated reason:** Postgres was saturated.

## 6. Projection: services with more than one endpoint

This service has two endpoints that share almost all of their request path: POST and GET `/events`. Real
services usually have many. The hurdles above scale with that, roughly as follows.

**Model.** The request-path methods that need C2 are F + K × E:
- **F** is shared request infrastructure: HTTP server, NIO, JSON core, JDBC driver, pool and JDK concurrency;
- **K** is the number of endpoints with distinct code;
- **E** is each endpoint's own code: its controller and service, its DTOs' (de)serializers, its queries.

Calibration: K = 2 must reproduce this service's 488 methods and 3.1 C2 CPU-seconds, which gives F ≈ 400 and
E ≈ 45. The application package alone contributed 17 methods, and endpoint-specific Jackson and PgJDBC paths
make up the rest. I use E = 20–150 for leaner to heavier endpoints. Compile CPU is scaled from the measured 3.1
CPU-seconds.

These are assumptions for illustration, not measurements of another service.

![Projection](graphs/projection.png)

| Endpoints (K) | Request-path methods needing C2 (E = 20 / 45 / 150) | C2 CPU-seconds, same three cases |
|---|---|---|
| 1 | 420 / 445 / 550 | 2.7 / 2.8 / 3.5 |
| 2 (this service) | 440 / 490 / 700 | 2.8 / 3.1 / 4.4 |
| 10 | 600 / 850 / 1,900 | 3.8 / 5.4 / 12 |
| 50 | 1,400 / 2,650 / 7,900 | 9 / 17 / 50 |

How many of those methods each approach gets to C2 before the endpoint's first request:
- **The `skip` approach on JDK 25, even with fix A: about none.**
  - The 21% in D, plus whatever share of A's methods depend only on request-initialized classes, become eligible
    at the endpoint's first request and are compiled in a wave right then.
  - The roughly 57% in B, C and "other" wait for this JVM's invocation counters, i.e. for load.
  - So every endpoint's first traffic carries its own compile backlog, and a cold endpoint reached for the first
    time hours later pays it then.
  - The "about none" doesn't depend on K, but the backlog grows linearly with K, and so does the window in
    which the instance has no CPU headroom.
- **The `warm` approach, with a warm-up that exercises every endpoint: about all of them,** but the same compile
  CPU moves into startup. This service already spends about 60 s of warm-up. That grows with K, and warm-up
  traffic has to stay representative of, and isolated from, real data for every endpoint.
- **AOT code with preload (JEP 544, JDK 28 mainline): most of them, at little CPU.** This is a rough guess, not
  measured.
  - Code is compiled at AOT-cache creation and loaded, not compiled, in production.
  - "AP4" preload code carries class-initialization checks, so it can run while its classes are only loaded and
    linked; the first check that finds a class uninitialized triggers its normal initialization. That removes
    the dependency gate behind A and D, and since nothing has to wait for a level-2 compile, B also goes away.
  - C becomes a smaller problem when its methods are inlined into callers that have AOT code.
  - I'd expect 65–90% of the request path to have C2-quality code before the first request.
  - The remaining 10–35% is my estimate of methods with no top-level C2 compile in training and not covered
    through inlining. It's an extrapolation from C's 35% share here.

## 7. Conclusions

- On JDK 25, AOT training data plus SkipTier2IfPossible and eager compilation C2-compile startup and shared code
  well. They can't C2-compile the request path before traffic: those compiles depend on classes that the AOT
  cache preloads but that only requests initialize. Training size doesn't matter: 10k, 20k and 40k training
  calls gave the same counts.
- Of `skip`'s problems, the ones we could fix are fixed:
  - training no longer records dependencies on the warm-up client (A);
  - an idle-period GC keeps the expensive startup GCs out of the burst.
  B is a HotSpot policy race that could be reported upstream. C and D are inherent to JIT-only replay.
- For this service on JDK 25, `warm` remains the reliable way to meet the SLA. For services with many endpoints,
  the `warm` approach's startup cost grows with the number of endpoints, and so does the `skip` approach's
  backlog. The way out is AOT code that can run before class initialization, which is what JEP 544's preload
  code targets.

## Data

Experiments, in the work-trial repository's `evaluations/`, which is not published:
- jit-5s: 52 runs of `warm` and `skip`;
- training-deps: the instrumented `skip` run, the idle-only `PrintTieredEvents` run and the training-data dump;
- depfix-gc: fix A plus the idle-period GC, 4 runs per branch.

The figures are generated by `evaluations/20261119-skip-vs-warm-report/figures.py` there.
