# PART 5 — ARCHITECTURE

## From Objects to Systems

Parts 1–4 operated at the level of *objects* — classes collaborating within a single process. Part 5 zooms out to *systems*: how patterns compose into the large-scale structure of an application, and how the *same forces* (coupling, cohesion, OCP, DIP) that govern objects govern entire architectures. The patterns don't disappear at scale — they *reappear, magnified*. A Strategy becomes a pluggable service; an Observer becomes a message bus; a Proxy becomes an API gateway; a Command becomes an event in an event-sourced log. **Architecture is design patterns viewed through a telescope instead of a microscope.**

This Part progresses through the architectural spectrum — layered monolith → modular monolith → distributed systems → microservices → event-driven — and at each level shows which patterns dominate, which forces drive the structure, and what new failure modes emerge that single-process patterns never faced.

The through-line: **the principles are scale-invariant; the mechanisms and the costs are not.** DIP at the object level is a constructor parameter; DIP at the system level is an API contract and a network hop. The cost difference is the million-fold gap Part 3 measured (nanoseconds of dispatch vs. milliseconds of network) — which is why architectural decisions are *higher-stakes versions of the same bets* you make about objects.

---

## 5.1 — Layered Architecture: Patterns as Organizing Boundaries

The most fundamental architectural structure is **layering** — organizing a system into horizontal layers, each depending only on the layer below, each a *boundary* enforcing separation of concerns. This is cohesion-and-coupling (Part 1) applied to the whole codebase.

### The Classic Layers

```
┌─────────────────────────────────────────────────────┐
│  PRESENTATION  (controllers, UI, API endpoints)      │  ← handles input/output
│     - depends only on ↓                              │     knows nothing of the DB
├─────────────────────────────────────────────────────┤
│  APPLICATION   (use cases, orchestration, services)  │  ← coordinates the work
│     - depends only on ↓                              │     no business RULES, just flow
├─────────────────────────────────────────────────────┤
│  DOMAIN        (entities, business rules, logic)     │  ← the heart: pure business logic
│     - depends on NOTHING (the stable core)           │     no framework, no DB, no HTTP
├─────────────────────────────────────────────────────┤
│  INFRASTRUCTURE (DB, messaging, external APIs, files)│  ← technical details
│     - implements interfaces DEFINED by domain        │     the volatile outer ring
└─────────────────────────────────────────────────────┘
```

**Which patterns live where, and why:**
- **Presentation ↔ Application:** **Facade** (the application service is a facade over the domain for the controllers) and **DTO** (data transfer objects — flat structures crossing the boundary, decoupling the API's shape from the domain's shape, so you can change internal models without breaking the API contract).
- **Application layer:** **Command** (each use case is often a command — `PlaceOrderCommand` handled by a `PlaceOrderHandler`), **Mediator** (a command bus routes commands to handlers — MediatR in .NET, the "CQRS command side").
- **Domain ↔ Infrastructure:** **Repository** (domain defines `OrderRepository` interface; infrastructure implements `PostgresOrderRepository`), **Unit of Work**, **Adapter** (wrapping external APIs into domain-friendly interfaces), and crucially **Dependency Inversion** — note the arrow direction.

### The Dependency Inversion Twist (The Most Important Architectural Insight)

Naive layering has dependencies pointing *downward*: domain depends on infrastructure (domain calls the database). But this is **backwards** — it makes your *most valuable, most stable* code (business logic) depend on your *most volatile, most replaceable* code (database, frameworks). When you swap Postgres for DynamoDB, your business logic shouldn't flinch.

**The fix is DIP at architectural scale** — invert the dependency so it points *inward*:

```
   NAIVE (dependency points down — domain enslaved to infrastructure):

     Domain ──calls──► Database     ← change DB, domain breaks. WRONG.

   INVERTED (dependency points IN — infrastructure serves domain):

     Domain defines:  «OrderRepository» (interface, lives IN the domain)
                            ▲
                            │ implements (arrow points INWARD, toward domain)
     Infrastructure: PostgresOrderRepository

     ← The domain owns the CONTRACT; infrastructure obeys it.
       Swap the DB → only infrastructure changes; domain untouched.
```

This inversion is the foundation of **Hexagonal Architecture (Ports & Adapters)**, **Clean Architecture**, and **Onion Architecture** — three names for the same core idea:

```
HEXAGONAL / PORTS & ADAPTERS:

              ┌─────────────────────────────────┐
   incoming   │         APPLICATION CORE         │   outgoing
   adapters   │   ┌─────────────────────────┐   │   adapters
   (driving)  │   │      DOMAIN (pure)      │   │   (driven)
              │   └─────────────────────────┘   │
   HTTP ──────┼──►│ «port» (interface)      │◄──┼────── PostgresAdapter
   CLI  ──────┼──►│                         │◄──┼────── KafkaAdapter
   Tests ─────┼──►│ defines what it NEEDS    │◄──┼────── StripeAdapter
              │   └─────────────────────────┘   │
              └─────────────────────────────────┘

   "Ports" = interfaces the core defines (Repository, EventPublisher, etc.)
   "Adapters" = implementations that plug into ports (DB, queue, API, UI)
   The core depends ONLY on ports it owns; adapters depend on the core.
   Everything points INWARD. The business logic is the stable center.
```

**Why this is the dominant modern architecture for non-trivial monoliths:** the business logic — your competitive advantage, your hardest-won correctness — becomes a *pure, framework-free, infrastructure-free core* that you can test in milliseconds (no DB, no HTTP, just inject fake adapters) and that survives every technology migration. The volatile stuff (frameworks, databases, message brokers, the web) lives in *replaceable adapters* at the edge. This is Repository, Adapter, DI, and DIP composed into a *system shape* — the architectural payoff of every object-level principle from Part 1.

**The tradeoff (honest):** ports-and-adapters adds indirection and ceremony — interfaces, adapters, DTOs, mapping between layers. For a CRUD app with no real business logic, it's over-engineering (the business "logic" is just shuttling data to the DB, so the pure core is empty and the layers add nothing but boilerplate). It pays off precisely when there's *substantial, valuable, long-lived business logic worth protecting* from infrastructure churn. The bet (Part 1): will this domain logic be complex and long-lived enough to justify insulating it? For a banking core, yes; for a CRUD admin panel, no.

---

## 5.2 — The Monolith and Its Modular Evolution

### The Monolith (Honestly Assessed)

A **monolith** is a single deployable unit — all code in one process, one codebase, one deployment. The industry spent the 2010s treating "monolith" as a slur; the mature view (and the current pendulum swing) recognizes it as **the correct default for most systems.**

```
MONOLITH:
   ┌──────────────────────────────────────┐
   │           Single Process              │
   │  ┌────────┐ ┌────────┐ ┌────────┐    │
   │  │ Orders │ │ Users  │ │Payments│    │   all modules,
   │  │ module │ │ module │ │ module │    │   one process,
   │  └────────┘ └────────┘ └────────┘    │   one deploy,
   │       └─────────┼─────────┘          │   in-process calls
   │           ┌─────────┐                │   (nanoseconds)
   │           │ Database│                │
   │           └─────────┘                │
   └──────────────────────────────────────┘
```

**Monolith advantages (substantial):** in-process calls are *free* (nanoseconds — Part 3) versus network calls (milliseconds); one transaction spans everything (true ACID, no distributed-transaction nightmare); one deploy, one thing to monitor, trivial local development; refactoring across module boundaries is a simple IDE operation. **For a startup or any system without proven scale/team pressures, the monolith ships faster and breaks less.**

**Monolith danger:** without discipline, it rots into a **Big Ball of Mud** — modules reach into each other's internals (no encapsulation across modules), coupling becomes total, and a change anywhere risks breaking everywhere (Part 1's tight-coupling nightmare at codebase scale). The problem is never "monolith"; it's "*unstructured* monolith."

### The Modular Monolith (The Sweet Spot)

The **modular monolith** keeps the monolith's deployment simplicity while imposing *internal module boundaries* as strict as if they were separate services — but enforced by code structure, not network:

```
MODULAR MONOLITH:
   ┌──────────────────────────────────────────────┐
   │              Single Process                    │
   │  ┌──────────────┐      ┌──────────────┐       │
   │  │ Orders Module│      │Payments Module│      │
   │  │ ┌──────────┐ │      │ ┌──────────┐ │       │
   │  │ │ internal │ │      │ │ internal │ │       │
   │  │ │ (private)│ │      │ │ (private)│ │       │
   │  │ └──────────┘ │      │ └──────────┘ │       │
   │  │ PUBLIC API ──┼──────┼─► via explicit│      │
   │  │ (interface)  │ ONLY │   contract    │      │
   │  └──────────────┘      └──────────────┘       │
   │   Modules talk ONLY through published          │
   │   interfaces — internals are INVISIBLE         │
   │   across module boundaries.                     │
   └──────────────────────────────────────────────┘
```

**How patterns enforce modularity:** each module exposes a **Facade** (its public API — the only entry point; internals are package-private/internal/hidden), modules communicate via **interfaces** (DIP — a module depends on another's published contract, never its internals) or via an **in-process event bus** (Observer/Mediator — Orders publishes `OrderPlaced`, Payments subscribes, with no direct coupling). Each module can own its own database tables (no cross-module DB joins — a discipline that *pre-positions you for extraction*).

**Why modular monolith is often the right answer:** it gives you the monolith's operational simplicity (one deploy, in-process speed, simple transactions) *and* the microservice's clean boundaries (each module independently understandable, replaceable, even *extractable later*). The boundaries are enforced by the compiler/module system rather than the network — so you get the discipline without the distributed-systems tax. **And it's the safe migration path:** start modular-monolith; if a module genuinely needs independent scaling/deployment, extract *that one module* to a service later — the boundary is already clean. This is the modern consensus: *"monolith first, modular always, microservices only when forced."*

---

## 5.3 — Crossing the Network: Why Distribution Changes Everything

The moment you split a system across processes/machines, you cross a threshold where **the patterns still apply but the physics inverts.** Part 3.8 quantified it: an in-process call is nanoseconds; a network call is milliseconds — a *million-fold* difference — and, more importantly, a network call can **fail, time out, arrive twice, arrive out of order, or partially succeed** in ways an in-process call never can.

### The Fallacies Made Architectural

Part 3.8's Eight Fallacies of Distributed Computing become *architectural constraints*:

```
In-process call:                  Network call:
  • always returns                  • may time out, may never return
  • succeeds or throws (binary)     • may PARTIALLY succeed (sent but no ack)
  • instant                         • slow + variable latency
  • same memory                     • data must be serialized/copied
  • caller & callee live/die        • either side can be down independently
    together                          while the other runs
```

**The single most important consequence — partial failure:** in a monolith, if a function call is happening, the whole process is alive; either the call completes or the process crashed (taking your caller with it). In a distributed system, the *callee can be dead while the caller is perfectly healthy*, or the network between them severed, or the callee processed the request but the *response* was lost (so you don't know if it succeeded). **This is the defining difficulty of distributed systems, and it's why distributed patterns are fundamentally about *managing failure*, not hiding it.** (Recall Part 2B's warning about remote Proxy "hiding costs" — naive distribution that pretends the network is local is the original sin that patterns like Circuit Breaker exist to atone for.)

### The CAP Theorem (The Inescapable Tradeoff)

When you distribute *data*, you face a mathematical limit. **CAP theorem:** in the presence of a network **P**artition (which *will* happen — networks fail), you must choose between **C**onsistency (every read sees the latest write) and **A**vailability (every request gets a response):

```
        Partition happens (unavoidable in distributed systems).
                          │
            ┌─────────────┴─────────────┐
            ▼                           ▼
   CP: Consistency              AP: Availability
   refuse to answer if          answer with possibly-stale
   you can't guarantee          data rather than refuse;
   freshness (the node          reconcile later
   on the wrong side of         (eventual consistency)
   the partition rejects)
   e.g. banking balances        e.g. shopping cart, social feed
```

You cannot have both during a partition — physics forbids it. **This forces a per-feature decision:** a bank balance needs **C** (better to reject than show wrong money); a social-media like-count tolerates **A** (better to show a slightly-stale count than an error). Real systems mix both — *consistency where it's worth the availability cost, availability where staleness is acceptable.* **Eventual consistency** (AP systems converge to correctness *given time without new writes*) is the dominant model for scale, and it ripples into every pattern: you can no longer assume a write is immediately visible everywhere, which breaks naive Observer/Repository assumptions and forces patterns like Saga (5.5) and CQRS (5.6) to embrace asynchrony and reconciliation.

---

## 5.4 — Microservices: Patterns at System Scale

**Microservices** decompose a system into independently deployable services, each owning a business capability and its own data, communicating over the network. This is the architecture most associated with "scale" — and the most *over-applied* in the industry.

```
MICROSERVICES:
   ┌──────────┐   ┌──────────┐   ┌──────────┐
   │ Orders   │   │ Payments │   │ Inventory│   each: own process,
   │ Service  │   │ Service  │   │ Service  │   own deploy, own DB,
   │ ┌──────┐ │   │ ┌──────┐ │   │ ┌──────┐ │   own team, own scaling
   │ │ DB   │ │   │ │ DB   │ │   │ │ DB   │ │
   │ └──────┘ │   │ └──────┘ │   │ └──────┘ │
   └────┬─────┘   └────┬─────┘   └────┬─────┘
        └──── network (HTTP/gRPC/messaging) ────┘
                       │
              ┌────────────────┐
              │  API Gateway   │  ← single entry (Facade/Proxy at scale)
              └────────────────┘
                       │
                    clients
```

### The Object Patterns, Reincarnated as Infrastructure

Every microservice infrastructure component is a Part 2 pattern at system scale:

- **API Gateway = Facade + Proxy.** One entry point (Facade) that routes, authenticates, rate-limits, and aggregates calls to internal services (Proxy controlling access). Clients see one simple interface; the messy service topology hides behind it. *This is literally the Facade and Proxy patterns, deployed as infrastructure.*
- **Service Discovery = a runtime Registry/Factory.** Services register themselves; callers look up "where is the Payments service right now?" (instances come and go with autoscaling) — the registry-based selection of Part 4.4, at network scale.
- **Message Broker (Kafka/RabbitMQ) = Observer/Mediator + Command.** Pub/sub is Observer across the network (publishers don't know subscribers); a broker routing messages is a Mediator; each message is often a Command or Event. *The entire event-driven backbone is behavioral patterns made distributed.*
- **Sidecar / Service Mesh (Istio/Envoy) = Decorator + Proxy.** A proxy deployed alongside each service that transparently adds cross-cutting concerns (mTLS, retries, metrics, tracing) *without the service code knowing* — Decorator and Proxy at the deployment level, wrapping every service call.
- **Adapter** reappears for every external integration; **Strategy** reappears as pluggable service implementations.

### The Resilience Patterns (New at This Scale)

Distribution's partial-failure reality demands *new* patterns that single-process code never needed — these are the genuinely novel architectural patterns:

**Circuit Breaker** (the most important). When a downstream service is failing, *stop calling it* — repeatedly calling a dead service wastes resources, piles up timeouts, and can cascade the failure upstream (your threads all block waiting → you die too → *cascading failure*, how one service outage takes down the whole system). The circuit breaker wraps calls and trips like an electrical breaker:

```
   CLOSED (normal) ──failures exceed threshold──► OPEN (fail fast)
      ▲                                              │
      │                                         after timeout
   success                                           ▼
      │                                       HALF-OPEN (test
      └──────────── trial succeeds ──────────  with one request)
                    trial fails → back to OPEN
```
- **Closed:** calls pass through; failures are counted.
- **Open:** the breaker "trips" — calls fail *immediately* without hitting the dead service (fail fast, protect your threads, give the downstream room to recover).
- **Half-open:** after a cooldown, let *one* trial call through; if it succeeds, close (recovered); if it fails, re-open.

This is a **State pattern** (Part 2C) — three states with transition rules — operating as a resilience mechanism. Hystrix, Resilience4j, and every service mesh implement it. It's the canonical example of *embracing failure*: instead of pretending the network is reliable, it assumes failure and contains it.

**Other resilience patterns (briefly, all failure-embracing):** **Retry with exponential backoff + jitter** (retry transient failures, but back off and randomize to avoid synchronized retry storms that re-DDoS the recovering service); **Bulkhead** (isolate resources per dependency — separate thread pools — so one slow dependency can't consume *all* threads and sink everything, like watertight ship compartments); **Timeout** (never wait forever — an un-timed-out call is a thread held hostage); **Idempotency** (because a request may arrive twice — lost ack → retry → duplicate — operations must be safe to repeat, e.g., "set balance to X" not "add X"; this is *essential* and pervasive in distributed design).

### Microservices: The Honest Tradeoff

**Microservices buy:** independent deployment (teams ship without coordinating), independent scaling (scale only the hot service), fault isolation (one service down ≠ whole system down, *if* you built resilience), technology heterogeneity (each service picks its stack), and team autonomy (Conway's Law — your architecture mirrors your org; microservices let teams own services end-to-end).

**Microservices cost (severe, routinely underestimated):** you trade *all* the monolith's simplicity for a *distributed systems problem* — network latency and partial failure everywhere, no distributed transactions (you can't `BEGIN TRANSACTION` across services — hence Saga, 5.5), eventual consistency headaches, distributed debugging (a single user request fans out across ten services — you *need* distributed tracing just to see what happened), operational explosion (dozens of deployments, service discovery, mesh, observability stack), and data consistency challenges that are genuinely hard computer science. **The latency math alone is brutal:** an operation that was ten in-process calls (10 × nanoseconds ≈ instant) becomes ten network calls (10 × milliseconds = seconds, if serial) — Part 3's cost model is now your architecture's dominant constraint.

**The mature verdict (current industry consensus, post-hype):** *microservices solve organizational and scaling problems, not technical ones, and only at a scale where the monolith's limits are genuinely binding.* Adopt them when team count, deployment-coordination pain, or differential-scaling needs *force* the split — not because they're fashionable. **Most systems that adopted microservices didn't need them and paid the distributed-systems tax for no benefit.** The famous failure mode is the **distributed monolith** — services so chatty and tightly coupled that you have the monolith's coupling *and* the network's cost: the worst of both worlds. The modular monolith (5.2) gives most of the benefit at a fraction of the cost; reach for microservices only when you've outgrown it.

---

## 5.5 — The Saga Pattern: Transactions Without Transactions

The deepest *new* problem distribution creates: **you cannot have an ACID transaction across services** (no shared database, no distributed lock you'd want at scale). Yet business operations span services — "place order" must reserve inventory, charge payment, and create a shipment, *atomically-ish.* If payment fails after inventory is reserved, you must *undo* the reservation. **Saga** solves this with a sequence of *local* transactions plus *compensating actions* (undo operations) — and this is **Command's `undo()` (Part 2C/4.5) elevated to the architectural scale.**

```
SAGA (orchestrated):  place order across 3 services

  ┌──────────────┐
  │ Orchestrator │  drives the sequence, handles failures
  └──────┬───────┘
   1. Reserve inventory ──► Inventory svc ✓
   2. Charge payment    ──► Payment svc   ✗ FAILS!
   3. (rollback) ──► COMPENSATE: release inventory (undo step 1)
                     ↑ the "undo" — a Saga is a chain of commands,
                       each with a compensating command, run in
                       reverse on failure. Pure Command-pattern undo
                       at distributed scale.
```

Two flavors: **orchestration** (a central coordinator — a Mediator at system scale — drives each step and triggers compensations on failure; explicit, traceable, but a coordinator to maintain) vs. **choreography** (each service emits events others react to — Observer at scale; no central coordinator, fully decoupled, but the overall flow is *implicit and hard to follow* — exactly Observer's debuggability cost from Part 2C, magnified across a distributed system).

**The hard truth Saga forces you to accept:** compensations are *not* perfect rollbacks. You can release reserved inventory, but you can't un-send a shipped package or un-charge a card without a *new* refund transaction (which may itself fail). Sagas give **eventual consistency with compensation**, not true atomicity — there are intermediate states where the system is *temporarily inconsistent* (inventory reserved but payment not yet charged), which the business must tolerate and the UI must account for. This is the price of distribution: you trade ACID guarantees for availability and independence, and you *design for the in-between states* rather than pretending they don't exist.

---

## 5.6 — Event-Driven Architecture, CQRS, and Event Sourcing

The architectural climax — where Command, Observer, and Memento from Part 2 fuse into a system-defining style.

### Event-Driven Architecture (EDA)

Instead of services *calling* each other (commands — "do this"), services *emit events* ("this happened") that other services react to. This is **Observer as the system's fundamental communication model:**

```
   Order Service ──emits──► [ OrderPlaced event ] ──► Message Broker (Kafka)
                                                          │ broadcasts to subscribers
                            ┌─────────────────────────────┼──────────────┐
                            ▼                             ▼              ▼
                      Inventory svc              Email svc        Analytics svc
                      (reserve stock)         (send confirmation)  (record metrics)
                      ← each reacts independently; the Order service
                        doesn't know or care who's listening (Observer!)
```

**Why EDA scales and decouples:** the emitter is *completely decoupled* from consumers (add a new consumer — a fraud-detection service — without touching the emitter; pure OCP at system scale), consumers process *asynchronously* (the emitter doesn't wait — high throughput), and the broker *buffers* (a slow consumer doesn't block; messages queue — natural backpressure handling). **The cost is Observer's cost, magnified:** *implicit control flow* — "what happens when an order is placed?" has no single answer you can read in one place; the flow is scattered across every subscriber, discoverable only by tracing events through the system. Debugging "why didn't the email send?" means tracing an event through a broker to a consumer that may have silently failed. This is Part 2C's "implicit flow is hard to debug" at its most extreme — and why EDA *requires* heavy investment in observability (distributed tracing, event monitoring, dead-letter queues) to be operable.

### Event Sourcing (Memento + Command, Made the Source of Truth)

Conventional systems store *current state* (the account balance is $100). **Event Sourcing** stores the *full log of events* that produced the state (deposited $150, withdrew $50) and *derives* current state by replaying them. This is **Command's "log + replay" undo strategy and Memento's snapshots (Part 2C/4.5) elevated to the system's source of truth:**

```
   Traditional:  store CURRENT state, overwrite on change
                 balance: $100   ← history LOST; can't answer "what was it last Tuesday?"

   Event Sourced: store the EVENT LOG (append-only, immutable)
                 [Deposited $150][Withdrew $50][Deposited $... ]
                  ────────────────────────────────────────────►
                 current state = fold/replay events from the start
                 (or from a periodic SNAPSHOT — Memento! — to avoid
                  replaying millions of events)
```

**What Event Sourcing buys (powerful):** a *perfect audit log* (every change, forever — the events *are* the history; invaluable for finance, compliance, debugging); **time travel** (reconstruct state at *any* past moment by replaying to that point — "what was the balance last Tuesday?" is trivial); the ability to *derive new views retroactively* (add a new read model and replay all of history into it); and natural fit with EDA (the events you store are the events you publish). Git is event sourcing (commits are events, current state is their fold); accounting ledgers are event sourcing (append-only, never edit a posted entry, corrections are new entries — Memento/Command thinking that predates computers by centuries).

**What it costs (significant):** the log grows forever (mitigated by snapshots — Memento — at intervals so you replay from the last snapshot, not the beginning); querying current state requires replay or a maintained projection (hence CQRS, below); **you can never delete an event** (immutable log) — which collides with GDPR's "right to be forgotten" (a genuine, hard problem requiring encryption-with-key-deletion tricks); and *event schema evolution* is brutal (a 5-year-old event must still be replayable, so changing event structure requires versioning and upcasting). It's a heavyweight pattern justified only when *audit, temporal queries, or event-driven integration* are first-class requirements — over-applied, it's a productivity catastrophe.

### CQRS (Command Query Responsibility Segregation)

Event Sourcing creates a problem: the event log is great for *writes* but terrible for *reads* (you can't query "all orders over $100" against an event log efficiently). **CQRS** separates the *write model* (commands → events, optimized for consistency and business rules — the Command pattern's domain) from the *read model* (queries against denormalized views optimized for fast reads):

```
CQRS:
   WRITES                                    READS
   ──────                                    ─────
   Command ──► Write Model ──► Events ──┐    Query ──► Read Model(s)
   (PlaceOrder) (business rules,        │           (denormalized,
                consistency)            │            fast, possibly
                                        │            multiple views
                                        └──project──► for different needs)
                                          (events update read models,
                                           often ASYNC → eventual consistency)

   Write side: normalized, consistent, command-focused (the Command pattern)
   Read side:  denormalized, fast, query-focused, possibly many tailored views
```

**Why separate them:** reads and writes have *opposite* optimization needs (writes want consistency and normalization; reads want speed and denormalization), often *wildly different* load (100x more reads than writes is common), and different scaling needs. CQRS lets you optimize and scale each independently — even use *different databases* (Postgres for writes, Elasticsearch for search reads, Redis for hot reads). It pairs naturally with Event Sourcing (events from the write side project into read models) and EDA (the projection is event-driven).

**The cost:** **eventual consistency between write and read sides** — you place an order (write side) and immediately query your orders (read side), but the read model hasn't been updated yet (the projection is async), so your new order *isn't there yet*. The UI must handle this ("your order is processing") — a real UX and correctness complication. Plus the obvious complexity of maintaining two models and the projection machinery. CQRS is justified when the read/write asymmetry is *severe*; for a balanced CRUD app, it's pure overhead — one of the most over-applied architectural patterns when teams adopt it cargo-cult style.

---

## 5.7 — Architecture Synthesis: Patterns Through the Telescope

The grand unification — every object pattern's architectural reincarnation, and the scale-invariant principles beneath:

```
OBJECT PATTERN (Part 2)        →  ARCHITECTURAL REINCARNATION (Part 5)
─────────────────────────────────────────────────────────────────────
Facade                         →  API Gateway, module public API, BFF
Proxy                          →  API Gateway, service mesh sidecar, CDN
Adapter                        →  integration adapters, anti-corruption layer
Strategy                       →  pluggable service implementations
Observer                       →  pub/sub, message broker, event-driven arch
Mediator                       →  message broker, saga orchestrator, command bus
Command                        →  CQRS command side, message/event, task queue
Command.undo()                 →  Saga compensating transactions
Command log + replay           →  Event Sourcing
Memento (snapshot)             →  Event Sourcing snapshots, DB savepoints
State                          →  Circuit Breaker, workflow/state-machine services
Chain of Responsibility        →  middleware pipeline, service mesh filter chain
Repository / Unit of Work      →  data-access layer, per-service data ownership
Composite                      →  nested service composition, aggregation
DIP / Dependency Injection     →  Ports & Adapters, Hexagonal/Clean architecture
```

**The four scale-invariant truths:**

**1. The principles never change; the costs change by a million-fold.** Coupling, cohesion, OCP, and DIP govern a two-class collaboration and a fifty-service mesh identically. What changes is the *cost of a boundary crossing* — nanoseconds in-process, milliseconds across the network — which is why architectural boundaries must be *coarser* and *fewer* than object boundaries. You can afford fine-grained objects; you cannot afford fine-grained chatty services. **Put boundaries where the value of independence exceeds the cost of the crossing** — and that cost is millions of times higher across the network, so architectural boundaries must justify themselves millions of times more.

**2. Distribution is not a pattern; it's a Faustian bargain.** Every step toward distribution (monolith → modular monolith → microservices → event-driven) buys independence and scalability at the price of *partial failure, eventual consistency, and operational complexity that no pattern can abstract away* — only manage. The patterns at this scale (Circuit Breaker, Saga, CQRS, idempotency) are fundamentally about *embracing and containing failure*, the opposite of the in-process world where calls simply succeed or throw. The naive instinct — make remote calls look like local calls (Part 2B's remote-Proxy warning) — is the original architectural sin.

**3. Complexity is conserved, only relocated.** A modular monolith puts complexity in *code structure* (module boundaries, interfaces). Microservices move that same complexity into *operational infrastructure* (network, deployment, observability, resilience). Event-driven systems move it into *implicit flow* (tracing events). **You never eliminate essential complexity; you choose where to pay it** — and the skill is choosing the location where your team and system can best afford it. Teams routinely move complexity from a place they understand (a monolith's codebase) to a place they don't (a distributed system's operations) and call it progress.

**4. The default should be the simplest architecture that meets the requirement.** YAGNI (Part 1) is *most* expensive to violate at the architectural scale — an unnecessary microservices migration costs years and can sink a company, where an unnecessary object pattern costs a few classes. The modern, scar-tissue-earned consensus: **start with a modular monolith with clean (hexagonal) boundaries; distribute individual modules into services only when a specific, proven force — team-scaling pain, differential-scaling need, fault-isolation requirement — makes the distributed tax worth paying.** Architecture is the discipline of *deferring* the expensive bets until the evidence demands them, while keeping boundaries clean enough that you *can* make them later. The clean boundary is cheap insurance; the premature distribution is a ruinous bet.

You can now reason from a single object's vtable dispatch (Part 3) all the way up to a fifty-service event-sourced mesh (Part 5), seeing the *same forces and the same patterns* operating at every scale, with the *costs* scaling from nanoseconds to milliseconds to days-of-operational-toil — and you can judge *which scale of structure a given problem actually warrants*, which is the essence of architectural judgment.

Part 6 (Performance) will now drill into the *quantitative* dimension — profiling, complexity, latency vs. throughput, and the all-important discipline of knowing *when optimization matters and when it's a waste* — turning Part 3's cost model and Part 5's latency math into concrete performance engineering.