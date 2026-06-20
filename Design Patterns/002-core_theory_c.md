# PART 2C — BEHAVIORAL PATTERNS

**The unifying force:** Behavioral patterns answer *"how do objects communicate, distribute responsibility, and coordinate behavior — without welding the participants tightly together?"* Where creational patterns hide *which* object is made and structural patterns manage *how objects are composed*, behavioral patterns govern *the flow of control and the assignment of responsibility between objects at runtime.* This is the largest GoF family (eleven patterns) and the one where Part 1's **language-relativity** insight bites hardest — Strategy, Command, Iterator, and Template Method nearly dissolve in languages with first-class functions.

```
What each behavioral pattern governs:

Strategy         → swap an ALGORITHM at runtime
Observer         → notify many dependents when one thing changes
Command          → turn a REQUEST into an object (undo, queue, log)
Iterator         → traverse a collection without exposing its guts
Template Method  → fix an algorithm's SKELETON, vary the steps
State            → change behavior when internal STATE changes
Chain of Resp.   → pass a request along a line of handlers
Mediator         → centralize many-to-many communication
Memento          → snapshot & restore state (undo)
Visitor          → add operations to a structure without modifying it
Interpreter      → represent & evaluate a grammar
```

Two clusters worth flagging up front, because they're easily confused:
- **Strategy / State / Template Method** all swap behavior — but Strategy swaps *interchangeable algorithms* the client picks, State swaps *behavior driven by internal state transitions*, Template Method swaps *steps within a fixed skeleton via inheritance*. Same mechanism (polymorphic behavior), different *intent and driver*.
- **Command / Memento** both capture something as an object — Command captures a *request/action*, Memento captures a *state snapshot*. They often collaborate to build undo systems.

---

## 2C.1 — Strategy

### The Force — The Cleanest Illustration of Pattern Thinking

You have a task that can be done by **multiple interchangeable algorithms**, and the choice varies by context, configuration, or runtime data. Hardcoding the choice with conditionals (`if (mode == FAST) quickSort() else if (mode == STABLE) mergeSort()`) violates OCP (every new algorithm edits the conditional), tangles unrelated algorithms in one class (violating SRP), and can't be swapped at runtime. Strategy resolves "extract each algorithm behind a common interface and inject the chosen one, so algorithms vary independently of the code that uses them."

### Intent

> Define a family of algorithms, encapsulate each one, and make them interchangeable. Strategy lets the algorithm vary independently from clients that use it.

### Motivation Story

A navigation app computes routes. The *routing algorithm* differs: fastest, shortest, scenic, avoid-tolls, walking, cycling. The map UI (zoom, render, mark waypoints) is identical regardless. You don't want the `Navigator` class swollen with a giant conditional over route types — adding "avoid-highways" would mean editing tested code and risking the others. Instead, define a `RouteStrategy` interface with `buildRoute(from, to)`; each algorithm is its own class (`FastestRoute`, `ScenicRoute`...). The `Navigator` holds a `RouteStrategy` reference and delegates. Pick or swap the strategy at runtime — even mid-session when the user toggles "avoid tolls." Adding a new strategy = adding a class, touching nothing existing.

### Structure

```
   ┌──────────────────┐  has-a   ┌──────────────────┐
   │     Context      │ ────────►│   «Strategy»     │
   │ - strategy       │          │ + execute(data)  │
   │ + doWork() ──────┼──┐       └────────▲─────────┘
   └──────────────────┘  │ delegates       │
            strategy.execute(data)         │
                                  ┌────────┼─────────┐
                          ┌───────┴──┐ ┌───┴────┐ ┌──┴──────┐
                          │StrategyA │ │StrategyB│ │StrategyC│
                          │(Fastest) │ │(Scenic) │ │(Walking)│
                          └──────────┘ └─────────┘ └─────────┘
```

### Internal Mechanics

The `Context` holds a reference to the `Strategy` *interface* (injected at construction or via a setter — composition + DIP). When work is needed, it delegates to `strategy.execute()`. Polymorphic dispatch routes to whichever concrete strategy is currently set. The context contains the *invariant* surrounding logic; the strategy contains the *varying* algorithm. Swapping is a one-line `setStrategy()` call. This is the textbook embodiment of "program to an interface" + "favor composition" + OCP — which is why it's the pattern most often used to *teach* pattern thinking.

### The Language-Relativity Payoff (from Part 1, section 1.9)
In a language with first-class functions, **Strategy is just a function parameter:**
```python
def navigate(start, end, route_fn):   # route_fn IS the strategy
    return route_fn(start, end)

navigate(a, b, fastest_route)         # no interface, no class, no Context boilerplate
```
The entire class hierarchy collapses to "pass the function." `sorted(data, key=...)`, JavaScript's `arr.sort(comparator)`, Go's `sort.Slice(s, less)` — all are Strategy without anyone calling it that. The full OO version earns its weight only when strategies are *stateful*, need *multiple methods*, or require *named, injectable, testable* identities.

### Advantages
- Swap algorithms at runtime; isolate each algorithm (SRP); add new ones without touching existing code (OCP).
- Eliminates sprawling conditionals; each algorithm independently testable.
- Replaces algorithm-selection inheritance with cleaner composition.

### Disadvantages
- More objects/classes (in the heavyweight version); over-engineering for two trivial branches.
- Client must know strategies exist to choose one (some knowledge leaks).
- Communication overhead if the strategy needs lots of data from the context.

### Tradeoffs
Strategy trades a small structural overhead for runtime swappability and OCP-clean extension. Worth it when algorithms genuinely multiply or change; overkill for a stable two-way `if`. In function-first languages, the overhead nearly vanishes — reach for it freely.

### When NOT to Use
Two fixed branches that won't grow (a plain `if` is clearer). When the "algorithm" never varies. When passing a function is more idiomatic than a class.

### Language Note
The canonical "pattern that's just a function." Heavyweight in Java/C++ (interface + classes), near-invisible in Python/JS/Go/Kotlin/Rust (closure or function value). Recognize the *force* (swappable behavior); reach for the *lightest local tool*.

---

## 2C.2 — Observer

### The Force

When **one object's state changes, an unknown number of other objects must react**, and you want this *without* the changing object knowing the concrete identities of its dependents. Hardcoding the notifications (`subject.updateDisplay(); subject.updateLog(); subject.updateCache();`) welds the subject to every dependent (tight coupling, OCP violation — each new dependent edits the subject) and makes the subject's responsibility balloon. Observer resolves "let objects subscribe to a subject and be notified automatically on change, with the subject knowing only an abstract `Observer` interface, not concrete dependents."

### Intent

> Define a one-to-many dependency between objects so that when one object changes state, all its dependents are notified and updated automatically.

### Motivation Story

A spreadsheet cell holds a value. A bar chart, a pie chart, and a sum-cell all display data derived from it. When the user edits the cell, *all three* must update — but the cell shouldn't know charts exist (today it's three views; tomorrow ten). The cell (`Subject`) maintains a list of `Observer`s. Views *subscribe*. On edit, the cell calls `notify()`, looping over observers calling `update()`. Each view refreshes itself. Add a new view? It just subscribes — the cell's code never changes. This is the foundation of all event systems, reactive UIs, and pub/sub.

### Structure

```
   ┌────────────────────┐  notifies all  ┌──────────────────┐
   │     «Subject»      │ ──────────────► │   «Observer»     │
   │ - observers: List  │                 │ + update(...)    │
   │ + subscribe(o)     │                 └────────▲─────────┘
   │ + unsubscribe(o)   │                          │ implements
   │ + notify() ────────┼─► for o in observers:    │
   │ + setState()       │      o.update(state)     │
   └────────▲───────────┘                ┌─────────┼─────────┐
            │ holds state          ┌─────┴───┐ ┌────┴───┐ ┌───┴────┐
   ┌────────┴────────┐             │ObserverA│ │ObserverB│ │ObserverC│
   │ ConcreteSubject │             │ (Chart) │ │ (Log)   │ │ (Cache) │
   └─────────────────┘             └─────────┘ └─────────┘ └─────────┘
```

### Internal Mechanics & Push vs. Pull

`subscribe`/`unsubscribe` mutate the observer list. `notify()` iterates and calls `update()` on each. Two models for *how data reaches observers*:
- **Push:** the subject *sends the changed data* in `update(newValue)`. Simpler for observers, but the subject must guess what each observer wants (may send too much/too little).
- **Pull:** the subject sends only `update(this)`; each observer *queries the subject* for exactly what it needs. More flexible, more coupling-to-subject, more calls.

**Critical hazards (interview gold):**
- **Lapsed listener / memory leak:** if an observer forgets to `unsubscribe`, the subject's list keeps it alive forever → leak. (Java mitigates with `WeakReference`; many bugs come from this.)
- **Notification storms / cascades:** an observer's `update` triggers another change, triggering more notifications — potential infinite loops or performance cliffs.
- **Ordering:** observers are notified in list order; relying on order is fragile.
- **Re-entrancy:** if an observer subscribes/unsubscribes *during* notification, you're mutating the list you're iterating → concurrent-modification bugs.
- **Threading:** which thread runs `update()`? Notifications across threads need synchronization.

### Advantages
- Loose coupling: subject knows only the `Observer` interface, not concrete observers (DIP).
- OCP: add new observers without touching the subject.
- Dynamic relationships: subscribe/unsubscribe at runtime.
- Foundation of event-driven and reactive architectures.

### Disadvantages
- The hazards above (leaks, cascades, ordering, re-entrancy, threading) are *real and common*.
- Notification order is unpredictable/implicit.
- Debugging is hard: control flow is *implicit* — "who updated this?" leads through invisible subscriptions (the core cost of event-driven systems).
- Can cause performance issues at scale (many observers, frequent changes).

### Tradeoffs
Observer trades *explicit, traceable control flow* for *loose coupling and dynamic, open-ended notification*. The decoupling is powerful and often essential, but you pay in debuggability (implicit flow) and a suite of subtle lifecycle/concurrency hazards. Worth it whenever dependents are many, changing, or unknown to the subject.

### When NOT to Use
When there's one fixed dependent (just call it directly). When the implicit control flow would obscure critical logic. When synchronous, ordered, traceable updates matter more than decoupling.

### Language Note
Built into many ecosystems: Java's `PropertyChangeListener`, Node's `EventEmitter`, the DOM's `addEventListener`, C#'s `event`/`delegate`, RxJS/Reactive Extensions (Observer on steroids — streams of events). Modern reactive frameworks (React, Vue, Svelte, MobX, signals) are industrial-strength Observer. You rarely hand-roll it; you use a framework's version — but understanding the hazards above explains most "why did my UI update twice / leak / loop?" bugs.

---

## 2C.3 — Command

### The Force

You want to **turn a request/action into a first-class object** — so you can store it, pass it around, queue it, log it, schedule it, or *undo* it — decoupling the object that *invokes* an operation from the object that *performs* it. Calling `receiver.doThing()` directly means the invoker is welded to the receiver and the action happens immediately, un-reified, un-undoable, un-queueable. Command resolves "encapsulate a request as an object, parameterizing senders with different requests and enabling queuing, logging, and undo."

### Intent

> Encapsulate a request as an object, thereby letting you parameterize clients with different requests, queue or log requests, and support undoable operations.

### Motivation Story

A text editor's toolbar, menu, and keyboard shortcut can all trigger "Bold." You don't want each UI element hardwired to the bolding logic. Instead, a `BoldCommand` object encapsulates the action (`execute()` calls the document's bolding) *and its inverse* (`undo()`). The button, menu, and shortcut all just hold a `Command` and call `execute()` — they don't know what it does. The editor keeps a *history stack* of executed commands; Ctrl+Z pops one and calls `undo()`. A macro = a list of commands executed in sequence. Queuing, logging for replay/audit, and remote execution all fall out naturally because the request is now a *thing you can hold*.

### Structure

```
┌──────────┐  holds   ┌─────────────┐  invokes  ┌──────────────┐
│ Invoker  │ ───────► │ «Command»   │           │  (Receiver)  │
│ (Button) │          │ + execute() │           │ + action()   │
│ + run()──┼─► cmd.execute()        │           └──────▲───────┘
└──────────┘          │ + undo()    │                  │
                      └──────▲──────┘                  │ calls
                             │                          │
                   ┌─────────┴────────┐                 │
                   │ ConcreteCommand  │─────────────────┘
                   │ - receiver       │ execute(){ receiver.action() }
                   │ - args (state)   │ undo()   { receiver.inverse() }
                   └──────────────────┘
```

### Internal Mechanics

A `ConcreteCommand` stores a reference to the **receiver** (who does the work) plus any **arguments/state** needed (and, for undo, the *prior state* to restore). `execute()` invokes the receiver's method with stored args; `undo()` reverses it (often by restoring a saved snapshot — note the **Memento** collaboration). The **invoker** holds commands abstractly and triggers `execute()` without knowing the receiver. Because commands are objects, they go into lists (macros/queues), stacks (undo history), logs (replay/audit), or across the wire (remote execution).

### Undo Strategies (the depth most courses skip)
- **Inverse operation:** `undo()` computes the reverse (undo "add 5" = "subtract 5"). Cheap but not always possible (irreversible ops).
- **State snapshot (Memento):** before executing, save the affected state; `undo()` restores it. General but memory-heavy.
- **Command log + replay:** to reach state N, replay commands 1..N from a known baseline (this is **event sourcing** at the architectural scale — a profound connection covered in Part 5).

### Advantages
- Decouples invoker from receiver (the invoker is reusable across any command).
- First-class requests → undo/redo, queues, logging, scheduling, macros, transactions, remote execution.
- OCP: new commands without changing invokers.
- Composable (macro commands = commands of commands — note the **Composite** synergy).

### Disadvantages
- A class per action → proliferation (many tiny command classes).
- Undo logic can be genuinely hard (irreversible operations, large state).
- Indirection overhead for simple direct calls.

### Tradeoffs
Command trades a proliferation of small objects for the powers of reification: undo, queuing, logging, scheduling, decoupling. Worth it precisely when you need *one or more* of those powers; pure overhead for a direct call that needs none of them. The undo requirement is usually the deciding factor.

### When NOT to Use
When you just need to call a method and never need undo/queue/log/decoupling. When a function/closure suffices (see below).

### Language Note
In function-first languages, a Command is often **a closure** — `() => doc.bold()` captures the receiver and args, *is* `execute()`, and can be stored in a list. JavaScript event handlers, Python callables in a task queue, Go funcs in a channel — all Command-without-the-class. The full class version earns its keep when commands need *multiple methods* (`execute` + `undo` + `serialize`), persistent identity, or rich state. Task queues (Celery, Sidekiq), job systems, and CQRS "commands" are Command at architectural scale.

---

## 2C.4 — Iterator

### The Force

You need to **traverse the elements of a collection sequentially without exposing its internal representation** (array? linked list? tree? hash table?), and you want a *uniform* traversal interface across different collection types, plus the ability to have *multiple simultaneous traversals*. Letting clients poke at the collection's internals (indexing into the backing array, walking node pointers) welds them to the structure — change the structure, break every traversal. Iterator resolves "provide a standard way to walk a collection's elements one by one, hiding how the collection is built."

### Intent

> Provide a way to access the elements of an aggregate object sequentially without exposing its underlying representation.

### Motivation Story

You have a custom `Tree` collection. Clients need to visit every element — but should they know it's a tree, and write recursive traversal themselves? No — that couples them to the structure and duplicates traversal logic everywhere. Instead, `Tree` produces an `Iterator` with `hasNext()`/`next()`. Clients write `while (it.hasNext()) process(it.next())` — *identical code* whether the underlying thing is a tree, array, or hash set. The tree's iterator internally knows the traversal (in-order, say); the client just consumes elements. Swap the tree for a list later — client code is untouched.

### Structure

```
┌──────────────────┐  creates  ┌──────────────────┐
│  «Aggregate»     │ ────────► │   «Iterator»     │
│ + createIterator()│          │ + hasNext():bool │
└────────▲─────────┘           │ + next(): Element│
         │                     └────────▲─────────┘
┌────────┴─────────┐                    │
│ConcreteAggregate │           ┌────────┴─────────┐
│ (Tree/List/Set)  │──creates─►│ ConcreteIterator │
│                  │           │ - position/cursor│
└──────────────────┘           │ (knows traversal)│
                               └──────────────────┘

Client:  it = collection.createIterator();
         while (it.hasNext()) { e = it.next(); ... }
```

### Internal Mechanics

The iterator holds **traversal state** (a cursor: array index, current node, stack for tree traversal) *external to the collection* — which is why you can have *multiple independent iterators* over one collection simultaneously (each with its own cursor). `next()` returns the current element and advances the cursor; `hasNext()` reports whether more remain. The collection stays oblivious; the iterator encapsulates *how to walk*. A key concern: **iterator invalidation** — if the collection is modified during iteration, the cursor may become invalid (Java throws `ConcurrentModificationException`; C++ iterators can dangle — undefined behavior). **Internal vs. external iterators:** *external* (client drives `next()` — more control) vs. *internal* (collection drives, you pass a function — `forEach`; less control, less error-prone).

### Advantages
- Decouples traversal from collection structure (SRP — collection stores, iterator walks).
- Uniform traversal interface across collection types (polymorphic iteration).
- Multiple simultaneous, independent traversals.
- Supports different traversal orders via different iterators (in-order, pre-order, reverse).

### Disadvantages
- Overkill for simple arrays you can just index.
- Iterator invalidation on concurrent modification (a real, common bug source).
- A separate iterator object per traversal (minor overhead).

### Tradeoffs
Iterator trades a small abstraction for structure-independent, uniform, multi-traversal access. The trade is almost always made *for you* by the language/stdlib. The cost is negligible; the invalidation hazard is the only real gotcha.

### When NOT to Use
Essentially never hand-rolled in modern languages — the built-in protocol does it. You'd only implement a *custom* iterator for a *custom* collection or a special traversal order.

### Language Note
**The most thoroughly absorbed-into-languages pattern of all.** Python's `__iter__`/`__next__` + `for x in collection`; Java's `Iterable`/`Iterator` + for-each; C#'s `IEnumerable` + `yield`; JavaScript's iterator protocol + `for...of` and generators; Go's `range`; Rust's `Iterator` trait. **Generators/coroutines** (`yield`) are an even more powerful evolution — lazy, on-demand iteration where the traversal *suspends and resumes*. You almost never write the GoF class version; you implement the language's iteration protocol. This is the textbook case of "a pattern becoming a language feature."

---

## 2C.5 — Template Method

### The Force

You have several variants of a process that share the **same overall sequence of steps**, but **individual steps differ**. Duplicating the whole sequence in each variant violates DRY and means a change to the *shared* skeleton must be made in every copy (error-prone). You want to write the invariant skeleton *once* and let variants override only the *differing* steps. Template Method resolves "define the skeleton of an algorithm in a base class, deferring specific steps to subclasses, so subclasses redefine steps without changing the algorithm's structure."

### Intent

> Define the skeleton of an algorithm in an operation, deferring some steps to subclasses. Template Method lets subclasses redefine certain steps of an algorithm without changing the algorithm's structure.

### Motivation Story

A data-processing pipeline always: (1) open source, (2) parse, (3) transform, (4) validate, (5) save. For CSV vs. JSON vs. XML, steps 1–5 happen *in the same order*, but *parsing* and *validating* differ per format. Write an abstract `DataProcessor.process()` (the **template method**) that calls the five steps in order — this is the fixed skeleton, written once. Make `parse()` and `validate()` abstract; `CsvProcessor` and `JsonProcessor` override only those. The sequence is guaranteed identical (no subclass can reorder it); only the varying steps differ. Change the *order*? Edit one place. This is the **Hollywood Principle: "Don't call us, we'll call you"** — the base class calls *down* into the subclass's steps, inverting normal control.

### Structure

```
┌─────────────────────────────────┐
│        AbstractClass            │
├─────────────────────────────────┤
│ + templateMethod()  ◄───────────┼── the FIXED skeleton (final, not overridable)
│     step1();                    │     calls steps in fixed order
│     step2();  // abstract       │
│     step3();                    │
│ # step1()   (concrete/default)  │
│ # step2()   (ABSTRACT) ─────────┼── subclasses MUST fill these
│ # step3()   (hook, optional)    │
└──────────────▲──────────────────┘
               │ overrides only the varying steps
       ┌───────┴────────┐
   ┌───┴──────┐   ┌──────┴─────┐
   │ SubclassA│   │ SubclassB  │
   │ +step2() │   │ +step2()   │
   └──────────┘   └────────────┘
```

### Internal Mechanics & Hooks

The `templateMethod()` is **non-overridable** (Java `final`) — it *enforces* the skeleton; subclasses can't break the sequence, only fill steps. Steps come in flavors:
- **Abstract steps:** subclass *must* implement (the genuine variation points).
- **Concrete steps:** shared default in the base, used as-is.
- **Hooks:** *optional* override points with empty/default bodies — subclasses *may* hook in extra behavior at defined moments (e.g., a `beforeSave()` hook). Hooks give controlled extensibility without forcing overrides.

The control inversion (Hollywood Principle) is the crux: instead of subclass code calling shared utilities, the *base class orchestrates* and calls *down* into subclass steps. This is also exactly **how frameworks work** — you fill in methods; the framework's skeleton calls them.

### Template Method vs. Strategy (the key contrast)
- **Template Method** uses **inheritance** — variation is *compile-time*, baked into subclasses, and you vary *individual steps within one fixed algorithm*. The skeleton is in a base class.
- **Strategy** uses **composition** — variation is *runtime*, swappable, and you replace the *whole algorithm*. The algorithm is in a held object.
Template Method is "inheritance-based, fine-grained, compile-time step variation"; Strategy is "composition-based, whole-algorithm, runtime swapping." Given Part 1's composition-over-inheritance lean, Strategy is often preferred — but Template Method shines when steps share substantial code and the skeleton genuinely shouldn't change.

### Advantages
- Eliminates duplication of the shared skeleton (DRY); the sequence lives in one place.
- Subclasses customize only what varies (focused, minimal).
- Enforces the algorithm's structure (subclasses can't reorder/break it).
- The framework extension mechanism (Hollywood Principle) — controlled, defined extension points.

### Disadvantages
- **Inheritance-based** → all its downsides: tight base-subclass coupling, fragile base class, compile-time fixity (Part 1's whole case against inheritance applies).
- Can't swap behavior at runtime (unlike Strategy).
- Skeleton changes can ripple to all subclasses.
- Too many hooks/steps → hard to follow which subclass does what.
- Subclasses are constrained by the base's design (the LSP discipline matters).

### Tradeoffs
Template Method trades inheritance's rigidity for DRY skeleton-sharing and enforced structure. Worth it when variants genuinely share a substantial fixed sequence and you want to *prevent* reordering. When you need runtime swapping or want to avoid inheritance, prefer Strategy (composition). Often these are two solutions to overlapping problems — pick by *inheritance vs. composition* and *step-level vs. whole-algorithm* variation.

### When NOT to Use
When variants don't share a common skeleton. When you need runtime behavior swapping (use Strategy). When inheritance's coupling is unacceptable.

### Language Note
In function-first languages, Template Method becomes a **higher-order function**: a function implementing the skeleton that takes the varying steps *as function arguments*. `def process(parse_fn, validate_fn): open(); parse_fn(); ...` — no inheritance, runtime-flexible, the skeleton-with-injected-steps without a class hierarchy. This is strictly more flexible (runtime + no inheritance coupling) and is the idiomatic choice where functions are first-class — making the inheritance-based Template Method largely a Java/C++/C# shape.

---

## 2C.6 — State

### The Force

An object's **behavior must change depending on its internal state**, and the naive implementation is a sprawl of conditionals (`if (state == DRAFT) ... else if (state == REVIEW) ... else if (state == PUBLISHED) ...`) *repeated in every method*. Each method re-branches on state; adding a new state means editing every method (OCP violation); the transition rules are scattered and implicit; and it's impossible to see "what can happen in REVIEW state" in one place. State resolves "encapsulate each state as an object with its own behavior and transition logic, so the context delegates to its current state object — and changing state means changing which object it delegates to."

### Intent

> Allow an object to alter its behavior when its internal state changes. The object will appear to change its class.

### Motivation Story

A document moves through `Draft → Moderation → Published`. The `publish()` action does different things in each: in Draft it submits for review; in Moderation (if you're an admin) it publishes, else does nothing; in Published it does nothing. The `edit()`, `publish()`, `reject()` methods each branch on state — a tangle. With State: define a `DocumentState` interface with `publish()`, `edit()`, etc. Implement `DraftState`, `ModerationState`, `PublishedState`, each with behavior *for that state only*, including *which state to transition to*. The `Document` (context) holds a `currentState` and delegates every call to it. `publish()` in `DraftState` sets the context's state to `ModerationState`. The state machine becomes explicit, each state is self-contained, and adding a state is adding a class.

### Structure

```
   ┌──────────────────┐  delegates  ┌──────────────────┐
   │     Context      │ ──────────► │    «State»       │
   │ - state: State   │             │ + handle(ctx)    │
   │ + request() ─────┼─► state.handle(this)           │
   │ + setState(s)    │◄────────────┼── states call back to
   └──────────────────┘  transition │    change the context's state
                                    └────────▲─────────┘
                              ┌──────────────┼──────────────┐
                        ┌─────┴────┐   ┌──────┴────┐   ┌─────┴─────┐
                        │DraftState│   │ReviewState│   │PublishedSt│
                        │handle(){ │   │handle(){  │   │handle(){  │
                        │ ctx.set( │   │ ...       │   │ no-op     │
                        │ Review)} │   │}          │   │}          │
                        └──────────┘   └───────────┘   └───────────┘
```

### Internal Mechanics

The context delegates each behavioral method to its `currentState` object. Each state object knows: (1) how to behave *in this state*, and (2) which state to transition to next (it calls `context.setState(nextState)`). **Where transitions live is a design choice:** in the state objects (decentralized — each state owns its outgoing transitions; most common) or in the context (centralized transition table). State objects are often stateless and shareable (Flyweight synergy) — they hold no per-context data, so one `PublishedState` instance can serve all documents.

### State vs. Strategy (the most confused pair in the catalog)
Structurally **identical** — a context delegating to a swappable behavior object. The difference is *intent and who drives the swap*:
- **Strategy:** the *client* picks the algorithm; strategies are *independent and unaware of each other*; the choice is "which way to do this one thing." Swapping is external.
- **State:** the object *transitions itself* between states based on events; states *know about and trigger transitions to other states*; it models a *state machine*. Swapping is internal and rule-driven.
Strategy is "pick an interchangeable algorithm." State is "I am a state machine and my behavior is whatever my current node says." Same skeleton, opposite story.

### Advantages
- Eliminates massive state-conditionals; each state's behavior is localized (SRP).
- Transitions are explicit and encapsulated; the state machine becomes readable.
- OCP: new states = new classes, no editing existing states.
- Makes illegal transitions impossible to express accidentally.

### Disadvantages
- A class per state → proliferation (overkill for 2–3 simple states).
- Transition logic spread across state classes can make the *whole* machine hard to see at once (the centralized-vs-decentralized tradeoff).
- Context and states are coupled (states call back to set context state).

### Tradeoffs
State trades a class explosion for an explicit, maintainable state machine with localized behavior. Worth it when states are many, behavior differs substantially per state, and transitions are complex. For a trivial two-state toggle, a boolean and an `if` is honest and simpler.

### When NOT to Use
Few states with trivial behavior differences. When a simple enum + switch is genuinely clearer (don't pattern-ize a light switch). When there's no real state-machine.

### Language Note
In languages with **sum types + pattern matching** (Rust, Scala, Haskell, Swift, TypeScript discriminated unions), state machines are often modeled as an enum of state variants matched in a transition function — type-checked exhaustively (the compiler forces you to handle every state). This is frequently *safer* than the OO State pattern. Dedicated state-machine libraries (XState in JS) formalize it further. The OO class-per-state version is the Java/C++/C# idiom.

---

## 2C.7 — Chain of Responsibility

### The Force

A request may be handled by **one of several handlers**, and you don't want the sender coupled to a specific receiver — you want to pass the request along a line of potential handlers until one handles it (or it's processed by several in sequence). Hardcoding "try handler A, else B, else C" in the sender couples it to all handlers and their order (OCP violation; rigid). Chain of Responsibility resolves "decouple sender from receiver by giving multiple objects a chance to handle the request, passing it along a chain until handled."

### Intent

> Avoid coupling the sender of a request to its receiver by giving more than one object a chance to handle the request. Chain the receiving objects and pass the request along the chain until an object handles it.

### Motivation Story

An HTTP request enters a web server and must pass through: authentication → rate-limiting → input validation → logging → the actual handler. Each is a separate concern. You don't want one giant function doing all five with nested `if`s. Instead, each is a **handler** with a reference to the *next* handler. A request enters the first; each handler either processes-and-stops, processes-and-passes-on, or passes-on-untouched. Auth checks the token (rejects or passes on); rate-limiter checks quota (rejects or passes); and so forth. This is exactly **middleware** — the most ubiquitous real-world form of this pattern (Express, ASP.NET, Django, every web framework).

### Structure

```
┌──────────┐   ┌──────────────────┐
│  Client  │──►│   «Handler»      │
└──────────┘   │ - next: Handler  │
               │ + handle(request)│
               └────────▲─────────┘
                        │
        ┌───────────────┼───────────────┐
   ┌────┴─────┐   ┌─────┴─────┐   ┌──────┴────┐
   │ HandlerA │──►│ HandlerB  │──►│ HandlerC  │──► null
   │ (Auth)   │   │(RateLimit)│   │(Validate) │
   │ handle(){│   │           │   │           │
   │  if mine │   │           │   │           │
   │   process│   │           │   │           │
   │  else    │   │           │   │           │
   │   next.  │   │           │   │           │
   │   handle}│   │           │   │           │
   └──────────┘   └───────────┘   └───────────┘

  Request flows left→right until someone handles it (or chain ends).
```

### Internal Mechanics

Each handler holds a reference to the *next* handler (a linked list). Its `handle()` decides: handle and stop, handle and forward, or forward untouched. Two chain semantics:
- **"First match wins"** (pure CoR): the first capable handler processes it and the chain stops (e.g., exception handlers, support-ticket escalation).
- **"Pipeline"** (every handler runs in sequence): each does its part and always forwards (e.g., middleware where auth *and* logging *and* validation all run). Modern middleware is usually this pipeline form, often with the ability to short-circuit (auth failure stops the chain).

A handler *can* be unable to handle and *must* forward — and if the chain ends with no handler, the request is unhandled (a case you must design for — silent dropping is a hazard).

### Advantages
- Decouples sender from the specific handler (sender just drops the request into the chain).
- OCP: add/remove/reorder handlers without touching others or the sender.
- Single Responsibility: each handler does one thing.
- Dynamic chains: build/reconfigure the pipeline at runtime.

### Disadvantages
- **No handling guarantee:** a request may fall off the end unhandled (silent failure if not designed for).
- Debugging is hard: control flow threads through many handlers; "where did my request die?" is non-obvious.
- Performance: a long chain means many hops per request.
- Order matters and is implicit (auth-before-logging vs. after — getting it wrong is a security bug).

### Tradeoffs
Chain of Responsibility trades a guarantee of handling and easy traceability for flexible, decoupled, reorderable processing pipelines. The trade is excellent for cross-cutting concerns (the middleware use case) where composability matters; the main risks are unhandled-request silent failures and order-dependent bugs.

### When NOT to Use
When exactly one known handler always handles it (call it directly). When every request *must* be handled and the chain's "might fall off the end" semantics are dangerous. When the processing order is critical and fragile.

### Language Note
**Middleware is Chain of Responsibility**, and it's everywhere: Express `app.use(mw)`, Django/Rails middleware, ASP.NET pipeline, gRPC interceptors, Redux middleware. In function-first style, handlers are often *functions* taking `(request, next)` rather than handler objects — the closure captures `next`. The OO linked-handler-objects version is the classic GoF shape; the functional middleware version dominates modern web frameworks.

---

## 2C.8 — Mediator

### The Force

A set of objects must **communicate in complex, many-to-many ways**, and if each object references every other directly, you get a tangled web: N objects with up to N² connections, each tightly coupled to many others, impossible to reuse in isolation or modify without ripple effects. Mediator resolves "centralize the many-to-many communication in one mediator object, so each component talks only to the mediator, not to each other — converting a tangled mesh into a clean hub-and-spoke."

### Intent

> Define an object that encapsulates how a set of objects interact. Mediator promotes loose coupling by keeping objects from referring to each other explicitly, and lets you vary their interaction independently.

### Motivation Story

A dialog form has fields: a checkbox that enables/disables a text field, a dropdown that changes which buttons show, a submit button that activates only when required fields are filled. If each widget directly manipulates the others (`checkbox.onChange → textField.enable(); submitButton.update()`), every widget knows about many others — adding a field means touching several widgets. Instead, a `DialogMediator` knows all the widgets and *all the interaction rules*. Each widget just notifies the mediator "I changed"; the mediator decides what else to update. Widgets become *dumb and reusable*; all the coordination logic lives in one place you can read and change.

### Structure

```
              ┌──────────────────┐
   ┌─────────►│   «Mediator»     │◄──────────┐
   │          │ + notify(sender, │           │
   │          │         event)   │           │
   │          └────────▲─────────┘           │
   │ notifies          │ coordinates         │ notifies
   │          ┌────────┴─────────┐           │
┌──┴───────┐  │ ConcreteMediator │  ┌─────────┴──┐
│Component A│  │ knows all comps, │  │ Component C │
│(checkbox)│  │ holds all rules  │  │ (button)    │
└──────────┘  └──────────────────┘  └────────────┘
       ┌────────────┴──────────────┐
       │   Component B (textfield)  │
       └───────────────────────────┘

  BEFORE (mesh):  A↔B, A↔C, B↔C, ... up to N² links
  AFTER  (hub):   A→M, B→M, C→M ... N links, mediator holds the logic
```

### Internal Mechanics

Each component holds *one* reference — to the mediator — and calls `mediator.notify(this, event)` when something happens. The mediator contains the **interaction logic**: a (often large) method that, given who-sent-what, decides which components to update and how. Components are decoupled from each other entirely; they only know the mediator interface. The complexity doesn't *disappear* — it *moves* from the scattered web into the mediator, where it's at least centralized and visible.

### Mediator vs. Observer vs. Facade
- **Observer:** *one-to-many*, *unidirectional* broadcast (subject → many observers), observers don't coordinate. **Mediator:** *many-to-many*, *bidirectional* coordination (the mediator orchestrates mutual interactions). (Mediator is sometimes *implemented with* Observer — components observe the mediator.)
- **Facade:** simplifies access to a subsystem, *unidirectional*, subsystem doesn't know the facade. **Mediator:** components *know* the mediator and actively talk *to* it; it's about *peer coordination*, not access simplification.

### Advantages
- Eliminates the N² coupling mesh → loose coupling (DIP).
- Centralizes interaction logic in one readable, changeable place.
- Components become reusable (they don't know about each other).
- OCP: change interactions by changing the mediator, not the components.

### Disadvantages
- **The mediator can become a god object** — a bloated, low-cohesion monster that knows everything and changes for every reason (the central risk; you've concentrated complexity, and if it grows unchecked, you've just moved the mess into one giant class).
- A single point of failure/complexity.
- Can hide the actual interaction flow inside one big method.

### Tradeoffs
Mediator trades the *distribution* of coupling (a tangled mesh) for the *concentration* of it (one coordinator). Worth it when many objects interact in complex ways and the mesh is genuinely unmanageable. The constant danger is the mediator bloating into a god object — discipline (splitting mediators, keeping them focused) is required. If interactions are simple or few, the mediator adds a needless middleman.

### When NOT to Use
Few objects with simple interactions (the mesh isn't a problem yet). When it would become an unfocused god object. When direct communication is clearer and the coupling is minimal.

### Language Note
Air traffic control (planes talk to the tower, not each other) is the canonical analogy. In practice: UI frameworks coordinating widgets, message brokers/event buses (a generalized mediator), chat-room servers, Redux-style stores (a central coordinator for state changes). The pattern is conceptual; no special language machinery — just "route everything through a coordinator."

---

## 2C.9 — Memento

### The Force

You need to **capture an object's internal state so it can be restored later** (undo, checkpoints, snapshots, transactions) — *without* exposing or violating the object's encapsulation. The naive approach — making the object's internals public so an outside "history" can read and restore them — destroys encapsulation (Part 1's cardinal sin): now anyone can corrupt the state, and the object can no longer guarantee its invariants. Memento resolves "let an object externalize a snapshot of its state into an opaque token that only the original object can read, preserving encapsulation while enabling save/restore."

### Intent

> Without violating encapsulation, capture and externalize an object's internal state so that the object can be restored to this state later.

### Motivation Story

A text editor needs undo. To undo, you must restore the document's prior state (text, cursor, selection, formatting). You could expose all internals so an undo-manager reads and rewrites them — but then the document can't protect its invariants, and the undo-manager is welded to the document's internal structure. Memento instead: the document creates a `Memento` — an opaque snapshot of its own state. The undo-manager (`Caretaker`) just *holds a stack of mementos* without being able to *read inside* them. To undo, it hands a memento back to the document, which restores itself from it. The caretaker stores; only the document reads/writes the contents. Encapsulation intact.

### Structure

```
┌──────────────────┐  creates   ┌──────────────────┐
│   Originator     │ ─────────► │     Memento      │
│ - state          │            │ - state (opaque  │
│ + save(): Memento│            │   to outsiders)  │
│ + restore(m)─────┼─reads──────┤ (only Originator │
│                  │◄───────────┤  can access      │
└──────────────────┘            │  contents)       │
                                └────────▲─────────┘
                                         │ stores (can't read)
                                ┌────────┴─────────┐
                                │    Caretaker     │
                                │ - history: Stack │  holds mementos,
                                │ + undo()         │  never inspects them
                                └──────────────────┘
```

### Internal Mechanics & The Encapsulation Trick

The crux is **controlled access:** the `Memento` exposes a *wide interface* to the `Originator` (full state read/write) but a *narrow interface* to everyone else (the `Caretaker` can only hold and pass it, not read it). Languages enforce this differently:
- **Nested classes** (Java/C++): `Memento` is an inner class of `Originator`, so only the originator accesses its private state; outsiders see only an opaque marker interface.
- **Friend classes** (C++): explicit friendship.
- **Convention/closures** (looser languages): the memento's internals are simply not exposed.

The caretaker manages *when* to save/restore (the history/policy) but never *what's inside* (the originator owns that). This separation — **caretaker manages timeline, originator manages content** — is the whole point.

### Memento + Command (the undo system)
Memento and Command frequently collaborate to build undo: a `Command`'s `execute()` first asks the originator for a memento (snapshot), then acts; `undo()` restores from that memento. Command captures *the action*; Memento captures *the state to revert to*. Together they form the standard undo/redo architecture.

### Advantages
- Save/restore state *without breaking encapsulation* (the defining benefit).
- Originator's internals stay private; caretaker is decoupled from state structure.
- Clean foundation for undo/redo, checkpoints, transactions, snapshots.

### Disadvantages
- **Memory cost:** snapshots can be large; frequent snapshots of big state consume lots of memory (mitigations: incremental/delta mementos storing only changes, or command-based inverse operations instead).
- The caretaker must manage memento lifecycle (when to discard old ones).
- In languages without good access control, "opaque to caretaker" is convention, not enforcement.

### Tradeoffs
Memento trades memory (storing snapshots) for encapsulation-preserving state restoration. Worth it when you need undo/snapshots *and* care about encapsulation. The memory cost drives the choice between full snapshots (simple, heavy) and deltas/inverse-commands (complex, light). At scale, you snapshot strategically (checkpoints) rather than every change.

### When NOT to Use
When state is trivial to reconstruct or you don't need restoration. When inverse operations (Command undo) are cheaper than snapshots. When the memory cost of snapshots is prohibitive and deltas are impractical.

### Language Note
Snapshots map to **immutable data + structural sharing** in functional languages: keep references to old immutable versions (cheap, since unchanged parts are shared — persistent data structures). Git is a giant memento system (each commit is a snapshot). Database savepoints, VM snapshots, and `useState`/undo libraries all embody it. Serialization (saving an object's state to restore later) is memento-like. The "opaque token" formality matters most in strict-encapsulation OO languages.

---

## 2C.10 — Visitor

### The Force — The Hardest GoF Pattern

You have a **stable object structure** (a fixed set of element types — e.g., AST nodes, shapes, file-system entries) and you need to **add many new, unrelated operations** over them *without modifying the element classes each time*. Putting every operation as a method on each element class means: (a) each element class accumulates dozens of unrelated methods (low cohesion), and (b) adding an operation edits *every* element class (OCP violation). Visitor resolves "represent an operation as a separate object (a 'visitor') that can be applied across the element structure, so new operations are new visitor classes — no element class changes."

### Intent

> Represent an operation to be performed on the elements of an object structure. Visitor lets you define a new operation without changing the classes of the elements on which it operates.

### Motivation Story

A compiler has an AST with node types: `NumberNode`, `AddNode`, `MultiplyNode`, `VariableNode`. You need many operations over it: *evaluate*, *pretty-print*, *type-check*, *optimize*, *generate code*. If each operation is a method on every node type, every node class has five+ unrelated methods, and adding "compute complexity" edits all of them. Instead: each operation is a `Visitor` (`EvaluateVisitor`, `PrintVisitor`, `TypeCheckVisitor`), each with a `visit(NumberNode)`, `visit(AddNode)`, etc. Nodes have one `accept(visitor)` method that calls back `visitor.visit(this)`. Add a new operation = add a new visitor class; *no node class changes*. The operations are pulled *out* of the elements into cohesive visitor objects.

### Structure & The Double-Dispatch Mechanism

```
┌──────────────────┐            ┌──────────────────┐
│   «Element»      │            │   «Visitor»      │
│ + accept(visitor)│            │ + visit(ElemA)   │
└────────▲─────────┘            │ + visit(ElemB)   │
         │                      │ + visit(ElemC)   │
  ┌──────┴──────┐               └────────▲─────────┘
┌─┴────────┐ ┌──┴───────┐         ┌──────┴───────┐
│ ElementA │ │ ElementB │   ┌─────┴──────┐ ┌──────┴─────┐
│ accept(v)│ │ accept(v)│   │ VisitorX   │ │ VisitorY   │
│ {v.visit │ │ {v.visit │   │(Evaluate)  │ │(PrettyPrint)│
│  (this)} │ │  (this)} │   │visit(A){..}│ │visit(A){..}│
└──────────┘ └──────────┘   │visit(B){..}│ │visit(B){..}│
                            └────────────┘ └────────────┘
```

**Double dispatch is the heart of Visitor — understand this precisely.** Most OO languages have *single dispatch*: the method that runs depends on *one* type (the receiver — `obj.method()` dispatches on `obj`'s type). But Visitor needs the operation to depend on *two* types: the *element type* AND the *visitor type*. The trick:
1. Client calls `element.accept(visitor)` → dispatches on **element's** type (single dispatch #1), landing in the right element's `accept`.
2. Inside that `accept`, it calls `visitor.visit(this)` → `this` is now *statically known* to be the concrete element type, so this dispatches on **visitor's** type (single dispatch #2) *and* selects the correct overloaded `visit(ConcreteElement)`.
Two single dispatches compose into **double dispatch** — the correct `(element type × visitor type)` method runs. This indirection is *why Visitor feels so convoluted*: it's manually simulating a feature (multiple dispatch) most OO languages lack.

### The Defining Tradeoff — The "Expression Problem"
Visitor embodies a fundamental tension. There are two axes of extension:
- **Adding new operations:** Visitor makes this *easy* (new visitor class, no element changes).
- **Adding new element types:** Visitor makes this *hard* (every existing visitor must add a `visit(NewElement)` method — OCP violation on the *element* axis).

This is the **Expression Problem**: OO classes make adding *types* easy but adding *operations* hard (put a method on each class); Visitor *inverts* this — easy to add operations, hard to add types. **You choose which axis to make cheap.** Use Visitor when *operations* change more than *element types* (the structure is stable, operations multiply — like a settled AST). Avoid it when *element types* churn (you'll edit every visitor constantly).

### Advantages
- Add new operations without touching element classes (OCP on the operation axis).
- Related operation logic is gathered in one cohesive visitor (vs. scattered across elements).
- Visitors can accumulate state while traversing (e.g., a running total).

### Disadvantages
- **Adding a new element type breaks every visitor** (the Expression Problem cost — the dominant downside).
- **Double-dispatch indirection is hard to understand** — the most conceptually difficult GoF pattern.
- Visitors often need access to element internals, which can break encapsulation (elements expose state to visitors).
- Verbose: every element needs `accept`; every visitor needs a `visit` per element type.

### Tradeoffs
Visitor trades the ability to easily add element *types* (and a heavy dose of indirection) for the ability to easily add *operations* over a stable structure. It's a *specialized* pattern — powerful exactly when you have a fixed type hierarchy and proliferating operations (compilers, interpreters, document processors), and actively harmful when types are unstable.

### When NOT to Use
When element types change frequently (you'll suffer the Expression Problem constantly). When there are few operations (just put methods on the elements). When the double-dispatch complexity isn't justified by the operation count.

### Language Note
Languages with **pattern matching on sum types** (Rust `match`, Scala, Haskell, F#, Swift, modern Java `switch` patterns) make Visitor *largely obsolete*: define operations as functions that `match` on the element type — adding an operation is a new function, and the compiler *exhaustively checks* you've handled every type. This is cleaner than double dispatch and gives the same "operations separate from data" benefit. Languages with **true multiple dispatch** (Julia, Clojure multimethods, CLOS) make Visitor unnecessary outright — you just define methods dispatching on multiple argument types. Visitor is most needed in single-dispatch OO languages *without* pattern matching (classic Java/C++/C#). It's the starkest example of a GoF pattern being a workaround for a missing language feature.

---

## 2C.11 — Interpreter

### The Force

You have a **simple, well-defined language or grammar** (arithmetic expressions, search queries, regex, business rules, configuration DSLs) and need to *evaluate sentences* in it. Hardcoding the evaluation logic ad hoc is unmaintainable as the grammar grows; you want a structured way to represent grammar rules as classes and interpret them recursively. Interpreter resolves "represent each grammar rule as a class, build a syntax tree of these classes for a given sentence, and interpret it by recursively evaluating the tree."

### Intent

> Given a language, define a representation for its grammar along with an interpreter that uses the representation to interpret sentences in the language.

### Motivation Story

You need to evaluate boolean rule expressions like `(age > 18) AND (country == "US" OR vip == true)` for a rules engine. Each grammar construct becomes a class: `GreaterThan`, `Equals`, `And`, `Or`, `Variable`, `Constant` — all implementing an `Expression` interface with `interpret(context)`. The expression `A AND B` becomes an `And(exprA, exprB)` object; `interpret()` recursively interprets both sides and combines. The whole rule is a *tree* of expression objects (an AST); evaluating = calling `interpret()` on the root, which recurses down. New operators = new expression classes.

### Structure

```
┌──────────────────────┐
│    «Expression»      │◄───────────────┐
│ + interpret(context) │                │ composites hold
└──────────▲───────────┘                │ sub-expressions
           │                            │ (recursion!)
   ┌───────┴────────────┐               │
┌──┴──────────────┐ ┌────┴────────────┐ │
│ TerminalExpr    │ │ NonTerminalExpr │─┘
│ (Variable,      │ │ (And, Or, Add)  │
│  Constant)      │ │ - left, right   │
│ interpret(){    │ │ interpret(){    │
│  return value}  │ │  left.interpret()│
│ (leaf, base case│ │  OP right.       │
│                 │ │  interpret() }   │
└─────────────────┘ └─────────────────┘

  Tree for "A AND (B OR C)":
            And
           /   \
          A     Or
               /  \
              B    C
```

### Internal Mechanics

The grammar maps to a class hierarchy: **terminal expressions** (leaves — literals, variables; base cases of the recursion) and **non-terminal expressions** (operators — composites holding sub-expressions). A sentence is parsed into an **abstract syntax tree** of these objects. `interpret(context)` is called on the root and recurses: non-terminals interpret their children and combine; terminals return their value (often looked up in the `context`, which holds variable bindings). **Note this is Composite applied to grammar** — the tree of expressions is a Composite structure, interpreted recursively. (Parsing the text *into* the tree is a separate concern — Interpreter covers evaluation, not parsing; real systems use a separate parser.)

### Advantages
- Grammar rules map cleanly to classes; easy to extend the grammar (new rule = new class, OCP).
- Each rule's logic is isolated and individually testable.
- Natural fit for simple, stable, well-defined languages.

### Disadvantages
- **Scales terribly:** complex grammars produce a huge number of classes (one per rule) — quickly unwieldy. For anything beyond simple grammars, real parser generators (ANTLR, yacc/bison, parser combinators) are vastly better.
- Performance: tree-walking interpretation is slow (vs. compilation); deep recursion.
- Doesn't address parsing (you need a separate parser to build the tree).
- Rarely the right tool — the *least used* GoF pattern in practice.

### Tradeoffs
Interpreter trades scalability for a clean OO representation of *simple* grammars. It's worth it *only* for small, stable DSLs where the grammar is genuinely simple. For real languages, use dedicated parsing tools — Interpreter is mostly of educational value (it teaches how ASTs and recursive evaluation work) and appears rarely in production beyond tiny embedded rule/query languages.

### When NOT to Use
Anything beyond a simple grammar (use ANTLR, parser combinators, etc.). When performance matters (compile or use optimized interpreters). It's the rare pattern whose default answer is "you probably want something else."

### Language Note
Functional languages express interpreters elegantly with **sum types + pattern matching** (an `Expr` ADT + a recursive `eval` function that matches on each variant) — this is the *standard* way to write interpreters in Haskell/Rust/Scala/ML, far cleaner than the OO class hierarchy. The OO Interpreter pattern is essentially the awkward translation of "ADT + recursive eval" into class-based languages. Internal DSLs in expressive languages (Ruby, Kotlin) and tools like ANTLR have largely superseded hand-rolled Interpreter.

---

## 2C — Behavioral Patterns: Comparison & Synthesis

```
PATTERN          GOVERNS…                        KEY MECHANISM          DISSOLVES INTO (FP)
─────────────────────────────────────────────────────────────────────────────────────────
Strategy         swappable algorithm              inject behavior obj    a function arg
Observer         1→many change notification       subscriber list        event streams/signals
Command          request as object (undo/queue)   reified action+receiver a closure
Iterator         traverse without exposing guts   external cursor         for-x-in / generators
Template Method  fixed skeleton, varying steps    inheritance + override  higher-order function
State            behavior per state machine       delegate to state obj   sum type + match
Chain of Resp.   request down a handler line       linked handlers         middleware functions
Mediator         many↔many coordination           central hub             event bus / store
Memento          snapshot & restore (encapsulated) opaque state token     immutable snapshots
Visitor          new ops on stable structure       double dispatch        pattern matching
Interpreter      evaluate a grammar               AST of rule classes     ADT + recursive eval
```

**The confusable clusters, resolved:**
```
Behavior-swapping trio (identical skeleton, different driver):
  Strategy  → CLIENT picks an interchangeable algorithm
  State     → OBJECT transitions itself; states know each other
  Template  → SUBCLASS overrides steps in a fixed inherited skeleton

Capture-as-object pair (collaborate for undo):
  Command   → captures the ACTION (execute/undo)
  Memento   → captures the STATE to revert to

Coordination trio (who knows whom):
  Observer  → 1→many, one-directional broadcast, observers ignorant of each other
  Mediator  → many↔many, components know the mediator, it holds the rules
  Chain     → linear pass-along until handled
```

**The meta-lesson for behavioral patterns:** They are fundamentally about **inverting and decoupling control flow** — instead of object A directly commanding object B, behavioral patterns insert an abstraction (a strategy, an observer list, a command object, a handler chain, a mediator) so that *who does what, and when, and in response to what* can vary without rewiring the participants. This is Part 1's "loose coupling" applied to *runtime behavior* rather than static structure. And this family is where language-relativity is most dramatic: **seven of the eleven largely dissolve into language features** (functions, closures, generators, pattern matching, event systems) in modern multi-paradigm languages. The expert recognizes the *coordination force* and reaches for the lightest local mechanism — which is increasingly *not* a class hierarchy.