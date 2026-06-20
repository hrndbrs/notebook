# PART 7 — SECURITY

## Patterns Through the Threat Lens

Every prior Part asked "does this code work, run fast, and scale?" Security asks a different, adversarial question: **"how does this code behave when someone is actively trying to break it?"** This inversion is the heart of security thinking. A pattern that is elegant under cooperative use can be a liability under adversarial use — the Decorator ordering choice that was a *correctness* concern in Part 4 becomes a *vulnerability* here; the Proxy that *controlled access* in Part 2 becomes a *trust boundary* that, if misplaced, leaks the system open; the Prototype/Memento that *cloned and snapshotted state* becomes a *deserialization attack surface*.

The defining mental shift: **stop assuming inputs are well-formed and actors are cooperative.** In every prior Part, the "client" of a pattern was a fellow programmer using it correctly. In security, the client may be an attacker probing every interface for a way in. Design patterns shape *where the boundaries are* and *what crosses them* — and security is, at root, the discipline of *trust boundaries*: where untrusted data meets trusted execution. Patterns either enforce those boundaries cleanly or smear them, and that difference is the difference between a secure and a breachable system.

```
The security reframe of every concept so far:

  Coupling/cohesion      → attack surface (each interface is a potential entry)
  Abstraction boundaries → trust boundaries (where untrusted meets trusted)
  Encapsulation          → the security guarantee (an object protecting its invariants
                            IS access control at the object level)
  Indirection            → can hide a vulnerability OR enforce a control
  "Program to interface" → the seam where you insert validation/auth (or forget to)
```

---

## 7.1 — The Foundational Security Principles (The Forces of This Part)

Just as SOLID generated the design patterns, a set of security principles generates the *defensive* patterns and the *judgments* about offensive risk. These are the forces; learn them before the threats.

**Defense in Depth.** Never rely on a single control. Layer them, so that breaching one doesn't breach the system. (Architecturally: this is the Chain of Responsibility / middleware pipeline of Part 5 — auth *and* validation *and* rate-limiting *and* output encoding, each a layer.) If the firewall fails, the auth check catches it; if auth fails, input validation catches it; if that fails, least privilege limits the blast radius. **No single point of security failure.**

**Least Privilege.** Every component, user, and process gets the *minimum* access needed to do its job — and no more. A service that only reads should have read-only DB credentials. A protection Proxy (Part 2B) enforcing least privilege is the pattern embodiment. The payoff: when something *is* compromised (assume it will be), the damage is *contained* to what that component could access.

**Fail Secure (Fail Closed).** When a security control fails or is uncertain, *deny*. Contrast Part 4's "fail safe" (don't crash) — security *fail closed* means an error in the auth check denies access rather than granting it. The catastrophic anti-pattern is `try { checkAuth() } catch { /* proceed */ }` — an exception in the auth check should *lock the door*, not open it.

**Zero Trust / Never Trust Input.** Treat all input — from users, other services, files, even internal callers — as potentially hostile until validated. The trust boundary is where this validation happens. "It came from our own frontend" is not a trust guarantee (the attacker can call your API directly, bypassing your frontend entirely).

**Secure by Default.** The default configuration must be the *secure* one. Encryption on by default, permissions closed by default, the safe Decorator ordering as the default. Users won't opt into security; they'll accept whatever the default is. (Recall Part 1's "make the right action the easy action" — secure-by-default makes the secure path the path of least resistance.)

**Minimize Attack Surface.** Every interface, endpoint, parameter, and feature is a potential entry point. The fewer there are, the less to defend. A Facade (Part 2B) that exposes one narrow, validated entry point *reduces* attack surface; a system that exposes every internal class *maximizes* it.

**Complete Mediation.** Every access to every resource must be checked — *every time*, not just the first. Caching an authorization decision and reusing it is how privilege-escalation bugs happen (the user's permissions changed, but the cached "allowed" persists). A Proxy enforcing complete mediation checks on *every* call.

These principles are the *why* behind everything that follows — both the offensive risks (where patterns violate these) and the defensive patterns (where patterns enforce these).

---

## 7.2 — Trust Boundaries: The Core Security Concept Patterns Shape

A **trust boundary** is the line where data or control passes from a less-trusted zone to a more-trusted one — and it's *the* place security controls must live. The single most important security insight about design patterns: **patterns determine where your trust boundaries are and whether they're enforced.**

```
   UNTRUSTED ZONE                    │ TRUST BOUNDARY │      TRUSTED ZONE
   (the internet, user input,        │ ◄── controls   │   (your business logic,
    other services, files)           │     live HERE  │    your database)
                                     │                │
   raw HTTP request ────────────────►│ validate       │──► domain logic
   (could be anything,               │ authenticate   │    (assumes clean,
    could be an attack)              │ authorize      │     authorized input)
                                     │ sanitize       │
                                     │ rate-limit     │
```

**How patterns enforce or leak trust boundaries:**

- **Facade / API Gateway (Part 2B/5)** — the *ideal* trust boundary enforcement point. One narrow entry where *all* validation, auth, and rate-limiting happen. Everything behind it can (carefully) assume cleaner input. The danger: if the Facade *doesn't* validate, it becomes a *false* boundary — code behind it assumes safety that was never enforced. **A Facade that doesn't validate is worse than no Facade, because it creates a false sense of a boundary.**

- **Proxy (protection proxy, Part 2B)** — *literally* an access-control pattern. The protection proxy *is* the trust-boundary enforcer: it checks permissions before delegating (complete mediation, least privilege). This is the pattern most directly aligned with security.

- **Adapter (Part 2B)** — the **Anti-Corruption Layer** (Part 5's DDD term) is an Adapter that *also* validates and sanitizes data from an external/untrusted system before it enters your domain. It protects the trust boundary by ensuring foreign data conforms to your domain's invariants *and* is safe.

- **Chain of Responsibility / Middleware (Part 2C/5)** — the *implementation* of defense-in-depth at the boundary: a pipeline of security layers (auth → validation → rate-limit → audit), each a handler, each a layer of the depth. **Order is security-critical** — authentication must precede authorization (you can't check *what* an unidentified user may do); validation must precede processing. A misordered chain is a vulnerability (Part 2C's "order matters" concern, now with security stakes).

**The pervasive failure mode — the "confused deputy" and missing-boundary bugs:** when a pattern's indirection makes it *unclear where the trust boundary is*, validation gets skipped ("I assumed the layer below validated"; the layer below assumed the layer above did). Indirection — the very thing patterns add — can *obscure* where the boundary is, leading each layer to assume another handles security. **Explicit trust boundaries, enforced at a known pattern (Facade/Proxy/middleware), with validation that doesn't rely on "someone else did it," is the defense.**

---

## 7.3 — The Injection Family: When Untrusted Data Becomes Code

The largest, oldest, and most devastating vulnerability class — and one that patterns both *cause* (when abstractions hide the danger) and *prevent* (when the right pattern enforces separation). **Injection happens when untrusted data is interpreted as code/commands** because data and code aren't kept separate across a trust boundary.

### SQL Injection (and the Repository's Role)

```
   VULNERABLE (string concatenation — data becomes SQL):
     query = "SELECT * FROM users WHERE name = '" + userInput + "'"
     attacker sends:  ' OR '1'='1' --
     resulting SQL:   SELECT * FROM users WHERE name = '' OR '1'='1' --'
     → returns EVERY user. Or:  '; DROP TABLE users; --  → destroys data.

   SAFE (parameterized query — data stays DATA, never code):
     query = "SELECT * FROM users WHERE name = ?"
     bind(1, userInput)
     → the DB driver treats userInput as a pure VALUE; it can NEVER
       become SQL syntax. The trust boundary is enforced by the
       parameterization mechanism.
```

**The pattern connection (critical):** the **Repository pattern (Part 2D)** is a security asset *when implemented correctly* — it centralizes all data access in one layer, so you enforce parameterized queries *in one place* and *guarantee* no concatenated SQL escapes it. But it's a security *liability* when its abstraction *hides* a raw-string query inside (the clean `repo.search(term)` interface conceals a concatenated query — the same "abstraction hides danger" lesson as Part 6's N+1, now with a vulnerability instead of a slowdown). **A Repository is a place to enforce a security invariant (all queries parameterized) — but only if you actually enforce it there; the pattern provides the *location* for the control, not the control itself.**

### The General Injection Principle (Applies to All Variants)

```
   Injection type     │ Untrusted data interpreted as...
   ───────────────────┼──────────────────────────────────
   SQL injection      │ database query syntax
   Command injection  │ OS shell commands  (os.system(user_input) — catastrophic)
   XSS                │ HTML/JavaScript in the browser
   LDAP injection     │ directory query syntax
   XXE                │ XML external entity references
   Template injection │ server-side template code
   Path traversal     │ filesystem paths (../../etc/passwd)

   The UNIVERSAL fix: keep DATA and CODE separate across the trust boundary.
   - parameterize (SQL)
   - use safe APIs that don't invoke a shell (command)
   - context-aware output ENCODING (XSS — encode for the destination context)
   - allow-list validation (never deny-list — attackers find what you didn't ban)
```

**XSS and the Decorator/output-encoding connection:** Cross-Site Scripting injects malicious JavaScript into a page viewed by other users. The defense is **context-aware output encoding** — encoding data differently depending on *where* it lands (HTML body vs. HTML attribute vs. JavaScript vs. URL), because each context has different dangerous characters. This is naturally a **Decorator/Strategy** concern: an output-encoding decorator/strategy applied at the rendering boundary, selected by context. And it echoes Part 4's Decorator-ordering security lesson: *encode at the right boundary, in the right context, or the protection is worthless.*

**The deep lesson:** injection is fundamentally a **trust-boundary failure** — data from the untrusted zone crossed into the trusted zone *without being kept separate from code.* Every defense is a way of enforcing "data stays data." Patterns that centralize the boundary crossing (Repository for DB, a rendering Facade for output) give you *one place* to enforce this — which is exactly why centralizing security-critical operations behind a pattern is a defensive strategy, *provided the pattern actually enforces the control rather than just hiding the operation.*

---

## 7.4 — Deserialization: The Prototype/Memento/Command Attack Surface

This is the security topic most *directly* tied to specific design patterns, and one of the most dangerous and underappreciated vulnerability classes. **Deserialization vulnerabilities** turn data-into-objects into a path for remote code execution — and they strike precisely at patterns that *reconstruct objects from external data*: Prototype (cloning), Memento (state restoration), Command (serialized commands in queues — Part 4.5), and any persistence that serializes objects.

### Why Deserialization Is Dangerous

```
   Serialization: object → bytes (to store/transmit/queue)
   Deserialization: bytes → object (to restore)

   THE ATTACK: if deserialization reconstructs ARBITRARY object types from
   attacker-controlled bytes, the attacker crafts a malicious byte stream that,
   when deserialized, instantiates dangerous objects and triggers code execution
   during the reconstruction process itself ("gadget chains" — chaining existing
   classes' deserialization side-effects into arbitrary code execution).

   Java's native serialization, Python's pickle, PHP's unserialize,
   .NET BinaryFormatter — all have enabled catastrophic RCE this way.
   (Python's docs literally warn: never unpickle untrusted data.)
```

**The pattern connection — this is where Part 2/4's patterns become attack surfaces:**

- **Command in a message queue (Part 4.5):** commands serialized into a queue and deserialized by a worker. If an attacker can inject into the queue, a malicious serialized "command" deserializes into a code-execution gadget. **This is why Part 4.5 noted commands should "store data, not object references" and be designed for safe serialization** — the security reason behind that design guidance is deserialization attacks.

- **Memento persisted for crash recovery / undo (Part 2C/4.5):** snapshots serialized to disk/DB and restored. Untrusted memento bytes → deserialization RCE.

- **Prototype via serialization-based deep copy (Part 2A):** a common deep-copy implementation serializes-then-deserializes to clone. If the object graph includes untrusted data, the clone operation is a deserialization sink.

- **Event Sourcing (Part 5):** events serialized to the log and replayed (deserialized). The event store becomes a deserialization attack surface if events can be tampered with.

### The Defenses

```
   1. NEVER deserialize untrusted data with a format that can instantiate
      arbitrary types. (No Java native serialization / pickle / BinaryFormatter
      on untrusted input — ever.)
   2. Use DATA-ONLY formats (JSON, Protobuf) that deserialize into KNOWN,
      explicit types — not arbitrary object graphs. The deserializer can only
      produce the types you declared, closing the gadget-chain door.
   3. ALLOW-LIST permitted types if you must use object serialization.
   4. Validate AFTER deserialization (the reconstructed object must still
      pass invariant checks — deserialization can bypass constructors, so
      an object's normal validation (Part 4's Builder.build() gate) may never
      have run! A deserialized object can be in an "impossible" state.).
   5. Sign/encrypt serialized data so tampering is detectable (integrity).
```

**The profound design lesson connecting back to Part 1:** **deserialization can bypass a class's constructor**, meaning the encapsulation guarantees (Part 1) and validation gates (Part 4's Builder) that the object *relies on* to maintain its invariants **never run.** An object that *cannot* be constructed in an invalid state via its normal API *can* be deserialized into exactly that invalid state. This is a direct attack on the encapsulation-as-invariant-protection principle from Part 1 — and the defense (validate after deserialization) is the acknowledgment that *the trust boundary must be re-enforced* whenever an object enters through a non-constructor path. **Any pattern that reconstructs objects from external bytes reopens the trust boundary the constructor was supposed to guard.**

---

## 7.5 — Access Control: The Proxy's Native Territory

Access control — *who can do what* — is the security concern most directly served by patterns, because the **protection Proxy (Part 2B)** is *literally* an access-control pattern. This section is where patterns are unambiguously a security *asset.*

### Authentication vs. Authorization (Never Conflate)

```
   AUTHENTICATION (AuthN): WHO are you? (prove identity — password, token, key)
   AUTHORIZATION (AuthZ):  WHAT may you do? (check permission for THIS action)

   ORDER MATTERS (security-critical, Chain-of-Responsibility ordering):
   AuthN must precede AuthZ — you can't decide what an UNKNOWN actor may do.
   A chain that authorizes before authenticating is broken.
```

### The Protection Proxy as Access-Control Enforcer

```java
// The protection Proxy enforces complete mediation + least privilege.
class SecureDocumentProxy implements Document {
    private final Document realDoc;
    private final SecurityContext ctx;

    public String read() {
        requirePermission(Permission.READ);   // checked EVERY call (complete mediation)
        return realDoc.read();
    }
    public void delete() {
        requirePermission(Permission.DELETE); // separate, finer permission (least privilege)
        realDoc.delete();
    }
    private void requirePermission(Permission p) {
        if (!ctx.currentUser().hasPermission(p))
            throw new AccessDeniedException();  // FAIL CLOSED — deny on missing permission
    }
}
```

**Why this is the security-ideal use of a pattern:** the proxy enforces **complete mediation** (every call checked, not cached), **least privilege** (separate fine-grained permissions per operation), and **fail closed** (missing permission → deny). The real document is *never* reachable except through the proxy's checks — the trust boundary is the proxy, enforced uniformly. This is Part 2B's Proxy with security as the explicit purpose, and it embodies the principles of 7.1 in code.

**The common authorization vulnerabilities the proxy must defend against:**
- **Broken Object-Level Authorization (BOLA/IDOR)** — the #1 API vulnerability. The user is authenticated and authorized to use the endpoint, but you forget to check they own *this specific object*: `GET /orders/12345` returns order 12345 to *anyone* logged in, so an attacker enumerates IDs and reads everyone's orders. The fix: authorize the *specific resource*, not just the endpoint (`does this user own order 12345?`) — complete mediation at the *object* level, which the proxy must enforce per-object, not per-method.
- **Privilege escalation** — caching an authorization decision and reusing it after permissions changed (violates complete mediation), or trusting a client-supplied role.
- **Confused deputy** — a privileged component performs an action on behalf of a less-privileged caller without checking the *caller's* rights (the proxy must check the *original* caller's permissions, not its own).

**The deeper pattern lesson:** access control implemented as a *cross-cutting* concern via Proxy/Decorator/middleware (rather than scattered `if (user.canDoThis())` checks throughout business logic) is both *more secure* (one place to get right, complete mediation guaranteed) and *more maintainable* (SRP — security separate from business logic). Scattered, inline authorization checks are a security anti-pattern: easy to forget one (→ BOLA), hard to audit, tangled with logic. **Centralize access control behind a pattern.** This is Part 5's service-mesh/API-gateway security at object scale.

---

## 7.6 — The Singleton, Shared State, and Concurrency-Security

Part 2A's most-criticized pattern has a *security* dimension beyond its design flaws: **shared mutable state is a security hazard.**

```
   A mutable Singleton (or any shared mutable state) accessed concurrently:
   - RACE CONDITIONS become security vulnerabilities (TOCTOU — Part 2A's
     check-then-act race, now adversarial): an attacker times requests to
     exploit the gap between a security CHECK and the protected ACTION.
     Classic: check "balance >= amount" then "deduct amount" — two concurrent
     requests both pass the check before either deducts → double-spend.
   - SHARED STATE LEAKS DATA ACROSS USERS: a Singleton holding per-request
     data (a "current user" field) under concurrency can serve User A's data
     to User B's request (a catastrophic data-isolation breach).
   - GLOBAL MUTABLE CONFIG is a tampering target: if any code can mutate the
     shared security config (the Singleton's global access — Part 2A), a
     compromised component can disable security for the whole system.
```

**The security case *reinforces* Part 2A's design case against Singleton:** prefer **immutable** shared state (can't be tampered with or raced — Part 4's immutability-by-default, now a security property) and **dependency injection** with **request-scoped** state (each request gets its own instance — no cross-user leakage) over global mutable Singletons. The TOCTOU race that Part 2A treated as a *concurrency bug* is, in a security context, an *exploitable vulnerability* — and the fix (atomic check-and-act, e.g., a single atomic DB `UPDATE ... WHERE balance >= amount`, or proper locking) is both a correctness and a security fix. **Security and good design converge here: the same shared-mutable-state that makes Singleton a design anti-pattern makes it a security liability.**

---

## 7.7 — Secrets, Logging, and the Information-Leakage Surface

Patterns that *cross-cut* the system — logging Decorators (Part 4.1), Observers emitting events, error-handling middleware — are prime **information-leakage** surfaces, where sensitive data escapes through channels meant for observability or convenience.

```
   LEAKAGE THROUGH CROSS-CUTTING PATTERNS:

   - Logging Decorator (Part 4.1): logs requests/responses for observability,
     but accidentally logs passwords, tokens, credit cards, PII, secrets.
     → Logs are a TOP data-breach vector (often less-protected than the DB,
       shipped to third-party log services, retained for years).
     → Defense: NEVER log secrets/PII; redact at the logging boundary;
       log IDENTIFIERS not sensitive VALUES (Part 4.1 logged the customer ID,
       not the customer's data — that was a security decision).

   - Error messages / exceptions crossing the trust boundary to the client:
     a stack trace or DB error leaks internal structure, file paths, library
     versions, SQL — reconnaissance gold for an attacker.
     → Defense: FAIL CLOSED with a GENERIC message to the client (Part 4.1's
       "internal error" to the user); log the DETAIL internally only.
       (Part 4.1's ObservableDecorator did exactly this — generic Rejected
       to the caller, full detail to the internal log. That split is security.)

   - Observer/event payloads: events broadcast to many subscribers (Part 5 EDA)
     may carry sensitive data to subscribers that shouldn't see it (over-sharing
     in the event payload). → Least privilege applied to event contents.

   - Timing side channels: a security check whose duration depends on secret
     data (e.g., string comparison that returns early on first mismatch) leaks
     the secret bit-by-bit via response timing. → Constant-time comparison for
     secrets/tokens (compare in time independent of where they differ).
```

**The cross-cutting lesson:** the patterns that add observability, error handling, and event distribution — the *conveniences* of Parts 4 and 5 — are exactly the patterns that leak data if not designed with the trust boundary in mind. **Every channel out of the trusted zone (logs, errors, events, response timing) is an information-leakage surface, and the cross-cutting patterns that create those channels must redact, genericize, and least-privilege their payloads at the boundary.** The Part 4.1 ObservableDecorator that logged IDs-not-PII and returned generic-errors-to-clients was, unannounced, demonstrating secure cross-cutting design.

### Secrets Management

```
   NEVER: hardcode secrets in code (they leak via source control — git history
          is forever; a committed-then-deleted secret is still in history).
   NEVER: put secrets in config files in the repo, or in client-side code
          (anything shipped to the browser/app is public — API keys in JS are exposed).
   USE:   secret managers (Vault, AWS Secrets Manager, KMS), injected at runtime
          (DI again — secrets injected as dependencies, never embedded), rotated
          regularly, encrypted at rest and in transit.
```

---

## 7.8 — Defensive Patterns: Security Expressed as Patterns

Just as Part 5 had resilience patterns, security has its own pattern vocabulary — defensive structures that *are* security controls. Several are patterns you already know, repurposed:

**The Validation/Sanitization Boundary (Facade/Adapter as gatekeeper).** All untrusted input validated at *one* explicit boundary (the Facade/API-gateway/Anti-Corruption-Layer), using **allow-lists** (define what's permitted, reject all else) never deny-lists (attackers find what you forgot to ban). Validate *structure, type, range, and format* — and re-validate after any non-constructor object creation (deserialization, 7.4).

**Defense in Depth via the Middleware Chain (Chain of Responsibility, Part 2C/5).** Layered security handlers — authentication, then authorization, then input validation, then rate-limiting, then audit logging — each a link, ordered correctly (authN before authZ, validation before processing), no single layer trusted to be sufficient.

**Rate Limiting & Circuit Breaking (State pattern, Part 5).** Rate limiters (defending against brute-force, credential-stuffing, DoS) and the security-relevant Circuit Breaker (containing a compromised/failing dependency) — State machines acting as security controls.

**The Bulkhead (Part 5) as blast-radius containment.** Isolating components so a compromise or failure in one can't cascade — least privilege and fault isolation as a security boundary.

**Immutability + DI (Part 4) as tamper-resistance.** Immutable security state can't be tampered with or raced; DI-injected, request-scoped state prevents cross-user leakage (7.6). The design disciplines from Part 4 are security controls.

**Idempotency (Part 5) as replay defense.** Idempotent operations resist replay attacks (an attacker re-sending a captured request can't double-charge if the operation is idempotent with a request key) — a resilience pattern doubling as a security control.

**The unifying insight:** **many security controls ARE design patterns applied with adversarial intent** — Proxy for access control, Chain of Responsibility for defense-in-depth, Adapter for anti-corruption, State for rate-limiting/circuit-breaking, immutability+DI for tamper-resistance. Security is not a separate skillset bolted on; it is *the same patterns and principles you already know, applied under the assumption that someone is attacking.* The engineer who understands patterns deeply has most of the *structural* vocabulary of security already — they need only add the adversarial *mindset* and the specific *threat knowledge.*

---

## 7.9 — The Security Mindset and Where It Meets Pattern Design

Beyond specific threats, security is a *way of thinking* that should infuse pattern selection:

**Threat modeling — ask the adversarial question at design time.** For each pattern/component, ask: *What's the trust boundary? What crosses it? What's the worst an attacker who controls this input/component could do?* The structured version (STRIDE) checks six threat categories: **S**poofing (identity), **T**ampering (integrity), **R**epudiation (deniability), **I**nformation disclosure (leakage — 7.7), **D**enial of service (availability), **E**levation of privilege (access — 7.5). Run a pattern through these: a deserialization point (7.4) screams Tampering + Elevation; a logging Decorator (7.7) screams Information disclosure; a Singleton (7.6) screams Tampering + Information disclosure under concurrency.

**Assume breach.** Don't design only to *prevent* compromise; design to *contain* it (least privilege, bulkheads) and *detect* it (audit logging — the Observer/Decorator emitting security events to a monitoring system). The question shifts from "can they get in?" to "when they get in, how limited is the damage and how fast do we know?"

**Security is a property of the whole, not a feature.** It can't be a module you add; it's emergent from *every* trust boundary being enforced, *every* input validated, *every* access mediated, *every* output sanitized. One unguarded boundary (the one Facade that didn't validate, the one Repository method with concatenated SQL, the one endpoint missing object-level authz) breaches the system regardless of how secure everything else is. **Security is the minimum over all boundaries, not the average** — which is why the *consistency* that patterns provide (one place to enforce each control) is itself a security property.

**The convergence with good design (the Part's deepest theme).** Throughout this Part, the security-correct choice was *also* the design-correct choice from prior Parts: immutability (Part 4) is tamper-resistance; DI (Part 2D/4) prevents shared-state leakage and injects secrets safely; centralizing access control behind a Proxy (SRP, Part 1) is complete mediation; the validating Facade (Part 2B) is the trust boundary; logging IDs-not-PII (Part 4.1) is leakage prevention; parameterized queries in the Repository (Part 2D) is injection defense. **Good design and good security are not in tension — they largely converge, because both are about clean boundaries, single responsibility, least coupling, and explicit, enforced contracts.** The messy, tightly-coupled, boundary-smearing code that Part 1 warned against for *maintainability* reasons is *also* the insecure code, because smeared boundaries are exactly where trust-boundary enforcement gets forgotten.

---

## 7.10 — Security Synthesis

Consolidate the threat-lens view:

**1. Security is the adversarial reframe of every prior concept.** Stop assuming cooperative actors and well-formed input. Coupling becomes attack surface; abstraction boundaries become trust boundaries; encapsulation becomes access control; indirection either hides a vulnerability or enforces a control. The same structural thinking, under attack.

**2. Trust boundaries are the core concept, and patterns determine where they are and whether they're enforced.** A Facade/Proxy/middleware-chain at the boundary, *actually enforcing* validation/auth/sanitization (not just hiding the operation), is the foundation. A pattern that creates a *false* boundary — looks like a checkpoint but doesn't check — is worse than none.

**3. Injection is a trust-boundary failure: untrusted data interpreted as code.** Keep data and code separate (parameterize, encode for context, use safe APIs). Centralizing the boundary crossing behind a pattern (Repository for DB, rendering Facade for output) gives one place to enforce "data stays data" — *if* the pattern enforces it rather than concealing a raw operation.

**4. Deserialization is the pattern-specific RCE surface.** Prototype (clone), Memento (restore), Command (queued), and Event Sourcing (replay) all reconstruct objects from external bytes — bypassing constructors and their invariant guards (Part 1/4). Never deserialize untrusted data into arbitrary types; use data-only formats into known types; re-validate after, because the trust boundary the constructor guarded was reopened.

**5. Access control is the Proxy's native territory and patterns' clearest security win.** The protection Proxy enforces complete mediation (every call), least privilege (fine-grained, per-object — defend against BOLA/IDOR), and fail-closed (deny on uncertainty). Centralize authorization as a cross-cutting concern (Proxy/middleware), never scatter inline checks (which get forgotten).

**6. Shared mutable state is a security hazard; immutability + DI + request-scoping is the defense.** The Singleton's design flaws are also security flaws — TOCTOU races become exploitable, shared state leaks across users, global config is a tampering target. Security and good design converge against shared mutable state.

**7. Cross-cutting patterns (logging, errors, events) are leakage surfaces.** Redact secrets/PII at the logging boundary (log IDs, not values); return generic errors to clients while logging detail internally; least-privilege event payloads; constant-time secret comparison. Every channel out of the trusted zone must be designed to leak nothing sensitive.

**8. Many security controls ARE patterns applied adversarially, and security largely converges with good design.** Proxy (access), Chain of Responsibility (defense-in-depth), Adapter (anti-corruption), State (rate-limit/circuit-break), immutability+DI (tamper-resistance), idempotency (replay defense). The deep engineer already has the structural vocabulary; security adds the adversarial mindset (threat modeling, STRIDE, assume-breach) and the threat knowledge. And the secure choice is overwhelmingly the *same* choice good design already recommended — because both demand clean, explicit, enforced boundaries, and security is the *minimum* over all of them, making the consistency patterns provide a security property in itself.

You can now examine any pattern, architecture, or system through the threat lens — locate its trust boundaries, identify where untrusted data crosses into trusted execution, recognize the injection/deserialization/access-control/leakage surfaces it creates or closes, and reach for the defensive patterns that enforce the foundational principles (defense in depth, least privilege, fail closed, complete mediation, secure by default). Most importantly, you understand that security is not adversarial *to* good design but *continuous with* it — the clean boundaries that make systems maintainable, performant, and comprehensible are the same boundaries that make them defensible.

Part 8 (Advanced Topics) will push to the frontier — edge cases, rare failure modes, the pattern combinations and tensions that arise at the limits, functional and reactive pattern paradigms, concurrency patterns, and the modern developments (and critiques) that are reshaping how patterns are understood.