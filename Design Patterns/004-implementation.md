# PART 4 — IMPLEMENTATION

## From Naive to Production-Grade

Parts 1–3 gave you the *why* (forces, principles), the *what* (the catalog), and the *how it runs* (internals). Part 4 is where you build. We take a small number of patterns and implement each through **four ascending tiers** — Basic, Intermediate, Advanced, Production-grade — explaining every line, every design decision, every alternative, and every tradeoff. The goal is not to re-implement all 29 patterns (that's reference material; you have the catalog) but to teach the *craft of implementation*: how a staff engineer turns a textbook diagram into code that survives production.

We focus on patterns that best teach implementation craft and that you'll actually write: **Strategy, Observer, Decorator, Factory/DI, Command (with undo), and a thread-safe Singleton-via-DI.** Each reveals different production concerns — thread safety, error handling, resource lifecycle, testability, generics, and idiomatic language fit.

The recurring discipline at every tier:
```
BASIC        → makes it work (the textbook shape)
INTERMEDIATE → makes it reusable (generics, interfaces, config)
ADVANCED     → makes it robust (errors, edge cases, concurrency)
PRODUCTION   → makes it survive (observability, lifecycle, testing, ops)
```

---

## 4.1 — Strategy: Four Tiers

Strategy is our teaching vehicle for the most important implementation lesson — *the same pattern collapses or expands depending on how much the situation demands.*

### Tier 1 — Basic (the textbook shape)

```java
// The contract every strategy fulfills.
interface DiscountStrategy {
    double applyDiscount(double price);
}

// Concrete algorithms.
class NoDiscount implements DiscountStrategy {
    public double applyDiscount(double price) { return price; }
}
class PercentageDiscount implements DiscountStrategy {
    private final double percent;
    PercentageDiscount(double percent) { this.percent = percent; }
    public double applyDiscount(double price) {
        return price * (1 - percent / 100);
    }
}

// The context holds and delegates to a strategy.
class Checkout {
    private DiscountStrategy strategy;
    Checkout(DiscountStrategy strategy) { this.strategy = strategy; }
    double total(double price) { return strategy.applyDiscount(price); }
}
```

**Line-by-line reasoning:**
- `interface DiscountStrategy` — *the seam* (Part 1). Everything depends on this abstraction, not concrete classes. This single line is what buys OCP and testability.
- `final double percent` — `final` makes the strategy **immutable**: once constructed, it can't change. Immutable strategies are *safely shareable* (one `PercentageDiscount(10)` can serve all threads — recall Part 3's Flyweight/State note) and have no hidden state bugs. Make strategies immutable by default.
- `Checkout(DiscountStrategy strategy)` — **constructor injection** (Part 2D). The dependency arrives from outside; `Checkout` never does `new PercentageDiscount()`, so it's not welded to any concrete strategy (DIP).
- `strategy.applyDiscount(price)` — the polymorphic call (the vtable dispatch of Part 3.3).

**What's wrong with stopping here:** Nothing, *for a small fixed case.* But it's verbose for what may be trivial logic, gives no way to *select* a strategy by name/config, and the discount has no context (it only sees `price`, not the customer or cart).

### Tier 2 — Intermediate (selectable, richer context, idiomatic)

Two improvements: let strategies see *more context*, and provide a *selector* so callers choose by key rather than constructing concretes (which would re-couple them).

```java
// Richer context object — strategies often need more than one value.
record PricingContext(double price, Customer customer, int quantity) {}

@FunctionalInterface
interface DiscountStrategy {
    double apply(PricingContext ctx);   // now sees full context
}

// A registry maps keys → strategies, so callers select by name (from config, DB, request).
class DiscountRegistry {
    private final Map<String, DiscountStrategy> strategies = new HashMap<>();
    DiscountRegistry register(String key, DiscountStrategy s) {
        strategies.put(key, s); return this;     // fluent
    }
    DiscountStrategy get(String key) {
        return strategies.getOrDefault(key, ctx -> ctx.price());  // Null Object default!
    }
}
```

**Decisions explained:**
- `record PricingContext(...)` — a **context object** instead of a long parameter list. When a strategy needs many inputs, pass *one* immutable context rather than widening the interface every time a new input appears (this is forward-compatible — add a field to the record, not a parameter to every strategy). Java `record` gives immutability + equals/hashCode free.
- `@FunctionalInterface` — declares the interface has *one* abstract method, so strategies can be **lambdas**: `ctx -> ctx.price() * 0.9`. This is Part 1's language-relativity made real — in modern Java, a Strategy *is* a lambda; you rarely write a named class. The interface still exists for *naming and typing*, but implementations collapse to one-liners.
- `DiscountRegistry` — solves "how do callers pick a strategy without coupling to concretes?" They ask the registry by key (a string from config/DB/request). This is the seam between *selection* (data-driven) and *implementation* (code). Adding a new discount = `register("BLACK_FRIDAY", ctx -> ...)` — pure OCP.
- `getOrDefault(key, ctx -> ctx.price())` — a **Null Object** (2D.4) inline: unknown key → "no discount" strategy, never `null`. No caller null-checks; no `NullPointerException`.

**Tradeoff introduced:** The registry adds a layer and a failure mode (typo'd keys silently return the default — is silent the right behavior, or should an unknown key *throw*? Depends on whether discounts are open-ended config or a fixed known set). This is a real production decision, not a detail.

### Tier 3 — Advanced (robust: validation, errors, composition)

Now we harden it. Strategies can fail (a discount that calls a pricing service), must validate, and may need to *compose* (stack a loyalty discount on a seasonal one).

```java
sealed interface DiscountResult permits Applied, Rejected {}
record Applied(double finalPrice, String reason) implements DiscountResult {}
record Rejected(String reason) implements DiscountResult {}   // explicit failure, not exception

interface DiscountStrategy {
    DiscountResult apply(PricingContext ctx);
}

// Composition: combine strategies (Decorator/Composite synergy).
class CompositeDiscount implements DiscountStrategy {
    private final List<DiscountStrategy> chain;
    CompositeDiscount(DiscountStrategy... s) { this.chain = List.of(s); }
    public DiscountResult apply(PricingContext ctx) {
        double price = ctx.price();
        var reasons = new StringBuilder();
        for (var s : chain) {
            var r = s.apply(new PricingContext(price, ctx.customer(), ctx.quantity()));
            if (r instanceof Rejected rej) return rej;     // short-circuit on failure
            if (r instanceof Applied a) {
                price = a.finalPrice();
                reasons.append(a.reason()).append("; ");
            }
        }
        return new Applied(price, reasons.toString());
    }
}
```

**Why each choice:**
- `sealed interface DiscountResult permits Applied, Rejected` — a **result type** instead of throwing exceptions for *expected* failures (a discount being invalid is a normal business outcome, not an exceptional one). `sealed` means the compiler *knows the complete set* of subtypes, so the `instanceof` checks below are **exhaustive** — this is the modern, type-safe alternative to Visitor (Part 2C.10) and the right way to model "success or known-failure." Reserving exceptions for *truly exceptional* conditions (programming bugs, infra failures) is a senior discipline — exceptions are expensive (stack capture) and control-flow-obscuring; expected outcomes belong in the return type.
- `CompositeDiscount` — strategies *compose*. This is the moment Part 2's "patterns combine" becomes code: Composite + Strategy + a touch of Chain of Responsibility (short-circuit). Each composed strategy sees the *running* price, so discounts stack correctly and *order matters* (a documented, deliberate property — exactly Decorator's ordering concern from Part 2B).
- **Short-circuit on `Rejected`** — fail fast; if any discount is invalid, the whole composition fails with that reason, no partial application.

**The lesson of this tier:** robustness is mostly about *honestly modeling failure* (result types), *making the complete set of outcomes visible to the compiler* (sealed types → exhaustiveness), and *composing* small pieces safely.

### Tier 4 — Production-grade (observable, testable, operable)

Production code must be *observable* (you must see what it did when a customer disputes a charge), *resilient* (a strategy calling an external service must time out and fall back), and *testable* (every path covered).

```java
class ObservableDiscount implements DiscountStrategy {
    private final DiscountStrategy delegate;
    private final Metrics metrics;
    private final Logger log;
    private final String name;

    ObservableDiscount(String name, DiscountStrategy delegate, Metrics m, Logger l) {
        this.name = name; this.delegate = delegate; this.metrics = m; this.log = l;
    }

    public DiscountResult apply(PricingContext ctx) {
        long start = System.nanoTime();
        try {
            DiscountResult r = delegate.apply(ctx);
            metrics.timing("discount.apply", System.nanoTime() - start,
                           "strategy", name, "outcome", r.getClass().getSimpleName());
            log.debug("discount {} for customer {} → {}", name, ctx.customer().id(), r);
            return r;
        } catch (Exception e) {
            metrics.increment("discount.error", "strategy", name);
            log.error("discount {} failed for customer {}", name, ctx.customer().id(), e);
            return new Rejected("internal error");   // fail safe — never crash checkout
        }
    }
}
```

**Production reasoning:**
- This is itself a **Decorator** (Part 2B) wrapping any `DiscountStrategy` to add **observability** without touching the strategy's logic — the cleanest possible demonstration of "add cross-cutting behavior by wrapping." Logging, metrics, and timing are *separate concerns* (SRP); they don't belong inside each discount's business logic.
- `metrics.timing(...)` / `metrics.increment(...)` — **you cannot operate what you cannot see.** In production, when revenue dips, you query "which discount strategies fired, how often, with what latency, how many errors?" If it's not instrumented, you're blind. This instrumentation is *not optional* in real systems.
- **Structured logging** with the customer id — when a specific customer disputes their discount, you grep their id and replay exactly what happened. Note: *log the id, not PII* (Part 7 — Security covers why; logs are a data-leak vector).
- `catch (Exception e) → return Rejected("internal error")` — **fail safe.** A bug in a discount strategy must *never* crash the checkout (revenue!). It logs, alerts (via the error metric → dashboard → pager), and degrades gracefully to "no discount." Deciding the *safe* fallback for each failure is core production design. (Note also: the *user-facing* reason is generic — never leak internal error details to clients, another Part 7 point.)
- Resilience for external-service-calling strategies would add a **timeout + circuit breaker** here (Part 5/8) — wrap the delegate so a slow pricing service can't hang every checkout.

**Testing this tier (non-negotiable):**
```java
@Test void rejectedStrategyDoesNotCrashAndIsObserved() {
    var failing = (DiscountStrategy) ctx -> { throw new RuntimeException("boom"); };
    var metrics = new FakeMetrics();          // injected fake (DI pays off here!)
    var strat = new ObservableDiscount("test", failing, metrics, new FakeLogger());

    var result = strat.apply(sampleContext());

    assertInstanceOf(Rejected.class, result);                    // failed safe
    assertEquals(1, metrics.count("discount.error"));            // observed the failure
}
```
The fakes are injectable *precisely because* we used DI and programmed to interfaces from Tier 1 — **testability is the dividend of good structure, not a separate activity.** This is why "favor composition + program to an interface" pays off: the seams that enable swapping implementations in production are the *same seams* that enable swapping in test doubles.

**The four-tier arc, summarized:** the *pattern* (delegate to a swappable strategy object) is constant across all four tiers; what grows is the *production scaffolding around it* — context objects, registries, result types, composition, observability, resilience, and tests. A junior writes Tier 1 and thinks they're done. A staff engineer knows the pattern is the *easy 20%* and the production hardening is the *essential 80%*.

---

## 4.2 — Observer: Implementing Around the Hazards

Part 2C catalogued Observer's hazards (leaks, re-entrancy, threading, ordering); Part 3 showed them as memory-level facts. Implementation is the art of *defusing* each one.

### Tier 1 — Basic

```java
interface Observer<T> { void onChange(T value); }

class Subject<T> {
    private final List<Observer<T>> observers = new ArrayList<>();
    private T value;
    void subscribe(Observer<T> o) { observers.add(o); }
    void unsubscribe(Observer<T> o) { observers.remove(o); }
    void set(T newValue) {
        this.value = newValue;
        for (Observer<T> o : observers) o.onChange(newValue);  // notify
    }
}
```

`Observer<T>` is **generic** — one Observer abstraction for any value type, not a new interface per type. But this naive version has *every* hazard from Part 2C live: re-entrancy crashes (mutating `observers` during the loop), no thread safety, leaks (no weak refs), one observer's exception kills the rest.

### Tier 2–3 — Intermediate + Advanced (defusing the hazards)

We jump straight to defusing, because in Observer the hazards *are* the implementation challenge:

```java
class Subject<T> {
    // CopyOnWriteArrayList: thread-safe, and iteration sees a STABLE snapshot,
    // so re-entrant subscribe/unsubscribe during notification can't corrupt the loop.
    private final List<Observer<T>> observers = new CopyOnWriteArrayList<>();
    private volatile T value;

    void subscribe(Observer<T> o) { observers.add(Objects.requireNonNull(o)); }
    void unsubscribe(Observer<T> o) { observers.remove(o); }

    void set(T newValue) {
        this.value = newValue;
        for (Observer<T> o : observers) {       // iterates the snapshot — re-entrancy safe
            try {
                o.onChange(newValue);
            } catch (Exception e) {
                // ISOLATE failures: one bad observer must not stop the others.
                log.error("observer {} threw", o, e);
            }
        }
    }
}
```

**Each defense, mapped to its Part 2C hazard:**
- **Re-entrancy** (an observer subscribes/unsubscribes mid-notification → `ConcurrentModificationException` in Part 3's loop) → `CopyOnWriteArrayList`. Iteration runs over an immutable snapshot taken at loop start, so concurrent mutation is invisible to the in-flight notification. (Cost: each mutation copies the whole array — fine for read-heavy/write-rare observer lists, which is the typical case.)
- **Threading** (which thread runs `onChange`, and is the list safe under concurrent access?) → the concurrent list handles structural thread-safety; `volatile value` ensures visibility of the latest value across threads. *Remaining decision:* do observers run on the caller's thread (synchronous, simple, but a slow observer blocks the setter) or on an executor (asynchronous, isolated, but reordered and harder to reason about)? This is a deliberate architectural choice — see Tier 4.
- **One observer crashing the rest** → the per-observer `try/catch`. Isolation is essential: in a system with ten observers, observer #3 throwing must not deprive #4–#10 of their notification. (Contrast: the naive version's uncaught exception aborts the whole `set()`.)
- **Null observer** → `requireNonNull` fails fast at subscribe time, not later with a confusing NPE deep in the loop.

**The leak hazard** (lapsed listener — Part 2C/3.7) needs a deliberate choice:
```java
// Option A: WeakReference observers — GC can reclaim observers nobody else references,
// preventing the classic leak. Cost: observers vanish if not strongly held elsewhere
// (a footgun — a lambda observer with no other reference disappears immediately).
// Option B: explicit lifecycle — return a Subscription handle the caller must close.
interface Subscription extends AutoCloseable { void close(); }

Subscription subscribe(Observer<T> o) {
    observers.add(o);
    return () -> observers.remove(o);   // closing unsubscribes — try-with-resources friendly
}
```
**Production strongly prefers Option B** (explicit `Subscription`/`Disposable` — exactly what RxJava, Reactor, and most modern event libraries do): the caller gets a handle and closes it (often via try-with-resources or a framework lifecycle hook), making subscription lifetime *explicit and deterministic* rather than relying on GC timing (Option A's weak refs make *when* an observer stops receiving events nondeterministic — a debugging nightmare).

### Tier 4 — Production (async, ordered, backpressured)

Production observers often must be *asynchronous* (don't block the publisher) yet *ordered* (events processed in sequence) and *backpressure-aware* (a slow observer mustn't cause unbounded memory growth):

```java
class AsyncSubject<T> {
    private final List<Observer<T>> observers = new CopyOnWriteArrayList<>();
    // Per-subject single-thread executor: events run async (don't block publisher)
    // but SERIALLY (preserving order — critical for correctness).
    private final ExecutorService notifier = Executors.newSingleThreadExecutor();

    void set(T newValue) {
        // Snapshot the observers NOW (on the publishing thread) for consistency,
        // then dispatch the notification work to the executor.
        var snapshot = List.copyOf(observers);
        notifier.submit(() -> {
            for (var o : snapshot) {
                try { o.onChange(newValue); }
                catch (Exception e) { log.error("observer failed", e); }
            }
        });
    }
}
```

At this point you're re-implementing what **reactive libraries** (RxJava, Project Reactor, Akka Streams) provide industrially — with *backpressure* (signaling a fast producer to slow when consumers lag), *schedulers* (controlling which threads run what), and *operators* (map/filter/buffer on event streams). **The production lesson: for non-trivial async Observer needs, don't hand-roll — use a reactive library**; but understanding *why* it's built the way it is (every design choice answers a hazard you now understand from first principles) is what lets you use it correctly and debug it when it misbehaves.

---

## 4.3 — Decorator: Implementation Craft and the Identity Trap

Decorator's implementation teaches a subtle production lesson most tutorials miss: the **identity/interface-erosion problem**.

### Basic → Advanced

```java
interface DataSource {
    void write(String data);
    String read();
}

class FileDataSource implements DataSource {           // the concrete base
    private final String path;
    FileDataSource(String path) { this.path = path; }
    public void write(String data) { /* write to file */ }
    public String read() { /* read from file */ return ""; }
}

// Base decorator: holds a wrapped DataSource, forwards everything by default.
abstract class DataSourceDecorator implements DataSource {
    protected final DataSource wrapped;       // composition (Part 1)
    DataSourceDecorator(DataSource wrapped) { this.wrapped = wrapped; }
    public void write(String data) { wrapped.write(data); }   // default: pass through
    public String read() { return wrapped.read(); }
}

class EncryptionDecorator extends DataSourceDecorator {
    EncryptionDecorator(DataSource w) { super(w); }
    public void write(String data) { wrapped.write(encrypt(data)); }     // add behavior
    public String read() { return decrypt(wrapped.read()); }
    private String encrypt(String s) { /* ... */ return s; }
    private String decrypt(String s) { /* ... */ return s; }
}

class CompressionDecorator extends DataSourceDecorator {
    CompressionDecorator(DataSource w) { super(w); }
    public void write(String d) { wrapped.write(compress(d)); }
    public String read() { return decompress(wrapped.read()); }
    private String compress(String s) { /* ... */ return s; }
    private String decompress(String s) { /* ... */ return s; }
}
```

**The `abstract DataSourceDecorator` base** is the key implementation craft: it provides *pass-through* defaults for every method, so each concrete decorator overrides *only* the methods it modifies. Without it, every decorator must implement *every* interface method just to forward them — brutal for wide interfaces (this is exactly why Java's `FilterInputStream` exists as a base decorator in `java.io`).

**Usage and the ordering hazard made physical:**
```java
DataSource source = new CompressionDecorator(new EncryptionDecorator(new FileDataSource("x")));
// write path: compress → encrypt → file.    read path: file → decrypt → decompress.
```
**Order is correctness here, not preference.** Compress-then-encrypt vs. encrypt-then-compress give *different bytes* — and encrypt-then-compress is actively *broken* (encrypted data is high-entropy and won't compress; worse, compression-then-encryption can leak information via compressed size — the CRIME/BREACH attack class, a real Part 7 security issue). The decorator chain's order is a *security-relevant decision* the construction site silently encodes. A production system would *enforce* valid orderings (e.g., a builder that only permits safe stacks) rather than trusting callers to nest correctly.

### The Identity Trap (Production Subtlety)

```java
FileDataSource original = new FileDataSource("x");
DataSource decorated = new EncryptionDecorator(original);

decorated instanceof FileDataSource   // → FALSE! the decorator is NOT a FileDataSource
decorated == original                 // → FALSE! different object identity
// If FileDataSource had a method getPath() not on the DataSource interface,
// you CANNOT call it through `decorated` — the decorator's interface hides it.
```
This is Decorator's documented disadvantage (Part 2B) as a concrete code trap: **a decorated object is not equal to, not type-identical to, and exposes only the *interface* of, the thing it wraps.** Code that relies on object identity (caches keyed by object, `==` checks), `instanceof` the concrete type, or concrete-only methods *breaks* when a decorator is introduced. Production mitigations: keep the *interface* complete enough that concrete-only methods aren't needed; if unwrapping is required, add an `unwrap()` method to the decorator base (as some frameworks do — e.g., JDBC's `Wrapper.unwrap()`); and never key caches/identity-maps on objects that might be decorated. Knowing this trap *before* it bites in production is exactly the value of studying implementation at depth.

---

## 4.4 — Factory + Dependency Injection: Wiring a Real Application

This section implements the patterns that *actually dominate* real codebases (Part 2D) and shows how an application is *wired together* — the implementation skill juniors most lack.

### The Problem: Object Graph Construction

A real service needs a database, a cache, a message publisher, and several other services — each depending on others. *Who constructs all this, and how?* The naive answer (each object `new`s its own dependencies) creates the welded, untestable mess DIP warns against. The implementation answer is **a composition root + dependency injection.**

### Tier 1–2 — Manual DI (the foundation everyone should understand first)

```java
// Each component declares its dependencies via the constructor (constructor injection).
class OrderService {
    private final OrderRepository repo;       // depends on ABSTRACTION (interface)
    private final PaymentGateway payments;    // abstraction
    private final EventPublisher events;      // abstraction
    OrderService(OrderRepository repo, PaymentGateway payments, EventPublisher events) {
        this.repo = repo; this.payments = payments; this.events = events;
    }
    // ... business logic uses repo, payments, events ...
}

// THE COMPOSITION ROOT — the ONE place that knows concrete types and wires everything.
// This is `main()` or a dedicated configuration class. Everywhere else stays abstract.
class CompositionRoot {
    static OrderService buildOrderService(Config cfg) {
        // construct leaves first, then objects that depend on them (bottom-up)
        DataSource ds = new HikariDataSource(cfg.dbUrl());          // connection pool
        OrderRepository repo = new PostgresOrderRepository(ds);     // concrete repo
        PaymentGateway payments = new StripePaymentGateway(cfg.stripeKey());
        EventPublisher events = new KafkaEventPublisher(cfg.kafkaБrokers());
        return new OrderService(repo, payments, events);           // inject all
    }
}
```

**The crucial concept — the Composition Root:** *all* concrete-type knowledge and wiring is concentrated in **one place** (`main`/`CompositionRoot`). Every other class depends only on interfaces and receives its collaborators via constructor. This means: (1) the *entire* application's dependency structure is readable in one file; (2) every class is trivially testable (inject fakes); (3) swapping Postgres→MySQL or Stripe→Adyen is a *one-line change in the root*, nothing else moves (OCP, DIP fully realized). **Understanding manual DI is mandatory before using a DI framework** — frameworks just *automate this wiring*; if you don't grasp what they're automating, you can't debug them.

**Note the `HikariDataSource`** — this is the legitimate "single instance" of Part 2A. The connection pool is created *once* in the root and injected wherever needed. *That's* how you get "one shared instance" without the Singleton anti-pattern: the composition root owns the single instance; DI distributes it. No global static, no `getInstance()`, fully testable.

### Tier 3–4 — Container-Based DI and Factories for Runtime Choices

When the object graph grows to hundreds of components, manual wiring becomes unwieldy, and a **DI container** (Spring, Guice, Dagger, .NET DI) automates it:

```java
// The container reads these declarations and auto-wires the graph by type.
@Component class PostgresOrderRepository implements OrderRepository { 
    PostgresOrderRepository(DataSource ds) { /* container injects ds */ }
}
@Component class OrderService {
    OrderService(OrderRepository r, PaymentGateway p, EventPublisher e) { /* auto-injected */ }
}
// The container resolves the entire graph: sees OrderService needs OrderRepository,
// finds PostgresOrderRepository, sees IT needs DataSource, provides the configured one, etc.
```

**Factories remain essential for *runtime* decisions the container can't make at startup** — when *which* implementation is chosen *per-request* based on data:

```java
// A Factory for choices that depend on runtime data (container can't decide this at boot).
@Component
class PaymentGatewayFactory {
    private final Map<Currency, PaymentGateway> gateways;   // container injects all gateways
    PaymentGatewayFactory(List<PaymentGateway> all) {
        this.gateways = all.stream().collect(toMap(PaymentGateway::currency, identity()));
    }
    PaymentGateway forCurrency(Currency c) {                 // runtime selection
        var g = gateways.get(c);
        if (g == null) throw new UnsupportedCurrencyException(c);  // explicit failure
        return g;
    }
}
```

**The division of labor (a key production insight):** the **DI container** handles *startup-time, type-based* wiring (the stable skeleton — "OrderService always needs an OrderRepository"); **factories** handle *runtime, data-based* selection (the dynamic choices — "this EUR order needs the European gateway"). Conflating them — using factories for everything (manual, verbose) or trying to make the container decide runtime-data-dependent choices (it can't) — is a common implementation mistake. Use each for what it's for.

**Container tradeoffs (be honest about them):** DI containers add "magic" — the wiring is implicit, errors surface at startup (or worse, runtime) as confusing reflection stack traces, and there's a learning curve and framework lock-in (Part 3 foreshadowed the debugging cost). Compile-time DI (Dagger, Spring's AOT) mitigates by generating wiring code at build time (errors at compile, no reflection cost, traceable). The tradeoff: container convenience and ecosystem vs. manual DI's explicitness and zero-magic. For small apps, manual DI is often *better* (no framework, fully explicit); for large apps, the container's automation wins.

---

## 4.5 — Command with Undo: Implementing Reversible Operations

Command (2C.3) with undo is the richest implementation challenge — it must capture *enough state to reverse itself*, which forces the Memento collaboration into real code.

### The Full Implementation

```java
interface Command {
    void execute();
    void undo();
}

// A command that captures the prior state needed to reverse itself (Memento collaboration).
class SetTextCommand implements Command {
    private final Document doc;
    private final String newText;
    private String previousText;        // the memento — captured at execute time

    SetTextCommand(Document doc, String newText) {
        this.doc = doc; this.newText = newText;
    }
    public void execute() {
        this.previousText = doc.getText();   // SNAPSHOT before mutating (this is the memento)
        doc.setText(newText);
    }
    public void undo() {
        doc.setText(previousText);           // RESTORE the snapshot
    }
}

// The invoker + history manager.
class CommandHistory {
    private final Deque<Command> undoStack = new ArrayDeque<>();
    private final Deque<Command> redoStack = new ArrayDeque<>();

    void execute(Command cmd) {
        cmd.execute();
        undoStack.push(cmd);
        redoStack.clear();          // a new action invalidates the redo branch (critical!)
    }
    void undo() {
        if (undoStack.isEmpty()) return;
        Command cmd = undoStack.pop();
        cmd.undo();
        redoStack.push(cmd);        // undone commands become redoable
    }
    void redo() {
        if (redoStack.isEmpty()) return;
        Command cmd = redoStack.pop();
        cmd.execute();              // re-executing re-snapshots, so it's re-undoable
        undoStack.push(cmd);
    }
}
```

**The implementation insights:**
- **`previousText` captured *in* `execute()`, not the constructor** — the command snapshots state *at the moment it runs*, because the document's state when the command is *created* may differ from when it's *executed* (commands can be queued). This timing is a subtle, correctness-critical decision.
- **`redoStack.clear()` on new execute** — the moment you perform a new action after undoing, the "redo future" is gone (you've branched the timeline). Every undo system works this way; forgetting this `clear()` produces a corrupt redo that re-applies actions from an abandoned branch. This single line is a famous source of undo bugs.
- **The two-stack design** (undo + redo) is the standard undo/redo architecture; `redo()` re-executes (which re-snapshots), keeping the invariant that anything on the undo stack can be undone again.

**The state-capture tradeoff in code (Part 2C.9 made concrete):**
- *This version* stores the **full previous text** — simple, but for a large document, every keystroke command holds a full-document copy → memory explosion (the Memento memory cost, live). 
- **Alternative — inverse operations:** store only the *delta* (`InsertCommand` stores "inserted 'x' at position 5"; `undo()` deletes 1 char at 5). Tiny memory, but only works for *reversible* operations and is more complex per command.
- **Alternative — command log (event sourcing):** store the *sequence of commands*; to reach any past state, replay from a checkpoint. This is the architectural-scale version (Part 2D/5).
A production editor uses **deltas for fine-grained edits** (memory-efficient) and **periodic full snapshots/checkpoints** (so undo doesn't replay thousands of deltas) — a hybrid, chosen by profiling memory vs. CPU. The implementation choice is a direct application of Part 3's cost model.

**Production additions:** command **coalescing** (merging 50 single-character commands into one "typed 'hello'" command, so Ctrl+Z undoes a word not a letter — a UX necessity), **transactional/macro commands** (Composite — group commands so they undo atomically), and **persistence** (serializing the command log for crash recovery — which requires commands to be serializable, influencing their design: store data, not object references). Each is a real feature built *on* the pattern's foundation.

---

## 4.6 — Cross-Language Implementation: The Same Pattern, Different Idioms

To cement Part 1's language-relativity at the implementation level, here is **Strategy** implemented idiomatically in four languages — note how the *same force* yields radically different *code weight*.

**Go (interface + struct, or just a function):**
```go
// Idiomatic Go: small interface, but a func type is even more idiomatic for single-method.
type Discount func(price float64) float64

func percentage(p float64) Discount {
    return func(price float64) float64 { return price * (1 - p/100) }
}
func checkout(price float64, d Discount) float64 { return d(price) }
// checkout(100, percentage(10)) — no classes, the function IS the strategy.
```

**Python (just pass a callable — the pattern nearly vanishes):**
```python
def checkout(price, discount):       # discount is any callable
    return discount(price)

checkout(100, lambda p: p * 0.9)     # the entire "pattern" is a lambda argument
# Strategy is invisible in Python — it's just first-class functions.
```

**Rust (static dispatch by default — zero runtime cost):**
```rust
trait Discount { fn apply(&self, price: f64) -> f64; }

// Generic = monomorphized = compiled to a direct call, no vtable (Part 3.6).
fn checkout<D: Discount>(price: f64, d: &D) -> f64 { d.apply(price) }
// Or `dyn Discount` to opt INTO vtable dispatch when you need runtime polymorphism.
```

**TypeScript (function type — like Go/Python):**
```typescript
type Discount = (price: number) => number;
const checkout = (price: number, d: Discount) => d(price);
checkout(100, p => p * 0.9);         // function as strategy
```

**The implementation truth across all four:** in every modern language with first-class functions (Go, Python, Rust, TS, Kotlin, Swift, JS), the *heavyweight Java class-hierarchy Strategy collapses to "pass a function."* The Java version (4.1) is heavy *because Java historically lacked first-class functions* (pre-lambda) — and even modern Java's `@FunctionalInterface` + lambda is its way of catching up. **When you implement a pattern, implement it in your language's idiom, not Java's** — writing Java-style class-hierarchy Strategy in Go or Python marks you as someone who learned patterns by rote and produces un-idiomatic code your reviewers will (rightly) reject. The *force* (swappable behavior) is universal; the *implementation* must be local.

---

## 4.7 — Implementation Synthesis: The Craft

Part 4's lessons, consolidated into the implementation judgment a staff engineer carries:

**1. The pattern is the easy 20%; production hardening is the essential 80%.** Every pattern's textbook shape is a few lines. Real implementation adds: context objects, result/error types, composition, immutability, thread safety, observability, resilience, lifecycle management, and tests. Juniors ship Tier 1 and think they're done; the gap between Tier 1 and Tier 4 *is* senior engineering.

**2. Model failure honestly.** Use result/sealed types for *expected* failures (business outcomes), reserve exceptions for *truly exceptional* conditions, and always define the *safe fallback* for each failure mode (fail-safe, never crash the critical path). Make the complete set of outcomes visible to the compiler where possible (sealed/sum types → exhaustiveness).

**3. Immutability is the default.** Immutable strategies, contexts, mementos, and value objects are thread-safe, shareable, and bug-resistant for free. Reach for mutability only when you've measured a need.

**4. Testability is the dividend of good structure, not a separate task.** The seams that enable production flexibility (program-to-interface + constructor DI) are the *same* seams that enable test doubles. If something is hard to test, the structure is wrong — fix the structure, and the test becomes easy.

**5. The Composition Root pattern wires the application.** Concentrate all concrete-type knowledge and wiring in one place; everywhere else depends on abstractions and receives collaborators via constructor injection. This realizes DIP fully and makes the entire dependency structure readable, swappable, and testable. Understand manual DI before reaching for a container; the container only automates what you should already understand.

**6. Defend against each pattern's documented hazards in code.** Observer's hazards (re-entrancy, threading, leaks, isolation) each map to a specific defensive technique (snapshot lists, explicit subscriptions, per-observer try/catch). Decorator's identity trap maps to "don't rely on identity/concrete type through a decorator." Knowing the hazard from Part 2 and the mechanism from Part 3 lets you write the defense in Part 4.

**7. Implement in your language's idiom.** The same force yields class hierarchies in classic Java, lambdas in modern Java, functions in Go/Python/TS, and compile-time generics in Rust. Match the local idiom; never transliterate Java patterns into function-first languages.

**8. Let your cost model (Part 3) drive structural choices.** Full snapshots vs. deltas (Command/Memento memory), sync vs. async (Observer), static vs. dynamic dispatch (Rust), container vs. manual DI — each is a tradeoff you now resolve from first principles rather than folklore.

You can now take any pattern from diagram to production code, hardening it through the four tiers, defending its hazards, choosing its idiom, and costing its tradeoffs. Part 5 (Architecture) zooms out from *objects* to *systems* — how these patterns compose into layered architectures, modular monoliths, and distributed/microservice systems, where the architectural-scale patterns (Repository, CQRS, Event Sourcing, Saga, Circuit Breaker) we've been foreshadowing finally take center stage.