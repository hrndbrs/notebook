# PART 6 — PERFORMANCE

## The Quantitative Discipline

Parts 3 and 5 gave you the *cost model* — dispatch is nanoseconds, cache misses are ×100, syscalls are microseconds, network is milliseconds, and the cost of a pattern is dominated by the indirection and I/O it sits near, not its instruction count. Part 6 turns that model into *engineering practice*: how to measure, where patterns actually cost performance, when to optimize, and — most importantly — **when not to.** 

The defining discipline of performance engineering is not making things fast. It is **knowing what to leave alone.** A staff engineer's performance skill is 10% optimization technique and 90% judgment about *where the bottleneck actually is* and *whether it's worth touching.* This Part is built around that judgment, because the most expensive performance mistakes are optimizing the wrong thing — adding complexity and removing clarity for a speedup the user never perceives.

```
The performance hierarchy of what actually matters (Part 3, as priorities):

  Algorithm/complexity (O(n²) vs O(n))  ──► can be 1000x+   ← almost always the real story
  I/O: network, disk, database          ──► ms vs ns         ← the usual true bottleneck
  Memory: cache misses, allocation/GC   ──► ×10–100          ← matters in hot loops
  Dispatch: virtual calls, indirection  ──► ×1–2 (often ~0)  ← almost never the story
  ─────────────────────────────────────────────────────────
  Patterns live mostly at the BOTTOM (dispatch), yet beginners
  "optimize" by removing them — fixing the cheapest layer while
  ignoring the expensive ones above. This inversion is the core
  error this Part exists to correct.
```

---

## 6.1 — The First Principle: Measure, Don't Guess

### Why Intuition Fails

Human intuition about performance is *systematically wrong*, and this is not a personal failing — it's structural. The cost model has factors spanning *fifteen orders of magnitude* (nanoseconds to seconds), modern CPUs do invisible things (out-of-order execution, speculation, caching, JIT devirtualization — Part 3), and the actual bottleneck is usually somewhere you didn't look. Donald Knuth's famous line is the foundation of the entire discipline:

> "Premature optimization is the root of all evil." — and the *less-quoted, crucial* full context: "We **should** forget about small efficiencies, say about 97% of the time; yet we should not pass up our opportunities in that critical 3%."

The full quote is the whole lesson: **97% of code doesn't need optimizing (and optimizing it is actively harmful — adding complexity for nothing); 3% is critical and must be optimized — and you find that 3% by measuring, not guessing.** The error isn't optimization; it's optimizing *blindly* and *everywhere* instead of *measuring* to find the 3% and optimizing *there.*

### The Profiling Workflow (Non-Negotiable)

```
   1. ESTABLISH a baseline + a target. ("p99 latency is 800ms; we need <200ms.")
      ← Without a target, you optimize forever with no definition of "done."
   2. PROFILE under realistic load. Find where time/memory ACTUALLY goes.
      ← The bottleneck is almost never where you guessed.
   3. IDENTIFY the dominant cost (the hotspot consuming the most).
   4. OPTIMIZE that one thing.
   5. MEASURE again. Confirm it helped. (Sometimes it doesn't, or moves the bottleneck.)
   6. REPEAT until the target is met, then STOP.
      ← "Fast enough" is a real state. Stop there. More is waste.
```

**Amdahl's Law makes step 3 quantitative** — it tells you the *ceiling* of any optimization:
```
  If a part takes fraction P of total time, and you speed THAT part by factor S,
  overall speedup = 1 / ((1 - P) + P/S)

  Example: a function is 5% of runtime. You make it INFINITELY fast (S = ∞):
     speedup = 1 / (0.95 + 0) = 1.05x  →  a mere 5% overall gain, for infinite effort.

  Example: a function is 80% of runtime. You make it 2x faster:
     speedup = 1 / (0.20 + 0.40) = 1.67x  →  huge win for modest effort.
```
**The lesson: optimize the big fraction, never the small one** — even a *perfect* optimization of a 5%-of-runtime component yields at most 5%. This is *why* measuring first is mandatory: it tells you which fraction is big. Optimizing a beautiful, virtual-dispatch-free hot path that's 2% of runtime is mathematically near-worthless, yet it's exactly what "I removed the patterns to make it fast" engineers do.

### Profiling Tools (What to Reach For)

- **Sampling profilers** (Java async-profiler/JFR, Go pprof, Linux perf, py-spy): periodically sample the call stack; low overhead; show where time is spent. *Start here.* **Flame graphs** are the standard visualization — width = time spent, so the widest bars are your hotspots, readable at a glance.
- **Allocation/memory profilers** (heap profilers, GC logs): find allocation hotspots and GC pressure (the cache/GC costs of Part 3).
- **Distributed tracing** (Jaeger, Zipkin, OpenTelemetry): for the Part 5 distributed world — shows where a request spends time *across services*, exposing the network bottlenecks that dominate distributed systems.
- **Benchmarking harnesses** (JMH for Java, Go's `testing.B`, criterion for Rust): for microbenchmarks — but beware, microbenchmarks *lie* constantly (JIT warmup, dead-code elimination, unrealistic cache state), so they need careful setup and shouldn't be trusted over real-load profiling.

---

## 6.2 — Complexity: The Optimization That Actually Matters

Before any micro-concern, **algorithmic complexity dominates everything else** — and it's where patterns can *accidentally* introduce catastrophic costs that dwarf any dispatch overhead by factors of thousands.

### Big-O as the First Lens

```
  n = 1,000,000 elements, how many operations:
  O(1)        →  1                    (constant — a hash lookup)
  O(log n)    →  20                   (binary search, balanced tree)
  O(n)        →  1,000,000            (single scan)
  O(n log n)  →  20,000,000           (good sort)
  O(n²)       →  1,000,000,000,000    (nested loop — a TRILLION; effectively hung)
  O(2ⁿ)       →  beyond astronomical  (naive recursion — never finishes)
```

The gap between O(n) and O(n²) at a million elements is *a million-fold* — vastly larger than every micro-optimization in this Part combined. **No amount of cache-tuning or dispatch-removal rescues an O(n²) algorithm; switching it to O(n log n) is the only fix, and it's worth more than everything else.** This is why complexity is the *first* thing to examine: a single algorithmic mistake outweighs a thousand pattern-dispatch concerns.

### How Patterns Hide Complexity Traps

Patterns can *conceal* algorithmic disasters behind clean interfaces — the abstraction that aids clarity also hides the cost. Three classic pattern-induced complexity traps:

**1. The N+1 query (Repository/ORM — Part 3.8 foreshadowed this).** The single most common production performance disaster. A clean Repository call hides a query; iterating and lazy-loading a relation per element turns one logical operation into N+1 round-trips:
```java
List<Order> orders = orderRepo.findAll();        // 1 query
for (Order o : orders) {
    o.getCustomer().getName();                   // lazy-load → 1 query PER ORDER
}
// 1000 orders = 1001 queries, each a network+disk round-trip (Part 3.8 cost).
// The Repository's clean interface HID 1000 milliseconds of I/O.
// Fix: eager/batch fetch → 1 or 2 queries. A 500x improvement.
```
The Repository pattern didn't cause this — but its abstraction *hid* it, which is the lesson: **clean abstractions can conceal expensive operations, so you must profile *through* them.** This is the dark side of the convenience Part 2D praised.

**2. Observer notification storms (Part 2C/3.7).** An O(n) observer list where each `update()` triggers more changes can cascade to O(n²) or worse — one change fires n notifications, each firing more. At scale, a single state change can avalanche into millions of notifications. The pattern's loose coupling hides the cascade's cost.

**3. Decorator chain depth.** Each decorator adds a layer (Part 3.7's stack-unwinding); a deep chain in a hot path multiplies per-call cost by the chain length. Usually negligible, but in a tight loop with a 10-deep chain, measurable.

**The discipline:** when profiling reveals a hotspot *behind* a pattern's interface, look *through* the abstraction at what it's actually doing — the clean method call may hide a query, a cascade, or a chain. The pattern's job is clarity; performance analysis must pierce that clarity to see the real operations.

---

## 6.3 — Latency vs. Throughput: The Distinction That Governs Optimization

Two performance metrics, *frequently confused*, requiring *opposite* optimizations — confusing them is a common senior-level error.

```
  LATENCY    = time for ONE operation to complete (how fast is one request?)
               measured in time: ms, µs. "p99 latency = 200ms."
  THROUGHPUT = operations completed per unit time (how many can we do?)
               measured in rate: req/s, ops/s. "50,000 req/s."

  Analogy — a highway:
    Latency    = how long YOUR car takes to drive the route.
    Throughput = how many cars pass through per hour.
  They TRADE OFF: adding lanes (throughput) doesn't make your car faster
  (latency); a faster speed limit (latency) might reduce total cars if
  spacing increases. Optimizing one can HURT the other.
```

**Why this governs pattern/architecture choices:**
- **Batching** improves throughput, *worsens* latency (you wait to accumulate a batch before processing — each item waits longer, but more items/sec overall). The Unit of Work (Part 2D) batches DB writes — great for throughput, adds latency to any single write.
- **Caching** improves *both* (cache hits are fast AND free up capacity).
- **Async/queuing** (Observer, message brokers — Part 5) improves throughput and *perceived* latency (the caller doesn't wait) but worsens *end-to-end* latency (the work happens later) and adds eventual-consistency complexity.
- **Parallelism** improves throughput, and *can* improve latency (if one task splits across cores) or *worsen* it (coordination/contention overhead).

**You must know which you're optimizing for, because they conflict.** A real-time trading system optimizes *latency* (microseconds matter per trade; never batch). A batch analytics pipeline optimizes *throughput* (total records/hour; batch aggressively, latency irrelevant). A web API optimizes *tail latency* (p99 — the *slow* requests, because the user who waits 2 seconds is the one who leaves, even if the median is 50ms). **Optimizing the wrong metric is wasted effort that can degrade the metric you actually care about** — adding batching to a latency-critical path makes it worse while improving a throughput number nobody needed.

### Percentiles, Not Averages (A Critical Subtlety)

```
  NEVER optimize for the average latency. Optimize for the TAIL (p95, p99, p99.9).

  Why: averages hide the pain. If median = 20ms but p99 = 2000ms,
  1% of requests take 2 seconds — and at scale, EVERY user hits p99
  regularly (a page making 100 calls almost certainly hits one p99).
  The average (maybe 40ms) looks fine while users suffer.

  Tail latency is dominated by: GC pauses, cache misses, lock contention,
  cold caches, retries, the slowest dependency. These are exactly the
  Part 3 cost-model effects — they don't show in averages, they show in tails.
```

This is why production systems alert on p99, not mean — and why "the average is fine" is a dangerous phrase. *The tail is the user experience.*

---

## 6.4 — Where Patterns Actually Cost Performance (And Where They Don't)

Now the central question of this Part, answered with Part 3's cost model: **when do design patterns genuinely hurt performance, and when is worrying about them a waste?**

### The Two Regimes (Memorize This Decision)

```
  REGIME A — HOT, CPU-BOUND, IN-MEMORY INNER LOOPS
  (game render loop, physics sim, numerical kernel, parsing millions
   of records, high-frequency trading hot path)
  → Patterns CAN cost real performance here (Part 3.4):
     - virtual dispatch in a megamorphic tight loop (mispredictions)
     - pointer-chasing through composed objects (cache misses, ×100)
     - per-element allocation (GC pressure)
     - lost vectorization/prefetching from scattered objects
  → Here, prefer data-oriented design: contiguous arrays, no virtuals
    in the hot path, struct-of-arrays, minimal indirection.

  REGIME B — EVERYTHING ELSE
  (web request handlers, business logic, anything touching DB/network/disk,
   I/O-bound code, human-paced operations, request-scoped work)
  → Patterns cost effectively NOTHING here:
     - dispatch is nanoseconds against milliseconds of I/O (×1,000,000)
     - the DB query / network call / disk read dwarfs ALL dispatch
     - JIT devirtualization (Part 3.5) makes monomorphic calls ~free anyway
  → Here, optimize for CLARITY. Use patterns freely. Removing them for
    "speed" gains nothing measurable and costs maintainability.
```

**The single most important performance judgment about patterns:** *the vast majority of code is Regime B, where pattern overhead is invisible; only a tiny, identifiable fraction is Regime A, where it matters — and you find that fraction by profiling, not assuming.* The beginner error is treating all code as Regime A (stripping patterns everywhere "for performance," gaining nothing, losing clarity). The opposite error is treating all code as Regime B (heavy patterns in a render loop, tanking framerate). **Knowing which regime a given piece of code is in — which is almost always obvious (is there a network/DB call? then it's B) — is the entire skill.**

### Quantifying It

```
  A web request handler that does:
     1 virtual Strategy call    ≈ 3 ns       (Part 3.3)
     1 database query           ≈ 5,000,000 ns (5 ms)
  The Strategy call is 0.00006% of the request. Removing it for
  "performance" is optimizing 0.00006% — Amdahl's Law says the
  maximum possible gain is 0.00006%. It is, quite literally,
  a waste of engineering time that makes the code worse.

  A game loop processing 1,000,000 entities at 60 FPS:
     budget = 16.6 ms / frame = 16.6 ns PER ENTITY.
     A virtual call + cache miss per entity ≈ 200+ ns.
     → blows the frame budget 12x over. Here patterns DO matter,
       and data-oriented design (no virtuals, contiguous data) is essential.
```

The same Strategy pattern: **invisible in the web handler, fatal in the game loop.** Context determines everything. This is Part 3's cost model applied as performance judgment.

---

## 6.5 — When Patterns *Improve* Performance

A crucial counterpoint juniors miss: **several patterns exist *specifically* to improve performance** — they're optimizations *expressed as patterns.* Removing them to "speed things up" does the opposite.

- **Flyweight (2B.6)** — the canonical performance pattern. Sharing intrinsic state collapses memory (gigabytes → megabytes), which *also* improves cache locality (more fits in cache — directly attacking Part 3.4's cache-miss cost) and reduces GC pressure. It *is* a memory optimization.
- **Object Pool** (a creational pattern we noted in Part 1) — reuse expensive-to-create objects (DB connections, threads, large buffers) instead of constructing/destroying them. HikariCP (connection pool) exists because opening a DB connection is *enormously* expensive (TCP handshake + auth + ~ms); pooling amortizes it. A massive, essential optimization, expressed as a pattern.
- **Proxy (virtual/caching) (2B.7)** — a caching proxy makes repeated calls free (cache hit); a virtual proxy defers expensive loads until needed (lazy init — don't pay for what you don't use). Both are performance patterns.
- **Lazy initialization / lazy Singleton** — defer expensive construction until first use; if never used, never pay.
- **Decorator for caching** — wrap an expensive operation with a memoizing decorator; transparent caching (Part 4.1's observable-decorator structure, repurposed for memoization).

**The lesson: patterns are not uniformly a performance *cost* — some are the performance *solution.* The competent engineer knows which is which** and reaches for Flyweight/Pool/caching-Proxy *to optimize*, while only stripping dispatch-heavy patterns from proven Regime-A hotspots.

---

## 6.6 — The Memory Dimension: Allocation, GC, and Cache

Part 3.4 established cache as the dominant micro-cost. Performance engineering operationalizes managing memory — often a bigger lever than CPU in managed-language systems.

### Garbage Collection Pressure

In GC languages (Java, Go, C#, JS), *allocation is cheap but collection is not* — and **GC pauses are a primary cause of tail latency (6.3).** Every allocated object eventually must be collected; high allocation rates → frequent GC → pauses that spike p99. Pattern-heavy code allocates more (every Decorator wrapper, every Command object, every Memento snapshot, every Strategy instance is an allocation):

```
  Allocation-heavy pattern usage in a hot path:
     - new Command per operation  → garbage
     - new Memento per undo       → garbage (and large!)
     - decorator chains rebuilt    → garbage
  → high allocation rate → GC pressure → tail-latency spikes (p99).

  Mitigations (Regime A only — don't do this in Regime B):
     - reuse objects (Object Pool, Flyweight)
     - avoid per-element allocation in hot loops
     - prefer value types / stack allocation where the language allows
       (Go structs, Java records/Valhalla, C# structs, Rust stack values)
     - escape analysis (JIT) sometimes eliminates allocations automatically
```

**Again, regime-dependent:** in a web handler, allocating a few Command/DTO objects per request is *nothing* (collected trivially, dwarfed by I/O). In a million-iteration hot loop, per-iteration allocation is *death* by GC pressure. Profile allocation (heap profiler) to know which you have.

### Cache-Conscious Design (Regime A)

When you *are* in a hot loop, Part 3.4's cache lesson becomes the optimization:
```
  AoS (Array of Structs / objects — pattern-friendly, cache-hostile):
     [obj{a,b,c,vptr}][obj{a,b,c,vptr}]...  scattered, each access a potential miss

  SoA (Struct of Arrays / data-oriented — cache-friendly, pattern-hostile):
     a: [a0,a1,a2,a3...]   ← processing all 'a's reads contiguous cache lines,
     b: [b0,b1,b2,b3...]     prefetcher kicks in, near-zero misses, vectorizable
     c: [c0,c1,c2,c3...]

  → In hot numerical/entity loops, SoA + no virtuals can be 10-50x faster (Part 3.4).
    This is WHY game engines (ECS architectures) and HPC reject OO patterns
    in their hot paths — not dogma, measured cache behavior.
```
The tradeoff is stark: SoA/data-oriented is fast but *loses* the OO clarity patterns provide. So you apply it *only* in the profiled hot path (Regime A), keeping clean patterns everywhere else (Regime B). Modern game engines literally do this — clean OO for game logic, brutal data-oriented ECS for the hot simulation/render loop.

---

## 6.7 — Distributed Performance: The Network Dominates (Part 5 Quantified)

In distributed systems (Part 5), *the network is the bottleneck*, and every Part-3.8 cost is now your dominant constraint. Pattern-dispatch performance is utterly irrelevant here; the optimizations are entirely about *reducing and tolerating network cost.*

```
  Distributed performance priorities (in order):
  1. REDUCE round-trips. Each is ms (vs ns in-process) — Part 5's million-fold gap.
     - batch/aggregate calls (BFF pattern, GraphQL — fetch in one round-trip)
     - avoid the distributed N+1 (calling a service per item in a loop — fatal)
     - co-locate data with computation
  2. CACHE aggressively (caching Proxy at scale — CDN, Redis). A cache hit
     turns a 50ms cross-region call into a 1ms local read.
  3. PARALLELIZE independent calls (don't serialize 10 calls = 500ms;
     fire them concurrently = 50ms). Fan-out/fan-in.
  4. TOLERATE latency: async (Observer/events), so the user doesn't wait
     for slow downstream work.
  5. TAIL latency is brutal here: a request fanning to 100 services hits
     each service's p99 — so the OVERALL p99 is dominated by the slowest
     of 100 dependencies. (This is why p99.9 of dependencies matters and
     why hedged requests / backup requests exist.)
```

**The defining distributed-performance insight:** an operation that was acceptable as ten in-process calls (Part 5) becomes a performance catastrophe as ten *serial* network calls (10 × 50ms = 500ms) — so distributed performance is largely about *minimizing round-trips and parallelizing the unavoidable ones.* This is a *direct* consequence of the cost model: the boundary you cross determines the cost, and network boundaries cost a million times more, so you cross them as rarely and as concurrently as possible. The chatty distributed monolith (Part 5.4) is slow *precisely because* it ignores this.

---

## 6.8 — When NOT to Optimize (The Most Important Section)

The discipline that separates senior from junior performance engineering is **restraint.** Most optimization is *negative-value* — it adds complexity, removes clarity, introduces bugs, and consumes engineering time, all for a speedup nobody perceives. Knowing when to *stop* (or never start) is the master skill.

### The Rules of Not Optimizing

**1. If it's not measured, it's not a bottleneck — leave it alone.** Never optimize on suspicion. Profile first; optimize only proven hotspots. Code that *feels* slow but profiles as 0.1% of runtime should *never* be touched.

**2. If it's fast enough, stop.** "Fast enough" is a real, definable state (the target from 6.1). A web endpoint at 50ms when the requirement is <200ms is *done* — making it 30ms is pure waste (the user can't perceive it, and you've spent time + added complexity for nothing). Optimization without a target is a bottomless time-sink.

**3. In Regime B (I/O-bound, the majority of code), pattern overhead is invisible — optimize for clarity, not speed.** Removing a Strategy or Decorator from a web handler "for performance" is the canonical waste: Amdahl's Law caps the gain at ~0%, and you've made the code worse. *The clarity patterns provide is worth more than nanoseconds you can't measure.*

**4. Optimization has costs that must be weighed:** optimized code is usually *less readable* (data-oriented SoA loses OO clarity; manual inlining obscures intent; caching adds invalidation bugs — "there are only two hard problems: cache invalidation and naming things"), *harder to maintain* (more invariants, more edge cases), *more bug-prone* (concurrency optimizations especially), and *consumes engineer-time* (the scarcest resource). An optimization is only worth it when the *measured* gain on a *proven* hotspot exceeds *all* these costs.

**5. Hardware is often cheaper than engineering.** Sometimes the right "optimization" is a bigger instance, more RAM, or another replica — vastly cheaper than weeks of engineer time micro-optimizing. Scaling out/up is a legitimate, often *optimal*, answer (until it isn't — at extreme scale, efficiency compounds into real money, which is why hyperscalers *do* micro-optimize).

**6. Premature optimization corrupts design.** Optimizing before you understand the problem locks in structures (denormalized data, caches, manual memory layout) that fight every future change. The clean design (good patterns, clear boundaries) is *easier to optimize later* — when you actually know the hotspot — than a prematurely-optimized mess is to *re-*optimize. **A clean design defers optimization cheaply; a premature optimization makes everything expensive forever.**

### The Decision Framework

```
   Should I optimize this?
   ├─ Have I PROFILED and confirmed it's a real hotspot?
   │     NO → STOP. Don't optimize. (You're guessing.)
   │     YES ↓
   ├─ Is it a significant fraction of total time? (Amdahl)
   │     NO (small %) → STOP. Max gain is tiny. Not worth it.
   │     YES ↓
   ├─ Am I already meeting the performance TARGET?
   │     YES → STOP. "Fast enough" is done.
   │     NO ↓
   ├─ Is the cheapest fix more hardware / a config change?
   │     YES → do that first (often cheaper than engineering).
   │     NO ↓
   ├─ Does the measured gain exceed the cost in clarity/maintainability/bugs/time?
   │     NO → STOP. Net-negative optimization.
   │     YES → OPTIMIZE. Then MEASURE again. Then STOP when target met.
```

This framework *is* the performance discipline. Most honest answers to the first question are "no, I haven't profiled" — which means **most contemplated optimizations should never happen.** The senior engineer's instinct is *skepticism toward optimization*, not enthusiasm for it.

---

## 6.9 — Performance Synthesis

Consolidate the quantitative discipline:

**1. Measure, never guess.** Human performance intuition is systematically wrong across fifteen orders of magnitude of cost. Profile under realistic load; let flame graphs and traces find the *actual* bottleneck (always somewhere you didn't expect). Amdahl's Law: optimize the big fraction, ignore the small one.

**2. The cost hierarchy is: algorithm ≫ I/O ≫ memory/cache ≫ dispatch.** Patterns live at the *bottom* (dispatch), yet beginners "optimize" by removing them — fixing the cheapest layer while the expensive ones (an O(n²) loop, an N+1 query, a serial fan-out of network calls) go ignored. Look *up* the hierarchy first.

**3. Two regimes, and knowing which you're in is the whole skill.** Regime A (hot, CPU-bound, in-memory loops — games, sims, kernels): patterns *can* cost real performance via dispatch and cache misses; prefer data-oriented design. Regime B (everything I/O-bound — web, business logic, distributed): pattern overhead is invisible against I/O; optimize for *clarity*, use patterns freely. The same Strategy is invisible in a web handler and fatal in a render loop — context is everything.

**4. Latency ≠ throughput, and they trade off.** Know which you're optimizing; optimizing the wrong one is wasted effort that can degrade the one you need. Optimize *tail* latency (p99), never averages — the tail *is* the user experience, and it's dominated by exactly the GC/cache/contention effects of the cost model.

**5. Some patterns ARE optimizations.** Flyweight, Object Pool, caching Proxy, lazy init — these *improve* performance; removing them slows things down. Patterns are not uniformly a cost; know which solve performance problems.

**6. In distributed systems, the network is the only thing that matters.** Minimize round-trips, parallelize the unavoidable ones, cache aggressively, tolerate latency with async. Dispatch is irrelevant; the million-fold network cost (Part 3.8/Part 5) is your entire performance story.

**7. The master skill is restraint — knowing when NOT to optimize.** Most optimization is net-negative: unmeasured, insignificant (Amdahl), past the "fast enough" target, or costing more in clarity/maintainability/bugs/time than the gain. Profile-or-don't-touch; "fast enough" is done; clean designs defer optimization cheaply while premature optimization corrupts design permanently. **The senior instinct is skepticism toward optimization, not enthusiasm for it** — because the most expensive performance mistakes are optimizing the wrong thing, and the second most expensive is optimizing the right thing past the point of value.

You can now reason quantitatively about performance from algorithm to dispatch, locate the true bottleneck by measurement rather than intuition, judge whether a pattern's overhead matters in a given context (almost always: no), reach for the patterns that *are* optimizations when appropriate, and — most valuably — *resist* the overwhelmingly common urge to optimize things that don't matter. This turns Part 3's cost model and Part 5's latency math into disciplined engineering judgment about where effort actually pays.

Part 7 (Security) will examine patterns through the threat lens — how patterns create or close attack surfaces (the Decorator ordering vulnerability from Part 4, deserialization in Prototype/Memento, the trust boundaries a Proxy/Facade enforces or leaks), and the defensive patterns that secure systems.