# PART 3 — INTERNALS

## What Happens Beneath the Patterns

Every pattern in Part 2 rests on one machine-level mechanism: **polymorphic dispatch** — the ability to call `obj.method()` and have the *right* code run based on the object's actual type at runtime. Strategy swaps it, Decorator chains it, Visitor double-dispatches it, State delegates through it. If you understand what polymorphism *costs the machine* — in memory, CPU cycles, cache behavior, and indirection — you understand the *true price tag* of every pattern, which is the difference between a senior who says "patterns add indirection" as a slogan and a staff engineer who can say *exactly how much* indirection and *when it matters*.

This Part descends through the layers: memory layout of objects, the vtable mechanism, what the CPU does on a virtual call, cache effects, then the OS/network/database layers where architectural-scale patterns live. We trace real execution step by step.

---

## 3.1 — The Foundation: How Objects Live in Memory

Before dispatch, you must know what an object *is* in memory. Patterns manipulate object references; to understand the cost of that manipulation, picture the actual bytes.

### A Plain Object's Layout

Consider a simple class with no inheritance, no virtual methods:

```java
class Point {
    int x;   // 4 bytes
    int y;   // 4 bytes
}
```

In memory (conceptually, in a JVM/C++-like runtime), an instance is a contiguous block:

```
Address    Contents
─────────────────────────────────
0x1000  ┌──────────────────┐
        │ object header    │  ← bookkeeping (see below)
0x1008  ├──────────────────┤
        │ x = 5  (4 bytes) │
0x100C  ├──────────────────┤
        │ y = 7  (4 bytes) │
0x1010  └──────────────────┘
```

The **object header** is metadata the runtime needs. In the HotSpot JVM it's ~12–16 bytes containing:
- **Mark word:** identity hash, GC age bits, lock state (for synchronization).
- **Class pointer (klass pointer):** *a pointer to the object's class metadata* — this is the crucial one for polymorphism. It answers "what type am I, really?"

In C++, a plain struct with no virtual methods has **no header at all** — it's just the fields, raw. The moment you add a `virtual` method, C++ silently inserts a hidden pointer (the vptr — section 3.2). This is why C++ developers care about "is this class polymorphic?" — it changes the object's size and layout.

### The Two Memory Regions: Stack vs. Heap

Objects and references live in different places, and patterns traffic in *references*, so you must distinguish them:

```
        STACK (per-thread)              HEAP (shared)
   ┌─────────────────────┐       ┌──────────────────────┐
   │ local variable:     │       │                      │
   │  point ──────────────┼──────►│  Point object        │
   │  (a reference/       │  ref  │  header | x=5 | y=7  │
   │   pointer, 8 bytes)  │       │                      │
   │                      │       │                      │
   │ fast, LIFO,          │       │ slower, GC-managed,  │
   │ auto-cleaned         │       │ dynamically sized    │
   └─────────────────────┘       └──────────────────────┘
```

- **Stack:** holds local variables, function parameters, return addresses. Fast (just move a stack pointer), automatically reclaimed on function return, small, thread-local. In Java, *references* (and primitives) live here; in C++, objects *can* live entirely on the stack.
- **Heap:** holds dynamically allocated objects. Slower (allocation involves finding free space; reclamation involves GC or manual `free`), shared across threads, flexible size.

**Why this matters for patterns:** Every pattern that "holds a reference to another object" — which is *almost all of them* (composition!) — creates a **pointer indirection**: the holding object stores a reference; to use the held object, the CPU must *follow that pointer* to the heap. A Decorator chain of five wrappers is *five pointer-follows*, each potentially a cache miss (3.4). This is the literal, physical meaning of "indirection has a cost."

```
Decorator chain in memory:

  outerWrapper ──►[Whip   |inner─┐]
                                 └──►[Caramel|inner─┐]
                                                    └──►[Milk |inner─┐]
                                                                     └──►[Espresso]

  cost() must follow 4 pointers, hopping across the heap,
  before reaching the base object. Each hop = potential cache miss.
```

---

## 3.2 — The Heart of It All: Virtual Method Tables (vtables)

This is the single most important internal mechanism in this entire course. **Every polymorphic call — every `strategy.execute()`, every `observer.update()`, every `state.handle()` — resolves through a vtable.** Understand this and you understand the machinery of *all* behavioral and most structural patterns.

### The Problem Dispatch Solves

```java
PaymentProcessor p = getProcessor();  // could be Stripe, PayPal, anything
p.charge(amount);                     // which charge() runs??
```

At *compile time*, the compiler does **not know** the concrete type of `p` — `getProcessor()` might return any `PaymentProcessor`. So it cannot hardcode the address of the `charge` method. The decision must be made at *runtime*, based on the object's *actual* type. This is **dynamic dispatch** (a.k.a. late binding, runtime polymorphism). The vtable is how it's implemented.

### What a vtable Is

A **virtual method table (vtable)** is a per-*class* array of function pointers — one slot per virtual method, each holding the address of that class's implementation of the method. There is **one vtable per class** (not per object), shared by all instances of that class. Each *object* carries a hidden pointer (the **vptr**) to its class's vtable.

```
Classes:                          vtables (one per class, in static memory):

interface Shape {                 Shape's subtypes each get a vtable:
    double area();
    void draw();                  Circle_vtable:        Square_vtable:
}                                 ┌──────────────────┐  ┌──────────────────┐
                                  │[0] area → Circle  │  │[0] area → Square  │
class Circle implements Shape{    │      ::area@0x4000│  │      ::area@0x5000│
    double area(){...}@0x4000     │[1] draw → Circle  │  │[1] draw → Square  │
    void draw(){...}  @0x4010     │      ::draw@0x4010│  │      ::draw@0x5010│
}                                 └──────────────────┘  └──────────────────┘
class Square implements Shape{          ▲                      ▲
    double area(){...}@0x5000           │                      │
    void draw(){...}  @0x5010    Objects point to their class's vtable:
}
                                 circle object:        square object:
                                 ┌──────────────┐      ┌──────────────┐
                                 │ vptr ────────┼──────│ vptr ────────┼──┐
                                 │ radius = 5   │  │   │ side = 4     │  │
                                 └──────────────┘  │   └──────────────┘  │
                                                   └─► Circle_vtable      └─► Square_vtable
```

**Key facts to burn in:**
- The **vtable is per-class**, created once (at class-load in Java, at compile/link in C++).
- The **vptr is per-object**, set when the object is constructed, pointing to its class's vtable.
- Each virtual method has a **fixed slot index** in the vtable, *consistent across all subtypes* — `area` is always slot 0, `draw` always slot 1, for every `Shape` subtype. This consistency is what makes dispatch a simple array index.

### How a Virtual Call Executes — Step by Step

When you write `shape.area()` and `shape` could be any `Shape`, the compiler generates roughly this (in pseudo-assembly):

```
;  shape.area()   where shape is in register R1 (points to the object)

(1)  vptr   = LOAD [R1 + 0]        ; read the vptr from object header (offset 0)
(2)  method = LOAD [vptr + 0]      ; read slot 0 (area) from the vtable
(3)  CALL   method                 ; indirect call to that address
```

Three steps:
1. **Load the vptr** from the object (one memory read).
2. **Load the method address** from the vtable at the known slot offset (a second memory read).
3. **Indirect call** to that address (jump to a location held in a register — the CPU can't know the target until step 2 completes).

Contrast with a **non-virtual (static) call**, where the compiler knows the exact method at compile time:

```
;  point.staticMethod()  — compiler knows the address

(1)  CALL  0x3000                  ; direct call to a known, hardcoded address
```

One step, no memory reads for dispatch, and the CPU knows the target instructions in advance (can prefetch them). **This difference — two extra memory loads plus an unpredictable indirect jump versus a single direct call — is the literal CPU cost of polymorphism, and therefore the literal cost floor of every pattern built on it.**

### Single Dispatch and Why Visitor Needs Two

Notice the vtable dispatches on **exactly one type** — the type of the object whose vtable we read (`shape`). This is **single dispatch.** The method selected depends on *one* receiver's runtime type. This is *precisely* why the Visitor pattern (2C.10) is so convoluted: it needs the method to depend on *two* runtime types (element AND visitor), but the machine only gives you one-type dispatch per call. Visitor's "double dispatch" is literally **two vtable lookups chained**: `element.accept(v)` does one vtable dispatch (on element's type), landing in code where the element type is now statically known, which then calls `v.visit(this)` doing a *second* vtable dispatch (on visitor's type). Two single dispatches = the double dispatch the machine won't give you directly. Now you see Visitor's complexity is *forced by the vtable's single-dispatch nature* — it's a workaround for a hardware/language limitation, exactly as Part 1 claimed.

---

## 3.3 — Execution Trace: A Strategy Call, End to End

Let's trace a complete polymorphic pattern call at the machine level, so the abstract becomes concrete. We'll use Strategy because it's the cleanest.

```java
interface RouteStrategy { Route build(Point a, Point b); }
class FastestRoute implements RouteStrategy { public Route build(...){...} }

class Navigator {
    private RouteStrategy strategy;     // a reference (8-byte pointer)
    Route navigate(Point a, Point b) {
        return strategy.build(a, b);    // THE polymorphic call
    }
}

// somewhere:
Navigator nav = new Navigator(new FastestRoute());
nav.navigate(home, work);
```

**Full execution trace of `strategy.build(a, b)`:**

```
STATE BEFORE THE CALL:
  Heap:
    Navigator obj @0x2000: [ vptr | strategy=0x3000 ]
    FastestRoute obj @0x3000: [ vptr=0x9000 ]          (no instance fields)
    FastestRoute_vtable @0x9000: [ slot0: build → 0xA100 ]

  Registers (inside navigate()):
    R1 = 0x2000   (this — the Navigator)

STEP 1 — Load the strategy reference from the Navigator:
    R2 = LOAD [R1 + offset_of_strategy]   ; R2 = 0x3000 (the FastestRoute object)
    ► one memory read (the field load)

STEP 2 — Load the vptr from the strategy object:
    R3 = LOAD [R2 + 0]                    ; R3 = 0x9000 (FastestRoute's vtable)
    ► second memory read (vptr load) — likely a different cache line than step 1

STEP 3 — Load the method address from the vtable:
    R4 = LOAD [R3 + 0]                    ; R4 = 0xA100 (FastestRoute::build)
    ► third memory read (vtable slot load) — yet another cache line

STEP 4 — Set up arguments (a, b into argument registers), then:
    CALL R4                               ; indirect call to 0xA100
    ► the CPU jumps to an address it only learned one instruction ago
      → branch predictor must GUESS the target (see 3.5)

STEP 5 — Execute FastestRoute::build, return the Route.
```

So a single Strategy call costs, at minimum: **one field load + two dispatch loads + one indirect (hard-to-predict) call**, versus a direct function call's single jump. That's the physical reality of "Strategy adds indirection." In a hot loop calling this millions of times, it's measurable; in a request handler called once per HTTP request, it's *utterly negligible* (the network round-trip dwarfs it by a factor of millions). **This is the foundation of pattern performance judgment: the cost is real but tiny, so it matters only in hot paths — a theme section 3.5 and Part 6 develop fully.**

---

## 3.4 — Cache Effects: Where Patterns Actually Cost You

The vtable's *instruction count* is small. The real performance story is **memory and cache** — and this is where patterns can genuinely hurt at scale, in ways most engineers never learn.

### The Memory Hierarchy (Know These Numbers)

Modern CPUs are *starved for data*. The CPU is vastly faster than main memory, so it relies on layers of cache:

```
                  Approx. latency      Size            Analogy (if L1 = 1 second)
  ─────────────────────────────────────────────────────────────────────────────
  CPU register     0 cycles            ~hundreds bytes  instant
  L1 cache         ~1-4 cycles         ~32-64 KB        1 second
  L2 cache         ~10-15 cycles       ~256KB-1MB       ~10 seconds
  L3 cache         ~40-60 cycles       ~8-32 MB         ~1 minute
  Main memory(RAM) ~200-300 cycles     GBs              ~5 minutes
  SSD              ~100,000+ cycles    TBs              ~days
  Network/disk     millions+ cycles    -                ~months
```

The jump from cache to RAM is **~100x**. A **cache miss** — needing data that isn't in cache, forcing a fetch from RAM — stalls the CPU for hundreds of cycles, during which it could have executed hundreds of instructions. **Cache misses, not instruction count, dominate the performance of pointer-heavy code** — and pattern-heavy OO code is pointer-heavy by nature.

### Cache Lines and Spatial Locality

Memory is fetched in **cache lines** (typically 64 bytes), not individual bytes. When you read one byte, the CPU loads the whole 64-byte line containing it. This rewards **spatial locality** (data used together stored together) and **temporal locality** (recently-used data reused soon).

### How Patterns Sabotage the Cache

Here's the crux. Consider processing a million objects:

**Pattern-heavy / OO layout (array of pointers to scattered heap objects):**
```
  array:  [ptr0][ptr1][ptr2]...   (contiguous, but just pointers)
            │     │     │
            ▼     ▼     ▼
  heap:  [obj0 @0x...][...gap...][obj1 @0x...][...][obj2 @0x...]
         scattered ALL OVER the heap (allocated at different times)

  Iterating + calling a virtual method on each:
   - follow ptr0 → CACHE MISS (obj0 not in cache, fetch from RAM: ~200 cyc)
   - load vptr → maybe another line
   - load vtable slot → another line (vtable is elsewhere)
   - indirect call → possible instruction-cache miss + misprediction
   - follow ptr1 → CACHE MISS again (obj1 elsewhere)
   ... a million times. Mostly STALLING, not computing.
```

**Data-oriented layout (contiguous array of plain values, no virtuals):**
```
  data:  [v0][v1][v2][v3][v4]...   all packed contiguously, 64-byte lines
          ───────────────────►
  Iterating:
   - read v0 → loads a whole cache line (v0..v15 in one fetch)
   - v1..v15 are ALREADY IN CACHE → ~1 cycle each (free!)
   - direct, predictable processing, no dispatch
   - hardware PREFETCHER detects the linear scan, loads ahead
   ... blazing fast, CPU fully utilized.
```

The pattern-heavy version can be **10x–50x slower** in tight loops — not because of vtable *instructions* (those are cheap) but because of **cache misses from chasing scattered pointers** plus lost prefetching and branch misprediction. This is the deep reason game engines, high-frequency-trading systems, and numerical code often *reject* heavy OO patterns in hot paths in favor of **data-oriented design** (structure-of-arrays, no virtuals). It's also the deep justification for **Flyweight** (2B.6) — sharing intrinsic state shrinks the data so more fits in cache.

**The staff-engineer synthesis:** The cost of patterns is not the few extra instructions of dispatch — it's **memory indirection destroying cache locality** when you process *many* objects in a *hot loop*. Therefore:
- In **hot inner loops over millions of objects** (rendering, physics, number-crunching): patterns can be genuinely expensive; prefer data-oriented layouts.
- In **request-scoped, I/O-bound, or human-paced code** (web handlers, business logic, anything touching a DB or network): the dispatch and cache cost is *invisible* — nanoseconds against milliseconds — so use whatever patterns maximize *clarity*. The bottleneck is the database/network, never the vtable.

Knowing *which regime you're in* is the entire skill. Beginners optimize the wrong one (removing patterns from web code "for speed," gaining nothing, losing clarity) or ignore the right one (heavy patterns in a render loop, tanking framerate).

---

## 3.5 — CPU Internals: Branch Prediction and Speculative Execution

One more CPU mechanism completes the dispatch-cost picture: how the CPU handles the *indirect call* in step 4 of our trace.

### The Pipeline and Why Indirect Calls Hurt

Modern CPUs are **pipelined** — they work on many instructions simultaneously, each at a different stage (fetch, decode, execute, etc.), like an assembly line processing ~15+ instructions in flight. To keep the pipeline full, the CPU must know *which instruction comes next* — but a **virtual call's target is unknown until the vtable load completes.** The CPU can't just wait (that would stall the pipeline). Instead it **speculates** — guesses the target via the **branch predictor** (specifically, the Branch Target Buffer for indirect jumps) — and races ahead executing the guessed path.

```
Direct call:    target known → pipeline stays full → fast.

Virtual call:   target unknown until vtable loads →
   CPU GUESSES the target (predicts based on history) →
   ├─ guess RIGHT (call site usually hits the same type):
   │    near-free — speculation paid off, pipeline full
   └─ guess WRONG (call site sees varying types):
        MISPREDICTION → flush the pipeline (~15-20 cycles wasted) →
        restart on the correct path
```

### Monomorphic vs. Polymorphic Call Sites (The Key Insight)

The branch predictor's accuracy depends on **how varied the types are at a given call site:**
- **Monomorphic** call site (always the same concrete type, e.g., a Strategy that's always `FastestRoute` in practice): the predictor learns it instantly, guesses right ~always → virtual call is **nearly as cheap as a direct call.**
- **Polymorphic** call site (a few types): predictor manages, occasional misses.
- **Megamorphic** call site (many types churning through, e.g., a Visitor over a wildly varied structure, or a Strategy genuinely swapping among dozens): predictor fails often → frequent mispredictions and pipeline flushes → **the genuinely slow case.**

### JIT Devirtualization (Why Java/JS Virtual Calls Are Often Free)

Here's the twist that surprises even experienced engineers: in **JIT-compiled runtimes (JVM, V8, .NET)**, the just-in-time compiler *watches your program run*, observes that a particular Strategy call site has only ever seen `FastestRoute`, and **devirtualizes** it — replacing the vtable lookup with a *direct, inlined* call guarded by a cheap type check (and if the guess proves wrong later, it deoptimizes). The result: **a polymorphic pattern call that is monomorphic in practice often costs essentially nothing after JIT warmup** — the abstraction is "compiled away." This is *why* idiomatic, pattern-heavy Java/Kotlin/Scala/JS code runs fast despite all the virtual dispatch: the JIT erases the cost for the common monomorphic case. C++ does similar work at compile time via **inlining** and devirtualization when it can prove the type, plus `final`/`override` hints and templates (which are *compile-time polymorphism* — Strategy with zero runtime dispatch).

**The complete dispatch-cost model (memorize this):**
```
Virtual call cost depends on:
  1. Is the call site monomorphic? → predictor + JIT make it ~free
  2. Is it in a hot loop over scattered objects? → cache misses dominate (expensive)
  3. Is it megamorphic? → mispredictions hurt
  4. Is it I/O-adjacent (web/DB)? → cost is invisible, ignore it entirely

Therefore: patterns are cheap almost everywhere EXCEPT hot, megamorphic,
cache-hostile inner loops — which are exactly where you'd profile and
optimize anyway. Elsewhere, optimize for clarity, not dispatch.
```

---

## 3.6 — Language Runtime Differences: Where Dispatch Lives

The same pattern dispatches through *different machinery* depending on the language, which changes both cost and behavior. A staff engineer knows these differ.

**C++ — vtables, compile-time-ish, zero-overhead philosophy.** Virtual methods use vtables exactly as described; non-virtual methods have zero dispatch cost. Templates provide *compile-time polymorphism* (Strategy resolved entirely at compile time — no vtable, fully inlinable). C++'s rule: "you don't pay for what you don't use" — a class with no virtuals has no vptr. This is why C++ pattern code can be as fast as C.

**Java — vtables + aggressive JIT.** Every non-`final`, non-`static`, non-`private` method is virtual by default (dispatched via the JVM's vtable-equivalent). But the JIT (HotSpot C2/Graal) devirtualizes, inlines, and deoptimizes adaptively — so *measured* cost is usually far below the naive vtable cost. Java pattern code is fast because the JIT is brilliant.

**Python — dynamic dictionary lookup (the slow one).** There are no vtables. `obj.method()` is a **runtime dictionary lookup**: search the instance `__dict__`, then the class `__dict__`, then walk the **MRO** (Method Resolution Order — the linearized inheritance chain) until found. Every attribute access is a hash-table lookup. This is *far* slower than a vtable index — but Python is interpreted and typically I/O-bound, so it rarely matters; and this same dynamism is *why* Python barely needs the GoF patterns (you can swap methods, pass functions, monkey-patch — the language is dynamic enough that patterns dissolve, per Part 1). Trade-off made explicit: maximum flexibility, slowest dispatch.

**JavaScript (V8) — hidden classes + inline caches.** Objects are dynamic dicts in principle, but V8 invents **hidden classes** (a.k.a. "shapes"/"maps") behind the scenes: objects with the same structure share a hidden class, giving them a *vtable-like fixed layout*, so property/method access becomes a fast offset lookup via **inline caches** at each call site. Result: JS dispatch approaches vtable speed *when objects have stable shapes* — and degrades to dictionary lookup when you add/remove properties chaotically (deopting the hidden class). This is why "don't change object shape after creation" is a JS performance rule — it preserves the vtable-like fast path.

**Go — interface tables (itabs).** Go has no inheritance; interface dispatch uses an **itab** (interface table) pairing a concrete type with the interface's method set — a vtable-like structure. A Go interface value is a *two-word pair*: `(pointer to itab, pointer to data)`. Method calls index the itab. Go deliberately keeps this simple and predictable (no JIT — it's AOT-compiled), favoring small interfaces and composition over deep hierarchies — the language design *embodies* "favor composition" and "small interfaces" (ISP) at the runtime level.

**Rust — static dispatch by default, dynamic when asked.** Rust resolves trait methods at **compile time via monomorphization** by default (generic code is specialized per concrete type — zero-cost, fully inlinable, like C++ templates). You *opt into* dynamic dispatch explicitly with `dyn Trait` (trait objects), which uses a vtable. So Rust makes the cost *visible and chosen*: `impl Trait`/generics = free static Strategy; `dyn Trait` = vtable dynamic Strategy. Pattern dispatch cost becomes an explicit, type-level decision.

```
DISPATCH MECHANISM BY LANGUAGE:

  C++     vtable (runtime) OR template (compile-time, zero-cost)
  Rust    monomorphized (compile-time, default) OR dyn vtable (opt-in)
  Java    vtable + JIT devirtualization (fast after warmup)
  Go      itab (interface table, AOT, predictable)
  C#      vtable + JIT (like Java)
  JS(V8)  hidden classes + inline caches (vtable-like when shapes stable)
  Python  dict/MRO lookup (slowest, but flexibility dissolves patterns)
```

**The synthesis:** the *same pattern* — say Strategy — has wildly different machine costs across these: free at compile time in Rust/C++ templates, near-free after JIT in Java/JS, a cheap itab index in Go, and a (rarely-mattering) dict walk in Python. The pattern's *structure* is constant; its *runtime embodiment* is language-specific. This is the internals-level proof of Part 1's language-relativity thesis.

---

## 3.7 — Internals of Specific Patterns: Execution Traces

Now we apply the machinery to trace what *specific* patterns do internally, beyond generic dispatch.

### Observer — The Notification Loop in Memory

```java
subject.notifyObservers();  // what actually happens
```

```
Heap state:
  Subject @0x4000: [ vptr | observers → 0x5000 ]
  observers List @0x5000: [ size=3 | [0]→0x6000 [1]→0x6100 [2]→0x6200 ]
  Observer objects scattered: ChartView@0x6000, Logger@0x6100, Cache@0x6200

Execution of notify():
  load observers list ref         → 0x5000  (cache miss likely)
  load size                       → 3
  LOOP i = 0..2:
    load observers[i]             → 0x6000 (then 0x6100, 0x6200)  ← scattered, misses
    load that observer's vptr     → its vtable
    load update() slot            → method address  ← MEGAMORPHIC if observers differ in type!
    indirect CALL update()        → may mispredict (different concrete types each iteration)
    ► each observer's update() may itself trigger more work / cascades
```

Internal insights this reveals: (1) the notification is a **loop of virtual calls over a scattered list** — the cache and megamorphic-dispatch costs of 3.4/3.5 apply if observers are many and varied; (2) the **lapsed-listener leak** (2C.2) is now visible — a dead observer still sits in that list at `0x6xxx`, kept alive by the reference, never GC'd; (3) **re-entrancy hazard** is physical — if an `update()` modifies the `observers` list mid-loop, the loop is iterating a structure being mutated → crash or corruption. The pattern's documented hazards are *consequences of this memory-level mechanism.*

### Decorator — Stack Unwinding Through the Chain

```java
new Whip(new Caramel(new Milk(espresso))).cost();
```

```
cost() on outermost Whip:
  Whip.cost():
    needs inner.cost() → follow inner ptr → Caramel  (pointer hop, cache miss)
      Caramel.cost():
        needs inner.cost() → follow ptr → Milk        (hop, miss)
          Milk.cost():
            needs inner.cost() → follow ptr → Espresso (hop, miss)
              Espresso.cost(): return 2.00            (base case)
            return 2.00 + 0.50  = 2.50
        return 2.50 + 0.70  = 3.20
    return 3.20 + 0.40 = 3.60

  This is a CALL STACK 4 frames deep, each frame a virtual call,
  each requiring a pointer-follow to a scattered heap object.
```

This makes concrete why deep decorator chains have a cost (stack depth + pointer-chasing + N virtual calls) and why they're hard to debug (the logic is *spread across the call stack*, unwinding inward then summing outward). It also shows why **order matters physically** — the chain is a literal linked list; reordering wrappers reorders the execution nesting.

### Singleton — The Memory-Barrier Reality of Double-Checked Locking

Recall Part 2A's DCL with mandatory `volatile`. The *why* is an internals story:
```java
instance = new Config();   // looks atomic; is NOT
```
This single line compiles to roughly three steps:
```
  (1) allocate memory for Config       → addr 0x7000
  (2) write 0x7000 into `instance`      ← reference published HERE
  (3) run Config's constructor on 0x7000 ← fields initialized HERE
```
The CPU/compiler may **reorder (2) before (3)** for performance (memory-reordering — legal under the memory model unless prevented). Then another thread sees `instance != null` (step 2 done), uses it, but the constructor (step 3) *hasn't run* — it reads a **half-constructed object** (garbage fields). `volatile` inserts a **memory barrier** forbidding that reorder, guaranteeing the constructor completes before the reference publishes. This is why DCL was subtly broken in Java before the JMM was fixed (JSR-133, Java 5) and why `volatile` is non-negotiable. The pattern's "gotcha" is, at root, a **CPU memory-ordering and cache-coherence** phenomenon — pure internals.

---

## 3.8 — Down the Stack: OS, Network, and Database Internals

The architectural-scale patterns (Repository, Proxy-as-remote, CQRS, Event Sourcing from Part 2D) reach below the language runtime into the OS, network, and database. Completeness demands we trace these layers too.

### OS Level: What a "Remote Proxy" Call Really Costs

A remote proxy (2B.7) makes `userService.getUser(id)` *look* like a local method call, but internally it crosses the **user/kernel boundary** and the network. The full cost the proxy hides:

```
proxy.getUser(id):   // looks like a method call; is actually:
  1. SERIALIZE the request (marshal id + method into bytes)        [CPU]
  2. SYSTEM CALL (send) → trap into the OS kernel                  [user→kernel switch, ~1µs]
  3. Kernel copies data to socket buffer, TCP/IP stack processes  [kernel]
  4. NIC sends packets over the wire                              [hardware]
  5. ... network transit (LAN ~0.5ms, cross-region ~50-150ms) ... [WALL CLOCK: enormous]
  6. Remote OS receives, wakes the server process (context switch)
  7. Server deserializes, runs the real getUser, serializes reply
  8. ... return network transit ...
  9. Local kernel receives, copies to user space, wakes our thread
  10. DESERIALIZE the response → return the User object
```

The vtable dispatch of the proxy call (3.2) is **~3 nanoseconds**; the network round-trip it hides is **~1–100 milliseconds** — a factor of *a million to thirty million*. This is the staff-engineer's perspective crystallized: **the proxy's dispatch cost is utterly irrelevant; what matters is the I/O it conceals.** It also exposes the *danger* of remote proxies (2B.7's "hidden cost" disadvantage): an innocent-looking `user.getName()` might trigger that entire million-times-costlier sequence. The abstraction hides a catastrophe-of-scale difference in cost — convenient but perilous.

### Network Level: Why Patterns Can't Hide the Fallacies

Architectural patterns (microservices, remote facades) operate over the network, which obeys the **Eight Fallacies of Distributed Computing** — false assumptions that *no pattern can abstract away*:
1. The network is reliable. (It isn't — packets drop.)
2. Latency is zero. (It isn't — physics: speed of light, ~50ms cross-continent minimum.)
3. Bandwidth is infinite. (It isn't.)
4. The network is secure. (It isn't.)
5. Topology doesn't change. (It does.)
6. There is one administrator. (There isn't.)
7. Transport cost is zero. (Serialization/transit cost real CPU and money.)
8. The network is homogeneous. (It isn't.)

A remote Proxy or Facade *syntactically* makes remote calls look local, but it **cannot make them behave local** — they can fail, time out, arrive out of order, or partially succeed. This is the internals-level reason that naive "make it look like a local call" distributed patterns (old-style RPC/CORBA) caused disasters, and why modern distributed patterns (Circuit Breaker, Retry, Bulkhead, Saga — Part 5/8) explicitly *embrace* failure rather than hiding it. **The lesson: a pattern that hides the network's nature is lying, and the lie has consequences at the protocol level.**

### Database Level: Unit of Work and the Transaction Machinery

Unit of Work (2D.3) maps directly onto **database transaction internals.** When the unit of work calls `commit()`, the database engine engages:

```
commit() triggers, inside the DB engine:
  • WAL (Write-Ahead Log): changes written to a durable log BEFORE data files
    (so a crash mid-commit can be recovered/rolled back) — this is the D in ACID (Durability)
  • Locking / MVCC: the DB ensures Isolation — either by locking rows or by
    Multi-Version Concurrency Control (keeping versioned snapshots so readers
    don't block writers). The Unit of Work's "resolve concurrency" maps here.
  • Atomicity: all the unit of work's tracked changes apply together or not at all,
    enforced by the transaction's commit/rollback (the A in ACID).
  • fsync to disk: the durability barrier — the commit isn't "done" until the
    log is physically on stable storage (an OS/hardware-level flush, ~ms).
```

The **ORM's "session/identity map"** (the Unit of Work's change-tracking) also implements a performance internal: it **dedupes and batches** — if you modify an object thrice, one UPDATE is issued at commit, not three; and an **identity map** ensures one in-memory object per database row (avoiding duplicate loads and inconsistent copies). The "N+1 query problem" (a notorious ORM performance bug) is a Unit-of-Work/Repository internals failure — lazy-loading a collection issues one query per element instead of one bulk query, turning 1 logical operation into N+1 round-trips (each with the OS/network cost above). Understanding the pattern's internals (lazy loading via virtual proxies + per-access queries) is exactly what lets a senior engineer *diagnose and fix* N+1 — which they do by changing the proxy's fetch strategy (eager/batch fetching).

---

## 3.9 — Internals Synthesis: The Cost Model of Patterns

You now have the complete machine-level picture. Consolidate it into the mental cost model a staff engineer carries:

**1. Every pattern's runtime cost traces to polymorphic dispatch, which is a vtable lookup: two memory loads + an indirect call.** In isolation, ~nanoseconds — negligible.

**2. The real cost is memory indirection and cache behavior, not instruction count.** Composition (holding references) means pointer-chasing; in hot loops over many scattered objects, cache misses (~100x RAM penalty) and lost prefetching dominate, making pattern-heavy code 10–50x slower *in that specific regime*. Flyweight and data-oriented design exist to fix exactly this.

**3. Branch prediction and JIT devirtualization usually erase dispatch cost** for monomorphic call sites (the common case). Megamorphic, cache-hostile, hot loops are the genuine danger zone. JIT runtimes (Java/JS) and compile-time polymorphism (Rust/C++) make most pattern dispatch effectively free in practice.

**4. The dispatch mechanism is language-specific** (vtable / itab / hidden class / dict-MRO / monomorphization), confirming at the metal level that a pattern's *weight* is language-relative even though its *structure* is universal.

**5. As you descend the stack, the costs explode by orders of magnitude** — dispatch (ns) ≪ cache miss (×100) ≪ syscall (µs) ≪ disk fsync (ms) ≪ network round-trip (ms–100ms). **Therefore the pattern-dispatch cost is invisible the moment any I/O is involved** — which is why web, business-logic, and distributed code should be optimized for *clarity* (use patterns freely), while only hot, CPU-bound, in-memory inner loops should be optimized for *dispatch/cache* (use patterns sparingly).

**6. Architectural patterns can't abstract away physics.** Remote proxies, facades, and distributed patterns hide *syntax* but not *reality* — latency, partial failure, and the distributed-computing fallacies leak through, which is why robust distributed patterns embrace failure rather than hiding it.

**The single sentence that captures Part 3:** *A pattern's cost is dominated not by the few instructions of virtual dispatch but by the memory indirection and — far more — the I/O it sits near; so spend pattern-indirection freely everywhere except hot, cache-sensitive, CPU-bound inner loops, because everywhere else the machine has bigger costs to worry about than your vtable.*

This grounds every performance claim in Part 6 and every "is this pattern worth it?" judgment in physical reality rather than folklore. You can now reason about pattern cost from first principles — registers to cache to syscalls to the network — which is precisely the depth that separates a staff engineer's "it depends, and here's the actual mechanism" from a junior's cargo-culted "patterns are slow" or "patterns are free."

Part 4 (Implementation) will now build patterns in real, production-grade code — beginner to advanced — applying this internals knowledge to write implementations that are not just *correct* but *appropriately costed* for their context.