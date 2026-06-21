# PART 11 — HANDS-ON PROJECTS

## Cementing Mastery in Working Code

Reading about patterns builds *recognition*; building with them builds *judgment*. This Part presents four projects of escalating difficulty — Beginner, Intermediate, Advanced, Expert — each designed not just to *use* patterns but to *make you feel the forces* that generate them, *encounter the costs* firsthand, and *experience the decay and tradeoffs* the prior Parts described in the abstract. 

The pedagogical design is deliberate: each project is structured so that you'll be *tempted to over-engineer*, *forced to confront a pattern's hazard*, and *required to make a judgment call with no clean answer* — because those are exactly the experiences that convert textbook knowledge into the staff-engineer judgment this course is built around. A project you breeze through teaches nothing; these are designed to make you struggle at precisely the points where understanding lives.

For each project: **Goals** (what mastery it builds), **Requirements** (what to build), **Architecture** (the shape and patterns), **Implementation Plan** (the build sequence), **Common Mistakes** (the traps — read these *before* building, then again after), and **Extensions** (how to push further). Build them in whatever language you're strongest in — and ideally, build at least one in a *function-first* language (Go/Python/Rust/TS) to *feel* the language-relativity of Part 1.9 firsthand.

```
   THE PROJECT ARC (what each cements):
   Beginner:     the core patterns + the "force first" instinct (Strategy,
                 Factory, Observer, Decorator) — and the over-engineering temptation
   Intermediate: composition of patterns + hazard handling + testability
                 (the wrapping family, Command+undo, DI) — the Part 4 tiers
   Advanced:     architecture-scale patterns + failure modes + operability
                 (Repository, hexagonal, resilience) — Parts 5/6/7 made real
   Expert:       a pattern-rich domain with genuine tensions + decay + the
                 un-patterning skill — the Part 8 frontier in working code
```

---

## 11.1 — BEGINNER PROJECT: A Text Adventure Game Engine

### Goals

Build a small text-adventure engine. This project cements the *four foundational patterns* (Strategy, Factory, Observer, Command) and — more importantly — trains the **"identify the force first"** instinct (Part 1) by giving you a domain where the forces are *obvious and tangible*. You'll also encounter your *first over-engineering temptation*, which is the point.

A text adventure is ideal for beginners because the patterns map to *concrete, intuitive* game concepts: a room's behavior varies (Strategy), monsters get created from data (Factory), the UI reacts to game-state changes (Observer), and player actions are reified for undo and history (Command).

### Requirements

A playable text adventure where:
- The player navigates rooms, picks up items, fights monsters, solves simple puzzles.
- Rooms, items, and monsters are defined in *data* (JSON/YAML), not hardcoded — so a designer can add content without touching code.
- The player types commands (`go north`, `take sword`, `attack goblin`, `look`).
- Different monsters have different combat behaviors.
- The game displays state changes (health, inventory, location) to the player.
- Support `undo` for the last action (for movement/inventory, not combat).

### Architecture

```
   ┌─────────────────────────────────────────────────────────┐
   │  CommandParser → Command objects (Command pattern)       │
   │     "go north" → MoveCommand{direction: north}           │
   │     each Command has execute() and undo()                │
   └──────────────────────┬──────────────────────────────────┘
                          ▼
   ┌─────────────────────────────────────────────────────────┐
   │  GameWorld (the state: rooms, player, monsters)          │
   │     - rooms/items/monsters built by EntityFactory        │
   │       from JSON data (Factory pattern)                   │
   │     - monsters have a CombatStrategy (Strategy pattern):  │
   │       AggressiveStrategy, CowardlyStrategy, BossStrategy  │
   └──────────────────────┬──────────────────────────────────┘
                          ▼ notifies on change
   ┌─────────────────────────────────────────────────────────┐
   │  GameObservers (Observer pattern):                       │
   │     - ConsoleView (prints state changes)                 │
   │     - StatsTracker (records moves, kills)                │
   │     - (later) AutoSaver                                  │
   └─────────────────────────────────────────────────────────┘
```

**The force-to-pattern mapping (internalize this — it's the whole point):**
- *Force: monster behavior varies and we'll add more monster types* → **Strategy** (`CombatStrategy` injected into each monster). Adding a behavior = new strategy, no existing code touched (OCP).
- *Force: entities are created from data, and we'll add content without recompiling* → **Factory** (`EntityFactory.create(jsonData)` returns the right entity type). New content = new JSON, no code change.
- *Force: many parts of the UI/system must react to state changes* → **Observer** (the view, the stats tracker, the autosaver all subscribe to the game world).
- *Force: actions must be undoable and form a history* → **Command** (each action is a reified `Command` with `execute`/`undo`).

### Implementation Plan

```
   1. Core domain FIRST, no patterns: Room, Item, Monster, Player as plain
      classes. Get a hardcoded game playable. (Resist patterns until you feel
      the force — this is the discipline being trained.)
   2. Add the Command parser + Command objects when you implement input.
      Feel how reifying actions ENABLES undo (you couldn't undo a method call).
   3. Add the Factory when you move entities to JSON. Feel how it DECOUPLES
      content from code (a designer can now add a dragon without you).
   4. Add Strategy when you have a SECOND monster behavior. (Critically: do NOT
      add it for the first monster — wait for the second. Feel the "two cases
      before you abstract" heuristic of Parts 9/10.)
   5. Add Observer when the UI needs to react to state. Feel the decoupling —
      the game logic doesn't know the view exists.
   6. Add undo (Command.undo + a history stack). Snapshot what you need.
```

**The critical pedagogical instruction:** build step 1 with *no patterns at all*, and add each pattern *only when you feel its force* in steps 2–6. This trains the single most important instinct — *patterns resolve forces; add them when the force appears, not before.* If you add Strategy before the second monster, you've over-engineered, and you should *notice yourself doing it.*

### Common Mistakes

```
   ✗ Over-engineering the entity hierarchy: deep inheritance (Entity→Creature→
     Monster→Goblin) when composition + data would do. Feel the fragile-base-
     class pain if you do this, then refactor to composition (Part 1.7 made real).
   ✗ Adding Strategy for ONE monster behavior (the YAGNI violation — wait for two).
   ✗ Making the Command parser a giant switch instead of a command registry
     (the Simple-Factory-vs-Factory-Method lesson, Part 2A.2).
   ✗ Observer memory leak: views that never unsubscribe (Part 4.2). On a small
     game it won't bite, but NOTICE that you didn't handle it — that's the lesson.
   ✗ Undo that doesn't snapshot enough state, or snapshots too much. Feel the
     Memento memory/correctness tradeoff (Part 2C.9/4.5).
   ✗ Combat Strategy that needs so much game state it's tightly coupled to the
     world — feel the "communication overhead" Strategy disadvantage.
```

### Extensions

Add save/load (Memento — serialize game state); add a quest system (State machine for quest progress); add item effects via Decorator (a "flaming sword" decorates a "sword"); add a scripting layer (a tiny Interpreter for puzzle logic — feel why Interpreter is heavyweight even for simple grammars); rebuild the whole thing in a function-first language and *watch Strategy and Command collapse into functions and closures* (Part 1.9 felt firsthand — this single exercise teaches language-relativity better than any reading).

---

## 11.2 — INTERMEDIATE PROJECT: A Data Pipeline / ETL Framework

### Goals

Build a configurable data-processing pipeline (Extract-Transform-Load). This project cements **pattern composition** (multiple patterns working together), **the wrapping family** (Adapter, Decorator, Proxy used together so you *feel* their distinct intents), **Command with robust undo**, **Dependency Injection** for testability, and the **Part 4 tiered-implementation discipline** (basic → production-grade). It's where you graduate from single patterns to *systems of patterns*.

### Requirements

A pipeline framework that:
- Reads data from multiple *source types* (CSV file, JSON API, database) through a uniform interface.
- Applies a configurable *chain of transformations* (filter, map, validate, enrich, dedupe), composable in any order.
- Writes to multiple *sink types* (file, database, message queue).
- Supports cross-cutting concerns — logging, metrics, retry — *without* polluting transformation logic.
- Is configured declaratively (a pipeline definition in YAML), so non-programmers can assemble pipelines.
- Handles failures gracefully (a bad record shouldn't kill the batch; a transient source failure should retry).
- Is fully unit-testable (sources/sinks mockable).

### Architecture

```
   SOURCES (Adapter — wrap heterogeneous inputs into one interface):
     CsvAdapter, ApiAdapter, DbAdapter  →  all implement DataSource

   PIPELINE (Chain of Responsibility / Decorator for transforms):
     Source → [Filter] → [Validate] → [Enrich] → [Dedupe] → Sink
              each transform implements Transformer; composable; ordered
              (feel that order matters — validate before enrich, etc.)

   CROSS-CUTTING (Decorator — wrap any stage without touching its logic):
     LoggingTransformer(RetryTransformer(ActualTransformer))
     MetricsSink(RealSink)  — observability + resilience by wrapping (Part 4.1)

   RESILIENCE (Proxy/Decorator):
     RetryingSource (wraps a flaky API source with retry+backoff)
     CircuitBreakerSink (State pattern — stops hammering a dead DB)

   CONSTRUCTION (Builder + Factory + DI):
     PipelineBuilder reads YAML → Factory creates sources/transforms/sinks →
     DI wires them → an executable Pipeline. Composition root pattern (Part 4.4).

   COMMAND (each pipeline RUN is a Command — for scheduling, logging, replay)
```

**The wrapping-family payoff (the key learning):** you'll use Adapter, Decorator, and Proxy *in the same system*, and *feel* their different intents — Adapter *converts* the CSV/API/DB interfaces to a common one, Decorator *adds* logging/metrics/retry by wrapping, Proxy/CircuitBreaker *controls access* to flaky externals. This is Part 2B's disambiguation made tactile: same mechanism (wrap-and-hold), three distinct purposes, used together.

### Implementation Plan

```
   1. Define the core interfaces: DataSource, Transformer, DataSink. Program
      to these from the start (the seam, Part 1.7). Build ONE concrete each,
      hardcoded-wired, end-to-end. Get data flowing.
   2. Add a second source type → extract the Adapter pattern (feel why: the
      CSV and API have totally different native interfaces).
   3. Build the transform chain — make transforms composable and ordered.
      Feel that order matters (dedupe-then-filter ≠ filter-then-dedupe).
   4. Add cross-cutting Decorators (logging, metrics) around stages. Feel how
      Decorator adds observability WITHOUT touching transform logic (SRP).
   5. Add resilience: RetryingSource, CircuitBreakerSink. Feel the Proxy/State
      patterns embracing failure (Part 5).
   6. Build the Builder + Factory to construct pipelines from YAML. This is
      your composition root — feel DI making it all testable.
   7. WRITE TESTS with mocked sources/sinks. Feel the dividend: testability
      comes FREE because you programmed to interfaces and injected dependencies
      (Part 4.7). If something's hard to test, the structure is wrong — fix it.
   8. Tier it up (Part 4): add structured error handling (Result types for bad
      records), batching for throughput, observability for production.
```

### Common Mistakes

```
   ✗ Conflating the wrapping patterns: using Decorator where Adapter is needed
     (or vice versa). If you find yourself confused which to use, ask the
     intent — convert? (Adapter) add? (Decorator) control access? (Proxy).
   ✗ Putting cross-cutting concerns (logging, retry) INSIDE transform logic
     instead of in Decorators → violates SRP, tangles concerns. The fix is
     wrapping, and feeling WHY wrapping is better is the lesson.
   ✗ A pipeline that dies on one bad record (no per-record error isolation) —
     feel the need for fail-safe handling (Result types, Part 4.3).
   ✗ Retry without backoff/jitter → retry storms (Part 5). Or retrying
     non-transient errors (retrying a validation failure forever).
   ✗ Circuit breaker with no half-open state (can't recover) — implement the
     full State machine (Part 5).
   ✗ Hardcoded `new CsvSource()` somewhere, breaking testability. Everything
     through DI/the composition root.
   ✗ The N+1 problem if the enrich step queries a DB per record instead of
     batching (Part 6) — feel the performance trap behind a clean interface.
```

### Extensions

Add parallelism (process records concurrently — feel the latency/throughput tradeoff and concurrency hazards of Parts 6/8); add a dead-letter queue for failed records (Part 5 operability); add backpressure (feel the reactive-streams force of Part 8.6); add pipeline composition (pipelines as sources for other pipelines — Composite); make transforms hot-swappable at runtime; add full observability (tracing through the pipeline — the 3 AM test, Part 9); benchmark and find the bottleneck (Part 6's measure-don't-guess — it's almost certainly I/O, not your patterns).

---

## 11.3 — ADVANCED PROJECT: A Task/Job Scheduling System

### Goals

Build a distributed-style task scheduler (think a mini Celery/Sidekiq/cron). This project cements **architecture-scale patterns** (Repository, Unit of Work, hexagonal architecture), **resilience patterns** (Circuit Breaker, retry, idempotency, bulkhead), **the Command pattern at scale** (jobs as serializable commands in a queue), and forces you to confront **failure modes, concurrency, and operability** as first-class concerns (Parts 5/6/7/9). This is where patterns meet the hard reality of distributed systems and partial failure.

### Requirements

A job scheduler that:
- Accepts jobs (units of work — send an email, generate a report, call an API) submitted via an API.
- Schedules jobs: immediate, delayed, or recurring (cron-style).
- Persists jobs durably (survives a restart — a crashed scheduler resumes pending jobs).
- Executes jobs via a pool of workers, with configurable concurrency.
- Handles failure: retries with backoff, dead-letters permanently-failed jobs, and is *idempotent* (a job that runs twice due to a crash/retry doesn't double-execute its effect).
- Isolates failure: a misbehaving job type can't starve all workers (bulkhead).
- Protects external dependencies with circuit breakers.
- Is observable: job status, queue depth, success/failure rates, latencies — all visible.
- Has a clean, framework-free core (hexagonal — testable without a real DB or queue).

### Architecture

```
   HEXAGONAL CORE (the scheduling domain — pure, no infra, Part 5.1):
     ┌──────────────────────────────────────────────────┐
     │  SchedulingDomain (pure logic):                   │
     │   - Job (a reified Command — serializable)        │
     │   - Schedule (when/how-often)                     │
     │   - retry policy, idempotency keys                │
     │   defines PORTS (interfaces it needs):            │
     │     «JobRepository» «JobQueue» «Clock» «Executor» │
     └──────────────────────────────────────────────────┘
              ▲ adapters implement ports (point INWARD)
     ┌────────┴──────────────────────────────────────────┐
     │ ADAPTERS (infra):                                  │
     │   PostgresJobRepository (Repository + Unit of Work)│
     │   RedisJobQueue / in-memory queue                  │
     │   ThreadPoolExecutor (worker pool, bulkheaded)     │
     │   HttpExecutor wrapped in CircuitBreaker + Retry    │
     └────────────────────────────────────────────────────┘

   JOB LIFECYCLE (State pattern — explicit state machine):
     PENDING → SCHEDULED → RUNNING → (SUCCEEDED | FAILED → RETRYING | DEAD)

   RESILIENCE LAYER:
     - Retry decorator (backoff + jitter) around execution
     - Circuit breaker (State) per external dependency
     - Bulkhead: separate worker pools per job-type so one type can't
       starve others (Part 5)
     - Idempotency: each job has a key; the executor checks "already done?"
       before executing (defends against double-run on crash/retry, Part 5/7)
```

**The architecture lessons made tactile:**
- *Hexagonal (Part 5.1):* the scheduling logic is pure and testable in milliseconds (inject in-memory adapters); swap Postgres→MySQL or Redis→RabbitMQ by swapping adapters, core untouched. *Feel* the dependency-inversion payoff.
- *Repository + Unit of Work (Part 2D):* job persistence behind a clean interface, with transactional state changes (claim-a-job + mark-running atomically, or you'll double-execute under concurrency — feel the TOCTOU hazard of Part 7.6).
- *Command at scale (Parts 2C/4.5/5):* jobs are serializable commands in a queue — *feel* why they must "store data, not object references" (serialization + the deserialization security concern of Part 7.4).
- *Idempotency (Parts 5/7):* the single hardest and most important distributed concept — *feel* why "a job might run twice" forces every job to be safe-to-repeat.

### Implementation Plan

```
   1. Build the pure core with in-memory adapters: submit a job, it runs.
      No DB, no real queue. Test the scheduling LOGIC in isolation (the
      hexagonal payoff — fast tests, no infra). This proves your ports are right.
   2. Add the Job state machine (State pattern). Make illegal transitions
      impossible (you can't go RUNNING→PENDING).
   3. Add persistence (Repository + Unit of Work). CRITICAL: make "claim a job
      for execution" ATOMIC (a single transactional UPDATE...WHERE status=PENDING)
      or two workers grab the same job → double execution. FEEL this race; it's
      the heart of the project.
   4. Add the worker pool + concurrency. Now confront real concurrency hazards.
   5. Add retry+backoff, dead-lettering, and the circuit breaker. Embrace failure.
   6. Add IDEMPOTENCY. Make a job that runs twice harmless. This is the lesson
      that separates this project from the others — distributed reality.
   7. Add bulkheading (separate pools per job type).
   8. Add FULL observability (the 3 AM test, Part 9): queue depth, job states,
      failure rates, latencies. You cannot operate this blind.
   9. Test failure scenarios explicitly: kill the scheduler mid-job (does it
      resume? double-run?); make a dependency fail (does the breaker trip?);
      submit a poison job (does it dead-letter without killing workers?).
```

### Common Mistakes

```
   ✗ THE BIG ONE — non-atomic job claiming → two workers run the same job
     (the TOCTOU race, Part 7.6). The fix (atomic claim) is the project's core
     lesson. If you don't hit this bug, you built it too carefully — most people
     hit it, and the fix teaches concurrency-security viscerally.
   ✗ No idempotency → crashes/retries cause double-sends, double-charges. The
     defining distributed mistake.
   ✗ Letting infra leak into the core (importing the DB driver in the domain) —
     breaks hexagonal, makes the core untestable. Keep the core PURE.
   ✗ Retry without a max → infinite retries of a permanently-failing job. Need
     dead-lettering.
   ✗ No bulkhead → one slow job type's jobs fill all workers, starving everything
     (Part 5). Feel the cascading-failure risk.
   ✗ Serializing jobs with native serialization on untrusted input (Part 7.4
     deserialization RCE) — use data-only formats.
   ✗ Building it observable-LAST or not at all → can't debug failures. Observability
     is a first-class concern, not an afterthought (Part 9).
   ✗ Circuit breaker without half-open → never recovers.
```

### Extensions

Make it genuinely distributed (multiple scheduler instances coordinating via the DB/queue — feel CAP, Part 5); add priority queues; add job dependencies (a DAG — Composite + topological scheduling); add a Saga for multi-step jobs with compensation (Part 5); add exactly-once semantics (and feel why it's *theoretically very hard* — the distributed-systems frontier); build a dashboard (Observer feeding a real-time view). This project, fully extended, is a real, portfolio-worthy distributed system that demonstrates Parts 5–9 in working code.

---

## 11.4 — EXPERT PROJECT: A Rules Engine / Workflow Orchestrator with Plugin Architecture

### Goals

Build a configurable business-rules-and-workflow engine where users define rules and workflows declaratively, and third parties extend it via plugins. This is the *expert* project because it's a genuinely **pattern-rich domain with real, unresolvable tensions** — you'll confront the **Expression Problem** (Part 8.2), make **Visitor-vs-pattern-matching** decisions, build an **Interpreter** and feel its scaling pain, design a **plugin architecture** (Strategy + Factory + DI at the limit), and — most importantly — **experience pattern decay and the un-patterning skill** (Part 8.3) as the system grows. It deliberately has *no clean architecture*; the learning is in *navigating the tensions.*

### Requirements

A rules/workflow engine that:
- Lets users define **rules** declaratively: conditions (`age > 18 AND country IN [US, CA]`) → actions (`approve`, `flag`, `route to manager`).
- Lets users define **workflows**: multi-step processes with branching, parallel steps, and state (an approval workflow, an onboarding flow).
- Evaluates rules against input data and executes the resulting actions.
- Supports a **plugin architecture**: third parties add new condition types, action types, and data sources *without modifying the core*.
- Persists workflow state (long-running workflows survive restarts).
- Is observable and debuggable (you can see *why* a rule fired or didn't — explainability).
- Handles rule/workflow *versioning* (a workflow in flight must complete on the version it started, even as new versions deploy).

### Architecture

```
   RULE EVALUATION (Interpreter + Composite — feel Interpreter's limits):
     Rules parse into an AST of Condition objects (Composite):
       And(GreaterThan(age,18), In(country,[US,CA]))
     Each Condition.evaluate(context) → boolean (Interpreter pattern)
     FEEL: this works for simple rules but scales BADLY as the grammar grows
     (Part 2C.11) — at some point you'll want a real parser or a different
     approach. Experiencing that limit IS the lesson.

   THE EXPRESSION PROBLEM (Part 8.2 — the central tension):
     You have CONDITION TYPES (GreaterThan, In, Regex...) and OPERATIONS over
     them (evaluate, explain-why, optimize, serialize, validate).
     - New condition type? New operation? You CAN'T make both cheap.
     - You must CHOOSE: OO methods (easy new types, hard new operations) vs.
       Visitor/pattern-matching (easy new operations, hard new types).
     - Plugins add new TYPES (3rd parties add conditions) AND you add new
       OPERATIONS (explain, optimize). BOTH axes vary. This is the genuine,
       unresolvable tension — navigate it consciously.

   WORKFLOW ORCHESTRATION (State + Command + Memento + Saga):
     - Workflow = State machine (each step a state, transitions on events)
     - Steps = Commands (executable, with compensation for rollback — Saga)
     - State persistence = Memento (snapshot workflow state durably)
     - Long-running = the Part 5 distributed concerns

   PLUGIN ARCHITECTURE (Strategy + Factory + DI + Adapter, at the limit):
     - Plugins register new Conditions/Actions/Sources via a registry
     - Loaded dynamically, wired via DI, isolated (a bad plugin can't crash core)
     - Anti-Corruption Layer (Adapter, Part 7) validates plugin I/O at the boundary

   EXPLAINABILITY (Visitor or pattern-matching over the rule AST):
     "Why didn't this rule fire?" → traverse the AST, show which condition
     failed. A NEW OPERATION over the condition types — the Expression Problem
     bites HERE (adding "explain" means touching every condition, OR a Visitor).
```

**Why this project is expert-level:** there is *no clean answer*. The Expression Problem (Part 8.2) is genuinely present and forces a conscious tradeoff. The Interpreter pattern *will* hit its scaling wall and you'll feel the pull toward a real parser. The plugin architecture pushes Strategy+Factory+DI to their limits and surfaces security concerns (untrusted plugin code, Part 7). And as you add features (explain, optimize, version, parallelize), you'll watch your initially-clean patterns *decay* — and have to *refactor and un-pattern* (Part 8.3). The project is a controlled environment for experiencing the frontier.

### Implementation Plan

```
   1. Build simple rule evaluation: Condition interface, a few conditions
      (Composite + Interpreter), evaluate against data. Hardcode rules first.
   2. Add rule parsing (text → AST). Start with a hand-rolled parser; FEEL the
      pain as the grammar grows (this teaches WHY Interpreter doesn't scale and
      why parser generators exist, Part 2C.11).
   3. Add the FIRST new operation over conditions: "explain why." HERE you hit
      the Expression Problem. Make a conscious choice (OO methods vs. Visitor vs.
      pattern matching, depending on your language) and DOCUMENT WHY (an ADR,
      Part 9). Feel the tradeoff: whichever you pick makes one axis painful.
   4. Build the workflow engine (State + Command). Add persistence (Memento).
   5. Build the plugin architecture. Register conditions/actions via a registry;
      wire via DI; isolate plugins. NOW the Expression Problem bites hard —
      plugins add TYPES, you add OPERATIONS, both axes move. Live with the
      tension you chose in step 3, or refactor (un-patterning, Part 8.3).
   6. Add versioning (in-flight workflows on old versions). Feel the
      serialization/schema-evolution pain (Parts 7.4/8.8).
   7. Add explainability, optimization (constant-folding the rule AST), and
      observability.
   8. STEP BACK and assess decay: which patterns have rotted? Is your Mediator/
      registry a god object? Is an abstraction now wrong? REFACTOR — including
      INLINING wrong abstractions back to duplication and re-extracting (the
      advanced un-patterning move, Part 8.3). This reflective refactor is the
      capstone lesson.
```

### Common Mistakes

```
   ✗ Choosing the Expression Problem axis UNCONSCIOUSLY, then fighting the
     structure forever. The lesson is to choose CONSCIOUSLY and document why —
     and to recognize that BOTH axes varying means you'll have pain somewhere
     no matter what (there's no clean win; accept the tension).
   ✗ Pushing Interpreter too far instead of switching to a real parser when the
     grammar outgrows it. Knowing WHEN to abandon a pattern is the skill.
   ✗ A plugin system with no isolation/sandboxing → a bad plugin crashes the
     core, or a malicious plugin executes arbitrary code (Part 7 — untrusted
     code at a trust boundary).
   ✗ Over-engineering the plugin architecture before you have a SECOND plugin
     (YAGNI at the expert level — even here, wait for the real second case).
   ✗ Not handling versioning → a workflow in flight breaks when you deploy a new
     version. The schema-evolution problem (Part 8.8).
   ✗ Letting the rule registry / orchestrator become a god object as it grows
     (Mediator decay, Part 8.3) — and NOT noticing/refactoring it.
   ✗ The WRONG ABSTRACTION: forcing conditions and actions into a shared
     abstraction because they "look similar," then drowning in special cases.
     The expert move is recognizing this and INLINING back to duplication
     (Part 8.3) — most people won't do this; doing it is the mastery signal.
   ✗ Deserializing untrusted rule/workflow definitions unsafely (Part 7.4).
```

### Extensions

Add a visual workflow builder (Observer-driven UI); add a DSL with a proper grammar (replace the hand-rolled Interpreter — feel the upgrade); add distributed workflow execution (Saga across services, Part 5); add A/B testing of rule versions; add a rule-conflict detector (two rules that contradict); make it multi-tenant (per-tenant rules with isolation — Part 7); add ML-based rule suggestions (the AI-era frontier, Part 8.7 — a probabilistic component in a deterministic system, with all the new forces that brings). Fully realized, this is a sophisticated product demonstrating *every* Part of this course — and, more importantly, demonstrating the *judgment* to navigate genuine tensions, recognize decay, and un-pattern when needed.

---

## 11.5 — How to Get the Most From These Projects

The projects only build mastery if approached deliberately. The meta-discipline:

```
   FOR EACH PROJECT:
   1. Build the NO-PATTERN version first where possible. Feel the PAIN that
      the pattern relieves. A pattern learned by feeling its absence is
      understood; a pattern applied from the start is memorized.

   2. Add each pattern only when you FEEL its force. Notice the temptation to
      add patterns speculatively — and RESIST it. The resistance is the training.

   3. When you hit a pattern's HAZARD (the Observer leak, the TOCTOU race, the
      wrong abstraction), don't just fix it — PAUSE and connect it to the
      relevant Part. The visceral "oh, THIS is what Part 4.2 meant" is where
      abstract knowledge becomes owned knowledge.

   4. Make every pattern decision an ADR (Part 9). Writing "I chose X because
      force Y, rejected Z because W, accepting cost C" trains the force-thinking
      and the communication skill interviews test (Part 10).

   5. Build at least one project in a function-first language. WATCH Strategy,
      Command, Template Method collapse into functions/closures. This single
      experience teaches language-relativity (Part 1.9) more deeply than any reading.

   6. After the advanced/expert projects, REFLECT on decay: revisit your early
      pattern choices after the system grew. Which rotted? Refactor them —
      including inlining wrong abstractions (Part 8.3). This reflective loop is
      where senior becomes staff.

   7. Profile the projects (Part 6). Find the real bottleneck. It will almost
      NEVER be your patterns — it'll be I/O. Feel that lesson firsthand: you'll
      stop fearing pattern overhead and start respecting I/O cost.
```

**The unifying instruction:** these projects are not about *finishing* — they're about *experiencing the forces, costs, hazards, and tensions* in your own hands. A project where you smoothly applied patterns from memory taught you little; a project where you over-engineered and felt it, hit a race condition and debugged it, chose the wrong Expression-Problem axis and suffered, or built a wrong abstraction and had to inline it — *that* project converted the course's knowledge into judgment. **Struggle at the decision points is the curriculum; the working code is just the byproduct.**

---

## 11.6 — Hands-On Projects Synthesis

**1. Four projects, escalating to cement progressively deeper mastery.** Beginner (a game engine — the four core patterns + force-first instinct + the over-engineering temptation); Intermediate (an ETL pipeline — pattern composition + the wrapping family + tiered implementation + testability); Advanced (a job scheduler — architecture-scale patterns + distributed failure modes + idempotency + the TOCTOU race + operability); Expert (a rules/workflow engine — the Expression Problem + Interpreter's limits + plugin architecture + pattern decay + the un-patterning skill).

**2. Each is designed to make you FEEL the forces, costs, hazards, and tensions** the prior Parts described abstractly — the over-engineering temptation, the Observer leak, the concurrency race, the wrong abstraction, the unresolvable Expression Problem. The struggle at the decision points is the actual curriculum.

**3. The meta-discipline converts code into judgment:** build the no-pattern version first to feel the pain; add patterns only when the force appears (resisting speculation); pause at every hazard to connect it to its Part; write ADRs to train force-thinking and communication; build one project in a function-first language to feel language-relativity; reflect on decay and practice un-patterning; profile to learn that I/O — not your patterns — is the real cost.

**4. The projects, fully extended, are portfolio-worthy real systems** demonstrating every Part of the course in working code — but their deeper value is the *experiential* conversion of recognition into judgment, which is the only path from senior to staff that reading alone cannot provide.

You now have a complete program for *building* mastery, not just reading about it — four projects calibrated to make the forces tangible, the hazards visceral, and the tensions unavoidable, with a meta-discipline that turns every project into deliberate practice of the judgment this entire course is built to develop.

Part 12 (Mastery Roadmap) — the final Part — will consolidate everything into a learning sequence, competency checklist, and milestone map from Beginner through Staff and Expert, so you have a concrete path to follow and a way to assess where you are on it.