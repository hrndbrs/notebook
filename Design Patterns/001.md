# PART 1 — FOUNDATIONS

## Design Patterns: From Absolute Beginner to Staff Engineer

---

## 1.0 Before We Begin: What This Part Covers

Part 1 does not teach a single design pattern. That is intentional. The most common failure mode in learning design patterns is memorizing twenty-three named recipes without understanding the soil they grow in. You end up a person with a hammer, seeing nails everywhere, applying Singleton to problems that never needed it.

Instead, this Part builds the *prerequisite mental machinery*: what a pattern actually is, where the concept came from, the object-oriented and design principles that patterns are built on top of, the vocabulary professionals use, and the intellectual traps beginners fall into. By the end, you will understand *why patterns exist as a category of knowledge* before you learn any individual one.

We will use code in multiple languages (mostly Java, Go, Python, and TypeScript) because patterns manifest differently across language paradigms — a point most textbooks dangerously ignore.

---

## 1.1 What Is a Design Pattern?

### Definition

A **design pattern** is a *named, reusable solution to a commonly recurring problem within a given context in software design.* It is not code. It is not a library. It is a *description of a solution structure* — a template for how to arrange classes, objects, and their interactions — that you adapt to your specific situation.

Read that definition again, because four words in it carry enormous weight:

- **Named** — it has a shared label ("Observer", "Factory") so engineers can communicate complex ideas in one word.
- **Reusable** — the same shape solves the problem across many different projects.
- **Recurring** — it addresses a problem that shows up again and again, not a one-off.
- **Context** — it is only appropriate *within a particular set of conditions*. Outside that context, it is wrong.

### Mental Model

Think of design patterns the way an architect (the building kind) thinks of structural solutions. An architect doesn't reinvent "how to span a gap" every time — they reach for an *arch*, a *truss*, a *cantilever*, or a *suspension* system. Each is a named solution with known properties, known load characteristics, known costs, and known contexts where it shines or fails. A truss is wonderful for a bridge and absurd for a doorway.

A design pattern is the software equivalent: a *known structural arrangement* with known forces it resolves and known tradeoffs it imposes.

Here's a sharper mental model: **a design pattern is a compressed conversation.** When a senior engineer says "let's make this a Strategy," they have, in three words, communicated:

```
"Let's extract this varying behavior into its own interface,
 inject an implementation at runtime, so we can swap algorithms
 without touching the calling code, and so each algorithm is
 independently testable and open for extension."
```

That entire paragraph compresses into one word because both engineers share the pattern vocabulary. **This compression is the single most underrated benefit of patterns, and we'll return to it repeatedly.**

### Intuition

Why does this concept even need to exist? Because in software, the *same shapes of problems* recur constantly:

- "I need to create objects but I don't want the calling code to know which concrete class it's getting." → recurring creation problem.
- "I need many parts of my system to react when one thing changes." → recurring notification problem.
- "I need to add behavior to objects without rewriting them." → recurring extension problem.

Before patterns were cataloged, every engineer solved these from scratch, badly, in incompatible ways, with no shared name to discuss them. Patterns are humanity's accumulated answer key to the recurring exam of software structure.

### A Critical Clarification: Pattern vs. Algorithm vs. Library

Beginners conflate these. Let's separate them precisely:

| Concept | What it is | Granularity | Example |
|---------|-----------|-------------|---------|
| **Algorithm** | A precise sequence of computational steps to produce an output | Implementation-level | Quicksort, Dijkstra |
| **Data structure** | A way of organizing data in memory | Implementation-level | Hash table, B-tree |
| **Design pattern** | A structural template for arranging objects/responsibilities | Design-level | Observer, Adapter |
| **Architectural pattern** | A structural template at the whole-system level | System-level | MVC, Microservices, Event Sourcing |
| **Library / Framework** | Actual reusable *code* you call or that calls you | Concrete code | React, Spring, Express |

The key distinction: **an algorithm tells you the steps to compute a result; a design pattern tells you how to organize the code that does the computing so the system stays flexible, understandable, and maintainable.** A pattern is about *structure and communication between parts*, not about computing a value.

A design pattern is *language-agnostic structure*; a library is *language-specific code*. You can't `npm install` the Observer pattern — but `EventEmitter` in Node.js is an *implementation* of it.

### Common Misconceptions (Read These Carefully)

**Misconception 1: "More patterns = better code."**
False, and dangerously so. Patterns add *indirection*. Indirection has a cost: more classes, more files, more cognitive hops to follow the logic. A pattern is justified only when the flexibility it buys exceeds the complexity it costs. Junior engineers over-apply patterns; senior engineers remove them.

**Misconception 2: "Patterns are language features I must use."**
False. Patterns are *workarounds for missing language features as much as they are timeless wisdom.* In a language with first-class functions, the "Strategy pattern" is often just... passing a function. We'll dedicate a whole section (1.9) to this profound point.

**Misconception 3: "There are exactly 23 design patterns."**
False. The famous "Gang of Four" book cataloged 23, but that was one book in 1994 describing patterns *they observed*. There are dozens more (Repository, Unit of Work, Dependency Injection, Null Object, Specification, CQRS...) and new ones emerge as software evolves. The number 23 is historical, not fundamental.

**Misconception 4: "Patterns are about object-oriented programming only."**
Mostly historical but increasingly false. Functional programming has its own patterns (monads, functors, lenses). Even the GoF patterns have functional equivalents. Patterns are about *recurring structure*, which transcends paradigm.

**Misconception 5: "If I memorize the patterns, I understand design."**
The most harmful misconception. Memorizing the 23 patterns is like memorizing chess openings without understanding *why* they work. Mastery is knowing the *forces* (the principles in section 1.6) that *generate* patterns — so you can recognize when to apply, adapt, combine, or *invent* one.

### Real-World Examples (Concrete, Not Toy)

- **Java's `java.util.Collections.unmodifiableList()`** returns a *Proxy/Decorator* that wraps a list and throws on mutation.
- **React's Context API + hooks** is essentially the *Observer* and *Dependency Injection* patterns implemented at the framework level.
- **Stripe's API client libraries** use the *Facade* pattern — one clean `stripe.charges.create()` call hides dozens of HTTP requests, retries, idempotency keys, and serialization.
- **Database connection pools** (HikariCP, pgbouncer) are *Object Pool* patterns — expensive-to-create connections are reused rather than recreated.
- **Kubernetes controllers** implement the *Observer/Reconciliation* pattern — they watch desired state and react to converge actual state toward it.

---

## 1.2 Historical Background: Where Did This Come From?

You cannot understand patterns without understanding *why the field invented them*. This is not trivia — the history explains the philosophy.

### The Origin: A Building Architect, Not a Programmer

The concept of a "pattern language" did not originate in software. It came from **Christopher Alexander**, a building architect, in his 1977 book *A Pattern Language* and 1979's *The Timeless Way of Building*.

Alexander was wrestling with a question: *why do some towns, buildings, and rooms feel alive and humane, while others feel dead and sterile?* He observed that good design recurred in *patterns* — like "Light on Two Sides of Every Room" (rooms lit from two directions feel better than rooms lit from one). He defined a pattern as:

> "A solution to a problem in a context."

And crucially, he described patterns as having a **rule of thumb structure**: a problem statement, the forces (competing pressures) at play, and a solution that resolves those forces. He believed patterns could be *composed into languages* — vocabularies that let ordinary people, not just expert architects, build humane spaces.

**Why this matters for software:** Alexander's core insight — that recurring problems have recurring good solutions, and that naming them creates a shared design vocabulary — is the entire intellectual foundation of software patterns. The emphasis on *forces* (competing pressures that the solution must balance) is something most software developers tragically forget. A pattern isn't just a shape; it's a *resolution of tensions*.

### The Bridge to Software: OOPSLA and the Hillside Group

In the late 1980s, software engineers — notably **Kent Beck** and **Ward Cunningham** — began applying Alexander's ideas to software. Cunningham later invented the wiki specifically to host a repository of software patterns (the original WikiWikiWeb, the ancestor of Wikipedia, was born to document patterns).

### The Defining Moment: The Gang of Four (1994)

In 1994, four authors — **Erich Gamma, Richard Helm, Ralph Johnson, and John Vlissides** — published:

> *Design Patterns: Elements of Reusable Object-Oriented Software*

This book, universally abbreviated **GoF** (Gang of Four), is the single most influential software-engineering book of its era. It cataloged **23 patterns** observed in real C++ and Smalltalk systems, organized into three categories (Creational, Structural, Behavioral — we'll define these in 1.8).

What GoF did brilliantly:
- Gave each pattern a *memorable name*, creating shared vocabulary.
- Used a rigorous template for each (Intent, Motivation, Structure, Participants, Consequences, Implementation, Known Uses).
- Grounded each in *real* systems, not invented examples.

What GoF accidentally caused (the dark side):
- A generation of engineers treated the 23 as a *checklist* to apply, rather than a *sample* of a way of thinking.
- The patterns were shaped by the *limitations of C++ and Smalltalk in 1994*. Many exist to work around the absence of features (first-class functions, reflection, mixins) that modern languages have. This created decades of "pattern cargo-culting" — applying heavyweight class hierarchies in languages where a closure would do.

### The Backlash and Maturation (2000s–present)

By the mid-2000s, thoughtful engineers pushed back. **Peter Norvig** famously showed that 16 of the 23 GoF patterns were "invisible or simpler" in dynamic languages like Lisp. **Paul Graham** argued patterns were often a sign of *missing language abstraction* — "a sign that I'm generating by hand what a more powerful language would generate for me." This is sometimes called the **"patterns are a smell" critique.**

The mature, modern view — the one a staff engineer holds — synthesizes both camps:

> Patterns are *neither sacred recipes nor code smells.* They are a **shared vocabulary** and a **catalog of force-resolutions**. Use them to communicate and to recognize structure. Apply them only when the underlying *forces* are present, and prefer the lightest mechanism your language offers to resolve those forces.

This synthesized view is the lens through which this entire course teaches. Hold onto it.

### Why It Exists (Summarized)

Design patterns as a discipline exist because:
1. **Recurring problems** waste effort if re-solved from scratch.
2. **Communication** between engineers needs a high-bandwidth shared vocabulary.
3. **Accumulated wisdom** about tradeoffs needs a place to live and be taught.
4. **Quality** improves when proven structures replace ad-hoc invention.

---

## 1.3 Prerequisite: Abstraction (The Mother of All Concepts)

You cannot understand patterns without deeply understanding **abstraction**, because *every pattern is an exercise in choosing what to hide and what to expose.*

### Definition

**Abstraction** is the deliberate suppression of detail — representing something by its *essential characteristics and behavior* while hiding its *implementation*. It is the act of drawing a boundary and saying: "On this side, you only need to know *what* it does; *how* it does it is hidden."

### Mental Model

A **steering wheel** is an abstraction over the steering system. You turn it left, the car goes left. You don't know (or need to know) whether it's a rack-and-pinion mechanism, hydraulic power steering, or drive-by-wire electronics. The wheel exposes an *interface* (turn to steer) and hides the *implementation* (the mechanical or electronic details).

The power of this: the implementation can change entirely — gas car to electric, hydraulic to electric power steering — and *your knowledge of how to drive doesn't change at all.* That decoupling of "what" from "how" is the source of all flexibility in software.

### Intuition

The human mind can hold only ~7 things in working memory. Software systems have millions of details. The *only* way humans build large software is by abstraction: hiding details behind boundaries so that at any level, we deal with a handful of concepts, not millions.

Every layer of computing is abstraction stacked on abstraction:

```
Your function call:        order.process()
   ↓ hides
Business logic:            validate, charge, ship
   ↓ hides
Library calls:             stripe.charge(), db.save()
   ↓ hides
HTTP / SQL protocols:      POST /charges, INSERT INTO...
   ↓ hides
TCP/IP packets:            SYN, ACK, payloads
   ↓ hides
Ethernet frames / signals: electrical voltages
   ↓ hides
Physics:                   electrons in copper
```

Each arrow is a boundary where detail is suppressed. **Design patterns are tools for drawing these boundaries well at the object-design level.**

### Two Kinds of Abstraction You Must Distinguish

1. **Data abstraction** — hiding *how data is stored*. A `Stack` exposes `push`/`pop`; whether it's backed by an array or linked list is hidden.
2. **Control/behavioral abstraction** — hiding *how something is done*. A `sort()` function hides whether it's quicksort or mergesort.

Patterns use both, but most GoF patterns are fundamentally about **behavioral abstraction** — hiding *which* behavior runs, *which* object is created, or *how* objects collaborate.

### Common Misconceptions

- **"Abstraction means making things vague."** No. Good abstraction is *precise about what it promises* and *silent about how it delivers.* Vagueness is the opposite of good abstraction.
- **"More abstraction is always better."** No. Each abstraction layer has a cost: indirection, cognitive load, and sometimes performance. *Premature or excessive abstraction* (the "AbstractFactoryFactoryBean" joke about Java) is a real disease. The art is *abstracting at the right level for the change you expect.*

### The Single Most Important Idea in This Entire Course

> **You abstract along the axis where you expect change.**

If payment methods will change, abstract over payment. If storage will change, abstract over storage. If nothing will change, don't abstract — you'll pay complexity for flexibility you never use. *Every design pattern is a pre-built abstraction along a specific axis of change.* Choosing a pattern = predicting where your system will need to flex. Get the prediction right, and patterns are magic. Get it wrong, and you've built rigid machinery to flex an axis that never moves while the axis that *does* move is cemented shut. **Hold this idea above all others.**

---

## 1.4 Prerequisite: Encapsulation, Coupling, and Cohesion

These three concepts are the *vocabulary of structural quality*. Patterns are tools to improve them. You must know them cold.

### Encapsulation

**Definition:** Bundling data with the methods that operate on that data, and *restricting direct access* to the internal state, so the object controls its own invariants.

**Mental model:** A vending machine. You can press buttons (the public interface). You cannot reach inside to take the money or sodas directly (the internals are hidden). The machine guarantees its own rules: no soda without payment. If you could reach in, those guarantees would collapse.

**Why it exists:** Without encapsulation, any code anywhere can change any data, so *no object can guarantee its own correctness.* Bugs become impossible to localize because the cause of a bad value could be anywhere. Encapsulation creates **invariant boundaries** — places where you can *prove* a rule holds.

```java
// WITHOUT encapsulation — anyone can corrupt state
class BankAccount {
    public double balance; // anyone can set this to -9999
}

// WITH encapsulation — the object guards its invariant
class BankAccount {
    private double balance; // hidden

    public void withdraw(double amount) {
        if (amount > balance) {
            throw new InsufficientFundsException();
        }
        balance -= amount; // invariant: balance never goes negative
    }
}
```

The second version can *guarantee* balance is never negative because every path that touches `balance` goes through guarded methods. That guarantee is *the* point.

### Coupling

**Definition:** The degree to which one module *depends on* the details of another. The more one component must know about another's internals to function, the *tighter* the coupling.

**Mental model:** Coupling is like *how much two people must coordinate to do their jobs.* If changing how you do your job forces your coworker to change how they do theirs, you're tightly coupled. If you each have a clean handoff (an interface) and can change internally without disturbing the other, you're loosely coupled.

**The spectrum (you must know these terms):**

```
TIGHT (bad, usually)  ←─────────────────────────→  LOOSE (good, usually)

Content coupling   Common   Control   Stamp   Data    Message
(reach into        coupling coupling  coupling coupling coupling
 internals)        (shared  (pass a   (pass    (pass    (just send
                   globals) flag that  whole    only     a message,
                            controls    struct   needed   no shared
                            behavior)   when you data)    types)
                                        need 2
                                        fields)
```

- **Content coupling (worst):** Module A modifies Module B's internal data directly. Catastrophic.
- **Data coupling (good):** Module A passes Module B exactly the data it needs as parameters.
- **Message coupling (loosest):** Modules communicate only via messages/events with no shared state.

**Why it matters:** Tight coupling is the primary enemy of changeability. When components are tightly coupled, a change ripples outward — touch one thing, break ten others. This is called the **"ripple effect"** or **"shotgun surgery."** Most design patterns exist *specifically to reduce coupling* by inserting an abstraction (interface) between collaborating parts.

**Critical nuance for senior engineers:** You cannot eliminate coupling — components must interact to form a system. The goal is not *zero* coupling (that's a system that does nothing) but **loose, intentional, and visible** coupling along well-chosen interfaces. *Some coupling is essential complexity; the skill is removing only the accidental coupling.*

### Cohesion

**Definition:** The degree to which the elements *inside* a single module belong together — how focused and singular its purpose is.

**Mental model:** A well-organized toolbox where the screwdriver drawer holds only screwdrivers (high cohesion), versus a junk drawer holding batteries, tape, a sandwich, and car keys (low cohesion). High cohesion means everything in the module serves one clear, related purpose.

**The spectrum (best to worst):**
- **Functional cohesion (best):** Every element contributes to a single well-defined task. A `PasswordHasher` class that only hashes passwords.
- **Sequential / Communicational cohesion (good):** Elements work on the same data or feed each other.
- **Temporal cohesion (weak):** Grouped because they happen at the same time (e.g., a `startup()` that inits logging, DB, and cache — unrelated except timing).
- **Coincidental cohesion (worst):** Elements grouped for no real reason — a `Utils` class with `parseDate()`, `sendEmail()`, and `calculateTax()`.

**Why it matters:** Low cohesion means a module has multiple unrelated responsibilities, so it changes for multiple unrelated reasons, gets touched by many people, and becomes a bug magnet. High cohesion means a module is *understandable in isolation* and changes for *one reason.* (This directly anticipates the Single Responsibility Principle in 1.6.)

### The Golden Rule These Three Imply

> **Strive for high cohesion within modules and loose coupling between modules.**

This single sentence is the north star of structural software design. *Nearly every design pattern is a maneuver to increase cohesion, decrease coupling, or both.* When you later ask "should I apply this pattern?", the real question is almost always "does this improve cohesion and/or coupling enough to justify the added indirection?"

### Visualizing the Goal

```
LOW COHESION, TIGHT COUPLING (nightmare):

  ┌─────────┐         ┌─────────┐
  │ Module A│◄═══════►│ Module B│
  │ does 5  │╲       ╱│ does 6  │
  │ unrel-  │ ╲     ╱ │ unrel-  │
  │ ated    │  ╲   ╱  │ ated    │
  │ things  │   ╲ ╱   │ things  │
  └────┬────┘    X    └────┬────┘
       │        ╱ ╲        │
       ▼       ╱   ╲       ▼
  ┌─────────┐         ┌─────────┐
  │ Module C│◄═══════►│ Module D│
  └─────────┘         └─────────┘
  (everything knows everything; change anything, break everything)


HIGH COHESION, LOOSE COUPLING (the goal):

  ┌─────────┐  clean   ┌─────────┐
  │ Module A│  inter-  │ Module B│
  │ one job │─face────►│ one job │
  └─────────┘          └─────────┘
       │ clean interface
       ▼
  ┌─────────┐  clean   ┌─────────┐
  │ Module C│─inter───►│ Module D│
  │ one job │  face    │ one job │
  └─────────┘          └─────────┘
  (each understandable alone; changes stay local)
```

---

## 1.5 Prerequisite: The Object-Oriented Toolkit

The GoF patterns are expressed in OO terms. You must understand the OO building blocks they manipulate. Even if you primarily write Go or functional code, this vocabulary is the *lingua franca* of pattern literature.

### Class vs. Object (Instance)

- **Class:** A *blueprint/template* defining structure (fields) and behavior (methods). Exists at "compile time" conceptually.
- **Object/Instance:** A concrete *thing* created from the class, living in memory at runtime, with its own field values.

**Mental model:** The class is the *cookie cutter*; objects are the *cookies*. One cutter, many cookies, each potentially decorated differently.

### Interface (The Most Important OO Concept for Patterns)

**Definition:** A *contract* — a set of method signatures with no implementation — that classes can promise to fulfill. It declares *what* can be done, never *how*.

**Mental model:** A *power outlet standard.* The outlet is an interface: any device with the right plug works, regardless of what's inside the device. The wall doesn't know or care if you plug in a lamp, laptop, or toaster — only that the plug conforms to the contract.

```java
interface PaymentProcessor {
    PaymentResult charge(Money amount, Card card);
    // No body. Pure contract. "Whoever implements me CAN charge."
}
```

**Why interfaces are the beating heart of design patterns:** An interface is *the seam along which you decouple.* The calling code depends on the *interface* (the what), not the concrete class (the how). This means you can swap implementations freely. **Virtually every GoF pattern works by having clients depend on an interface, with concrete classes hidden behind it.** If you understand *deeply* why interfaces enable swappability and decoupling, you already understand 60% of why patterns work.

### Inheritance

**Definition:** A mechanism where a class (subclass/child) *derives* fields and methods from another class (superclass/parent), establishing an "is-a" relationship.

**Mental model:** A `Dog` *is-a* `Animal`, inheriting `eat()` and `sleep()`, adding `bark()`.

```java
class Animal {
    void eat() { /* ... */ }
}
class Dog extends Animal { // Dog is-a Animal
    void bark() { /* ... */ }
}
```

**Why it exists:** To reuse code and express genuine taxonomic relationships.

**The crucial warning (you must internalize this now):** Inheritance is *the most overused and dangerous* OO mechanism. It creates the *tightest possible coupling* — a subclass depends on the *internal implementation details* of its parent, not just its interface. Change the parent, and subclasses break in surprising ways. This is the **"fragile base class problem."** The GoF authors themselves wrote one of the most important sentences in the book:

> **"Favor object composition over class inheritance."**

We unpack this in 1.7. For now, just register: *inheritance looks like the obvious tool for reuse, but it's frequently the wrong one, and patterns often exist specifically to replace it with composition.*

### Composition

**Definition:** Building objects by *containing references to other objects* (a "has-a" relationship) and delegating work to them, rather than inheriting.

**Mental model:** A `Car` *has-a* `Engine`, *has-a* `Transmission`. The car delegates "produce power" to its engine. You can swap a V6 engine for an electric motor *at runtime* without changing the car's class — because the car holds a *reference* to an `Engine` interface, not a hardcoded engine type.

```java
class Car {
    private Engine engine; // has-a (composition)

    Car(Engine engine) {        // engine injected
        this.engine = engine;
    }
    void start() {
        engine.ignite();        // delegate to the contained object
    }
}
```

**Why composition beats inheritance for flexibility:**
- **Runtime flexibility:** You choose the contained object *when you create the car*, even swap it later. Inheritance is fixed at compile time.
- **Looser coupling:** `Car` depends only on the `Engine` *interface*, not implementation internals.
- **No fragile base class:** Changing `Engine` internals can't silently break `Car` as long as the interface holds.
- **Avoids combinatorial explosion:** With inheritance, `Car` + flavors (Electric, Sport, Luxury) × (V6, V8, Electric) breeds dozens of subclasses. With composition, you mix and match a few parts.

### Polymorphism

**Definition:** The ability for *different types to be treated through a common interface*, with the *correct behavior selected at runtime* based on the actual object type. Literally "many forms."

**Mental model:** You call `animal.speak()`. If `animal` is a `Dog`, you get "Woof"; if a `Cat`, "Meow." Same call, different behavior, resolved by the *actual object*, not the *declared type*.

```java
PaymentProcessor p = getProcessor(); // could be Stripe, PayPal, etc.
p.charge(amount, card);   // calls the RIGHT charge() based on actual type
```

**Why this is the engine of patterns:** Polymorphism is *the mechanism* by which "depend on the interface" actually works at runtime. When client code holds a `PaymentProcessor` reference and calls `charge()`, polymorphism dispatches to whichever concrete implementation is actually present. **Patterns are, mechanically, organized applications of polymorphism.** (In Part 3, "Internals," we'll trace exactly how the CPU resolves this via vtables. For now, understand it conceptually: the *right method runs based on the real object.*)

### Abstract Class vs. Interface

A frequent source of beginner confusion:

| | **Interface** | **Abstract Class** |
|--|--------------|-------------------|
| Contains | Only contracts (mostly) | Contracts *and* some implemented methods/state |
| "Is-a" relationship | Looser ("can-do") | Stronger ("is-a-kind-of") |
| Multiple inheritance | Usually allowed | Usually not |
| Use when | Many unrelated types share a capability | Related types share *implementation* + contract |

Some patterns (Template Method) use abstract classes; others (Strategy) prefer interfaces. The choice reflects whether you're sharing *implementation* or just a *contract.*

---

## 1.6 Prerequisite: Design Principles — The Forces That Generate Patterns

Here is the deepest idea in Part 1: **Patterns are not fundamental. Principles are.** Patterns are *consequences* — they are recurring solutions that happen to satisfy good principles. If you understand the principles, you can derive, adapt, and even invent patterns. If you only memorize patterns, you're lost the moment a situation doesn't match a textbook diagram.

We cover the **SOLID** principles plus the critical non-SOLID ones.

### S — Single Responsibility Principle (SRP)

**Statement:** *A class should have only one reason to change.*

**What it really means:** Not "do one thing" (a common misreading) but *"be answerable to one stakeholder / one axis of change."* If your `Invoice` class changes when the *tax law* changes AND when the *PDF layout* changes AND when the *database schema* changes, it has three reasons to change — three responsibilities tangled together.

**Why:** When responsibilities tangle, a change for one reason risks breaking another. Separating them = high cohesion + localized change.

**Connection to patterns:** Strategy, Visitor, and others exist to *separate a responsibility* (an algorithm, an operation) out of a class so it can change independently.

```java
// VIOLATION: three reasons to change in one class
class Invoice {
    void calculateTax() { }   // changes when tax law changes
    void renderPdf() { }      // changes when layout changes
    void saveToDb() { }       // changes when persistence changes
}

// FOLLOWING SRP: each changes for one reason
class TaxCalculator { }
class InvoiceRenderer { }
class InvoiceRepository { }
```

### O — Open/Closed Principle (OCP)

**Statement:** *Software entities should be open for extension but closed for modification.*

**What it really means:** You should be able to *add new behavior* by *adding new code*, not by *editing existing, tested, working code.* You extend by plugging in new implementations, not by cracking open and rewiring the old ones.

**Mental model:** A power strip. To add a new device, you plug it in (extension). You don't rewire the strip's internals (modification). The strip is *open* for new devices, *closed* for internal change.

**Why:** Editing working code risks introducing bugs into things that currently work. Adding new code isolated behind an interface can't break the old paths.

```java
// VIOLATION: adding a shape means editing this method (and risking the existing cases)
double area(Shape s) {
    if (s instanceof Circle) return ...;
    else if (s instanceof Square) return ...;
    // every new shape = edit here = risk
}

// OCP: add a new shape = add a new class, touch nothing existing
interface Shape { double area(); }
class Circle implements Shape { public double area() { return ...; } }
class Square implements Shape { public double area() { return ...; } }
// New Triangle? Just add class Triangle implements Shape. Done.
```

**Connection to patterns:** OCP is the *single most pattern-generating principle.* Strategy, Decorator, Factory, Observer, Visitor — nearly all exist to make some axis *open for extension, closed for modification.* When you feel the pain of "I have to edit this switch statement every time," you're feeling the force that OCP-satisfying patterns resolve.

### L — Liskov Substitution Principle (LSP)

**Statement:** *Subtypes must be substitutable for their base types without altering the correctness of the program.* (Named for Barbara Liskov, 1987.)

**What it really means:** If code works with a `Bird`, it must work with *any* subtype of `Bird` *without knowing which one* and without breaking. A subtype must honor the *behavioral contract* of its parent — not just the method signatures, but the *promises*.

**The classic violation — Rectangle/Square:**
```
A Square "is-a" Rectangle mathematically. So Square extends Rectangle?
But Rectangle promises: setWidth(5) leaves height unchanged.
A Square can't honor that — setting width must change height too.
So code that relies on Rectangle's promise BREAKS when given a Square.
→ Square is NOT a behavioral subtype of Rectangle, despite the math.
```

**Why it matters for patterns:** Polymorphism — the engine of patterns — *only works if substitution is safe.* If a `PaymentProcessor` implementation throws on a method the interface promises works, every pattern relying on that interface becomes a landmine. LSP is the *discipline that makes polymorphism trustworthy.*

### I — Interface Segregation Principle (ISP)

**Statement:** *Clients should not be forced to depend on methods they do not use.* Prefer many small, focused interfaces over one large, general one.

**Mental model:** A restaurant menu vs. forcing every customer to read a 200-page tome covering breakfast, catering, and wholesale. Give each client exactly the interface it needs.

```java
// VIOLATION: a "fat" interface forces unused dependencies
interface Worker {
    void work();
    void eat();   // a RobotWorker doesn't eat — forced to implement a no-op/throw
}

// ISP: segregate into focused interfaces
interface Workable { void work(); }
interface Eatable { void eat(); }
// Robot implements Workable only. Human implements both.
```

**Why:** Fat interfaces couple clients to methods they don't care about, so changes to those methods ripple to clients that never used them. Segregation keeps coupling minimal and precise.

### D — Dependency Inversion Principle (DIP)

**Statement (two parts):**
1. *High-level modules should not depend on low-level modules. Both should depend on abstractions.*
2. *Abstractions should not depend on details. Details should depend on abstractions.*

**What it really means — and why "inversion":** Naively, high-level policy (e.g., `OrderService`) depends *downward* on low-level details (e.g., `MySqlDatabase`). DIP *inverts* this: both depend on an *abstraction* (`OrderRepository` interface) in the middle. The arrow of dependency now points *toward the abstraction* from both sides.

```
NAIVE (high-level chained to a specific low-level detail):

  OrderService ───depends-on──► MySqlDatabase
  (change the DB → OrderService breaks)


INVERTED (both depend on an abstraction):

  OrderService ──► «OrderRepository» ◄── MySqlRepository
   (high-level)      (abstraction)        (low-level detail)
   (swap MySql for Postgres → OrderService untouched)
```

**Why it's foundational:** DIP is *the* principle behind Dependency Injection, Factories, Repositories, and the entire idea of "plug in your implementation." It's what makes systems testable (inject a fake repository in tests) and flexible (swap real implementations). **DIP is arguably the most important principle for understanding real-world architecture**, and we will return to it constantly.

### Beyond SOLID: The Other Essential Principles

SOLID is famous but incomplete. These are equally important:

**DRY — Don't Repeat Yourself.** *Every piece of knowledge should have a single, authoritative representation.* Note the subtlety: DRY is about *knowledge*, not *code text*. Two lines that look identical but represent *different decisions that may change independently* are *not* a DRY violation — and merging them is a mistake (the "wrong abstraction" trap). Over-aggressive DRY is a real disease.

**KISS — Keep It Simple, Stupid.** The simplest solution that works is usually best. Patterns *add* complexity; KISS is the counterweight reminding you to justify it.

**YAGNI — You Aren't Gonna Need It.** Don't build flexibility for a future that may never come. This is the *direct antagonist of pattern over-application.* Most pattern abuse is a YAGNI violation: building elaborate plug-in machinery for an axis of change that never materializes.

**The profound tension you must hold:** OCP/DIP push you *toward* abstraction and flexibility (patterns). KISS/YAGNI push you *away* from it (simplicity). **Mastery is not picking a side — it's developing the judgment to know, for *this* specific case, which force wins.** A junior over-applies patterns (ignores YAGNI). A burned mid-level under-applies them (ignores OCP). A senior reads the *specific situation* and the *probability of each kind of change* and chooses. This judgment is the real subject of this entire course.

**Law of Demeter (Principle of Least Knowledge).** *Only talk to your immediate friends.* A method should call methods on: itself, its parameters, objects it creates, and its direct fields — not reach through chains like `order.getCustomer().getAddress().getCity().toUpperCase()`. Such "train wrecks" couple you to a deep structure; any link changing breaks you.

**Composition Over Inheritance.** (From 1.5, restated as a principle because it's that important.) Prefer "has-a" delegation over "is-a" derivation for behavior reuse.

---

## 1.7 Prerequisite: "Program to an Interface, Not an Implementation" & "Favor Composition"

The GoF book opens with *two* mottos that generate most of the patterns. We've touched both; now we make them airtight, because they are the *operational heart* of pattern thinking.

### "Program to an Interface, Not an Implementation"

**Meaning:** Write your code to depend on the *abstract type* (the contract), never the concrete class. Hold a `List`, not an `ArrayList`. Accept a `PaymentProcessor`, not a `StripeProcessor`.

```java
// Programming to an IMPLEMENTATION (rigid)
ArrayList<User> users = new ArrayList<>();
void process(ArrayList<User> users) { } // locked to ArrayList forever

// Programming to an INTERFACE (flexible)
List<User> users = new ArrayList<>();    // could become LinkedList freely
void process(List<User> users) { }       // accepts ANY List
```

**Why this is transformative:** The moment your code names a concrete type, it's *welded* to that type. Programming to the interface keeps a *seam* — a place where you can substitute a different implementation (a faster one, a test double, a remote one) without the client noticing. **This seam is where every pattern lives.** A pattern is, almost definitionally, *a disciplined way of arranging interface-seams so the system flexes where you want.*

### "Favor Composition Over Inheritance" — The Full Argument

We introduced this in 1.5. Now the *complete* reasoning, because it's the most important design heuristic in the OO world:

**The case against inheritance for reuse:**

1. **White-box reuse / broken encapsulation.** A subclass can see and depend on the parent's *protected* internals. The parent's *implementation* becomes part of the subclass's contract. This is "white-box" — the insides are exposed. Encapsulation between parent and child is broken.

2. **Static binding.** The inheritance relationship is fixed *at compile time.* You cannot change a `Dog` into a `Cat` at runtime. Composition lets you swap the contained object at runtime.

3. **The fragile base class problem.** Modifying a base class can break subclasses in ways the base-class author can't foresee — because subclasses depend on *how* the base does things, not just *what.*

4. **Combinatorial explosion.** Independent dimensions of variation multiply into a subclass explosion. (Coffee with milk, sugar, soy, whip → `MilkSugarSoyCoffee`, `MilkWhipCoffee`... — the exact problem the **Decorator** pattern solves with composition.)

**The case for composition:**
- **Black-box reuse.** You use the composed object only through its interface; its internals stay hidden. Encapsulation preserved.
- **Dynamic binding.** Swap composed parts at runtime.
- **Smaller, focused classes** that combine flexibly.

**The honest counterpoint (senior nuance):** Inheritance is not evil; it's *misused.* Inheritance is *correct* when there is a genuine, stable "is-a" relationship AND you want to inherit *and extend* a contract (this is how the Template Method pattern and most framework extension points work). The rule is "*favor*," not "*never use*." Use inheritance for *type relationships and contracts*; use composition for *behavior reuse and flexibility.*

---

## 1.8 Prerequisite: The Three Categories of GoF Patterns (The Map Before the Territory)

Before learning individual patterns (Part 2+), you need the *taxonomy* — the mental filing cabinet. GoF split their 23 patterns into three categories based on *what problem the pattern addresses.* Understanding the categories means that when you face a problem, you can ask "*which kind* of problem is this?" and narrow to the right shelf.

### Creational Patterns — "How are objects made?"

These patterns deal with **object creation mechanisms** — abstracting the *instantiation* process so the system is independent of *how its objects are created, composed, and represented.*

**The force they resolve:** Direct `new ConcreteClass()` calls weld your code to specific classes. Creational patterns decouple *the request to create* from *the concrete thing created.*

The five GoF creational patterns:
- **Singleton** — ensure one instance, global access. (The most famous, most overused, most criticized.)
- **Factory Method** — defer instantiation to subclasses via an overridable creation method.
- **Abstract Factory** — create *families* of related objects without specifying concrete classes.
- **Builder** — construct complex objects step by step, separating construction from representation.
- **Prototype** — create new objects by *cloning* an existing instance.

### Structural Patterns — "How are objects composed into bigger structures?"

These deal with **how classes and objects are composed** to form larger structures while keeping those structures flexible and efficient. They're about *relationships between entities.*

**The force they resolve:** You need to assemble objects into structures, bridge incompatible interfaces, add capabilities, or control access — *without* tight coupling or rigid hierarchies.

The seven GoF structural patterns:
- **Adapter** — make incompatible interfaces work together (a plug converter).
- **Bridge** — separate an abstraction from its implementation so both vary independently.
- **Composite** — treat individual objects and compositions of objects uniformly (tree structures).
- **Decorator** — add responsibilities to objects dynamically by wrapping (the composition answer to the subclass explosion).
- **Facade** — provide a simplified interface to a complex subsystem (the Stripe SDK example).
- **Flyweight** — share common state across many objects to save memory.
- **Proxy** — provide a placeholder/surrogate controlling access to another object (lazy loading, access control, remote calls).

### Behavioral Patterns — "How do objects communicate and distribute responsibility?"

These deal with **algorithms and the assignment of responsibilities between objects** — the *patterns of communication and interaction.* They're about *who does what, and how they talk.*

**The force they resolve:** Coordinating behavior, distributing responsibility, and managing communication *without* tight coupling between the interacting parts.

The eleven GoF behavioral patterns:
- **Strategy** — define a family of interchangeable algorithms, select at runtime.
- **Observer** — notify dependents automatically when subject state changes (events, pub/sub).
- **Command** — encapsulate a request as an object (undo, queues, logging).
- **Iterator** — access elements of a collection sequentially without exposing its internals.
- **Template Method** — define an algorithm's skeleton, defer steps to subclasses.
- **State** — alter an object's behavior when its internal state changes (state machines).
- **Chain of Responsibility** — pass a request along a chain of handlers (middleware).
- **Mediator** — centralize complex communication between objects.
- **Memento** — capture and restore an object's state (snapshots, undo).
- **Visitor** — add operations to object structures without modifying them.
- **Interpreter** — define a grammar and interpret sentences in it.

### The Mental Filing Cabinet

```
┌─────────────────────────────────────────────────────────────┐
│  WHEN YOU HAVE A PROBLEM, ASK: WHAT KIND IS IT?              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  "How do I CREATE this thing flexibly?"                      │
│        → CREATIONAL shelf                                    │
│        (Singleton, Factory Method, Abstract Factory,         │
│         Builder, Prototype)                                  │
│                                                              │
│  "How do I COMPOSE/STRUCTURE/WRAP these things?"             │
│        → STRUCTURAL shelf                                    │
│        (Adapter, Bridge, Composite, Decorator,               │
│         Facade, Flyweight, Proxy)                            │
│                                                              │
│  "How do these things COMMUNICATE / who's RESPONSIBLE?"      │
│        → BEHAVIORAL shelf                                    │
│        (Strategy, Observer, Command, Iterator, Template      │
│         Method, State, Chain of Responsibility, Mediator,    │
│         Memento, Visitor, Interpreter)                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

This map is enough for now. Parts 2 onward will descend into each pattern in full depth. But carry this taxonomy: *Creational = making, Structural = composing, Behavioral = communicating.*

---

## 1.9 Prerequisite: Patterns Are Relative to Language (The Insight That Separates Experts)

This section is what separates engineers who *understand* patterns from those who merely *recite* them. It is the most sophisticated idea in Part 1, and most textbooks omit it entirely.

### The Core Insight

> **Many design patterns exist to compensate for what a language *lacks*. The "right" pattern — or whether you need one at all — depends entirely on the language you're using.**

The GoF patterns were extracted from **C++ and Smalltalk circa 1994.** Those languages lacked features that modern languages have built-in. So some "patterns" are really *manual reconstructions of features that other languages give you for free.*

### Concrete Demonstrations

**Strategy pattern in Java (1994-style):**
```java
interface SortStrategy { void sort(int[] data); }
class QuickSort implements SortStrategy { public void sort(int[] d){ } }
class Sorter {
    private SortStrategy strategy;
    Sorter(SortStrategy s){ this.strategy = s; }
    void run(int[] d){ strategy.sort(d); }
}
// new Sorter(new QuickSort()).run(data);
```

**The same Strategy in a language with first-class functions (Python/JS/Go):**
```python
def sorter(data, strategy):   # strategy is just a function
    strategy(data)

sorter(data, quick_sort)      # no interface, no class, no ceremony
```

The entire Strategy pattern *evaporates* into "pass a function." The pattern was *scaffolding to simulate first-class functions* in a language that lacked them. In Python, JavaScript, Go, Rust, Kotlin, Swift — you just pass the function.

**More examples of patterns dissolving in capable languages:**

| GoF Pattern | Dissolves into... | In languages with... |
|-------------|-------------------|----------------------|
| Strategy | Passing a function | First-class functions |
| Command | A closure | Closures |
| Iterator | `for x in collection` | Built-in iteration protocols |
| Singleton | A module | Module systems (Python, Go, JS) |
| Factory | A function returning an object | First-class functions/dynamic typing |
| Decorator | Function composition / `@decorator` | Higher-order functions |
| Visitor | Pattern matching | Sum types + pattern matching (Rust, Scala, Haskell) |
| Prototype | `Object.create()` / `clone` | Prototypal inheritance / built-in clone |
| Template Method | Higher-order function | First-class functions |

### Norvig's Famous Finding

Peter Norvig analyzed the 23 GoF patterns against dynamic languages (Lisp/Dylan) and found **16 of 23 had qualitatively simpler implementations or were invisible** — absorbed into the language itself.

### The Three Levels of Understanding This Implies

This insight reframes what it means to "know a pattern":

1. **Beginner:** "I memorized the UML diagram for Strategy."
2. **Intermediate:** "I can implement Strategy in Java."
3. **Expert:** "I recognize the *force* Strategy resolves (swappable behavior), and I reach for the *lightest mechanism my current language offers* to resolve it — which might be a function, a closure, a trait object, a module, or, yes, sometimes a full class hierarchy. I know *why* the heavy version exists and *when* it's actually warranted."

**The expert doesn't ask "which pattern?" first. They ask "which force is present?" and then "what's the lightest tool in *this* language to resolve it?"** Sometimes that lightest tool *is* the classic pattern (in Java, with complex stateful strategies, the full class version genuinely earns its keep). Often it's a one-liner.

### Why This Matters for *Your* Career

If you cargo-cult Java-style patterns into Go or Python, you produce *over-engineered, un-idiomatic code* that your peers will reject in code review. Go programmers, for instance, deliberately favor small interfaces, struct embedding, and functions over GoF class hierarchies — applying Java patterns there marks you as someone who learned patterns by rote. *Idiomatic pattern usage is language-specific.* Mastery means knowing both the universal force *and* the local idiom.

### The Counter-Caution (Don't Over-Correct)

Some conclude "patterns are obsolete, just use functions." That's the *opposite* error. Even in functional/dynamic languages:
- The *vocabulary* still matters for communication ("this is basically an Observer").
- The *forces* still exist and must be recognized.
- *Stateful, complex* variants of patterns still genuinely need structure.
- *Architectural* patterns (Repository, CQRS, Event Sourcing) are language-independent and very much alive.

The lesson is *not* "patterns are dead." It's "**patterns are a vocabulary of forces and force-resolutions; the implementation weight varies by language, and the expert chooses the lightest sufficient mechanism.**"

---

## 1.10 Prerequisite: How to Read a Pattern (The GoF Template)

When you study each pattern in later Parts, you'll encounter a recurring structure. Learn the template now so you know *what questions to ask* of every pattern. The GoF used this rigorous format, and you should mentally fill it in for every pattern you meet:

1. **Intent** — One or two sentences: what does this pattern *do* and *why*? (The compressed essence.)
2. **Motivation** — A concrete scenario showing the problem and how the pattern solves it. (The story.)
3. **Applicability** — *When* should you use it? The conditions/context that signal "this pattern fits." (The *most practically important* section — it tells you *when*, the question beginners most need answered.)
4. **Structure** — A diagram of the classes/objects and their relationships. (The shape.)
5. **Participants** — The classes/objects involved and their responsibilities. (The cast.)
6. **Collaborations** — How the participants interact at runtime. (The choreography.)
7. **Consequences** — The results: what tradeoffs does it impose? What does it cost? (The *second most important* section — there is *always* a cost.)
8. **Implementation** — Practical techniques, pitfalls, language-specific notes.
9. **Sample Code** — Concrete implementation.
10. **Known Uses** — Real systems that use it. (Proof it's not academic.)
11. **Related Patterns** — Which patterns combine with or substitute for it.

**The two sections that matter most — and that beginners skip:** **Applicability** (when) and **Consequences** (tradeoffs). Anyone can copy a structure diagram. The skill is knowing *when it applies* and *what it costs.* Throughout this course, we will *dwell* on those two for every pattern.

### A Discipline to Adopt Now

Every time you learn a pattern, force yourself to articulate, in your own words:
- **What** force does it resolve?
- **Why** does that force hurt without it?
- **When** is it appropriate (and when is it overkill)?
- **Where** have you seen it in real systems?
- **What does it cost?** (Indirection, classes, runtime overhead, cognitive load.)

If you can't answer "what does it cost," you don't understand the pattern yet — you've only seen its happy path.

---

## 1.11 Prerequisite: Anti-Patterns and the Cost of Patterns

A *foundation* in patterns is incomplete without understanding their *dark mirror.*

### What Is an Anti-Pattern?

**Definition:** An **anti-pattern** is a *commonly-reached-for "solution" that looks reasonable but produces more problems than it solves.* It's a recurring *bad* response to a recurring problem — the negative space that the study of patterns illuminates.

The term was coined by Andrew Koenig (1995), deliberately mirroring "pattern."

### Pattern-Related Anti-Patterns You Must Know Now

**1. Golden Hammer ("when you have a hammer, everything's a nail").** Over-applying a favorite pattern everywhere. The engineer who just learned Singleton and now everything is a Singleton. The cure is a *diverse toolkit* and *force-based* (not familiarity-based) selection.

**2. Over-Engineering / Speculative Generality (a YAGNI violation).** Building elaborate pattern machinery for flexibility you don't need yet. Five layers of factories to create one object that will never have a second implementation. *The cost of a pattern is always paid up front; the benefit is only realized if the predicted change actually happens.*

**3. Cargo Cult Programming.** Applying patterns by ritual imitation without understanding *why.* Copying a Java pattern into Go because "that's how it's done," producing un-idiomatic code. (Directly connects to section 1.9.)

**4. The "Pattern Zealot."** Forcing real problems to fit textbook patterns rather than solving the actual problem. Distorting your domain to match a UML diagram.

**5. Lasagna / Ravioli Code.** *Too many* layers (lasagna) or *too many* tiny objects (ravioli) — the over-decoupled extreme. Indirection so deep that following a single request requires opening fifteen files. This is what happens when you apply patterns without restraint. Decoupling has a *navigational cost.*

### The Universal Cost of Every Pattern

Burn this into memory — it's the counterweight to everything in this course:

> **Every pattern adds indirection. Indirection is not free.**

The costs:
- **Cognitive cost:** More classes/interfaces to hold in your head; more hops to trace logic.
- **Navigational cost:** "Where does this actually happen?" becomes harder to answer.
- **Maintenance cost:** More moving parts to keep consistent.
- **Sometimes runtime cost:** Extra virtual dispatch, object allocation, indirection (usually negligible, occasionally not — covered in Part 6).

A pattern *pays for itself* only when the flexibility it provides is *actually exercised* by real change. **The decision to use a pattern is fundamentally a bet about the future:** you pay complexity now, betting that a specific kind of change will come later and the pattern will make it cheap. *Good engineers make this bet consciously and are often right; juniors make it unconsciously and are often wrong.*

### The Discipline of Restraint

The mature progression of an engineer with patterns:
1. **Ignorance:** doesn't know patterns; reinvents badly.
2. **Discovery:** learns patterns; applies them everywhere (over-engineering phase — *everyone* goes through this).
3. **Restraint:** learns the costs; applies patterns sparingly and deliberately.
4. **Mastery:** sees forces directly; reaches for the lightest sufficient tool; uses patterns as *vocabulary* and *option*, not *obligation.*

This course aims to take you to stage 4 *without* requiring you to spend years stuck in stage 2.

---

## 1.12 Foundations Synthesis: The Mental Model to Carry Forward

Before we proceed to Part 2, consolidate the foundation into a single coherent worldview. If you remember nothing else from Part 1, remember this synthesis:

**1. Software's central difficulty is managing change and complexity in systems too large for one mind.** We cope through *abstraction* — drawing boundaries that hide detail.

**2. The quality of those boundaries is measured by cohesion (high, within) and coupling (loose, between).** Good structure = focused modules with clean seams.

**3. Principles (SOLID, DRY, KISS, YAGNI, composition-over-inheritance, program-to-interface) are the fundamental forces.** They tell you *where* to put boundaries and *how* to keep them healthy. They sometimes *conflict* (OCP vs. YAGNI), and judgment in resolving those conflicts is the core skill.

**4. Design patterns are recurring, named *resolutions* of those forces** — pre-built abstraction structures along common axes of change. They are *consequences* of the principles, not axioms themselves.

**5. Patterns come in three families:** Creational (making), Structural (composing), Behavioral (communicating).

**6. The weight of a pattern's implementation is language-relative.** Many patterns dissolve into language features; the expert recognizes the *force* and reaches for the *lightest local idiom.*

**7. Every pattern costs indirection.** Its use is a *bet about future change.* Apply only when the bet is justified. The opposite of skillful pattern use is not "no patterns" but *thoughtless* patterns (anti-patterns, golden hammer, over-engineering).

**8. Therefore, mastery is judgment, not memorization.** Knowing the 23 diagrams is the *start*. Knowing *which force, which axis of change, which language idiom, and whether the bet is worth it* — that's the staff-engineer skill this course builds.

The mental loop of an expert facing a design decision:

```
   ┌─────────────────────────────────────────────────┐
   │ 1. What is the actual problem / force here?     │
   │    (swappable behavior? creation coupling?       │
   │     notification? incompatible interface?)       │
   └───────────────────────┬─────────────────────────┘
                           ▼
   ┌─────────────────────────────────────────────────┐
   │ 2. Is this axis of change REAL and LIKELY?      │
   │    (YAGNI check — or am I speculating?)          │
   └───────────────────────┬─────────────────────────┘
                           ▼
   ┌─────────────────────────────────────────────────┐
   │ 3. If real: what's the LIGHTEST mechanism in    │
   │    THIS language to resolve the force?           │
   │    (function? closure? interface? full pattern?) │
   └───────────────────────┬─────────────────────────┘
                           ▼
   ┌─────────────────────────────────────────────────┐
   │ 4. Does the flexibility gained EXCEED the        │
   │    indirection cost paid? (the bet)              │
   └───────────────────────┬─────────────────────────┘
                           ▼
   ┌─────────────────────────────────────────────────┐
   │ 5. Apply, name it (vocabulary), document the     │
   │    WHY for the next engineer.                    │
   └─────────────────────────────────────────────────┘
```

This loop *is* the expertise. Everything in Parts 2–12 fills in the details: the specific forces, the specific patterns, the specific costs, the specific language idioms, and the specific judgment calls.

---

### End of Part 1 — Foundations

You now possess the prerequisite machinery: what a pattern *is* and *isn't*, where the discipline came from, the abstraction/encapsulation/coupling/cohesion vocabulary, the OO toolkit, the SOLID + DRY/KISS/YAGNI principles that *generate* patterns, the two GoF mottos, the three-family taxonomy, the language-relativity insight, the reading template, and the anti-pattern cost-awareness that keeps you from over-applying everything you're about to learn.

Crucially, you have the *worldview*: **patterns are force-resolutions and vocabulary, applied with judgment, paid for in indirection, chosen as a bet about change, and weighted by language.** This frame will make every individual pattern in Part 2 *click* as an instance of a principle rather than a recipe to memorize.