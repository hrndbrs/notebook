# PART 1 — FOUNDATIONS

## Operating Systems: From Absolute Zero to First Principles

---

## 0. Before We Begin: What Is a Computer, Really?

Before we can talk about an operating system, we must understand what it operates. You cannot understand the manager without understanding the thing being managed.

### 0.1 The Definition

**WHAT:** A computer is a machine that performs computation by manipulating symbols according to rules, automatically, at high speed. Concretely, a modern computer is a collection of physical devices — a processor, memory, and input/output devices — wired together so that the processor can read instructions and data from memory, transform that data, and write results back.

**Mental model:** Picture a very fast, very literal, very stupid clerk sitting at a desk. The clerk has:

- A tiny notepad with a few slots for numbers (registers)
- A giant wall of numbered mailboxes, each holding one number (memory)
- A rulebook that says "look at the mailbox the instruction points to, do exactly what it says, then move to the next instruction" (the fetch-decode-execute cycle)

The clerk cannot get bored, cannot improvise, and cannot understand meaning. The clerk only follows the rulebook, billions of times per second. Everything a computer does — rendering a video, running an AI model, flying a spacecraft — is this clerk doing arithmetic and moving numbers between mailboxes very, very fast.

**Intuition:** The profound and counterintuitive truth of computing is that *meaning is not in the machine*. The machine moves numbers. We humans assign meaning to those numbers: "these 8 bits represent the letter A," "this number represents a pixel's redness," "this number is a memory address." The computer does not know or care. This separation between mechanism (moving numbers) and meaning (our interpretation) is the deepest idea in all of computer science, and the operating system is built entirely on top of it.

### 0.2 Historical Background

**WHY this matters historically:** The idea of a programmable machine predates electronics. In 1837, Charles Babbage designed the Analytical Engine, a mechanical computer with a "store" (memory) and a "mill" (processor) — the same separation we use today. Ada Lovelace, writing notes on it, understood that such a machine could manipulate *any* symbols, not just numbers — music, logic, anything. She saw the abstraction before the hardware existed.

The decisive theoretical moment came in 1936, when Alan Turing described an abstract machine (now the "Turing machine") with an infinite tape of cells, a head that reads and writes symbols, and a table of rules. Turing proved that this absurdly simple device could compute anything that is computable at all. This matters enormously for operating systems because it tells us the hardware can be dead simple, and *all the richness comes from the program* — the rules. An OS is, at its heart, a very sophisticated program that makes simple hardware usable.

In 1945, John von Neumann described an architecture (the "von Neumann architecture") where instructions and data live in the *same* memory. This is the model nearly every computer uses today, and understanding it is non-negotiable for understanding operating systems.

### 0.3 The von Neumann Architecture (The Bedrock)

This is the single most important diagram in this entire course. Memorize it.

```
        ┌─────────────────────────────────────────────┐
        │                    CPU                        │
        │   ┌──────────────┐      ┌─────────────────┐   │
        │   │ Control Unit │      │       ALU       │   │
        │   │ (decides what│◄────►│ (does the math: │   │
        │   │  to do next) │      │  add, compare)  │   │
        │   └──────────────┘      └─────────────────┘   │
        │   ┌──────────────────────────────────────┐    │
        │   │            Registers                 │    │
        │   │ (tiny, ultra-fast scratchpad slots)  │    │
        │   └──────────────────────────────────────┘    │
        └───────────────────────┬───────────────────────┘
                                │  System Bus
                                │  (wires carrying
                                │   addresses, data,
                                │   control signals)
        ┌───────────────────────┴───────────────────────┐
        │                                                 │
   ┌────┴──────────┐                          ┌──────────┴────┐
   │    MEMORY     │                          │  I/O DEVICES  │
   │ (RAM: holds   │                          │ (disk, keyboard│
   │  BOTH program │                          │  network, GPU) │
   │  instructions │                          └────────────────┘
   │  AND data)    │
   └───────────────┘
```

**WHAT each part does:**

- **CPU (Central Processing Unit):** The clerk. Executes instructions.
  - **Control Unit:** Decides which instruction to do next, and orchestrates the others.
  - **ALU (Arithmetic Logic Unit):** Does actual computation — addition, subtraction, AND, OR, comparison.
  - **Registers:** A handful (typically 16–32 on modern CPUs) of extremely fast storage slots *inside* the CPU. These are where the actual work happens. Reading a register takes essentially zero time; reading main memory takes ~100x longer.
- **Memory (RAM):** The wall of numbered mailboxes. Holds both the program's instructions and its data, mixed together, distinguished only by how the CPU uses them.
- **I/O Devices:** Everything else — the disk, keyboard, mouse, screen, network card, GPU. These connect the sealed CPU-and-memory world to the outside.
- **The Bus:** The bundle of wires connecting everything. Data, memory addresses, and control signals travel along it.

**WHY it's designed this way:** Storing instructions in the same memory as data was revolutionary because it means a program can be *changed like data*. You can load different programs into the same machine without rewiring it. This is why your laptop can run a browser one second and a game the next — programs are just numbers in memory, and the OS's job (foreshadowing!) is to load the right numbers and point the CPU at them.

**Common misconception:** Beginners often think the CPU "knows" what a program is doing. It does not. The CPU sees a stream of instruction-numbers and executes them one after another with no understanding of the overall goal. "Running Chrome" is a human description; from the CPU's view it is just billions of tiny arithmetic and memory operations.

### 0.4 The Fetch-Decode-Execute Cycle

**WHAT:** The fundamental heartbeat of every computer. The CPU repeats this loop forever, billions of times per second:

```
   ┌──────────────────────────────────────────────┐
   │                                                │
   │   1. FETCH                                      │
   │      Read the instruction at the address in    │
   │      the Program Counter (PC) register.        │
   │                  │                              │
   │                  ▼                              │
   │   2. DECODE                                     │
   │      Figure out what the instruction means.     │
   │      ("This is an ADD," "this is a JUMP.")      │
   │                  │                              │
   │                  ▼                              │
   │   3. EXECUTE                                     │
   │      Do it. (Add two registers, load from       │
   │      memory, etc.)                              │
   │                  │                              │
   │                  ▼                              │
   │   4. Advance the PC to the next instruction.    │
   │      (Or jump elsewhere if told to.)           │
   │                  │                              │
   └──────────────────┘  (repeat forever)
```

**The Program Counter (PC):** A special register holding the memory address of the *next* instruction. Mastering this concept is essential, because *the entire idea of a "process" and "multitasking," which is the core of an OS, is ultimately about controlling the Program Counter and the registers.* When the OS switches from running your browser to running your music player, what it physically does is save the current PC and registers, then load the music player's saved PC and registers. The CPU then "continues" as if it had been running the music player all along. Hold onto this — it is the seed of everything.

**Execution trace example.** Suppose memory holds this tiny program (addresses on left, made-up simple instructions on right):

```
Address  Instruction
100      LOAD  R1, [200]   ; copy the number at mailbox 200 into register R1
101      LOAD  R2, [201]   ; copy the number at mailbox 201 into register R2
102      ADD   R3, R1, R2  ; R3 = R1 + R2
103      STORE [202], R3   ; write R3 into mailbox 202
104      HALT

Data:
200      7
201      5
202      (empty)
```

Step-by-step, with PC starting at 100:

1. PC=100. **Fetch** instruction at 100: `LOAD R1, [200]`. **Decode:** load from memory. **Execute:** read mailbox 200 (value 7), put 7 in R1. PC→101.
2. PC=101. Fetch `LOAD R2, [201]`. Execute: R2 = 5. PC→102.
3. PC=102. Fetch `ADD R3, R1, R2`. Execute: R3 = 7+5 = 12. PC→103.
4. PC=103. Fetch `STORE [202], R3`. Execute: write 12 into mailbox 202. PC→104.
5. PC=104. Fetch `HALT`. Stop.

That is *literally* all a computer does. Every program, no matter how complex, decomposes into millions or billions of steps exactly like these.

---

## 1. The Storage Hierarchy: Why Speed and Size Trade Off

You cannot understand operating systems without understanding that **memory is not one thing**. It is a layered hierarchy, and the OS spends an enormous fraction of its effort shuffling data between layers. Here is why the hierarchy exists.

### 1.1 The Fundamental Tradeoff

**WHY:** There is an iron law in hardware: storage that is *fast* is *expensive* and therefore *small*; storage that is *cheap* is *slow* but can be *large*. No technology is simultaneously fast, cheap, and large. So engineers build a *hierarchy*: a tiny amount of blazing-fast storage close to the CPU, backed by progressively larger and slower layers.

```
            Speed         Size        Cost/byte   Analogy (if memory access
                                                   were a trip to get a tool)
  ┌──────┐  fastest       ~1 KB       highest
  │ Regs │  <1 ns         (few dozen)              The tool in your hand.
  └──┬───┘
  ┌──┴───┐
  │ L1 $ │  ~1 ns         ~64 KB                   Tool on your desk.
  └──┬───┘
  ┌──┴───┐
  │ L2 $ │  ~4 ns         ~512 KB                  Tool in the desk drawer.
  └──┬───┘
  ┌──┴───┐
  │ L3 $ │  ~15 ns        ~32 MB                   Tool in the closet down the hall.
  └──┬───┘
  ┌──┴───┐
  │ RAM  │  ~100 ns       ~16-128 GB               Drive to the hardware store.
  └──┬───┘
  ┌──┴───┐
  │ SSD  │  ~100 µs       ~1-4 TB                  Order it; arrives in days.
  └──┬───┘
  ┌──┴───┐
  │ HDD  │  ~10 ms        ~1-20 TB    lowest        Have it manufactured.
  └──────┘
```

*($ = "cache." ns = nanosecond = billionth of a second. µs = microsecond = millionth. ms = millisecond = thousandth.)*

**Intuition for the numbers:** These differences are *vast* and easy to underestimate. To make them human-scale, multiply everything so a register access takes 1 second:

- Register: **1 second**
- L1 cache: ~1 second
- RAM: ~100 seconds (~1.5 minutes)
- SSD: ~1.5 days
- HDD (spinning disk): ~6 months

This is why "is the data in cache or on disk?" is one of the most important performance questions in all of computing. A program that fits in cache versus one that constantly hits disk can differ in speed by a factor of a *million*. The OS is the entity that decides what lives where.

### 1.2 Caching: The Most Important Idea in Systems

**WHAT:** A cache is a small, fast store that keeps copies of data from a larger, slower store, betting that you'll want the same data again soon.

**WHY it works — Locality of Reference:** Programs don't access memory randomly. They exhibit:

- **Temporal locality:** If you used something recently, you'll probably use it again soon (e.g., a loop counter accessed every iteration).
- **Spatial locality:** If you used something, you'll probably use things *near* it soon (e.g., reading an array element, then the next element).

Because real programs have locality, a small cache can satisfy the *majority* of requests, making the whole system feel nearly as fast as the cache while being nearly as large as the slow backing store. This single idea — keep a small fast copy of frequently/recently used data — reappears everywhere: CPU caches, OS page cache, database buffer pools, CDNs, your browser's cache. Learn it once, recognize it forever.

**Mental model:** A librarian who keeps the most-requested 20 books on the desk instead of walking to the stacks every time. Most requests are for popular books, so most of the time the librarian never leaves the desk.

**Common misconception:** Caching is not "free speed." Caches add complexity: what to evict when full (eviction policy), how to stay consistent when the underlying data changes (cache invalidation — famously one of the hardest problems in computer science), and the cost of a "miss" (data not in cache, must fetch from slow store, sometimes slower than no cache at all due to overhead). We will return to all of these.

---

## 2. Binary, Bits, and How Numbers Become Everything

**WHY this is foundational:** Every single thing in a computer — every instruction, image, sound, and the operating system itself — is ultimately a pattern of bits. The OS manages memory in bits and bytes, defines permissions in bit flags, and represents addresses as binary numbers. You must be fluent here.

### 2.1 The Bit

**WHAT:** A *bit* (binary digit) is the smallest unit of information: a single value that is either 0 or 1. Physically it is a wire that is either low-voltage or high-voltage, a transistor that is off or on, a region of disk magnetized one way or the other.

**WHY binary and not decimal?** Because it is dramatically easier and more reliable to build hardware that distinguishes *two* states (on/off) than *ten* states (ten distinct voltage levels). Two states give a wide noise margin: a slightly fuzzy "on" is still clearly not "off." This reliability is why all digital computers are binary at the physical level.

### 2.2 From Bits to Numbers

**Bytes:** 8 bits grouped together form a *byte*. One byte can represent 2⁸ = 256 distinct values (0 to 255). The byte is the fundamental unit of addressable memory — when we said "numbered mailboxes," each mailbox holds one byte.

**Binary counting:** Just as decimal digits represent powers of 10, binary digits represent powers of 2. The binary number `1011` means:

```
  1      0      1      1
  ×8     ×4     ×2     ×1      (powers of two: 2³ 2² 2¹ 2⁰)
  =8  +  0   +  2   +  1   =  11 (decimal)
```

**Words:** A *word* is the natural unit a particular CPU works with — the size of its registers and the amount it moves in one go. A "64-bit computer" has 64-bit words and 64-bit registers. This number matters hugely: it determines, among other things, *how much memory the machine can address*, which we'll see is central to operating systems.

### 2.3 Hexadecimal (Why Addresses Look Like 0x7FFE)

**WHAT:** Hexadecimal ("hex") is base-16, using digits 0–9 then A–F (where A=10, B=11, … F=15). 

**WHY it exists:** Binary is correct but unwieldy for humans — `11111111` is hard to read. Each hex digit represents exactly 4 bits, so two hex digits represent exactly one byte. `11111111` becomes `FF`. Programmers and the OS use hex everywhere — memory addresses, color codes, error codes — because it is a compact, lossless shorthand for binary. The `0x` prefix (as in `0x7FFE`) just means "the following is hexadecimal."

**Common misconception:** Hex is not a different *kind* of number; it is the same number written in a different base. `0xFF`, `255`, and `0b11111111` are all the same quantity, just three notations.

### 2.4 Interpretation Is Everything

**The crucial point for OS understanding:** The byte `01000001` (decimal 65) might be:

- The number 65
- The letter 'A' (in ASCII text encoding)
- Part of a CPU instruction
- One channel of a pixel's color
- Part of a memory address

**Nothing in the bits tells you which.** The meaning comes entirely from *context* — how the program and OS choose to interpret it. This is why a corrupted file shows garbage: the bits are fine, but they're being interpreted with the wrong rules. And it's why security bugs exist: if an attacker can make the CPU interpret *data* as *instructions*, they can hijack the machine. The OS's job includes enforcing the boundaries of correct interpretation. Remember this — it underlies memory protection, executable-space protection, and many attacks we'll study in Part 7.

---

## 3. What Is an Operating System? (Now We Can Define It)

We now have enough foundation to define our actual subject precisely, from multiple angles.

### 3.1 The Core Definition

**WHAT:** An operating system is the layer of software that sits between the hardware and the application programs, and which:

1. **Manages hardware resources** (CPU time, memory, disk, devices) — deciding who gets what, when.
2. **Provides abstractions** that make the messy hardware easy and safe to use (files instead of disk sectors, processes instead of raw CPU, etc.).
3. **Isolates and protects** programs from each other and the system from programs.

```
   ┌───────────────────────────────────────────────┐
   │            Applications                         │
   │   (browser, editor, games, your code)           │
   └───────────────────────┬───────────────────────┘
                           │  System Calls
                           │  (the OS's API)
   ┌───────────────────────┴───────────────────────┐
   │            OPERATING SYSTEM                      │
   │   (kernel: scheduler, memory manager,           │
   │    file system, device drivers, etc.)           │
   └───────────────────────┬───────────────────────┘
                           │  Privileged hardware access
   ┌───────────────────────┴───────────────────────┐
   │            HARDWARE                              │
   │   (CPU, RAM, disk, network, devices)            │
   └───────────────────────────────────────────────┘
```

### 3.2 Three Lenses on What an OS Is

Senior engineers hold multiple mental models simultaneously. Here are the three classic views, each illuminating a different truth:

**(a) The OS as a Resource Manager / Government.** The hardware has finite resources (one CPU might serve hundreds of programs; RAM is limited; one disk serves everyone). Like a government allocating scarce public resources, the OS arbitrates competing demands fairly and efficiently. It decides which program runs on the CPU right now, how much memory each gets, who can use the network card. This lens emphasizes *policy, fairness, and efficiency*.

**(b) The OS as a Virtual Machine / Illusionist.** Raw hardware is brutal to program directly. The OS creates pleasant *illusions*:

- The illusion that each program has the CPU all to itself (when really dozens share it via rapid switching).
- The illusion that each program has a huge, private, contiguous memory (when really memory is shared, fragmented, and partly on disk) — this is *virtual memory*.
- The illusion of *files* (named, resizable byte-streams) on top of a disk that really only knows fixed-size numbered blocks.

This lens emphasizes *abstraction* — hiding ugly reality behind clean interfaces. Each abstraction answers: what messy hardware detail does this hide, and what convenient model does it present instead?

**(c) The OS as a Standard Library / Service Provider.** The OS provides common services every program needs — reading files, sending network packets, drawing to screen — so each program doesn't reimplement them. This lens emphasizes *reuse and standardization*.

All three are true at once. A request like "open this file" simultaneously involves the OS as illusionist (presenting a file abstraction), as government (checking you're allowed and scheduling the disk), and as service provider (running the shared filesystem code).

### 3.3 Historical Background: Why Operating Systems Emerged

Understanding the *why* requires the history, because each OS feature was invented to solve a concrete pain.

**Era 1 — No OS (1940s–early 1950s):** The programmer *was* the operator. You booked the entire machine for an hour, physically loaded your program via switches or punched cards, and watched lights. No abstraction; you spoke directly to hardware. **Pain:** Enormously wasteful — the multimillion-dollar machine sat idle while you set up.

**Era 2 — Batch systems (1950s):** A small resident program — the first primitive OS, the "monitor" — automatically loaded one job after another from a queue of card decks. **Pain solved:** Less idle time between jobs. **New pain:** While a job waited for slow I/O (reading cards, printing), the expensive CPU sat idle.

**Era 3 — Multiprogramming (1960s):** Keep *several* jobs in memory at once. When one job waits for I/O, the OS switches the CPU to another ready job. **Pain solved:** CPU stays busy. **This is the birth of the OS as a serious resource manager** — it now must decide *which* job runs, protect jobs from each other's memory, and manage shared resources. Most core OS concepts (scheduling, memory protection, concurrency) date from here.

**Era 4 — Time-sharing (1960s–70s):** Switch between jobs so rapidly that *interactive* users each feel they have the machine to themselves. MIT's CTSS and later Multics pioneered this. **Pain solved:** Interactivity — programmers could now type and get immediate responses instead of submitting card decks and waiting hours. **Legacy:** Multics was hugely influential but complex; in reaction, Ken Thompson and Dennis Ritchie at Bell Labs built a simpler system in 1969 and (punning on Multics) called it **UNIX**. UNIX's design — everything is a file, small composable tools, written in the new portable C language — became the foundation of Linux, macOS, Android, iOS, and the servers running essentially the entire internet. Nearly everything you'll learn descends from UNIX.

**Era 5 — Personal computers (1980s):** Computers became cheap and personal (MS-DOS, early Mac OS, Windows). Early PC OSes initially *lacked* the protection and multitasking of their mainframe ancestors (a single program could crash the whole machine), then gradually reacquired those features as PC hardware grew capable enough to support them.

**Era 6 — Mobile, cloud, virtualization (2000s–now):** The same core ideas, re-pressurized by new constraints: battery life and touch (Android/iOS, both UNIX-descended), and *virtualization* — running many virtual computers on one physical one, which underpins all cloud computing. We'll cover virtualization deeply later; note for now it's "an OS-like illusion applied to whole machines."

**The throughline:** Every OS feature exists to solve a real, specific pain — usually about using expensive hardware efficiently, letting multiple things coexist safely, or making hard things easy. When you later study a feature and wonder "why is this so complicated?", the answer is almost always a historical pain it was built to solve. Keep asking "what would break without this?"

### 3.4 Common Misconceptions About Operating Systems

- *"The OS is the thing with windows and icons."* No — that's the *graphical user interface (GUI)*, an application-level shell sitting on top. The real OS (the kernel) has no windows; it's invisible machinery. You can run a full OS with no GUI at all (most servers do).
- *"The OS is always running, like a program in a loop."* Subtle but important: the kernel is mostly *not* running. It springs into action in response to events (a program makes a request, a device signals, a timer fires), does its work, and gets out of the way so applications can run. It's more like an event-driven library than an always-spinning loop. (We'll make this precise when we cover interrupts and system calls.)
- *"More OS features = better."* Often the opposite. Every feature is code that can have bugs, consume resources, and widen the attack surface. Great OS design is frequently about *minimalism* and choosing the right abstractions, not piling on features.

---

## 4. The Two Worlds: User Mode and Kernel Mode

This is the central protection mechanism of modern operating systems, and one of the most important concepts in this entire course. Internalize it now; everything else leans on it.

### 4.1 The Problem It Solves

**WHY:** If every program could directly execute any instruction and touch any memory or device, then one buggy or malicious program could halt the CPU, overwrite another program's data, read your passwords from another program's memory, or reprogram your disk. There would be no protection, no isolation, no security. We need a way to let the OS do anything while restricting ordinary programs to a safe subset.

### 4.2 The Mechanism

**WHAT:** Modern CPUs have (at least) two *privilege modes*, enforced in hardware:

- **Kernel mode** (also "supervisor mode," "privileged mode," "ring 0"): Unrestricted. Code running here can execute *every* CPU instruction, access *all* memory, and control *all* devices. The OS kernel runs here.
- **User mode** ("ring 3"): Restricted. Certain *privileged instructions* (halt the CPU, talk directly to devices, change memory mappings, switch privilege modes) are *forbidden*. Memory access is limited to the program's own allotted regions. Application programs run here.

```
   ┌──────────────────────────────────────────────┐
   │              USER MODE (restricted)            │
   │                                                │
   │   Your apps run here. Can do math, use their   │
   │   own memory, call functions. CANNOT directly  │
   │   touch hardware, other apps' memory, or run   │
   │   privileged instructions.                     │
   │                                                │
   │   Need something privileged? ──┐               │
   └────────────────────────────────┼──────────────┘
                                    │ SYSTEM CALL
                                    │ (controlled gateway)
   ┌────────────────────────────────┼──────────────┐
   │              KERNEL MODE  ◄─────┘               │
   │                  (unrestricted)                 │
   │                                                 │
   │   The OS kernel runs here. Can do anything.     │
   │   Validates each request, does the privileged   │
   │   work safely, returns to user mode.            │
   └──────────────────────────────────────────────┘
```

**HOW it's enforced:** A bit in a special CPU register records the current mode. When user-mode code attempts a privileged instruction or an out-of-bounds memory access, the CPU hardware *refuses* and triggers a *trap* (an exception) that jumps into the kernel. The kernel then typically terminates the offending program (this is one cause of the dreaded "segmentation fault"). Crucially, *user-mode code cannot simply flip itself into kernel mode* — that itself is a privileged operation. The only way in is through controlled gateways the kernel defines.

### 4.3 Crossing the Boundary: System Calls

**WHAT:** A *system call* (syscall) is the controlled gateway by which a user program requests a privileged service from the kernel. It's the OS's API — the only legitimate door from user mode into kernel mode.

**HOW it works (step-by-step trace):** Suppose your program wants to read a file.

1. Your program (user mode) puts the request details (which file, how many bytes) into agreed-upon registers, then executes a special *trap instruction* (e.g., `syscall` on x86-64).
2. The trap instruction is the *one* legitimate way to elevate privilege. The CPU switches to kernel mode and jumps to a fixed, kernel-defined entry point (it cannot jump anywhere the program chooses — that would defeat protection).
3. The kernel's syscall handler runs (kernel mode). It *validates* the request: Are you allowed to read this file? Are the addresses valid? This validation is essential — the kernel must never trust user input blindly.
4. The kernel performs the privileged work (instructs the disk, copies data).
5. The kernel places the result in a register and executes a "return from trap" instruction, switching *back* to user mode and resuming your program right after its trap.

**WHY this design:** It enforces a strict, auditable boundary. Programs can only ask for predefined services through a narrow, validated interface. The kernel mediates every privileged action. This is *defense in depth*: even a malicious program is confined to making syscalls, each of which the kernel scrutinizes.

**Mental model:** User mode is the lobby of a secure building; kernel mode is the vault. You (a program) can't walk into the vault. You fill out a request slip (set up registers) and ring the bell (trap instruction). A guard (kernel) takes your slip, checks your credentials, fetches what you asked for *if you're authorized*, and hands it back through the window. You never get to roam the vault yourself.

**Real-world examples of syscalls:** `read`, `write`, `open`, `close` (files); `fork`, `exec`, `exit` (processes); `mmap` (memory); `socket`, `send`, `recv` (network). When you call a friendly library function like `printf` in C or `print` in Python, it eventually boils down to a `write` syscall. The libraries are a comfortable cushion over the raw syscall interface.

**Performance implication (foreshadowing Part 6):** Crossing the user/kernel boundary has real cost (saving state, switching modes, validating, switching back) — typically hundreds of nanoseconds. This is cheap for one call but expensive if done millions of times. A huge amount of systems-performance engineering is about *minimizing syscall crossings* (batching I/O, buffering, techniques like `io_uring`). Keep this in your pocket; it explains many real-world design decisions.

**Security implication (foreshadowing Part 7):** The syscall interface is the primary *attack surface* between untrusted programs and the all-powerful kernel. A bug in syscall validation can let an attacker escalate from user mode to kernel mode — "privilege escalation," the crown jewel of exploits. Sandboxing technologies (like `seccomp`) work by restricting *which* syscalls a program may make, shrinking that attack surface.

---

## 5. Interrupts: How the OS Regains Control

We said the kernel is mostly *not* running — it reacts to events. *Interrupts* are the hardware mechanism that makes this possible, and without them, multitasking and responsive I/O would be impossible.

### 5.1 The Problem

**WHY:** Two problems. First, devices are slow and unpredictable — a keystroke or network packet arrives at a moment the CPU can't predict. How does the CPU notice without wasting time constantly checking ("polling")? Second, if a program is happily running on the CPU, and the CPU only does what the program's instructions say, *how does the OS ever get control back* to switch to another program or handle that keystroke?

### 5.2 The Mechanism

**WHAT:** An *interrupt* is a signal that causes the CPU to pause whatever it's doing, save its place, and jump to a predetermined kernel routine (the *interrupt handler* or *ISR — interrupt service routine*), then later resume the interrupted work as if nothing happened.

**Types:**

- **Hardware interrupts:** Raised by devices. The network card got a packet; the disk finished a read; the *timer chip* fired. Asynchronous — they can occur between any two instructions.
- **Software interrupts / traps / exceptions:** Raised by the CPU itself due to executing an instruction. Includes the syscall trap (deliberate), and *faults* like divide-by-zero, invalid memory access (the "page fault," which is sometimes normal and sometimes an error), or attempting a privileged instruction in user mode.

**HOW (step-by-step trace) — a keypress while your editor runs:**

```
1. Your text editor is executing on the CPU (user mode).
2. You press a key. The keyboard controller raises an
   interrupt signal on a wire to the CPU.
3. The CPU finishes its current instruction, then —
   instead of fetching the next one — it:
     a. Saves the current PC and key registers.
     b. Switches to kernel mode.
     c. Looks up the handler address in the
        "interrupt vector table" and jumps there.
4. The kernel's keyboard interrupt handler runs:
   reads the key from the keyboard controller, stores it
   in a buffer, perhaps wakes the editor waiting for input.
5. The handler finishes ("return from interrupt"):
   restores the saved PC/registers, switches back to
   user mode. The editor resumes exactly where it left
   off — completely unaware it was ever paused.
```

### 5.3 The Timer Interrupt: The Heartbeat of Multitasking

**This is one of the most important ideas in the course.** Recall the puzzle: if a program is running and won't voluntarily give up the CPU (maybe it's in an infinite loop), how does the OS ever reclaim control?

**Answer:** Before handing the CPU to a program, the OS configures a *hardware timer* to fire an interrupt after a small interval (a "time slice," typically a few milliseconds). When the timer fires, the CPU is *forced* into the kernel's timer-interrupt handler — regardless of what the program wanted. Now the OS has control again. It can decide: let this program continue, or save its state and switch to another. This is *preemption*, and it's what makes true multitasking possible and prevents any one program from monopolizing or hanging the machine.

**Mental model:** The OS is like a teacher running a debate where each student may speak, but a kitchen timer dings every few minutes. When it dings, control returns to the teacher, who decides who speaks next. No student can filibuster forever, because the timer always brings control back to the teacher.

**WHY this matters:** Without the timer interrupt, a single buggy infinite loop would freeze your entire computer forever. With it, the OS always gets a chance to intervene. The combination of *user/kernel mode* (the OS is privileged) and the *timer interrupt* (the OS periodically gets control) are the two hardware features that together make a protected, multitasking OS possible. If you understand only two mechanisms from Part 1, understand these two.

---

## 6. The Process: The Central Abstraction

We've assembled the pieces; now we meet the OS's most important creation. Nearly all of operating systems revolves around this concept, so we introduce it carefully here in Foundations and will go deeper in later parts.

### 6.1 Program vs. Process

**WHAT:** A *program* is a passive thing — a file on disk containing instructions and initial data, just sitting there (like a recipe written in a cookbook). A *process* is a program *in execution* — an active, living entity with its own state (like actually cooking the recipe: you, in the kitchen, at a particular step, with particular ingredients in particular bowls).

**Intuition:** One program can spawn many processes. Open three browser windows: one program (the browser binary on disk), three (or more) processes running it, each with its own separate state. The recipe is one; the cooking sessions are many.

### 6.2 What a Process Consists Of

A process is the OS's bookkeeping bundle for one running program. Conceptually it includes:

```
   ┌─────────────────────────────────────┐
   │            A PROCESS                  │
   │                                       │
   │   ┌─────────────────────────────┐     │
   │   │  Address space (its memory): │     │
   │   │  ┌───────────────────────┐   │     │
   │   │  │ Code (the instructions)│  │     │
   │   │  ├───────────────────────┤   │     │
   │   │  │ Data (globals)         │  │     │
   │   │  ├───────────────────────┤   │     │
   │   │  │ Heap (grows up) ▲      │   │     │  dynamically
   │   │  │      ...               │   │     │  allocated memory
   │   │  │ Stack (grows down) ▼   │   │     │  function calls,
   │   │  └───────────────────────┘   │     │  locals
   │   └─────────────────────────────┘     │
   │                                       │
   │   CPU state: PC, registers (its       │
   │   place in execution)                 │
   │                                       │
   │   OS bookkeeping: process ID (PID),   │
   │   open files, permissions, parent,    │
   │   priority, etc.                      │
   └─────────────────────────────────────┘
```

- **Address space:** The process's private view of memory (the illusionist OS gives each process the illusion of its own large, contiguous memory). Divided into regions: *code* (the instructions), *data* (global variables), the *heap* (dynamically allocated memory that grows as needed), and the *stack* (for function calls and local variables). We'll dissect each later.
- **CPU state:** The PC and registers — exactly the things that must be saved and restored to pause and resume the process. (Now you see why we hammered the PC earlier.)
- **OS bookkeeping:** A unique process ID (PID), which files it has open, who owns it, its priority, its parent process, and more — all stored in a kernel data structure traditionally called the *Process Control Block (PCB)*.

### 6.3 The Context Switch (Foreshadowed, Now Named)

**WHAT:** A *context switch* is the act of saving the currently running process's CPU state (its "context" — PC, registers, etc.) into its PCB, and loading another process's saved state, so the CPU now continues that other process.

This is the physical reality behind the "illusion that each program has the CPU to itself." The CPU runs process A for a few milliseconds, the timer interrupt fires, the OS saves A's context and loads B's, and B runs for a few milliseconds, and so on — switching so fast that to a human it looks like everything runs simultaneously. On a single CPU core, *only one process truly runs at any instant*; the simultaneity is an illusion woven from rapid switching. (Multi-core machines genuinely run several at once — one per core — but typically far more processes than cores, so switching is still pervasive.)

**Tradeoff (foreshadowing Part 6):** Context switches aren't free — saving/restoring state and the resulting cache disruption cost time during which *no useful work happens*. Switch too rarely and the system feels unresponsive; switch too often and you waste capacity on overhead. Tuning this balance is a core scheduling concern we'll explore deeply.

### 6.4 Why the Process Abstraction Is Brilliant

The process unifies three of the OS's hardest jobs into one clean concept:

- **Virtualizing the CPU** (each process thinks it has its own CPU, via context switching).
- **Virtualizing memory** (each process thinks it has its own memory, via the address space).
- **Protection and isolation** (one process can't see or corrupt another's memory or state).

When you understand the process completely — and we will, exhaustively — you understand the spine of the operating system. Everything in later parts (scheduling, memory management, IPC, security) either builds a process, runs a process, isolates a process, or lets processes cooperate.

---

## 7. Concurrency vs. Parallelism: A Foundational Distinction

These two words are constantly confused, even by professionals, and the confusion causes real bugs and bad designs. Nail the distinction now.

**Concurrency:** Dealing with *many things at once* — a structure/composition property. Multiple tasks are *in progress* over the same period, but not necessarily executing at the literally same instant. A single CPU core rapidly switching between tasks is *concurrent* but not parallel.

**Parallelism:** Doing *many things at the literally same instant* — an execution property requiring multiple execution units (e.g., multiple CPU cores).

**Analogy (the classic one):**

- *Concurrency:* One barista taking your order, starting your coffee, taking the next person's order while the espresso brews, handing back to the first — juggling several orders by interleaving. One worker, multiple tasks in flight.
- *Parallelism:* Two baristas working two espresso machines, genuinely making two coffees at the same instant.

You can have concurrency without parallelism (one core, time-sliced), parallelism without much concurrency complexity (rare), or both (multi-core machine running many interleaved tasks across cores — the normal modern case).

**WHY this matters for OS:** The OS provides *concurrency* on every machine (via context switching) and exploits *parallelism* on multi-core machines (by scheduling different processes/threads onto different cores). The hardest problems in systems — race conditions, deadlocks, synchronization — arise from concurrency, whether or not true parallelism is present, because the trouble comes from tasks *interleaving in unpredictable orders and sharing data*. We'll devote serious attention to this later; for now, just hold the precise distinction.

---

## 8. The Mental Toolkit: Recurring Themes to Watch For

As we proceed through this course, the same deep ideas will recur in different costumes. Recognizing them as old friends accelerates mastery enormously. Watch for these:

1. **Abstraction:** Hiding complexity behind a clean interface (files over disk blocks, processes over raw CPU). Always ask: *what does this hide, and what does it present instead?*
2. **Indirection / Mapping:** Solving problems by adding a translation layer (virtual addresses → physical addresses). "All problems in computer science can be solved by another level of indirection" — David Wheeler. (The companion half: "except the problem of too many levels of indirection.")
3. **Caching:** Keep a small fast copy of frequently used data. Appears at every layer.
4. **Policy vs. Mechanism separation:** *Mechanism* = how you do something (the machinery of switching processes); *Policy* = the decision of what to do (which process to switch to). Good systems separate these so policy can change without rewiring mechanism. Watch for this split repeatedly.
5. **Time/space tradeoffs:** Use more memory to save time (caching, precomputation), or more time to save memory (compression, recomputation). Engineering is choosing where on this curve to sit.
6. **Trusting boundaries:** Who is trusted, who isn't, and how the boundary is enforced (user vs. kernel mode). Security lives here.
7. **The cost of crossing boundaries:** Syscalls, context switches, cache misses, network hops — each boundary crossing has a cost, and performance work is largely about managing these crossings.
8. **Amdahl's reality:** Adding resources (cores, machines) yields diminishing returns when parts of the work can't be parallelized or when coordination overhead grows. We'll formalize this in the performance and scaling parts.

Keep this list nearby. When you meet a new mechanism in Parts 2–12, try to classify it against these themes. Much of what looks like a pile of disconnected OS trivia is really these eight ideas, applied over and over.

---

## 9. Foundations Self-Check

Before continuing, you should be able to explain each of these *out loud, in your own words*. If any is shaky, re-read its section — later parts assume fluency here.

1. Why does the storage hierarchy exist, and roughly how many orders of magnitude separate a register from a disk?
2. Why is caching effective, and what is locality of reference?
3. Why do computers use binary, and why do bits have no inherent meaning?
4. Give the three lenses on "what an OS is," and a concrete example of each in one operation.
5. What problem does the user/kernel mode split solve, and how is it enforced in hardware?
6. Walk through a system call step by step, naming the mode transitions.
7. Why is the timer interrupt the linchpin of multitasking? What breaks without it?
8. Distinguish a program from a process, and list what a process contains.
9. What exactly is saved and restored in a context switch, and why is it not free?
10. Precisely distinguish concurrency from parallelism, with an example of each.

---

That completes **Part 1 — Foundations** at full depth. We've built the bedrock: how the hardware works, why operating systems exist, the two hardware mechanisms (privilege modes and interrupts) that make protected multitasking possible, and the process abstraction that sits at the center of everything to come. Every later part — scheduling, memory management, file systems, concurrency, security, distributed systems — rests on these foundations.