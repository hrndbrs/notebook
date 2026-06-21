# PART 2 — CORE THEORY

## The Theoretical Machinery of Operating Systems

In Part 1 we built the bedrock: hardware, privilege modes, interrupts, and the process abstraction. Part 2 develops the *theory* — the conceptual machinery that every OS implements in some form. We will cover the four great pillars of OS theory: **process & thread management**, **CPU scheduling**, **concurrency & synchronization**, and **memory management (including virtual memory)**, plus **deadlock theory** and **I/O & file system theory**. For each, we develop internal mechanics, mental models, ASCII diagrams, and a clear-eyed accounting of advantages, disadvantages, and tradeoffs.

This is the longest and most important conceptual part of the course. Take it slowly.

---

## 10. Processes In Depth: States, Lifecycle, and the PCB

We introduced the process in Part 1. Now we develop its full theory.

### 10.1 The Process State Model

**WHAT:** At any instant, a process is in exactly one *state*. The OS tracks this state to decide what to do with the process. The classic model has these states:

```
                          ┌──────────┐
        admitted          │          │      exit
   ┌─────────────────────►│   NEW    │
   │  (process created,   │          │
   │   being set up)      └────┬─────┘
   │                           │ admitted to ready queue
   │                           ▼
   │                     ┌──────────┐   scheduler dispatch   ┌──────────┐
   │                     │          │ ─────────────────────► │          │
   │                     │  READY   │                        │ RUNNING  │
   │                     │          │ ◄───────────────────── │          │
   │                     └──────────┘   timer interrupt /    └────┬─────┘
   │                           ▲        preempted                 │
   │                           │                                  │ I/O or
   │           I/O complete /  │                                  │ event wait
   │           event occurs    │                                  ▼
   │                     ┌─────┴──────┐                     ┌──────────┐
   │                     │            │ ◄────────────────── │          │
   │                     │  BLOCKED   │                     │          │
   │                     │ (waiting)  │                     └──────────┘
   │                     └────────────┘
   │                                                        ┌──────────┐
   └────────────────────────────────────────────────────── │TERMINATED│
                                                            │ (zombie) │
                                                            └──────────┘
```

**The five states:**

- **NEW:** The process is being created — the OS is allocating its data structures but it isn't yet ready to run.
- **READY:** Runnable and waiting for a CPU. It has everything it needs *except* a turn on the processor. Multiple processes sit here in the *ready queue*.
- **RUNNING:** Currently executing on a CPU core. On an *N*-core machine, at most *N* processes are RUNNING at once.
- **BLOCKED (a.k.a. WAITING):** Cannot proceed until some event occurs — disk read completes, network packet arrives, a lock is released, a timer expires. *Crucially, a blocked process is not on the CPU and not competing for it.*
- **TERMINATED (a.k.a. ZOMBIE):** Finished execution but its bookkeeping entry persists briefly so the parent can read its exit status.

### 10.2 The Critical Transitions (and Why They Matter)

The *transitions* between states are where all the action is. Master these:

**READY → RUNNING (dispatch):** The scheduler picks a ready process and gives it a CPU. This is a context switch *into* the process.

**RUNNING → READY (preemption):** The timer interrupt fires (or a higher-priority process becomes ready), and the OS forcibly takes the CPU away. The process did nothing wrong; its time slice simply expired. *It's still perfectly able to run — it just lost its turn.* This is the transition that makes multitasking fair.

**RUNNING → BLOCKED (the most important one to understand deeply):** The process *voluntarily* gives up the CPU because it asked for something slow (e.g., "read this file") and cannot continue until it arrives. 

**WHY this transition is the heart of efficient computing:** Recall the storage hierarchy — a disk read takes ~10ms, during which the CPU could execute *tens of millions* of instructions. If the process held the CPU while waiting, all that capacity would be wasted. Instead, the OS moves it to BLOCKED and hands the CPU to a READY process. *This is the entire reason multiprogramming exists* (recall Era 3 in the history). The blocked process is "parked" until its data arrives.

**BLOCKED → READY (wakeup):** The awaited event occurs (the disk interrupt fires signaling the read is done). The process is now runnable again — but it doesn't immediately run; it rejoins the ready queue and waits its turn. *Note: it goes to READY, not RUNNING* — a subtle point that trips up beginners. Being woken doesn't mean being scheduled.

**Common misconception:** Students often conflate BLOCKED with READY ("both are just 'not running'"). The distinction is vital: a READY process *wants the CPU and could use it right now*; a BLOCKED process *cannot use the CPU even if offered one* because it's waiting for something external. The scheduler only ever picks from READY processes. Offering a CPU to a blocked process would be useless — it has nothing to do until its event arrives.

### 10.3 The Process Control Block (PCB), Fully Specified

**WHAT:** The PCB (in Linux, the `task_struct`) is the kernel's data structure representing one process — its complete identity and state. When we say "the OS tracks a process," we mean it maintains this structure. It typically contains:

```
   ┌────────────────────────────────────────────┐
   │              PROCESS CONTROL BLOCK           │
   ├────────────────────────────────────────────┤
   │  Identification:                             │
   │    • PID (process ID)                        │
   │    • PPID (parent's PID)                      │
   │    • owner (user ID, group ID)               │
   ├────────────────────────────────────────────┤
   │  CPU state (saved when not running):         │
   │    • Program Counter                         │
   │    • all general registers                   │
   │    • stack pointer                           │
   │    • CPU flags / status register             │
   ├────────────────────────────────────────────┤
   │  Scheduling info:                            │
   │    • state (READY/RUNNING/BLOCKED/...)       │
   │    • priority                                │
   │    • accumulated CPU time, time slice left   │
   ├────────────────────────────────────────────┤
   │  Memory management:                          │
   │    • pointer to page table (address space)   │
   │    • memory region descriptors               │
   ├────────────────────────────────────────────┤
   │  I/O & resources:                            │
   │    • open file descriptor table              │
   │    • working directory                       │
   │    • signal handlers                         │
   ├────────────────────────────────────────────┤
   │  Accounting & relationships:                 │
   │    • parent/children pointers                 │
   │    • exit status                             │
   └────────────────────────────────────────────┘
```

The PCB is *the* embodiment of the process. A context switch is, concretely, "save CPU state into process A's PCB, load CPU state from process B's PCB." The PCBs live in kernel memory, organized in queues (the ready queue, various blocked queues — one per event being waited on) and tables.

### 10.4 Process Creation: fork and exec

**WHY this design is worth studying:** UNIX's process-creation model is elegant, influential, and initially counterintuitive. It splits "make a new process" into two separate operations.

**`fork()`:** Creates a new process by *cloning the caller*. The new process (child) is a near-exact copy of the parent — same code, same data, same open files, same everything — except it gets a new PID. After `fork()`, *two* processes return from the single call, distinguished only by the return value: the child sees `0`, the parent sees the child's PID.

```
   Before fork():            After fork():

   ┌─────────────┐           ┌─────────────┐   ┌─────────────┐
   │   Parent    │           │   Parent    │   │   Child     │
   │   PID 100   │   ───►     │   PID 100   │   │   PID 101   │
   │             │           │ fork()→101  │   │ fork()→0    │
   └─────────────┘           └─────────────┘   └─────────────┘
                              (identical copies, run independently
                               from the point of the fork)
```

**`exec()`:** Replaces the current process's memory image with a *new program*. The PID stays the same, but the code, data, heap, and stack are wiped and reinitialized from a program file on disk. The process "becomes" a different program.

**The combo — how a shell launches a program:**

```
1. Shell calls fork()  → now two shells (parent + child).
2. Child calls exec("/bin/ls") → child's memory is replaced
   by the `ls` program; it's now running ls.
3. Parent calls wait() → blocks until the child finishes,
   then collects its exit status.
```

**WHY split it into two calls instead of one "spawn"?** The gap *between* fork and exec is a powerful design point: after forking but before exec-ing, the child can adjust its environment (redirect its output to a file, change permissions, close file descriptors) and those changes carry into the new program. This is how shell features like `ls > output.txt` (redirection) and `a | b` (pipes) are implemented cleanly. **Tradeoff:** elegance and flexibility vs. the apparent waste of copying a whole process only to immediately overwrite it. That waste is largely eliminated by *copy-on-write* (Section 19.6), a beautiful optimization we'll cover under memory. (Windows, by contrast, uses a single `CreateProcess` call — fewer steps, but less composable.)

**The zombie and orphan edge cases (failure modes worth knowing):**

- **Zombie:** A process that has terminated but whose parent hasn't yet called `wait()` to collect its status. It holds no resources except its PCB entry, which lingers so the exit status isn't lost. Too many zombies (from a buggy parent that never reaps) leak PCB slots.
- **Orphan:** A process whose parent died first. Orphans are "adopted" by the `init` process (PID 1), which dutifully reaps them. This prevents permanent zombies.

---

## 11. Threads: Concurrency Within a Process

### 11.1 The Motivation

**WHY threads exist:** A process gives isolation — its own address space, separate from all others. But isolation has a cost: processes can't easily share data, and creating/switching them is relatively heavy. Often you want *multiple concurrent activities that share the same memory* — e.g., a web server handling 1,000 connections, or a browser rendering a page while running JavaScript while fetching images. Spawning a whole isolated process per activity is wasteful when they all want to cooperate on the same data. **Threads** solve this.

### 11.2 What a Thread Is

**WHAT:** A thread is an independent stream of execution *within* a process. A process can contain multiple threads, all sharing the process's address space (code, data, heap) but each having its *own* stack and its *own* CPU state (PC, registers).

```
   ┌──────────────────────────────────────────────┐
   │                  PROCESS                        │
   │                                                 │
   │   SHARED among all threads:                     │
   │   ┌──────────┬──────────┬──────────┐           │
   │   │  Code    │  Data    │  Heap    │           │
   │   └──────────┴──────────┴──────────┘           │
   │                                                 │
   │   PRIVATE to each thread:                       │
   │   ┌─────────────┐  ┌─────────────┐              │
   │   │  Thread 1   │  │  Thread 2   │              │
   │   │  • own stack│  │  • own stack│              │
   │   │  • own PC   │  │  • own PC   │              │
   │   │  • own regs │  │  • own regs │              │
   │   └─────────────┘  └─────────────┘              │
   └──────────────────────────────────────────────┘
```

**WHY each thread needs its own stack:** The stack holds function-call frames and local variables — "where am I in my call chain and what are my locals." Two threads executing different functions simultaneously need separate call chains, hence separate stacks. But they *share* the heap and globals, which is exactly what lets them cooperate on shared data — and exactly what creates the danger of race conditions (Section 13).

### 11.3 Threads vs. Processes: The Tradeoff Table

| Property | Process | Thread |
|---|---|---|
| Address space | Private, isolated | Shared with siblings |
| Communication | Hard (needs IPC) | Easy (shared memory) |
| Creation cost | High | Low |
| Context-switch cost | Higher (swap address space, flush caches) | Lower (same address space) |
| Crash isolation | One crash doesn't kill others | One thread corrupting memory can crash the whole process |
| Safety | Strong (isolation) | Weak (shared mutable state ⇒ bugs) |

**The core tradeoff:** Threads buy you cheap communication and cheap switching at the cost of *isolation*. Because threads share memory, a bug in one thread can corrupt another's data, and concurrent access to shared data requires careful synchronization (the subject of Section 13). Processes are safer but heavier. This is one of the most consequential design choices in systems engineering: *isolation vs. efficiency*.

### 11.4 User-Level vs. Kernel-Level Threads

**WHERE are threads managed?** Two models, with a hybrid:

- **Kernel-level threads:** The OS kernel knows about and schedules each thread. **Advantage:** if one thread blocks (e.g., on I/O), the kernel can run another thread of the same process; true parallelism across cores is possible. **Disadvantage:** every thread operation (create, switch, sync) involves the kernel — a syscall crossing — so it's heavier.
- **User-level threads:** A library in user space manages threads; the kernel sees only one process. **Advantage:** ultra-fast thread operations (no kernel crossing). **Disadvantage (fatal flaw):** if one thread makes a blocking syscall, the kernel blocks the *entire process* (it doesn't know there are other threads to run), and the library can't use multiple cores.
- **Hybrid (M:N):** Map M user threads onto N kernel threads. Complex; tried and largely abandoned for general use, though it lives on in modern language runtimes' lightweight concurrency — Go's *goroutines* and Java's *virtual threads* are essentially sophisticated M:N user-scheduling over kernel threads. **Tradeoff:** combines the best of both but is notoriously hard to implement correctly.

**Why Hernando will care:** In Go (your trading bot, your SNAP API work), goroutines are M:N green threads multiplexed over an OS-thread pool by the Go runtime scheduler. Understanding this section is precisely understanding why a blocking syscall in one goroutine doesn't freeze your whole program (the runtime parks the goroutine and runs others on the same OS thread), and why CPU-bound goroutines need `GOMAXPROCS` cores to run in parallel. We'll connect this explicitly in later parts.

---

## 12. CPU Scheduling Theory

Now we confront the central *policy* question: given many READY processes and limited CPUs, *which one runs next, and for how long?* This is the **scheduling problem**, and it is pure policy sitting atop the context-switch mechanism (recall: policy vs. mechanism, theme #4).

### 12.1 Scheduling Goals (Which Are in Conflict)

Schedulers optimize several metrics, and **they conflict** — you cannot maximize all at once. This conflict *is* the subject:

- **Throughput:** Jobs completed per unit time. (Maximize.)
- **Turnaround time:** Time from job submission to completion. (Minimize.)
- **Response time:** Time from request to *first* response. (Minimize — this is what makes interactive systems feel snappy.)
- **Waiting time:** Total time spent in the READY queue. (Minimize.)
- **Fairness:** Every process gets a reasonable share; none starves.
- **CPU utilization:** Keep the CPU busy. (Maximize.)

**The central tension:** Optimizing throughput often means running long jobs to completion without interruption — but that murders response time for interactive users. Optimizing response time means switching constantly — but context-switch overhead reduces throughput. Optimizing average turnaround can starve some jobs (unfairness). *There is no universally optimal scheduler;* the right choice depends on the workload (batch server vs. interactive desktop vs. real-time controller). This is a recurring lesson: **scheduling is about tradeoffs, not a single right answer.**

### 12.2 Preemptive vs. Non-Preemptive

- **Non-preemptive (cooperative):** Once a process gets the CPU, it keeps it until it voluntarily yields (blocks or finishes). Simple, low-overhead, but one misbehaving process hangs everything. (Early Windows/Mac worked this way — recall a single frozen app freezing the machine.)
- **Preemptive:** The OS can forcibly take the CPU (via the timer interrupt). Necessary for responsiveness and fairness, but introduces the possibility of preemption mid-operation, which is the root of synchronization problems (Section 13).

### 12.3 The Classic Algorithms

We'll examine each with mechanics, an ASCII Gantt chart, and tradeoffs. Throughout, *arrival time* = when a job enters; *burst time* = CPU time it needs.

#### FCFS — First-Come, First-Served

**Mechanics:** A simple FIFO queue. Run jobs in arrival order, each to completion. Non-preemptive.

Example: jobs P1 (burst 24), P2 (burst 3), P3 (burst 3), all arriving ~together in that order.

```
 | P1                       | P2  | P3  |
 0                          24    27    30
 Wait times: P1=0, P2=24, P3=27.  Average wait = 17.
```

**The convoy effect (its fatal flaw):** A long job at the front makes everyone behind it wait, tanking average wait time. If the *same* jobs arrived as P2, P3, P1:

```
 | P2  | P3  | P1                       |
 0     3     6                          30
 Avg wait = (0+3+6)/3 = 3.  Six times better!
```

Same jobs, drastically different result based purely on order. **Advantage:** dead simple, fair in the "no starvation" sense. **Disadvantage:** terrible average wait when burst times vary; one big job convoys everyone.

#### SJF — Shortest Job First

**Mechanics:** Pick the READY job with the smallest burst time. **Provably optimal for minimizing average waiting time** (intuition: doing quick jobs first gets many jobs "out of the way," and each waiting job's pain is minimized).

```
 Same jobs (P1=24, P2=3, P3=3), SJF picks shortest first:
 | P2  | P3  | P1                       |
 0     3     6                          30
 Avg wait = 3.  Optimal.
```

**The two fatal flaws:**
1. **You can't know burst times in advance.** Real schedulers must *estimate* future burst length (typically via an exponential moving average of recent bursts). The estimate can be wrong.
2. **Starvation:** A steady stream of short jobs means a long job *never* runs. Unfair to long jobs.

**Preemptive variant — SRTF (Shortest Remaining Time First):** If a new job arrives with a shorter remaining time than the running job, preempt. Even better average wait, even worse starvation.

#### Round Robin (RR)

**Mechanics:** FCFS + preemption with a fixed *time quantum* (e.g., 10ms). Each job runs for up to one quantum, then goes to the back of the queue. Preemptive.

```
 P1=24, P2=3, P3=3, quantum=4:
 | P1 | P2 | P3 | P1 | P1 | P1 | P1 | P1 |
 0    4    7   10   14   18   22   26   30
 (P1 keeps coming back; P2,P3 finish early)
```

**The quantum tradeoff (a beautiful, concrete example of a design knob):**
- **Quantum too large** → RR degenerates into FCFS (each job finishes within its quantum); poor response time.
- **Quantum too small** → excellent responsiveness but context-switch overhead dominates (if quantum ≈ switch cost, you spend half your time switching!).
- **Rule of thumb:** quantum should be large relative to context-switch cost (so overhead is small) but small relative to typical burst (so it's responsive) — commonly 10–100ms.

**Advantage:** fair, responsive, no starvation, good for interactive/time-sharing. **Disadvantage:** worse average turnaround than SJF; performance sensitive to quantum tuning.

#### Priority Scheduling

**Mechanics:** Each process has a priority; run the highest-priority READY process. Can be preemptive or not. (SJF is priority scheduling where priority = 1/burst.)

**The fatal flaw — starvation:** Low-priority processes may never run. **The fix — aging:** gradually raise the priority of waiting processes so they eventually run. (Famous lore: when MIT shut down an IBM 7094 in 1973, they found a low-priority job submitted in 1967 that had never run.)

#### Multi-Level Feedback Queue (MLFQ) — How Real Schedulers Actually Work

**This is the most important practical algorithm** — the conceptual ancestor of real schedulers in Windows, macOS, and older Linux. It cleverly *approximates SJF without knowing burst times in advance*, by *learning* from observed behavior.

**Mechanics:** Multiple queues, each with a different priority and time quantum. Rules:

```
   ┌──────────────────────────────────────────┐
   │  Q0 (highest priority, smallest quantum)  │ ← new jobs start here
   │     ▼ if uses full quantum (CPU-bound)    │
   ├──────────────────────────────────────────┤
   │  Q1 (medium)                              │
   │     ▼ if uses full quantum                │
   ├──────────────────────────────────────────┤
   │  Q2 (lowest priority, largest quantum)    │ ← CPU-hogs sink here
   └──────────────────────────────────────────┘

   Run highest non-empty queue (RR within a queue).
   • A job that uses its WHOLE quantum (CPU-bound) → demoted.
   • A job that yields early for I/O (interactive)  → stays high.
   • Periodically, BOOST all jobs back to top (prevents
     starvation & adapts to behavior changes).
```

**WHY this is brilliant:** It *infers* job type from behavior. Interactive jobs (which block frequently for I/O — keyboard, network) naturally stay in high-priority queues and get snappy response. CPU-bound jobs (which burn full quanta) sink to low priority where they run during the gaps, getting good throughput without hurting interactivity. *It approximates "short jobs first" using observed behavior as a proxy for burst length* — no fortune-telling required. The periodic priority boost prevents starvation and lets a job that *changes* behavior (was CPU-bound, becomes interactive) get re-evaluated.

**Tradeoffs:** Many tunable parameters (number of queues, quanta, boost interval). Vulnerable to *gaming*: a process could issue a pointless I/O right before its quantum expires to stay high-priority — modern variants defend against this by accounting total CPU time used at a level rather than per-burst behavior.

#### Linux CFS — A Modern Real Scheduler (Briefly)

Linux long used the **Completely Fair Scheduler (CFS)**, which abandons fixed queues for an elegant idea: track each process's *virtual runtime* (vruntime — roughly, CPU time consumed, weighted by priority), keep processes in a red-black tree ordered by vruntime, and always run the one with the *smallest* vruntime (the one most "starved" of CPU). This naturally gives every process a fair share over time, with priorities ("nice" values) skewing the weighting. (As of recent kernels Linux has moved to **EEVDF**, a refinement adding better latency guarantees. We'll revisit real schedulers in Part 8.)

### 12.4 Scheduling Summary Table

| Algorithm | Preempt? | Optimizes | Fatal flaw | Use case |
|---|---|---|---|---|
| FCFS | No | Simplicity | Convoy effect | Batch, simple |
| SJF | No | Avg wait (optimal) | Can't predict; starvation | Theoretical baseline |
| SRTF | Yes | Avg wait | Worse starvation | Theoretical |
| RR | Yes | Response, fairness | Tuning-sensitive | Time-sharing |
| Priority | Either | Importance | Starvation | Real-time, systems |
| MLFQ | Yes | Balanced | Complexity, gaming | General-purpose OS |
| CFS/EEVDF | Yes | Fairness | Complexity | Modern Linux |

---

## 13. Concurrency and Synchronization — The Hardest Part of OS Theory

This is where operating systems gets genuinely hard, where the subtlest and most expensive bugs in all of software live. We proceed with extreme care.

### 13.1 The Root of All Evil: The Race Condition

**WHAT:** A *race condition* occurs when two or more threads access shared data concurrently, at least one writes, and the final result depends on the *unpredictable timing* of their execution.

**The canonical example.** Two threads each increment a shared counter `x` (currently 5). In your source code, `x = x + 1` looks atomic — one indivisible step. But the CPU executes it as *three* separate instructions:

```
   LOAD  R, [x]     ; read x into register
   ADD   R, R, 1    ; increment register
   STORE [x], R     ; write register back to x
```

Now suppose Thread A and Thread B both run `x++`, and the timer interrupt preempts A mid-sequence:

```
   Thread A              Thread B            x in memory
   ─────────             ─────────           ───────────
   LOAD R(A)=5                                   5
   ADD  R(A)=6                                   5
   ── PREEMPTED ──►
                         LOAD R(B)=5             5
                         ADD  R(B)=6             5
                         STORE x=6               6
   ◄── RESUMED ──
   STORE x=6                                     6   ← WRONG!
```

Two increments happened, but `x` is 6, not 7. **One update was lost.** And the bug is *nondeterministic* — it only manifests when preemption lands in the tiny window between LOAD and STORE. The program might work a million times and fail the millionth-and-first, on a customer's machine, unreproducibly. *This is why concurrency bugs are the most feared class of bug in production software.*

**Hernando-relevant:** This is precisely the bug class lurking in your spread-monitoring module and any concurrent balance update in the SNAP payment API. Two goroutines updating a shared order book, or two requests debiting the same account, are textbook race conditions. The rest of this section is your defense.

### 13.2 The Critical Section and the Requirements for a Solution

**WHAT:** A *critical section* is a region of code that accesses shared resources and must not be executed by more than one thread at a time. The shared-counter update above is a critical section.

**The goal — mutual exclusion:** Ensure that when one thread is in the critical section, all others are excluded. A correct solution must satisfy three properties (Dijkstra's criteria):

1. **Mutual exclusion:** At most one thread in the critical section at a time. (Safety — nothing bad happens.)
2. **Progress:** If no thread is in the critical section and some want in, one must be allowed in (no needless blocking). (Liveness — something good eventually happens.)
3. **Bounded waiting:** A thread wanting in must eventually get in (no starvation).

Plus a practical fourth: it should work regardless of relative thread speeds or number of CPUs, and be efficient.

### 13.3 Building Blocks: From Atomic Hardware to Locks

**The chicken-and-egg problem:** To protect shared data we need a lock; but a lock is *itself* shared data, so protecting *it* seems to require another lock, ad infinitum. The recursion bottoms out in **hardware atomic instructions** — operations the CPU guarantees are indivisible (cannot be interrupted partway).

**Test-and-Set (the foundational atomic primitive):** A single hardware instruction that atomically reads a memory location *and* sets it to 1, returning the old value:

```
   // executed ATOMICALLY by hardware — cannot be split
   bool TestAndSet(bool *lock) {
       bool old = *lock;
       *lock = true;
       return old;
   }
```

With it, a **spinlock**:

```
   acquire(lock):
       while (TestAndSet(lock) == true)
           ;   // spin: keep trying until we get false (was unlocked)
       // we now hold the lock
   release(lock):
       *lock = false;
```

**WHY it works:** Only one thread can get the `false` (unlocked) return; the atomicity guarantees no two threads both see "unlocked" and both proceed. **The "spin":** if the lock is held, the thread loops, *busy-waiting*.

**The spinlock tradeoff:** Spinning *burns CPU* doing nothing while waiting. This is acceptable only when the wait is expected to be *very short* (shorter than the cost of a context switch) — e.g., inside the kernel protecting a brief critical section on a multi-core machine. For longer waits, spinning is wasteful; you want the thread to *block* (sleep) instead, which leads us to higher-level primitives. **Other atomics:** *Compare-and-Swap (CAS)* — "if this location equals expected, set it to new; report success" — is the workhorse of modern *lock-free* programming (Section 13.7) and underlies most language atomic types, including Go's `sync/atomic`.

### 13.4 Mutexes and the Sleep/Wakeup Distinction

**WHAT:** A *mutex* (mutual-exclusion lock) is the standard high-level lock. Unlike a spinlock, when a thread can't acquire a mutex, the OS puts it to **sleep** (moves it to BLOCKED), and wakes it when the lock frees. No CPU is burned while waiting.

```
   lock(m):  if free, take it; else BLOCK (sleep) until woken
   unlock(m): release; wake one waiting thread (→ READY)
```

**Spinlock vs. mutex — the core tradeoff:** Spinlock wastes CPU but avoids the context-switch cost (good for ultra-short critical sections, kernel internals). Mutex pays the context-switch cost but frees the CPU for others (good for anything longer). Choosing wrong is a real performance bug: a spinlock around a slow operation wastes massive CPU; a mutex around a 3-instruction operation pays more in switching than it protects. Some real mutexes are *adaptive* — spin briefly, then sleep if still waiting.

### 13.5 Semaphores — Generalizing Beyond Mutual Exclusion

**WHAT:** A *semaphore* (Dijkstra, 1965) is an integer counter with two atomic operations:

```
   wait(S)   (a.k.a. P, "down"):  while S<=0: block;  then S--
   signal(S) (a.k.a. V, "up"):    S++;  wake a waiter if any
```

**Two uses:**

1. **Binary semaphore (counter 0/1):** Acts like a mutex — mutual exclusion.
2. **Counting semaphore (counter N):** Allows up to *N* threads through — perfect for managing a pool of *N* identical resources (e.g., "10 database connections available"). `wait` to grab one (blocking if none free), `signal` to return one.

**The deeper power — ordering/signaling between threads:** Semaphores can enforce *ordering*, not just exclusion. Initialize a semaphore to 0; thread B does `wait(S)` (blocks immediately since S=0); thread A does `signal(S)` when it's done with some prerequisite, unblocking B. This guarantees "B's code runs only after A's signal" — solving coordination, not just contention.

**The producer-consumer problem (the canonical synchronization pattern — you will implement variants of this constantly):**

```
   Shared: a bounded buffer of size N.
   Producers add items; consumers remove them.
   Constraints: don't add to a full buffer; don't remove
                from an empty one; don't corrupt the buffer.

   Three semaphores:
     empty = N   // count of empty slots
     full  = 0   // count of filled slots
     mutex = 1   // protects buffer structure itself

   Producer:                    Consumer:
     wait(empty)  // need slot     wait(full)   // need item
     wait(mutex)                   wait(mutex)
       add item                      remove item
     signal(mutex)                 signal(mutex)
     signal(full) // one more item  signal(empty)// one more slot
```

**WHY the ordering matters (a critical, classic bug):** You must acquire `empty`/`full` *before* `mutex`. Reverse it — grab `mutex` then wait on `full` — and a consumer holding the mutex on an empty buffer sleeps forever while producers can't get the mutex to add anything. **Deadlock** (Section 14). This ordering subtlety is exactly why semaphores, though powerful, are *error-prone* — a misplaced operation causes deadlock or races, and the compiler can't catch it.

### 13.6 Monitors and Condition Variables — Safer Abstractions

**WHY they exist:** Semaphores are powerful but unstructured — easy to misuse (forget a `signal`, reverse an order, double-`wait`). **Monitors** (Hoare, 1974) bundle shared data *with* the procedures that access it, automatically enforcing that only one thread executes inside the monitor at a time. They're built into languages: Java's `synchronized`, C#'s `lock`, and condition-variable patterns in pthreads and Go.

**Condition variables:** Let a thread *wait inside* a monitor for a condition to become true, releasing the monitor lock while it waits (so others can make the condition true) and reacquiring it on wakeup:

```
   wait(cond, lock):   atomically release lock + sleep;
                       on wake, reacquire lock
   signal(cond):       wake one thread waiting on cond
   broadcast(cond):    wake all threads waiting on cond
```

**The mandatory idiom — always wait in a `while` loop, never an `if`:**

```
   lock(m)
   while (!condition_is_true)    // ← WHILE, not IF
       wait(cond, m)
   ... use the resource ...
   unlock(m)
```

**WHY `while` and not `if`:** Between being signaled and reacquiring the lock, *another* thread might have slipped in and changed the condition back (or a "spurious wakeup" might have occurred). Re-checking in a loop guarantees you only proceed when the condition truly holds. *This is one of the most common concurrency bugs in real code* — using `if` instead of `while`. Burn it into memory.

### 13.7 Lock-Free Programming and Memory Models (Advanced Preview)

**WHAT:** Locks have costs (contention, blocking, deadlock risk). *Lock-free* data structures use atomic primitives (especially CAS) to coordinate without locks, guaranteeing *some* thread always makes progress.

```
   // Lock-free increment via Compare-And-Swap:
   do {
       old = x;
       new = old + 1;
   } while (!CompareAndSwap(&x, old, new));
   // retry until no one changed x between our read and write
```

**Memory models and reordering (the deep, surprising part):** Modern CPUs and compilers *reorder* memory operations for speed. Two threads can observe each other's writes in *different orders* than the source code suggests. This means even seemingly correct lock-free code can break without **memory barriers/fences** that enforce ordering. This is genuinely one of the hardest topics in computing — even experts get it wrong. **Tradeoff:** lock-free code can be dramatically faster under contention and immune to deadlock, but is far harder to write correctly and reason about. The pragmatic industry wisdom: *use locks unless profiling proves you need lock-free, and even then prefer well-tested library implementations over hand-rolled.* (Go's memory model, which governs your bot's correctness, is precisely this topic; the `sync` package and channels exist so you rarely have to reason at the barrier level.)

---

## 14. Deadlock Theory

Synchronization prevents races but introduces a new failure mode: threads can get permanently stuck waiting for each other. This is **deadlock**, and it has a clean, complete theory.

### 14.1 What Deadlock Is

**WHAT:** A set of processes is deadlocked when each is waiting for a resource held by another in the set — a cycle of waiting that can never resolve.

**The intuitive image (Coffman's "deadly embrace"):** Two friends at dinner, one fork each, both needing two forks to eat. Each holds one and waits for the other's. Both wait forever.

```
   Thread A holds Lock 1, wants Lock 2.
   Thread B holds Lock 2, wants Lock 1.

        ┌─────────┐   wants    ┌─────────┐
        │ Thread A │ ─────────► │ Lock 2  │
        └─────────┘            └─────────┘
            ▲ holds                 │ held by
            │                       ▼
        ┌─────────┐   wants    ┌─────────┐
        │ Lock 1  │ ◄───────── │ Thread B │
        └─────────┘            └─────────┘
                  (a cycle ⇒ deadlock)
```

### 14.2 The Four Necessary Conditions (Coffman Conditions)

Deadlock can occur **only if all four** hold simultaneously. This is powerful: *break any one and deadlock becomes impossible.*

1. **Mutual exclusion:** Resources can't be shared; only one holder at a time.
2. **Hold and wait:** A process holding resources can request more while holding.
3. **No preemption:** Resources can't be forcibly taken; only voluntarily released.
4. **Circular wait:** A closed chain of processes, each waiting for one held by the next.

### 14.3 The Four Strategies

**(1) Prevention — make one Coffman condition impossible by design:**
- Break *circular wait* (the most practical): impose a **global ordering** on locks and require all threads acquire locks in that order. A cycle becomes impossible because you can't have A-then-B and B-then-A. *This is the single most useful deadlock-fighting technique in real code.* (In your payment API: always lock accounts in, say, ascending account-ID order when transferring between two, and you can never deadlock two opposing transfers.)
- Break *hold-and-wait:* acquire all needed resources at once, or none. (Hurts concurrency.)
- Break *no preemption:* allow taking resources back. (Often impossible — can't un-print a page.)

**(2) Avoidance — stay in "safe" states:** The **Banker's Algorithm** (Dijkstra) grants a request only if the system can still guarantee everyone *could* eventually finish (a "safe state"). **Tradeoff:** requires knowing maximum resource needs in advance and is computationally expensive; rarely used in practice outside specialized systems.

**(3) Detection and recovery:** Allow deadlocks, periodically detect them (find a cycle in the resource-allocation graph), then recover by killing/restarting a process or forcibly preempting a resource. Used by some databases (which detect deadlock among transactions and abort a "victim" — you'll see "deadlock detected, transaction rolled back" errors in PostgreSQL, directly relevant to your backend work).

**(4) Ignore it (the "ostrich algorithm"):** Pretend deadlocks can't happen; if the system hangs, reboot. **Surprisingly, this is what general-purpose OSes like Linux and Windows mostly do** for application-level deadlocks. **WHY:** deadlocks are rare in practice, and the overhead of prevention/avoidance/detection isn't worth it for the common case. *A profound engineering lesson: sometimes the right response to a rare failure is to not pay a constant tax to prevent it.* This is the policy-vs.-cost reasoning that distinguishes textbook completeness from production pragmatism.

### 14.4 Related Hazards

- **Livelock:** Threads aren't blocked but make no progress — like two people stepping side-to-side in a hallway, each politely moving the same way. They're *active* but stuck. (Can arise from naive deadlock-avoidance retries.)
- **Starvation:** A thread waits indefinitely because others keep being favored (recall priority scheduling). Not a deadlock — the system progresses — but unfair to the victim.
- **Priority inversion:** A high-priority thread waits on a lock held by a low-priority thread, which is itself preempted by medium-priority threads — so "high" effectively waits behind "medium." This famously nearly doomed NASA's 1997 Mars Pathfinder mission. **Fix:** *priority inheritance* — the lock-holder temporarily inherits the waiter's high priority so it can finish and release. A superb real-world example of theory meeting a $150M spacecraft.

---

## 15. Memory Management Theory

We now turn to the second great resource the OS virtualizes: memory. This is among the most elegant and important bodies of theory in all of computer science.

### 15.1 The Problems Memory Management Solves

Recall the illusionist OS. For memory, it must create the illusion that *each process has its own large, private, contiguous memory*, while in reality physical RAM is shared, finite, and fragmented. Specifically it must solve:

- **Relocation:** Programs are written without knowing where in physical RAM they'll land. Addresses must be translatable.
- **Protection:** Process A must not read or write process B's memory.
- **Sharing:** Yet sometimes processes *should* share memory (shared libraries, IPC) — controlled sharing.
- **Capacity:** Run programs needing more memory than physically exists.

### 15.2 The Foundational Idea: Address Spaces and Virtual Memory

**WHAT:** Each process is given a **virtual address space** — its own private range of addresses (e.g., 0 to 2⁶⁴−1 on a 64-bit machine) — as if it owned all of memory. The addresses the program uses (*virtual addresses*) are *not* real RAM locations; the OS+hardware *translate* every virtual address to a *physical address* on the fly.

```
   Process A's view          Translation          Physical RAM
   (virtual addresses)        (OS + MMU)           (real)
   ┌──────────────┐                              ┌──────────────┐
   │ 0x0000...    │ ───┐                          │  ...         │
   │ ...          │    │      ┌──────────┐        │  A's page    │
   │ 0x4000 (code)│ ───┼────► │   MMU    │ ─────► │  B's page    │
   │ ...          │    │      │ (maps V  │        │  A's page    │
   │ 0xFFFF...    │ ───┘      │  → P)    │        │  free        │
   └──────────────┘           └──────────┘        │  B's page    │
   Process B has an                                │  ...         │
   IDENTICAL-looking                               └──────────────┘
   virtual space, mapped
   to DIFFERENT physical RAM.
```

**This is theme #2 (indirection) in its grandest form.** Adding a translation layer between "addresses programs use" and "real RAM" solves relocation (the map can point anywhere), protection (A's map simply has no entry pointing to B's RAM — A *cannot even name* B's memory), sharing (two maps can point to the same physical page), and capacity (the map can say "this page is on disk, not RAM" — virtual memory proper).

### 15.3 The Mechanism: Paging

**WHAT:** Paging divides virtual memory into fixed-size **pages** (typically 4 KB) and physical memory into same-sized **frames**. The OS maintains a **page table** per process mapping each virtual page → a physical frame (or "not present").

**How a virtual address is translated:** Split the address into a *page number* (high bits) and an *offset* within the page (low bits). Use the page number to index the page table, get the physical frame number, and combine with the offset:

```
   Virtual address (e.g., 32-bit, 4KB pages):
   ┌──────────────────────┬──────────────────┐
   │  Page number (20 bit)│  Offset (12 bit)  │
   └──────────┬───────────┴────────┬─────────┘
              │                     │
              ▼                     │
        ┌───────────┐               │
        │ Page Table│               │
        │  [page#]  │ → frame#      │
        └─────┬─────┘               │
              ▼                     ▼
   ┌──────────────────────┬──────────────────┐
   │  Frame number        │  Offset (same)    │  ← physical address
   └──────────────────────┴──────────────────┘
```

(The offset passes through unchanged because pages and frames are the same size; only the page-to-frame mapping needs lookup.)

**WHY paging beats earlier schemes:** Before paging, systems allocated memory in *variable-size contiguous chunks*, which caused **external fragmentation** — free memory scattered in unusable small gaps between allocations (like a parking lot with single empty spaces between cars, no room for a bus despite plenty of total free space). Fixed-size pages eliminate external fragmentation: *any* free frame fits *any* page. The cost is minor **internal fragmentation** — the last page of a process is usually partly empty (average half a page wasted per memory region). A clean, favorable tradeoff.

### 15.4 The Page Table Problem and the TLB

**The space problem:** A 64-bit address space with 4 KB pages has an astronomical number of pages — a flat page table would be larger than RAM itself. **Solution: multi-level (hierarchical) page tables** — a tree of page tables where only the portions actually used are allocated. Most of the vast address space is unused, so most of the tree is absent. (x86-64 uses 4 or 5 levels.)

**The speed problem:** Every memory access now requires *extra* memory accesses to walk the page table (4–5 levels = 4–5 extra reads per access!). This would make every memory reference catastrophically slow. **Solution: the TLB (Translation Lookaside Buffer)** — a small, ultra-fast hardware cache *inside the CPU* holding recently used virtual→physical translations.

```
   CPU needs to translate virtual page V:
     • Check TLB (tiny, fast).
         HIT  → got the frame instantly (~1 cycle). Common case.
         MISS → walk the page table (slow), then cache result in TLB.
```

**WHY the TLB works:** Locality of reference again (theme #3) — programs repeatedly touch the same handful of pages, so a small TLB (64–1500 entries) achieves very high hit rates (often >99%). The TLB is one of the most important structures in the machine; without it, virtual memory would be unbearably slow. **Tradeoff/cost:** on a context switch, translations belong to the old process and may be invalid for the new one, so the TLB must be flushed or tagged — a hidden cost of context switching we flagged in Part 1. (Modern CPUs tag TLB entries with an address-space ID to avoid full flushes.)

### 15.5 Demand Paging and the Page Fault

**WHAT — virtual memory's killer feature:** Not all of a process's pages need to be in RAM at once. The OS loads pages **on demand** — only when actually accessed. A page table entry can be marked "not present" (the page is on disk, or never allocated).

**The page fault (a controlled, normal use of the exception mechanism from Part 1):**

```
1. Program accesses a virtual address whose page is "not present."
2. The MMU raises a PAGE FAULT (a trap → kernel mode).
3. The kernel's page-fault handler runs:
     a. Is this a legal access? (within an allocated region?)
        If not → segmentation fault, kill the process.
     b. If legal: find a free frame (or evict one — §15.6),
        read the needed page from disk into it,
        update the page table to "present."
4. Return from the fault; re-execute the faulting instruction,
   which now succeeds. The program never knew it paused.
```

**This is a profound payoff of the foundations:** the *same* trap mechanism that catches illegal accesses (errors) also implements demand paging (a feature) and lazy allocation. One mechanism, dual purpose. **Benefits:** programs start fast (load only what's touched), can be larger than RAM, and memory is used efficiently. **Cost:** the first access to each page incurs a slow disk read — but locality means this is rare after warmup.

### 15.6 Page Replacement: When RAM Is Full

When a page must be brought in but no frame is free, the OS must **evict** a resident page to disk. *Which one?* This is the **page replacement** problem — and it's the same conceptual problem as cache eviction (theme #3 again).

- **OPT (optimal):** Evict the page that won't be used for the longest time into the future. *Provably optimal but unimplementable* (requires knowing the future). Used only as a theoretical benchmark.
- **FIFO:** Evict the oldest-loaded page. Simple but dumb — can evict a hot, frequently-used page just because it's old. Suffers **Belady's anomaly**: *more frames can paradoxically cause more faults* — a famous counterintuitive result showing FIFO isn't well-behaved.
- **LRU (Least Recently Used):** Evict the page unused for the longest *past* time, betting (via locality) the past predicts the future. Excellent in practice but expensive to implement exactly (needs a timestamp or ordering updated on *every* access).
- **Clock (Second-Chance) — what real systems actually use:** An efficient LRU *approximation*. Pages sit in a circular list; each has a "referenced" bit set by hardware on access. A clock hand sweeps: if a page's bit is 1, clear it and give a "second chance" (skip); if 0, evict it. Approximates LRU at a fraction of the cost.

**Thrashing (a critical failure mode):** If the combined working sets of running processes exceed RAM, the system spends nearly all its time paging in/out instead of computing — performance collapses catastrophically. 

```
   Throughput
     ▲
     │        ╭──────╮
     │       ╱        ╲  ← cliff! thrashing
     │      ╱          ╲___________
     │     ╱
     │____╱
     └────────────────────────────► degree of multiprogramming
              (too many processes ⇒ working sets don't fit ⇒ collapse)
```

**The working set model:** A process's *working set* is the set of pages it's actively using in a recent time window. If the OS ensures each running process's working set fits in RAM (admitting fewer processes if needed), thrashing is avoided. **This is why a machine that's a bit short on RAM gets slow, but a machine that's badly short gets *catastrophically* slow** — you've fallen off the thrashing cliff. (Practical signal: when a server's load spikes and it starts swapping heavily, latency explodes nonlinearly — directly relevant to operating your trading bot and any production backend.)

### 15.7 Segmentation (Briefly, for Completeness)

**WHAT:** An alternative/complement to paging that divides memory into variable-size, *logically meaningful* segments (code segment, stack segment, heap segment) rather than fixed pages. **Advantage:** matches the program's logical structure, natural protection per segment. **Disadvantage:** reintroduces external fragmentation (variable sizes). Modern systems mostly use paging, sometimes with a thin segmentation layer (x86 historically combined both). The conceptual takeaway: *fixed-size units (pages) trade a little internal waste for no external fragmentation; variable-size units (segments) do the reverse.*

---

## 16. I/O and File System Theory

The last great pillar: how the OS manages devices and presents the **file** abstraction.

### 16.1 The I/O Problem and Its Abstractions

**WHY it's hard:** Devices are wildly diverse (disks, keyboards, networks, GPUs), slow and variable, and each has its own quirky control interface. The OS must present a uniform, simple interface over this chaos.

**The layered solution:**

```
   ┌─────────────────────────────────────┐
   │  Application: read(), write()        │  uniform API
   ├─────────────────────────────────────┤
   │  File system / device-independent    │  buffering, caching
   │  OS I/O layer                        │
   ├─────────────────────────────────────┤
   │  Device drivers                      │  device-specific code,
   │  (one per device type)               │  uniform interface up
   ├─────────────────────────────────────┤
   │  Hardware (device controllers)       │
   └─────────────────────────────────────┘
```

**The device driver — a key abstraction:** A *driver* is OS code that knows the gory specifics of one device but exposes a *standard interface* upward. This is why you can `read()` from a file on an SSD, a USB stick, or a network drive with identical code — different drivers, same interface. (Drivers are also, notably, the largest source of OS bugs and security holes, since they run in kernel mode but are often written by hardware vendors — a Part 7 concern.)

### 16.2 Three Ways to Do I/O (Performance Tradeoffs)

1. **Polling (programmed I/O):** The CPU repeatedly checks "are you done yet?" in a loop. Simple but wastes CPU busy-waiting. Acceptable only for very fast devices or when a response is imminent.
2. **Interrupt-driven I/O:** The CPU issues the request and goes off to do other work; the device raises an *interrupt* (Part 1!) when done. Efficient for slow devices — no busy-waiting. But one interrupt per data unit is costly for high-throughput devices.
3. **DMA (Direct Memory Access):** A dedicated DMA controller transfers data directly between device and RAM *without the CPU*, interrupting the CPU only once when the *whole* transfer completes. **Essential for high-bandwidth I/O** (disk, network, GPU) — the CPU sets up the transfer and is then free. Without DMA, moving a gigabyte would consume the CPU; with it, the CPU is nearly idle during the transfer.

**The throughline:** these three are a progression of *removing the CPU from the data path* — from total involvement (polling) to one-interrupt-per-chunk (interrupt-driven) to near-zero involvement (DMA). A clean illustration of offloading work to specialized hardware.

### 16.3 The File Abstraction

**WHAT:** A *file* is a named, persistent, ordered sequence of bytes. This abstraction (UNIX's great contribution) hides everything about the underlying storage — sectors, cylinders, flash blocks, network protocols — behind a handful of operations: `open`, `read`, `write`, `seek`, `close`.

**The file descriptor:** When you `open` a file, the OS returns a small integer — a *file descriptor* — that indexes into the process's open-file table. Subsequent operations use this handle. **The famous UNIX unification: "everything is a file."** Devices, pipes, network sockets, even kernel data are exposed through the *same* file-descriptor interface. You `read()` from a keyboard, a disk file, or a network socket with the same syscall. This radical uniformity is why UNIX-style tools compose so beautifully (theme: one good abstraction applied everywhere).

### 16.4 File System Internals

**WHAT a file system must track:** Given a disk that's just a giant array of numbered fixed-size *blocks*, the file system imposes structure: which blocks belong to which file, the directory hierarchy, free vs. used blocks, and metadata (size, owner, permissions, timestamps).

**The inode (index node):** The central data structure (in UNIX-family file systems). Each file has an inode storing its metadata *and the locations of its data blocks*. Directories are just special files mapping names → inode numbers.

```
   ┌──────────────────────────────────┐
   │  INODE (one per file)            │
   │  • permissions, owner, size      │
   │  • timestamps                    │
   │  • pointers to data blocks:      │
   │    ┌──────────────────────────┐  │
   │    │ direct ptr → data block 0│  │
   │    │ direct ptr → data block 1│  │
   │    │ ...                      │  │
   │    │ single-indirect ─► block │  │  ← block full of
   │    │   of more pointers       │  │     more pointers
   │    │ double-indirect ─► block │  │  ← for large files
   │    │   of blocks of pointers  │  │
   │    └──────────────────────────┘  │
   └──────────────────────────────────┘
```

**WHY the indirect-pointer scheme (a clever space/scalability tradeoff):** Most files are small, so a few *direct* pointers handle them with zero overhead. Rare large files use *indirect* blocks (pointers to pointers), scaling to huge sizes without bloating the common small-file case. The structure pays cost proportional to file size — small files stay cheap, large files remain possible.

**Free space management:** Bitmaps (one bit per block: free/used) or free lists track available blocks. **Directory structure:** a tree (actually a directed graph with hard links) mapping human names to inodes.

### 16.5 Consistency, Journaling, and Crash Recovery

**The crash problem:** A file operation often requires *multiple* block writes (update the inode, update the free bitmap, write the data). If the machine crashes *between* these writes, the file system is left **inconsistent** (e.g., a block marked used but belonging to no file, or vice versa). 

**Journaling (the dominant solution):** Before performing the real updates, write the intended changes to a *journal* (a log) on disk. Then apply them. After a crash, the OS *replays* the journal to finish or undo partial operations, restoring consistency. **This is the same idea as database write-ahead logging** (directly relevant to your PostgreSQL and payment-API work — atomicity and durability via logging is *the* universal technique). **Tradeoff:** journaling costs extra writes (you write changes twice — journal then final), so file systems offer modes: journal *everything* (safest, slowest) vs. journal *metadata only* (faster, data could still be lost). A direct safety-vs-performance knob.

**Copy-on-write file systems (modern approach):** ZFS, Btrfs never overwrite data in place; they write new versions to new locations and atomically flip a pointer. This gives crash consistency "for free" plus cheap snapshots. (Same copy-on-write idea we'll meet in memory — theme reuse.)

---

## 17. Part 2 Synthesis: How the Pillars Interlock

The four pillars are not independent — they form one machine:

```
   A process RUNS on the CPU (scheduling) within its
   ADDRESS SPACE (memory mgmt). It BLOCKS on I/O (file/IO
   theory), letting the scheduler run another process.
   Its THREADS share that address space and must
   SYNCHRONIZE (concurrency) to avoid corrupting it, while
   avoiding DEADLOCK. The whole dance is orchestrated by
   INTERRUPTS (Part 1) and protected by USER/KERNEL mode.
```

Every concept rests on the foundations: scheduling needs the timer interrupt; memory protection needs privilege modes; I/O needs interrupts and DMA; synchronization needs atomic hardware instructions. *The theory is the foundations, elaborated.*

---

## 18. Part 2 Self-Check

Explain each aloud in your own words:

1. The five process states and every transition between them — especially why BLOCKED→READY (not →RUNNING).
2. Why `fork`/`exec` are split, and what zombies and orphans are.
3. Threads vs. processes — the precise isolation/efficiency tradeoff.
4. Why scheduling goals conflict; trace FCFS, SJF, RR, and explain *why MLFQ approximates SJF without knowing the future*.
5. Walk through the lost-update race condition at the instruction level.
6. Mutual exclusion, the three correctness requirements, and why spinlock vs. mutex is a real tradeoff.
7. Counting semaphores and the producer-consumer pattern, including why lock ordering matters.
8. Why condition-variable waits must use `while`, not `if`.
9. The four Coffman conditions and a prevention strategy that breaks circular wait.
10. Virtual→physical translation via paging; the role of the TLB; why multi-level page tables exist.
11. The page-fault sequence, and how one mechanism serves both errors and demand paging.
12. Thrashing and the working-set model — why "a bit short" on RAM differs from "badly short."
13. Polling vs. interrupt-driven vs. DMA, and the inode indirect-pointer scheme.
14. Journaling and its connection to database write-ahead logging.

---

That completes **Part 2 — Core Theory** at full depth. We've built the four great pillars — processes/threads, scheduling, concurrency/synchronization (with deadlock), and memory management — plus I/O and file systems, all resting on Part 1's foundations. You now hold the complete conceptual machinery of an operating system.