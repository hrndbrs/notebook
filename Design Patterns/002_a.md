# PART 2 — CORE THEORY

## How to Read This Part

For every pattern, I follow a consistent deep structure: the **force** it resolves (from Part 1's principles), **intent**, a **motivation story**, **structure** (ASCII), **internal mechanics**, **advantages**, **disadvantages**, **tradeoffs**, **when NOT to use it**, and the **language-relativity note** (how heavy or light it is across languages). Keep Part 1's mental loop running: *what force, is the axis real, what's the lightest local mechanism, does benefit exceed indirection cost.*

---

# PART 2A — CREATIONAL PATTERNS

**The unifying force:** Direct `new ConcreteClass()` calls weld calling code to specific classes (violating DIP and OCP). Every creational pattern inserts a seam between *"I need an object"* and *"here is precisely which class and how it's built."* They differ in *what kind* of creation flexibility they buy.

```
The spectrum of "what varies" in creation:

Singleton        → how MANY instances exist (exactly one)
Factory Method   → WHICH class is instantiated (one product, deferred to subclass)
Abstract Factory → which FAMILY of classes (many related products)
Builder          → HOW a complex object is assembled (step by step)
Prototype        → creating by COPYING an existing object
```

---

## 2A.1 — Singleton

### The Force

Sometimes exactly one instance of a thing must exist, and many parts of the system need to reach it: one configuration registry, one connection pool, one logger sink. The naive solutions — a global variable (no control over creation, no lazy init, no invariant guard) or passing the object everywhere (tedious, pollutes signatures) — each fail. Singleton resolves "ensure one, provide controlled global access."

### Intent

> Ensure a class has only one instance, and provide a global point of access to it.

### Motivation Story

You're building an app with a configuration object loaded from disk. Loading is expensive and the config must be identical everywhere. If each module did `new Config()`, you'd parse the file repeatedly and risk divergent copies. You want: *load once, share one, reach it from anywhere.*

### Structure

```
┌──────────────────────────────┐
│         Singleton            │
├──────────────────────────────┤
│ - instance: Singleton  ◄─────┼── static, holds the one instance
│ - Singleton()                │   private constructor (no outside `new`)
├──────────────────────────────┤
│ + getInstance(): Singleton ──┼── static accessor; creates-or-returns
│ + businessMethod()           │
└──────────────────────────────┘
        ▲
        │ everyone calls Singleton.getInstance()
   ┌────┴────┬─────────┐
 ClientA  ClientB   ClientC
```

The mechanism has three parts: a **private constructor** (blocks outside instantiation), a **static field** (holds the sole instance), and a **static accessor** (`getInstance`) that lazily creates then returns it.

### Internal Mechanics & The Concurrency Trap

Naive lazy version:
```java
class Config {
    private static Config instance;
    private Config() { /* expensive load */ }
    public static Config getInstance() {
        if (instance == null) {        // (1) check
            instance = new Config();   // (2) create
        }
        return instance;
    }
}
```

This is **broken under concurrency.** Trace two threads:
```
Thread A: reads instance == null  → true
Thread B: reads instance == null  → true   (A hasn't assigned yet)
Thread A: creates instance #1
Thread B: creates instance #2     ← TWO instances. Invariant violated.
```
This is a **race condition** on the check-then-act sequence (a TOCTOU bug — time-of-check to time-of-use). Fixes, from heavy to idiomatic:

**1. Synchronize the whole method** (correct, slow — every call locks even after init):
```java
public static synchronized Config getInstance() { ... }
```

**2. Double-Checked Locking (DCL)** — lock only on first creation. Subtle: requires `volatile` or the JVM can let a thread see a *partially constructed* object due to instruction reordering:
```java
private static volatile Config instance;  // volatile is MANDATORY
public static Config getInstance() {
    if (instance == null) {                 // first check (no lock)
        synchronized (Config.class) {
            if (instance == null) {         // second check (locked)
                instance = new Config();
            }
        }
    }
    return instance;
}
```
The `volatile` prevents the reordering where the reference is published before the constructor finishes — a notorious bug that ran in production JVMs for years before being understood.

**3. Initialization-on-demand holder** (the idiomatic Java solution — lazy, thread-safe, lock-free, leverages JVM class-init guarantees):
```java
class Config {
    private Config() {}
    private static class Holder {           // loaded only when referenced
        static final Config INSTANCE = new Config();
    }
    public static Config getInstance() { return Holder.INSTANCE; }
}
```
The JVM guarantees a class is initialized exactly once, atomically, on first use — so the inner `Holder` class isn't loaded until `getInstance` is first called, and the JVM's own locking handles thread safety. Elegant: it offloads the hard problem to the runtime.

**4. Enum singleton** (Joshua Bloch's recommendation — serialization-safe and reflection-safe for free):
```java
enum Config { INSTANCE; public void method() {} }
```

### Advantages
- Guaranteed single instance with controlled lifecycle (lazy init possible).
- Global access without polluting every signature.
- Lazy initialization saves resources if the object is never needed.

### Disadvantages (This Is the Most-Criticized Pattern — Know Why)
- **It's a global, and globals are coupling magnets.** Any code can grab the singleton, creating hidden dependencies that don't appear in constructors or signatures. You can't tell what a class depends on by reading its interface.
- **Murders testability.** Tests can't easily substitute a fake. Static state persists *between tests*, causing order-dependent flakiness. This alone disqualifies it in many shops.
- **Violates SRP.** The class does its job *and* manages its own lifecycle/uniqueness — two responsibilities.
- **Hides dependencies → violates DIP.** Clients depend on the concrete `Config` class directly, not an abstraction.
- **Concurrency is genuinely hard** (see above).
- **Lifecycle rigidity.** "Exactly one" is a global assertion that's painful to relax later (e.g., when you discover you need one-per-tenant).

### Tradeoffs & The Modern View
The *uniqueness* requirement is often legitimate (one connection pool). The *global access mechanism* is what's toxic. **The modern resolution: keep "single instance" as a lifecycle decision but deliver it via Dependency Injection** — the DI container creates one instance and *injects* it where needed. You get singleness *without* global static access, *without* the testability and coupling damage. This is why senior engineers say "single*ton* bad, single *instance* fine" — distinguish the *intent* (one instance) from the *implementation* (global static accessor).

### When NOT to Use
When you reach for it merely for convenient global access (use DI), when the "one" might become "many" (per-tenant, per-request), or in test-heavy codebases. If you find yourself writing `X.getInstance()` scattered everywhere, that's the smell.

### Language Note
In Go, a package-level variable with `sync.Once` is the idiom (no class needed). In Python, a module *is* a singleton (imported once, cached in `sys.modules`) — just use module-level state. In most languages, the framework's DI container makes the explicit pattern unnecessary.

---

## 2A.2 — Factory Method

### The Force

A class needs to create objects, but it shouldn't hardcode *which concrete class* — because the concrete choice varies by context, subclass, or configuration. Hardcoding `new PdfReport()` violates OCP (adding a report type means editing the creator) and DIP (depending on a concrete). Factory Method resolves "let a method decide the concrete class, deferrable to subclasses."

### Intent

> Define an interface for creating an object, but let subclasses decide which class to instantiate. Factory Method lets a class defer instantiation to subclasses.

### Motivation Story

A `LogisticsApp` plans deliveries. Its `planDelivery()` logic is identical regardless of transport, but it must create a `Transport` object — a `Truck` for road logistics, a `Ship` for sea. You don't want `planDelivery()` cluttered with `if (road) new Truck() else new Ship()`. Instead, the base class calls an abstract `createTransport()`; `RoadLogistics` overrides it to return a `Truck`, `SeaLogistics` returns a `Ship`. The creation decision is *deferred to the subclass* while the surrounding algorithm stays put.

### Structure

```
┌─────────────────────────┐          ┌──────────────┐
│      Creator            │          │  «Product»   │
├─────────────────────────┤  creates ├──────────────┤
│ + someOperation()       │ ───────► │ + operation()│
│ + createProduct():Product│ (abstract)└──────────────┘
└───────────▲─────────────┘                 ▲
            │ overrides                      │ implements
   ┌────────┴─────────┐            ┌─────────┴────────┐
   │ ConcreteCreatorA │            │ ConcreteProductA │
   │ +createProduct() │──creates──►│                  │
   │   →new ProductA  │            └──────────────────┘
   └──────────────────┘

someOperation() {
    Product p = createProduct();  // calls the overridable factory method
    p.operation();                // works against the ABSTRACTION
}
```

### Internal Mechanics

The key trick is **polymorphic dispatch on the creation step.** `someOperation()` (in the base) calls `createProduct()`. At runtime, the *actual subclass's* override runs (vtable dispatch — Part 3 details this), returning the right concrete product. The base class never knows the concrete type; it only touches the `Product` interface. This is **the Template Method pattern applied to object creation** — the skeleton is fixed, one step (creation) is varied via override.

### Advantages
- Removes concrete-class coupling from the business logic (satisfies DIP).
- **Open/Closed:** new products = new subclass, no edits to existing code.
- Single Responsibility: creation code is isolated in one place.
- The creation logic can be reused/overridden cleanly.

### Disadvantages
- Introduces a parallel class hierarchy (a creator subclass per product) — class proliferation.
- Can be overkill when there's only ever one product type (YAGNI).
- The subclassing requirement is rigid — you must subclass the creator to vary the product, even if subclassing the creator is otherwise unwanted.

### Tradeoffs
Factory Method buys OCP-compliant extensibility at the cost of a doubled hierarchy. It's justified when the *surrounding algorithm* is substantial and shared, and the *only* variation is which product gets made. If the surrounding logic is trivial, a simple factory function is lighter.

### Factory Method vs. "Simple Factory" (a critical clarification)
The **Simple Factory** (a.k.a. static factory) is *not* a GoF pattern — it's just a method with a `switch` that returns products:
```java
static Transport create(String type) {
    return switch(type) { case "road" -> new Truck(); case "sea" -> new Ship(); };
}
```
This centralizes creation but *violates OCP* (every new type edits the switch). Factory Method *trades the switch for polymorphism* to restore OCP. Beginners conflate these constantly. The Simple Factory is fine and pragmatic for small, stable type sets; Factory Method earns its weight when extensibility-without-modification matters.

### When NOT to Use
When products don't vary, when a simple factory function suffices, or in languages where you'd just pass a constructor/function as a value.

### Language Note
In Go, you return interfaces from constructor functions (`func NewTransport(kind string) Transport`) — no inheritance. In Python/JS, a function returning objects, or passing the *class itself* as a first-class value (`create(cls)`), dissolves most of the ceremony. The full subclassing version is genuinely a Java/C++/C# shape.

---

## 2A.3 — Abstract Factory

### The Force

Factory Method makes *one* product. But sometimes you must create **families of related products that must be used together and stay consistent**, without coupling to their concrete classes. A GUI toolkit must produce a `Button`, `Checkbox`, and `Scrollbar` that all match — all "Windows-style" or all "macOS-style." Mixing a Windows button with a macOS checkbox is a bug. Abstract Factory resolves "create whole consistent families, swap the entire family at once."

### Intent

> Provide an interface for creating families of related or dependent objects without specifying their concrete classes.

### Motivation Story

A cross-platform app renders UI. On Windows it needs `WinButton`, `WinCheckbox`; on macOS, `MacButton`, `MacCheckbox`. The app logic shouldn't know which OS it's on. You define a `GUIFactory` interface with `createButton()` and `createCheckbox()`. `WinFactory` produces the Windows family; `MacFactory` the Mac family. At startup you pick one factory; the rest of the app calls `factory.createButton()` and *automatically gets a consistent family.* Switching OS = switching one factory object.

### Structure

```
┌────────────────────┐
│  «AbstractFactory» │
├────────────────────┤
│ +createButton()    │
│ +createCheckbox()  │
└─────────▲──────────┘
          │
   ┌──────┴───────┐
   │              │
┌──┴─────────┐ ┌──┴─────────┐
│ WinFactory │ │ MacFactory │
│+createBtn  │ │+createBtn  │──┐
│ →WinButton │ │ →MacButton │  │ each factory produces
│+createChk  │ │+createChk  │  │ a CONSISTENT family
│ →WinCheck  │ │ →MacCheck  │  │
└────────────┘ └────────────┘  │
                               ▼
   «Button»            «Checkbox»
   ▲      ▲             ▲      ▲
 WinBtn  MacBtn      WinChk  MacChk
```

Note: it's a **grid**. Rows = product types (Button, Checkbox). Columns = families (Win, Mac). Abstract Factory keeps each column internally consistent.

### Internal Mechanics

The client holds an `AbstractFactory` reference. Every `create*()` call dispatches polymorphically to the chosen concrete factory, which `new`s the matching concrete product. The *coordination guarantee* — that you never mix families — comes from the fact that *one factory object produces all members*, so they're structurally guaranteed to belong together.

### Advantages
- Guarantees product-family consistency (the core value — can't mix Win + Mac).
- Isolates concrete classes from client (DIP, OCP for whole families).
- Swapping families is a one-line change (inject a different factory).

### Disadvantages
- **Heavy.** A new *product type* (say, `Slider`) means adding a method to the abstract factory *and every concrete factory* — violating OCP along the *product-type* axis even as it supports OCP along the *family* axis. This asymmetry is the pattern's defining tension.
- Many classes/interfaces (the grid explodes: M product types × N families).
- Significant ceremony for small needs.

### Tradeoffs
Abstract Factory optimizes for *adding new families* (cheap: new column) at the expense of *adding new product types* (expensive: touch every factory). Choose it only when family-consistency is a real requirement and families change more than product types. If you only have one family, it's pure over-engineering.

### Relationship to Factory Method
Abstract Factory is often *implemented using* Factory Methods (each `create*` is a factory method) — or using Prototype (each factory holds prototype instances it clones). It's a "factory of factory methods."

### When NOT to Use
Single product family, or when product types churn more than families. The grid's rigidity bites hard if you misjudge which axis varies.

### Language Note
In dynamic languages, a "factory" can be a dict/map of constructor functions, or a module per family. The class-grid is a statically-typed-language artifact.

---

## 2A.4 — Builder

### The Force

Some objects are complex to construct: many parameters, some optional, some interdependent, requiring multi-step assembly and validation. The naive solutions fail: a constructor with 10 parameters is unreadable (`new Pizza(true, false, true, 3, null, "thin", ...)` — the **telescoping constructor** anti-pattern) and unsafe (positional args swap silently); a no-arg constructor + setters allows *invalid intermediate states* and isn't thread-safe or immutable. Builder resolves "assemble a complex object step by step, readably, with validation, producing a valid (often immutable) final product."

### Intent

> Separate the construction of a complex object from its representation, so the same construction process can create different representations.

### Motivation Story

You build an HTTP request: URL (required), method, headers (many, optional), body (optional), timeout (optional). A constructor for every combination is combinatorial madness. With a builder:
```java
HttpRequest req = new HttpRequest.Builder("https://api.com")
    .method("POST")
    .header("Auth", token)
    .header("Accept", "json")
    .body(payload)
    .timeout(5000)
    .build();   // validates, then constructs the immutable request
```
Readable, order-independent, optional-friendly, and `build()` is the single validation gate that guarantees a valid object emerges.

### Structure

```
┌──────────────┐     directs    ┌───────────────────┐
│  Director    │ ─────────────► │    «Builder»      │
│ (optional)   │                ├───────────────────┤
│ construct()  │                │ +buildPartA()     │
└──────────────┘                │ +buildPartB()     │
                                │ +getResult()      │
                                └─────────▲─────────┘
                                          │
                                 ┌────────┴─────────┐
                                 │ ConcreteBuilder  │
                                 │ assembles parts, │
                                 │ holds Product    │
                                 └──────────────────┘
                                          │ builds
                                          ▼
                                     ┌─────────┐
                                     │ Product │
                                     └─────────┘
```

The **Director** is optional — it encapsulates a *known construction recipe* (e.g., "build a standard car"). In modern fluent builders, the client usually acts as its own director, calling steps directly. The Director earns its keep only when the *same sequence of steps* is reused to build many objects.

### Internal Mechanics

Each step method mutates the builder's internal fields and **returns `this`** (enabling the fluent chain). `build()` runs cross-field validation (e.g., "if method is GET, body must be null"), then constructs the final object — often passing all fields to a *private constructor* of an immutable class, so the product has no setters and can't be mutated post-construction. The builder is the *mutable scaffold*; the product is the *immutable result*.

### Advantages
- Readable construction of complex objects; named steps beat positional args.
- Handles optional parameters gracefully (no constructor explosion).
- Single validation point (`build()`) guarantees valid objects; supports immutability.
- Same process can yield different representations (a `CarBuilder` and `CarManualBuilder` from the same Director steps).
- Step-by-step construction allows deferred/conditional assembly.

### Disadvantages
- More code: a whole builder class per product, often mirroring its fields.
- Indirection for simple objects is pure overhead (YAGNI).
- The mutable builder + immutable product duality is a concept beginners stumble on.

### Tradeoffs
Builder trades extra boilerplate for construction clarity and object validity. Justified when objects have *many optional/interdependent parameters* or *must be immutable*. For a 2-field object, it's overkill — use a constructor.

### When NOT to Use
Few parameters, no optionality, no immutability requirement. The trigger is "telescoping constructors" or "objects assembled in stages" — without those symptoms, skip it.

### Language Note
Languages with **named/default/keyword arguments (Python, Kotlin, Swift, C#)** dissolve much of Builder's purpose: `Pizza(size="large", cheese=True)` gives readability and optionality natively. Builder remains valuable in Java (no named args) and for genuinely staged/validated construction. Kotlin's `apply`/DSL builders and Go's "functional options" pattern (`NewServer(WithTimeout(5), WithTLS(cfg))`) are idiomatic local reincarnations of the same force.

---

## 2A.5 — Prototype

### The Force

Sometimes creating an object from scratch is expensive (heavy computation, DB load, deep config) or you need a *copy of an existing configured object* without coupling to its concrete class. You have an instance; you want another just like it. Re-running construction is wasteful or impossible (you may not know the exact class — you just have the object). Prototype resolves "create new objects by cloning an existing instance rather than constructing anew."

### Intent

> Specify the kinds of objects to create using a prototypical instance, and create new objects by copying this prototype.

### Motivation Story

A graphics editor has shapes the user has configured (color, size, rotation, embedded data). "Duplicate" must produce an identical independent copy. The editor doesn't know each shape's concrete class — it just holds `Shape` references. So `Shape` declares `clone()`; each concrete shape knows how to copy *itself*. The editor calls `shape.clone()` polymorphically and gets a correct copy without knowing the type. Similarly, a game spawns 1000 enemies from one configured "prototype enemy" by cloning, avoiding 1000 expensive constructions.

### Structure

```
┌────────────────┐
│  «Prototype»   │
├────────────────┤
│ + clone():Prototype │◄── the copy contract
└───────▲────────┘
        │
   ┌────┴─────────────┐
   │ ConcretePrototype│
   │ + clone()        │── returns a copy of itself
   │   →copy of this  │
   └──────────────────┘

Client: Prototype p2 = p1.clone();  // new object, same state, type unknown to client
```

### Internal Mechanics & The Deep/Shallow Trap

The critical subtlety is **shallow vs. deep copy:**

```
Original object:  ┌─────────┐
                  │ name:"A"│
                  │ tags: ──┼──► [list in heap]
                  └─────────┘

SHALLOW copy:     ┌─────────┐
                  │ name:"A"│
                  │ tags: ──┼──► [SAME list]   ← both share one list!
                  └─────────┘                     mutate via one → both change (bug)

DEEP copy:        ┌─────────┐
                  │ name:"A"│
                  │ tags: ──┼──► [NEW list copy] ← independent
                  └─────────┘
```

A **shallow copy** duplicates the top-level fields but *shares referenced sub-objects* — so mutating a nested object through the clone corrupts the original (aliasing bug). A **deep copy** recursively clones nested objects, yielding true independence — but is expensive and must handle **cycles** (object graphs that reference themselves, or you get infinite recursion). Production deep-copy uses a "visited" map to break cycles. This deep/shallow distinction is the entire difficulty of Prototype and a frequent interview probe.

### Advantages
- Clone without coupling to concrete classes (you only need the `clone()` contract).
- Cheaper than re-constructing expensive objects.
- Can produce complex pre-configured objects on demand; alternative to factory hierarchies.
- Lets you add/remove "product types" at runtime (register prototype instances).

### Disadvantages
- **Deep-copying complex graphs is hard** — cycles, shared references, resources (open files/sockets can't be naively cloned).
- Every class must implement `clone()` correctly (easy to get subtly wrong).
- Java's built-in `Cloneable`/`clone()` is famously broken (Bloch: "extralinguistic," shallow by default, awkward with final fields) — most use copy constructors or serialization instead.

### Tradeoffs
Prototype trades cloning-correctness-burden for cheap, type-agnostic copying. Justified when construction is expensive or you must copy configured-but-opaque objects. The deep-copy hazard is the price.

### When NOT to Use
When construction is cheap (just `new` it), when objects hold un-clonable resources, or when the object graph has nasty cycles/sharing semantics.

### Language Note
JavaScript is *built on* prototypes (`Object.create`, the prototype chain) — it's native. Python has `copy.copy`/`copy.deepcopy` built-in (the pattern is a stdlib call). Go has no built-in deep copy; you write it or use libraries. C#/Java use copy constructors or `MemberwiseClone`. The pattern as an *explicit class structure* is rarely needed where a language provides clone primitives.

---

## 2A — Creational Patterns: Comparison & Synthesis

```
PATTERN          VARIES…                  KEY COST              REACH FOR WHEN…
─────────────────────────────────────────────────────────────────────────────
Singleton        instance COUNT (=1)      global coupling,      one instance truly
                                          testability           required (prefer DI)
Factory Method   WHICH product class      parallel hierarchy    shared algorithm,
                 (1 product, via subclass)                      product varies by subclass
Abstract Factory product FAMILY           class grid explosion  consistent families,
                 (many related)           (rigid on new types)  swap whole family
Builder          construction PROCESS     builder boilerplate   many optional params,
                 (step by step)                                 immutability, validation
Prototype        creation by COPYING      deep-copy hazards     expensive construction,
                                                                type-agnostic duplication
```

**The meta-lesson:** All five attack the *same root coupling* — direct `new ConcreteClass()` — but along different axes (count, class, family, process, copying). In code review, when you see proliferating `new` calls causing rigidity, ask *which axis is actually varying*; that question selects the pattern. And recall Part 1: in languages with first-class functions, named arguments, and clone primitives, several of these (Factory, Builder, Prototype, Singleton) shrink dramatically or vanish into language features. The *force* is universal; the *weight* is local.