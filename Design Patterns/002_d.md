# PART 2D — ESSENTIAL NON-GoF PATTERNS

GoF (1994) is not the whole story. The industry runs on patterns GoF never cataloged — many from Martin Fowler's *Patterns of Enterprise Application Architecture* (2002) and the dependency-injection movement. These appear in real codebases *far more often* than Interpreter or Flyweight. A staff engineer must know them cold.

## 2D.1 — Dependency Injection (DI)

**The force:** Part 1's DIP says high-level code should depend on abstractions, and someone must supply the concrete implementation. DI is the *mechanism*: **a component receives its dependencies from outside rather than creating them itself.** Instead of `this.db = new MySqlDatabase()` (welded to MySQL, untestable), the database is *injected*: `Constructor(Database db) { this.db = db; }`. Now you inject MySQL in production, an in-memory fake in tests, Postgres later — the component never changes.

**Three injection styles:** *Constructor injection* (dependencies passed to the constructor — preferred: dependencies are explicit, mandatory, and the object is valid once built); *setter injection* (via setters — for optional dependencies); *interface injection* (rare). **Inversion of Control (IoC)** is the broader principle (the framework controls object creation/wiring); a **DI container** (Spring, .NET DI, Guice, Dagger) automates wiring by reading a configuration of "which concrete fulfills which interface" and constructing the whole object graph for you.

**Why it dominates modern code:** It's the practical engine of testability (inject mocks), flexibility (swap implementations), and clean architecture (dependencies point inward toward abstractions). It's how the *legitimate* "single instance" need is met without the Singleton anti-pattern (the container manages one instance and injects it). **DI is arguably the single most important pattern in modern application development** — more pervasive than any GoF pattern.

**Tradeoff:** Indirection (the object graph is wired elsewhere, sometimes "magically" by a container, which can obscure what's connected to what and complicate debugging — "where does this instance come from?"). Containers add framework lock-in and a learning curve. But the testability/flexibility payoff is so large it's standard practice.

## 2D.2 — Repository

**The force:** Domain/business logic shouldn't know *how data is persisted* (SQL? NoSQL? an API?). Mixing query logic into business code couples them and makes both hard to test and change. **Repository mediates between the domain and data-mapping layers, presenting a collection-like interface for accessing domain objects** — `userRepository.findById(id)`, `userRepository.save(user)` — hiding all persistence details behind it.

The business logic asks the repository for objects as if it were an in-memory collection; the repository translates that into SQL/queries/API calls. Swap Postgres for MongoDB, or a real DB for an in-memory fake in tests, by swapping the repository implementation — domain logic untouched. It's essentially a *Facade + Adapter over your persistence layer*, depended upon via an interface (DIP). Ubiquitous in DDD and clean/hexagonal architectures.

**Tradeoff:** Another abstraction layer; can become a leaky or anemic pass-through if it just mirrors the ORM. Over-abstracting simple CRUD adds ceremony. But for non-trivial domains, the decoupling of business logic from persistence is highly valuable and the standard approach.

## 2D.3 — Unit of Work

**The force:** When a business operation changes *multiple* objects, you need to track all changes and commit them *as one atomic transaction* — and avoid redundant database writes. **Unit of Work maintains a list of objects affected by a business transaction, coordinates writing out changes, and resolves concurrency** — then commits everything in one transaction (or rolls back together on failure). It works hand-in-hand with Repository (repositories register changes; the unit of work commits them). ORMs like Hibernate, Entity Framework, and SQLAlchemy implement this internally (the "session"/"context" *is* a unit of work — you make changes, then `commit()` flushes them atomically).

**Tradeoff:** Complexity in tracking changes and managing the transaction boundary; the "magic" of automatic change tracking can surprise. But it's essential for transactional consistency and is mostly handled by your ORM.

## 2D.4 — Null Object

**The force:** Returning `null` to mean "nothing here" forces callers to litter code with null checks (`if (x != null) x.doThing()`), and forgetting one causes `NullPointerException` — "the billion-dollar mistake." **Null Object provides a real object with neutral/do-nothing behavior to represent "no object,"** so callers can use it uniformly without null checks. Instead of returning `null` for "no logger," return a `NullLogger` whose methods do nothing; callers call `logger.log()` unconditionally — it's safe either way.

**Tradeoff:** Hides the "absence" (sometimes you *want* to know nothing was found); can mask bugs (a silent no-op where you expected action). Not always appropriate, but eliminates a huge class of null-check noise and crashes where a neutral default is genuinely correct. (Modern languages address the same force with `Option`/`Maybe` types and nullable-type checking — often a better solution.)

## 2D.5 — Specification

**The force:** Business rules for *selecting* or *validating* objects ("premium customers in the EU who ordered last month") get duplicated and tangled across queries, validation, and UI. **Specification encapsulates a business rule as a reusable, combinable object** with an `isSatisfiedBy(candidate)` method — and specifications combine via `and`, `or`, `not` into composite rules (note the Composite synergy). One `PremiumCustomerSpec` is defined once, reused for filtering, validation, and querying, and composed with others.

**Tradeoff:** More objects; can be over-engineered for simple, one-off conditions. Shines when complex business rules recur, must be combined dynamically, or need to be unit-tested in isolation.

## 2D.6 — CQRS & Event Sourcing (Architectural Descendants)

These operate at the *architectural* scale (Part 5 covers them fully), but they're the grown-up descendants of Command/Memento, so note them here. **CQRS (Command Query Responsibility Segregation)** separates the *write* model (commands that change state — Command pattern at scale) from the *read* model (queries) — different models, often different data stores, optimized independently. **Event Sourcing** stores state as an append-only *log of events* (state changes) rather than current state; current state is *derived by replaying events* — exactly the "command log + replay" undo strategy from Command (2C.3) elevated to the system's source of truth. Git, accounting ledgers, and Kafka-based systems embody it. Powerful for auditability and temporal queries; costly in complexity and eventual-consistency challenges.

---

# PART 2 — FULL SYNTHESIS

You now hold the complete catalog — 23 GoF patterns plus the six non-GoF patterns the industry actually lives on. Consolidate:

**1. Patterns cluster by what they let vary.** Creational vary *how objects are made* (count, class, family, process, copying). Structural vary *how objects compose* (translate, split-axis, tree, wrap-to-add, simplify, share, gate). Behavioral vary *how objects coordinate* (swap algorithm, notify, reify request, traverse, fix-skeleton, state-machine, pass-along, centralize, snapshot, add-operation, evaluate-grammar).

**2. The same few principles generate them all.** Nearly every pattern is OCP (extend without modifying), DIP (depend on abstractions), composition-over-inheritance, and high-cohesion/loose-coupling, applied along a specific axis. *Patterns are principles made concrete.*

**3. Intent distinguishes structurally-identical patterns.** Adapter/Decorator/Proxy/Facade share "hold an object"; Strategy/State/Template share "swap behavior"; Command/Memento share "capture as object." The *force and intent*, not the class diagram, name the pattern.

**4. Patterns combine.** Abstract Factory uses Factory Methods; Command uses Memento for undo and Composite for macros; Interpreter is Composite; State uses Flyweight; Decorator and Chain and Composite all build recursive structures. Real systems weave many patterns together.

**5. Language determines weight.** Strategy, Command, Iterator, Template Method, Visitor, State, and others *dissolve* into functions, closures, generators, and pattern matching in modern multi-paradigm languages. The *force* is eternal; the *implementation weight* is local. The expert names the force and reaches for the lightest local tool.

**6. Every pattern is a bet (Part 1).** Each adds indirection now, betting a specific change comes later. Apply when the axis of change is real and likely; refuse when it's speculative (YAGNI). The non-GoF patterns (DI, Repository, Unit of Work) are bets that pay off so reliably they're near-default in modern apps; Interpreter and Flyweight pay off so rarely they're near-never.

**7. The modern stack runs on non-GoF patterns most of all.** DI wires everything; Repository + Unit of Work handle persistence; these appear in nearly every serious application, while several GoF patterns appear mainly inside frameworks you consume rather than code you write.

With Part 2 complete, you possess the full *vocabulary* and the *force-resolution map*. Part 3 (Internals) will descend beneath the patterns to the machine — how polymorphic dispatch actually works in memory (vtables), what virtual calls cost the CPU, how object layout and indirection affect cache behavior, and what really happens when a Decorator chain or Visitor double-dispatch executes at the level of registers, memory, and the call stack.