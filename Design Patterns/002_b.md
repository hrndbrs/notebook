# PART 2B — STRUCTURAL PATTERNS

**The unifying force:** Structural patterns answer *"how do I compose objects and classes into larger structures without welding them rigidly together?"* Where creational patterns hide *which* object is made, structural patterns manage *relationships between objects already made* — bridging mismatched interfaces, wrapping to add behavior, simplifying access, sharing to save memory, controlling access, and building tree structures. Almost all of them are **composition in action** — the direct payoff of Part 1's "favor composition over inheritance."

```
What each structural pattern manipulates:

Adapter    → converts one interface to another (compatibility)
Bridge     → splits abstraction from implementation (two free axes)
Composite  → builds part-whole trees, treated uniformly
Decorator  → wraps to add behavior dynamically (the subclass-explosion cure)
Facade     → wraps a subsystem behind one simple interface
Flyweight  → shares state across many objects (memory)
Proxy      → stands in front of an object to control access
```

A crucial early distinction, because beginners conflate these four "wrapping" patterns: **Adapter changes an interface; Decorator adds to behavior while keeping the interface; Facade simplifies many interfaces into one; Proxy keeps the same interface but controls access.** Same mechanical shape (one object holds another), four different *intents*. Intent, not structure, names the pattern — a recurring theme.

---

## 2B.1 — Adapter

### The Force

You have a class that does what you need, but its *interface doesn't match* what your code expects — because it's a third-party library, legacy code, or a component built to a different contract. You can't (or shouldn't) modify it. Without adaptation you'd rewrite client code to match the foreign interface, scattering the foreign type everywhere (coupling) and violating OCP. Adapter resolves "make an existing interface usable through the interface a client expects, without modifying either."

### Intent

> Convert the interface of a class into another interface clients expect. Adapter lets classes work together that couldn't otherwise because of incompatible interfaces.

### Motivation Story

Your app processes payments through a `PaymentProcessor` interface (`charge(amount, card)`). You integrate a new provider, `FancyPayLib`, whose method is `executeTransaction(FancyRequest)` — totally different shape. You don't own FancyPayLib's code. You write a `FancyPayAdapter implements PaymentProcessor` that, inside `charge()`, builds a `FancyRequest`, calls `executeTransaction()`, and translates the response back. Your whole app keeps talking to `PaymentProcessor`; only the adapter knows FancyPay exists. Swap providers later by swapping adapters.

### Structure — Two Variants

**Object Adapter (composition — preferred):**
```
┌──────────┐  expects   ┌─────────────────┐
│  Client  │ ─────────► │   «Target»      │
└──────────┘            │ + request()     │
                        └────────▲────────┘
                                 │ implements
                        ┌────────┴────────┐    holds (composition)
                        │    Adapter      │──────────────┐
                        │ + request() ────┼──┐           ▼
                        └─────────────────┘  │     ┌──────────────┐
                                             │     │   Adaptee    │
                          translates & calls └────►│ +specificReq()│
                                                   │ (foreign API)│
                                                   └──────────────┘
```

**Class Adapter (inheritance — multiple-inheritance languages only):** the adapter *inherits* from both Target and Adaptee. Rare, more rigid, impossible in single-inheritance languages without interfaces. **Prefer the object adapter** — it follows composition-over-inheritance, works everywhere, and can adapt subclasses of the adaptee too.

### Internal Mechanics

The adapter's method body is pure **translation**: map incoming parameters to the adaptee's expected shape, call the adaptee, map the result back. It may also bridge *semantic* gaps (different units, error conventions — e.g., adaptee throws, target returns a result object, so the adapter catches and converts). It's a thin, mechanical layer with no business logic of its own.

### Advantages
- Integrate incompatible/legacy/third-party code without modifying it (OCP, and respects code you don't own).
- Isolates the foreign interface to one place (the adapter); the rest of the system stays clean.
- Single Responsibility: conversion logic lives in one dedicated class.
- Swap the adapted thing by swapping the adapter.

### Disadvantages
- Extra indirection and a new class per adapted type.
- Many adapters can accumulate ("integration sprawl").
- If the two interfaces are *semantically* far apart (not just syntactically), the adapter grows complex and leaky.

### Tradeoffs
Adapter trades a small indirection layer for the ability to integrate without modification and to quarantine foreign dependencies. Nearly always worth it for third-party integration — the alternative (foreign types leaking everywhere) is far worse. The cost is real only when the impedance mismatch is so large the adapter becomes a mini-translation-engine.

### When NOT to Use
When you *own* both sides and can simply align the interfaces. When the interfaces already match. When a one-off inline conversion is clearer than a class.

### Language Note
Universal and idiomatic everywhere — wrapping a foreign API in your own interface is just good engineering in any language. In Go, the small-interface culture makes adapters extremely common and lightweight. In dynamic languages, duck typing sometimes removes the *need* (if it walks like a duck), but explicit adapters still clarify intent.

---

## 2B.2 — Bridge

### The Force

You have **two independent dimensions that both vary**, and modeling them with inheritance causes a combinatorial explosion. Shapes (Circle, Square) × Rendering APIs (Vector, Raster) via inheritance gives `VectorCircle, RasterCircle, VectorSquare, RasterSquare` — and adding one shape *or* one renderer multiplies the whole grid. Bridge resolves "separate an abstraction from its implementation so the two hierarchies vary **independently**" — turning multiplication (M×N classes) into addition (M+N classes).

### Intent

> Decouple an abstraction from its implementation so that the two can vary independently.

### Motivation Story

You're building UI controls (`Button`, `Slider`, `Menu`) that must render on multiple platforms (`Windows`, `Linux`, `Web`). Inheritance would breed `WindowsButton`, `LinuxButton`, `WebButton`, `WindowsSlider`... an M×N nightmare where every new control means N new classes and every new platform means M. Bridge splits this: a `Control` abstraction hierarchy (Button, Slider, Menu) *holds a reference to* a `Renderer` implementation interface (WindowsRenderer, LinuxRenderer, WebRenderer). A `Button` *delegates* its drawing to whatever `Renderer` it was given. Now: new control = 1 class; new platform = 1 class. M+N, not M×N.

### Structure

```
   ABSTRACTION hierarchy            IMPLEMENTATION hierarchy
   (what the user thinks about)     (how it's actually done)

   ┌──────────────────┐   has-a    ┌──────────────────┐
   │   Abstraction    │ ─(bridge)─►│ «Implementor»    │
   │ - impl: Implementor          │ + operationImpl()│
   │ + operation() ───┼──┐         └────────▲─────────┘
   └────────▲─────────┘  │ delegates         │
            │            └─► impl.operationImpl()
   ┌────────┴─────────┐            ┌─────────┴──────────┐
   │RefinedAbstraction│        ┌───┴────┐         ┌─────┴────┐
   │ (Button, Slider) │        │ImplA   │         │ImplB     │
   └──────────────────┘        │(Windows)│        │(Linux)   │
                               └─────────┘        └──────────┘

  The "bridge" is the has-a arrow between the two hierarchies.
  Each can grow without touching the other.
```

### Internal Mechanics

The abstraction holds a reference to the implementor *interface* (injected at construction — note this is composition + DIP). Its methods contain *higher-level logic* and *delegate the primitive operations* to the implementor. The implementor exposes only *low-level primitives*; the abstraction composes them into meaningful behavior. The two hierarchies meet only at the implementor interface — the "bridge."

### Bridge vs. Adapter (the constant confusion)
**Adapter** is *reactive*: two things already exist with mismatched interfaces, and you bolt them together after the fact. **Bridge** is *proactive*: you design two hierarchies separately *up front* precisely so they can vary independently. Adapter fixes a mismatch; Bridge prevents an explosion. Same has-a mechanic, opposite timing and intent.

### Advantages
- Turns M×N class explosion into M+N (the headline benefit).
- Abstraction and implementation evolve independently (two free axes of change — OCP on both).
- Implementation can be swapped at runtime (it's a held reference).
- Hides implementation details from clients.

### Disadvantages
- Upfront complexity and indirection; only pays off when *both* dimensions genuinely vary.
- Requires foresight to identify the two orthogonal axes correctly — get the split wrong and it's wasted structure.

### Tradeoffs
Bridge trades immediate simplicity for long-term independent evolvability of two axes. The bet (Part 1!) is that *both* dimensions will grow. If only one varies, Bridge is over-engineering — a single hierarchy or Strategy suffices.

### When NOT to Use
When only one dimension varies (use plain inheritance or Strategy). When the two axes aren't truly orthogonal. Early, when you don't yet know whether both will grow (YAGNI — wait for the second dimension to appear).

### Language Note
Bridge is fundamentally "hold an interface reference and delegate" — trivial in any language with interfaces. In languages with first-class functions, the implementor side is sometimes just a set of injected functions rather than an interface hierarchy. The conceptual *separation of two axes* is the durable part; the class machinery is optional.

---

## 2B.3 — Composite

### The Force

You have a **part-whole hierarchy** — objects that contain other objects, recursively, forming a tree — and you want client code to treat *individual objects* and *groups of objects* **uniformly**, without writing `if (isLeaf) ... else (iterate children) ...` everywhere. A file system: files (leaves) and folders (which contain files *and* folders). Computing total size should work the same whether you ask a file or a folder. Composite resolves "treat individual and composite objects through one uniform interface, with recursion hidden inside the structure."

### Intent

> Compose objects into tree structures to represent part-whole hierarchies. Composite lets clients treat individual objects and compositions of objects uniformly.

### Motivation Story

A graphics editor has simple shapes and *groups* of shapes (and groups of groups). The user selects a group and drags — every shape inside must move. Without Composite, the editor constantly branches: "is this a single shape or a group?" With Composite, `Graphic` is the common interface with `move()` and `draw()`. A `Circle` (leaf) implements them directly; a `Group` (composite) implements them by *delegating to its children* (`for each child: child.move()`). The editor calls `graphic.move()` on *anything* — leaf or group — and the right thing happens recursively. The branching vanishes into the structure.

### Structure

```
         ┌──────────────────┐
         │   «Component»    │◄────────────┐
         ├──────────────────┤             │ children are
         │ + operation()    │             │ also Components
         │ + add(Component) │             │ (recursion!)
         │ + remove(...)    │             │
         └────────▲─────────┘             │
                  │                       │
        ┌─────────┴──────────┐            │
   ┌────┴─────┐      ┌───────┴──────┐     │
   │   Leaf   │      │  Composite   │─────┘
   │+operation│      │ - children[] │
   │ (does    │      │+operation()──┼─► for c in children:
   │  real    │      │+add/remove   │       c.operation()
   │  work)   │      └──────────────┘   (delegates recursively)
   └──────────┘

  Tree example:
        Composite
        /   |   \
     Leaf Composite Leaf
            /  \
         Leaf  Leaf
```

### Internal Mechanics

The magic is **recursive delegation.** A composite's `operation()` loops over its children calling *their* `operation()` — each of which might be a leaf (does real work, base case) or another composite (recurses). The recursion bottoms out at leaves. Computing folder size: folder asks each child its size; files return their bytes; subfolders recurse. The client triggers one call at the root; the structure unfolds the whole tree.

**The design tension — transparency vs. safety:** Where do `add(child)`/`remove(child)` live? 
- **Transparent** (GoF default): declare them in `Component`, so leaves and composites share *one identical interface* (maximally uniform) — but leaves must implement `add()` somehow (throw exception or no-op), which is a lie about their capability (an LSP smell).
- **Safe**: declare child-management only on `Composite`, so leaves genuinely can't have children — type-safe — but now clients must sometimes distinguish leaf from composite, partially defeating uniformity.
This is a real, unresolvable tradeoff; you pick which lie you can live with. GoF leans transparent (favoring uniformity); many modern practitioners lean safe.

### Advantages
- Uniform treatment of leaves and composites — client code is simple, no type-branching.
- New component types (leaf or composite) plug in without changing client code (OCP).
- Naturally models any recursive/hierarchical domain (file systems, org charts, UI trees, ASTs, menus).

### Disadvantages
- The transparency/safety dilemma (above) — neither option is clean.
- Can over-generalize: forcing a uniform interface on genuinely different things can be awkward (a leaf with no children pretending it could).
- Hard to restrict *what* can contain *what* (type constraints on the tree) without extra checks.

### Tradeoffs
Composite trades interface purity (the leaf-can't-really-add problem) for client simplicity and recursive uniformity. Worth it whenever the domain is genuinely a tree and you'll operate over it uniformly. Pointless for flat structures.

### When NOT to Use
When there's no real part-whole hierarchy. When leaves and composites are too dissimilar to share a meaningful interface. When you have no operations that benefit from uniform recursion.

### Language Note
Universal — any language with interfaces/polymorphism. In functional languages, recursive *sum types* (a `Tree = Leaf | Node(Tree, Tree)`) plus pattern matching express the same idea more directly and safely, with the recursion explicit in the match. Composite is the OO way to encode what algebraic data types encode natively.

---

## 2B.4 — Decorator

### The Force — The Subclass-Explosion Payoff from Part 1

This is the pattern Part 1 kept promising. You want to **add responsibilities to individual objects dynamically**, in arbitrary combinations, *without* subclassing for every combination. Coffee + optional milk, sugar, soy, whip, caramel via inheritance breeds `MilkCoffee, SugarCoffee, MilkSugarCoffee, MilkSoyWhipCoffee...` — 2ⁿ classes for n add-ons, fixed at compile time, impossible to combine freely. Decorator resolves "wrap an object in another object that adds behavior, stackable in any order, chosen at runtime" — replacing the exponential class hierarchy with linear, composable wrappers.

### Intent

> Attach additional responsibilities to an object dynamically. Decorators provide a flexible alternative to subclassing for extending functionality.

### Motivation Story

A coffee shop priced via objects. `Espresso` costs $2. Add milk (+$0.50), then caramel (+$0.70), then whip (+$0.40). With Decorator: start with `new Espresso()`, then `new Milk(new Espresso())`, then `new Caramel(new Milk(new Espresso()))`. Each decorator *wraps* a `Beverage`, *implements* `Beverage`, and in `cost()` returns `wrapped.cost() + itsOwnCost`. Calling `cost()` on the outermost wrapper cascades inward, summing the chain. Any combination, any order, decided at runtime, with zero combinatorial classes — just one decorator class per add-on.

### Structure

```
         ┌──────────────────┐
         │   «Component»    │◄──────────────────┐
         │ + operation()    │                   │ wraps a
         └────────▲─────────┘                   │ Component
                  │                             │ (same type!)
        ┌─────────┴───────────┐                 │
┌───────┴────────┐   ┌────────┴─────────┐       │
│ConcreteComponent│   │   «Decorator»    │───────┘
│ +operation()   │   │ - wrapped: Component
│ (the base obj) │   │ +operation()─────┼─► wrapped.operation()
└────────────────┘   └────────▲─────────┘   + extra behavior
                              │
                   ┌──────────┴──────────┐
            ┌──────┴───────┐    ┌─────────┴──────┐
            │ DecoratorA   │    │  DecoratorB    │
            │ (Milk:       │    │  (Whip:        │
            │  +cost,      │    │   +cost)       │
            │  +behavior)  │    │                │
            └──────────────┘    └────────────────┘

  Stacking:  Whip( Caramel( Milk( Espresso ) ) )
             outer──────────────────────►inner
  cost() cascades: Whip adds to Caramel adds to Milk adds to Espresso
```

### Internal Mechanics

A decorator **is-a** Component (so clients can't tell it apart) *and* **has-a** Component (the thing it wraps). Its method calls the wrapped object's method, then adds behavior before/after/around it. Because each decorator both implements and holds the component interface, decorators *nest arbitrarily deep* — each layer transparent to the next. The call cascades through the chain like a stack unwinding. This is the cleanest possible illustration of "favor composition over inheritance": behavior is added by *composing wrappers at runtime* instead of *fixing it in subclasses at compile time*.

### The Canonical Real Example — Java I/O Streams
`new BufferedReader(new InputStreamReader(new FileInputStream("f")))` — *that's three decorators.* `FileInputStream` reads raw bytes; `InputStreamReader` decorates it to convert bytes→chars; `BufferedReader` decorates *that* to add buffering and `readLine()`. Each adds one capability; you stack exactly what you need. The entire `java.io` package is a Decorator showcase — and a famous example of Decorator's main downside (lots of tiny wrappers, verbose construction).

### Advantages
- Add/remove responsibilities at runtime, in any combination (vastly more flexible than inheritance).
- Avoids the 2ⁿ subclass explosion — n add-ons = n decorator classes, combined freely.
- Single Responsibility: each decorator does *one* thing; compose for complex behavior.
- OCP: new behavior = new decorator, no existing code touched.

### Disadvantages
- Many small wrapper classes → can be hard to read; lots of `new Wrapper(new Wrapper(...))`.
- Order can matter (encrypt-then-compress ≠ compress-then-encrypt) — silent bugs if mis-ordered.
- A decorated object isn't *identical* to the original — identity checks (`==`), `instanceof`, and accessing the inner object's specific methods get awkward (the wrapper hides them).
- Debugging a deep stack is tedious (which layer did what?).

### Tradeoffs
Decorator trades a proliferation of small wrapper classes and some identity/ordering hazards for runtime-composable, explosion-free extension. The trade is excellent when you have *many independent, combinable optional features*; poor when you have few, fixed features (just subclass or add a field).

### When NOT to Use
Few fixed combinations (subclass or flags are simpler). When you need to access the concrete wrapped object's identity/specific methods often. When ordering subtleties would confuse more than help.

### Language Note
In languages with **higher-order functions**, decoration is often **function composition**: `withLogging(withRetry(handler))` wraps functions, not objects. Python's `@decorator` syntax is *literally this pattern* built into the language (`@cache`, `@login_required` wrap functions). JavaScript middleware (`app.use(...)`) and React HOCs are functional decorators. The OO class-based Decorator is the heavyweight version; functional languages get it nearly for free.

---

## 2B.5 — Facade

### The Force

A subsystem is **complex** — many classes, intricate interactions, a steep learning curve — but most clients need only a *simple subset* of its capabilities through a *clean entry point*. Forcing every client to learn and correctly orchestrate the subsystem's internals creates widespread coupling: every client depends on many internal classes, and any internal change ripples everywhere. Facade resolves "provide one simplified, high-level interface to a complex subsystem, so clients depend on the facade instead of the tangle behind it."

### Intent

> Provide a unified interface to a set of interfaces in a subsystem. Facade defines a higher-level interface that makes the subsystem easier to use.

### Motivation Story

Starting a car's engine *really* involves: prime fuel pump, engage starter motor, set ignition timing, open throttle, check oil pressure, engage ECU. You don't do all that — you press **Start**. The button is a facade over a complex subsystem. In software: a `VideoConverter.convert(file, format)` facade hides codecs, bitrate calculators, audio mixers, container muxers, and buffer management — dozens of classes. The client calls *one method*; the facade orchestrates the subsystem. (Stripe's SDK, from Part 1, is exactly this — `stripe.charges.create()` hides HTTP, retries, idempotency, serialization.)

### Structure

```
   ┌──────────┐
   │  Client  │
   └────┬─────┘
        │ only talks to Facade
        ▼
   ┌─────────────────────┐
   │      Facade         │  simple high-level methods
   │ + doComplexThing()  │
   └─────────┬───────────┘
             │ orchestrates the messy subsystem
   ┌─────────┼──────────┬──────────┐
   ▼         ▼          ▼          ▼
┌──────┐ ┌──────┐  ┌───────┐  ┌────────┐
│ClassA│ │ClassB│  │ClassC │  │ ClassD │   complex subsystem
│      │◄┤      │◄─┤       │◄─┤        │   (clients never see this)
└──────┘ └──────┘  └───────┘  └────────┘
```

### Internal Mechanics

The facade holds references to the subsystem's key classes. Each facade method runs a *recipe*: call subsystem class A, feed its result to B, configure C, invoke D in the right order with the right error handling. It's pure orchestration — it *adds no new functionality*, it just *packages existing functionality* behind a friendly door. Critically, the facade **doesn't forbid** direct subsystem access — advanced clients can still bypass it for fine control; the facade just serves the common 90% case simply.

### Facade vs. Adapter vs. Mediator (disambiguation)
- **Adapter** changes *one* interface to a *specific expected* interface (compatibility). **Facade** *simplifies many* interfaces into a *new convenient* one (ease of use). Adapter is about matching a contract; Facade is about reducing complexity.
- **Mediator** (a behavioral pattern, coming in 2C) coordinates *bidirectional* communication *among* subsystem objects that know about the mediator. **Facade** is *unidirectional* — clients call in; the subsystem doesn't know the facade exists.

### Advantages
- Shields clients from subsystem complexity → far less coupling (clients depend on one facade, not many internals).
- Subsystem internals can change freely as long as the facade's interface holds (OCP, layering).
- Provides a sane default entry point; lowers the learning curve.
- Defines clear layer boundaries in an architecture.

### Disadvantages
- The facade can become a **god object** — a bloated catch-all that knows about everything (low cohesion) if not disciplined.
- Adds a layer; over-thin facades that just forward one call add no value.
- Can become a bottleneck/coupling point if *every* interaction must route through it.

### Tradeoffs
Facade trades an extra layer for dramatically reduced client-subsystem coupling and a gentler interface. Almost always worth it for any non-trivial subsystem or library boundary — it's one of the highest-value, lowest-risk patterns. The only real danger is letting it bloat into a god object.

### When NOT to Use
When the subsystem is already simple (the facade adds nothing). When clients genuinely need fine-grained control over internals (though you can offer both). When it would become a dumping ground for unrelated methods.

### Language Note
Universal and ubiquitous — every well-designed library *is* a facade over its internals. Module systems make this natural: a package's public API is a facade; everything else is internal. No special language machinery needed; it's an architectural discipline more than a code structure.

---

## 2B.6 — Flyweight

### The Force

You need a **huge number of similar objects**, and the memory cost is prohibitive because each carries duplicated data. A document with a million characters, each an object storing its font, size, color, *and* glyph shape — gigabytes wasted, since most characters share the same font/glyph. The insight: most of each object's state is *identical across many instances* and can be **shared** rather than duplicated. Flyweight resolves "minimize memory by sharing the common (intrinsic) state across many objects, keeping only the unique (extrinsic) state per use."

### Intent

> Use sharing to support large numbers of fine-grained objects efficiently.

### Motivation Story

A text editor rendering a 1,000,000-character document. Naively, each character object stores its glyph bitmap, font, and style (intrinsic — same for all 'A's in Arial 12pt) plus its position (extrinsic — unique per character). Storing the glyph/font per character wastes enormous memory. Flyweight: create *one* shared `Character('A', Arial, 12pt)` flyweight reused for every 'A'; store only the *position* externally (passed in when drawing). A million characters now reference a few dozen shared flyweights plus a lightweight position array. Memory collapses from gigabytes to megabytes.

### Structure

```
                  ┌────────────────────┐
                  │  FlyweightFactory   │  ensures sharing
                  │ - pool: Map<key,FW> │
                  │ + getFlyweight(key) │── returns existing or creates+caches
                  └─────────┬───────────┘
                            │ manages
                            ▼
                  ┌────────────────────┐
   Client ──────► │    «Flyweight»     │
   passes         │ + operation(       │
   EXTRINSIC      │     extrinsicState)│◄── intrinsic state stored INSIDE
   state per call │   (uses intrinsic  │    (shared); extrinsic passed IN
                  │    + extrinsic)    │
                  └────────────────────┘

  INTRINSIC  = shared, context-independent (the 'A' glyph, font)  → stored in flyweight
  EXTRINSIC  = unique, context-dependent (position, current color) → passed by client
```

### Internal Mechanics

The defining move is **splitting state into intrinsic (shared, stored in the flyweight) and extrinsic (unique, supplied by the caller at use-time).** A `FlyweightFactory` maintains a pool/cache: when asked for a flyweight with given intrinsic state, it returns the *existing shared instance* if present, else creates and caches one. Flyweights are **immutable** (they must be — they're shared; mutation would corrupt every user). Extrinsic state is never stored in the flyweight; it's passed as a method parameter each call. The memory win comes from N objects collapsing to K unique flyweights (K ≪ N) plus a compact array of extrinsic state.

### Real Examples
- **String interning** (Java's String pool, Python's small-int and string caching): identical string literals share one object.
- **Game engines:** 10,000 trees share a few mesh/texture flyweights; only position/scale is per-instance.
- **Browsers:** shared glyph/font objects across all text.
- **`Integer.valueOf()` in Java** caches -128..127 as shared flyweights.

### Advantages
- Massive memory savings when many objects share state (the entire point).
- Can improve cache locality and reduce GC pressure (fewer objects).

### Disadvantages
- **Significant complexity** — splitting intrinsic/extrinsic state is unintuitive and invasive; client code must now *supply* extrinsic state everywhere it uses a flyweight.
- Trades CPU for memory: extrinsic state must be computed/passed each call; factory lookups add overhead.
- Flyweights *must* be immutable — a real constraint.
- Premature use is classic over-engineering; only justified at genuine scale.

### Tradeoffs
Flyweight trades code complexity and some CPU for large memory savings. The bet pays off *only* at high object counts with high state-sharing. At small scale it's pure overhead and obfuscation. This is the most "measure first" pattern — apply only when profiling shows memory is the actual bottleneck and sharing is high.

### When NOT to Use
Few objects, low sharing, or memory isn't constrained. When the intrinsic/extrinsic split would mangle your domain model for no measured benefit. Default to *not* using it until profiling demands it.

### Language Note
Many languages do this for you invisibly (string interning, integer caching). Explicit Flyweight is for *your* domain objects at scale. In data-oriented design (common in game/systems programming), the same goal is met via structure-of-arrays layouts rather than the OO flyweight structure. The *principle* (share intrinsic, externalize extrinsic) outlives the specific OO machinery.

---

## 2B.7 — Proxy

### The Force

You need to **control access to an object** — because accessing it directly is expensive (it's huge, remote, or slow to create), dangerous (needs permission checks), or needs surrounding behavior (caching, logging, lazy loading) — but you want clients to use it *exactly as if it were the real object*, with no code changes. Proxy resolves "provide a stand-in with the *same interface* as the real object, intercepting access to add control without the client knowing."

### Intent

> Provide a surrogate or placeholder for another object to control access to it.

### Motivation Story

A document contains high-resolution images that are slow to load from disk. Loading all of them when the document opens is wasteful — the user may never scroll to most. A **virtual proxy** stands in for each image: it implements the same `Image` interface, but holds only the filename. When `display()` is finally called (image scrolled into view), the proxy *lazily* loads the real image and delegates. Until then, no expensive load happens. The document code calls `image.display()` identically whether it's a real image or a proxy — it can't tell the difference.

### Structure

```
   ┌──────────┐
   │  Client  │
   └────┬─────┘
        │ depends on «Subject» (same interface for both!)
        ▼
   ┌─────────────────┐
   │   «Subject»     │
   │ + request()     │
   └────────▲────────┘
            │ both implement
     ┌──────┴────────────────┐
┌────┴────────┐      ┌────────┴────────┐   holds reference to
│ RealSubject │◄─────│     Proxy       │── the RealSubject
│ + request() │      │ + request() ────┼─► [control logic:
│ (real work) │      │                 │    check/cache/lazy/log,]
└─────────────┘      └─────────────────┘    then realSubject.request()
```

### The Four Classic Proxy Types (know these by name)
1. **Virtual proxy** — lazy initialization; defers creating/loading an expensive object until actually needed (the image example).
2. **Protection proxy** — access control; checks permissions before delegating (only admins may call `delete()`).
3. **Remote proxy** — local stand-in for an object in another address space/machine; hides network communication (RPC stubs, gRPC clients — the proxy marshals the call across the network).
4. **Smart proxy / smart reference** — adds bookkeeping: reference counting, caching results, logging access, lazy locking, lifecycle management.

### Internal Mechanics

The proxy implements the *same interface* as the real subject (so it's a drop-in substitute — LSP-compliant). It holds a reference to (or knows how to create/locate) the real subject. In each method it runs **pre/post control logic** — permission check, cache lookup, lazy instantiation, network marshaling, logging — and *conditionally* delegates to the real subject. The client, depending only on the interface, is oblivious.

### Proxy vs. Decorator vs. Adapter (the final disambiguation of the wrapping family)
All three wrap an object behind a held reference. The *intent* differs:
- **Adapter:** *changes* the interface (incompatible → expected).
- **Decorator:** *same* interface, *adds* behavior/responsibilities, designed to *stack*.
- **Proxy:** *same* interface, *controls access* (you don't choose to "stack" proxies for features; the proxy manages *whether/how* you reach the real object). Decorator enhances *what the object does*; Proxy manages *whether and how you get to it.*

### Advantages
- Control access transparently — clients unchanged (same interface).
- Enables lazy loading, caching, access control, remote access, logging without touching the real object (OCP, SRP).
- Can manage the real object's lifecycle (create on demand, dispose).

### Disadvantages
- Extra indirection (latency, a layer to maintain).
- Can hide costs — a method call that *looks* local might trigger a network round-trip (remote proxy) or a heavy load (virtual proxy); this surprise can bite (a `getName()` that silently does I/O).
- More classes; proxies for many types = boilerplate.
- Response time can become unpredictable (cache miss vs. hit, lazy load vs. cached).

### Tradeoffs
Proxy trades indirection and some hidden-cost risk for transparent access control. Excellent when the control concern (lazy/remote/security/cache) is real and you want clients oblivious. The hidden-cost danger is the main caution — make sure surprising I/O behind an innocent-looking call is acceptable.

### When NOT to Use
When no access-control concern exists. When hiding the real cost of an operation would mislead callers dangerously. When you can address the need more explicitly (e.g., explicit caching layer) without disguising it.

### Language Note
Many frameworks *generate* proxies dynamically: Java dynamic proxies / CGLIB power Spring AOP (transactions, security wrap your beans in proxies you never see); Hibernate uses virtual proxies for lazy entity loading; ORMs everywhere. Python's `__getattr__`/descriptors and JS's built-in `Proxy` object make proxies first-class. gRPC/RPC stubs are remote proxies by definition. The pattern is *pervasive* in framework internals even where you never write one by hand.

---

## 2B — Structural Patterns: Comparison & Synthesis

```
PATTERN     WRAPS/COMPOSES TO…           INTERFACE      KEY DISTINCTION
──────────────────────────────────────────────────────────────────────────────
Adapter     make incompatible fit         CHANGES it     converts contract
Bridge      split 2 axes (M×N→M+N)        n/a (delegates) proactive separation
Composite   build part-whole trees        uniform        recursion via tree
Decorator   add behavior dynamically      KEEPS, stacks   enhances what it DOES
Facade      simplify a subsystem          NEW simple one  reduces complexity
Flyweight   share state across many       n/a            memory via intrinsic share
Proxy       control access                KEEPS, controls gates whether/how you reach it
```

**The wrapping-family decision tree** (the four that look identical mechanically):
```
Does it hold one object and present an interface?
├─ Is the interface DIFFERENT from the wrapped one?      → ADAPTER
├─ SAME interface, ADDING features meant to stack?        → DECORATOR
├─ SAME interface, CONTROLLING access (lazy/remote/auth)? → PROXY
└─ Many interfaces collapsed into one NEW simple one?     → FACADE
```

**The meta-lesson:** Structural patterns are Part 1's *"favor composition over inheritance"* made concrete. Every one of them solves a structural problem by *holding a reference and delegating* rather than by inheriting — Adapter delegates to translate, Bridge delegates across an axis, Composite delegates to children, Decorator delegates and adds, Facade delegates to orchestrate, Proxy delegates conditionally, Flyweight shares the delegate. When you internalize "hold-and-delegate," all seven become variations on one theme, distinguished by *intent*. And as always: in languages with first-class functions, several (Decorator, Adapter, parts of Proxy) shrink to function wrapping — the force is universal, the weight is local.