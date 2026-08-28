# Process Management in Operating Systems — SDE Interview Notes

Covers: Process, Process Creation (fork/exec, zombie/orphan), PCB, Process States, Process Management, Process Table, Schedulers & Dispatcher, **CPU Scheduling Algorithms (with worked examples)**, Preemptive vs Non-Preemptive, Starvation & Aging, and a full "what happens when you run a program" walkthrough.

---

## 1. Process ⭐⭐⭐

**Definition:**
> A process is a program in execution. A program is a passive entity (just code on disk); a process is an active entity — it has a program counter, resources, and a current state.

**Example:** A **recipe** is a program — just instructions on paper. The moment a cook actually starts cooking it (using a stove, ingredients, following steps in order) — that's a process. Two cooks can follow the *same* recipe *at the same time* → two separate processes from one program.

### Memory Layout of a Process
```
High Address
 ┌─────────────────────┐
 │        Stack         │  ← function calls, local vars (grows DOWN ↓)
 │          ↓           │
 │                       │
 │          ↑           │
 │         Heap          │  ← malloc/new, dynamic memory (grows UP ↑)
 ├─────────────────────┤
 │   Data (global/static)│
 ├─────────────────────┤
 │   Text / Code (RO)    │  ← executable instructions
 └─────────────────────┘
Low Address
```
Stack and heap grow toward each other so they can share free space efficiently — if they collide, that's a **stack overflow**.

### Attributes of a Process (stored in the PCB)
1. Process ID (PID)
2. Process State
3. Priority / scheduling info
4. I/O status information
5. File descriptors (open files, sockets)
6. Accounting info (CPU time used, time limits)
7. Memory management info (page tables, base/limit registers)

**Interview Q&A:**
- **Program vs Process?** Program = passive instructions on disk. Process = that program loaded into memory and executing, with its own state, PC, and resources.
- **Can one program create multiple processes?** Yes — each execution gets its own PCB, memory space, PID, even with identical code.
- **Why does stack grow down and heap grow up?** So they can grow toward each other and use available memory efficiently until they collide (stack overflow).

---

## 2. Process Creation: `fork()`, `exec()`, Zombie & Orphan ⭐⭐⭐

### `fork()`
> System call (Unix/Linux) that creates a new process (**child**) by duplicating the calling process (**parent**). Both continue from the same point but as separate processes.

```
                 fork()
   Parent  ───────────┬───────────▶  Parent  (fork() returns child's PID)
   (PID 100)          │
                       └───────────▶  Child   (fork() returns 0)
                                      (PID 101, near-identical copy)
```

**Key facts**
- `fork()` returns **twice**: **child's PID** to parent, **0** to child. Negative = failure.
- Child gets a **copy** of parent's PCB, data, heap, stack — but own PID, own memory space.
- **Copy-On-Write (COW):** instead of copying all memory immediately, parent and child share the same physical pages (read-only) until either **writes** to one — only then is that page actually copied. Makes `fork()` fast/memory-efficient.

### `exec()` family
> Replaces the current process's memory image (code, data, stack, heap) with a new program. Does **not** create a new process — PID stays the same.

```
 Before exec()             After exec("ls")
 ┌───────────────┐         ┌───────────────┐
 │ PID 101        │         │ PID 101        │  ← same PID!
 │ Code: shell     │  ───▶   │ Code: ls binary │  ← entirely new program
 └───────────────┘         └───────────────┘
```

**Key facts**
- `exec()` never returns on success; returns only if it **fails**.
- Classic Unix pattern: **fork + exec** — fork a child, child execs a different program, parent continues (often calling `wait()`).
- **This is exactly how a shell works:** shell forks a child, child execs the command (`ls`), parent shell waits for it to finish.

### Zombie Process
> A finished process whose entry still exists in the process table because the **parent hasn't called `wait()`** yet to read its exit status.

**Example:** A finished food order sitting on the "ready" counter — kitchen (OS) is done cooking, but until the waiter (parent) picks it up (`wait()`), the ticket stays open.

- Consumes a PID and process table slot, but no memory/CPU.
- Fixed by parent calling `wait()`/`waitpid()`.
- If parent never calls `wait()`, zombies accumulate → can exhaust PID table (real production issue).

### Orphan Process
> A still-**running** child process whose **parent terminated** before it did.

- Automatically **re-parented** to `init`/`systemd` (PID 1), which reaps it when it eventually finishes.
- Orphan ≠ Zombie: orphan is still *running* under a new parent; zombie is already *dead* but not reaped.

**Interview Q&A:**
- **What does `fork()` return, and to whom?** Child's PID to parent; 0 to child; negative = failure.
- **What is Copy-On-Write and why does `fork()` use it?** Parent/child share read-only pages until either writes — only then is that page copied. Makes fork fast since most forked processes immediately exec() and never touch most copied memory.
- **`fork()` vs `exec()`?** fork() creates a new process (duplicate); exec() replaces the current process's image (same PID, new program).
- **What is a zombie, and how do you prevent it?** Finished process not yet reaped via `wait()`. Fixed by parent calling `wait()`/`waitpid()` (or handling `SIGCHLD`).
- **What is an orphan, and who adopts it?** Running child whose parent died first; re-parented to init/systemd.
- **Can a process be both orphan and zombie?** Not simultaneously — an orphan *becomes* a zombie briefly after finishing, until init reaps it (near-instant).

---

## 3. Process Control Block (PCB) ⭐⭐

> A data structure the OS maintains for every process, storing everything needed to manage, schedule, and resume it.

```
        ┌────────────────────────────┐
        │   Process Control Block    │
        ├────────────────────────────┤
        │ Process State               │
        │ Process ID (PID)             │
        │ Program Counter              │
        │ CPU Registers                │
        │ Memory limits / page tables  │
        │ Open file list                │
        │ Scheduling info (priority)   │
        │ Accounting info               │
        └────────────────────────────┘
```

- Protected — user processes cannot access another process's PCB.
- Often stored at the start of the process's **kernel stack** for security.
- **Process Table** = array/list of PCBs for all active processes (maps PID → PCB).
- On every context switch: OS **saves** outgoing process's registers into its PCB, **loads** incoming process's saved values from its PCB.

**Interview Q&A:**
- **What's stored in a PCB?** PID, state, program counter, CPU registers, memory info, open files, accounting/scheduling data.
- **PCB vs Process Table?** PCB = info for one process; Process Table = collection of PCBs for all processes.
- **Role in context switching?** Where CPU register state, PC, stack pointer of a preempted process are saved so execution resumes exactly where it left off.

---

## 4. Process States ⭐⭐⭐

### Five-State Model (most commonly asked)
| State | Meaning |
|---|---|
| New | Process being created; PCB being set up |
| Ready | In memory, waiting for CPU |
| Running | Currently executing (only 1 per core) |
| Blocked/Waiting | Waiting for an event (I/O, resource) |
| Terminated | Finished execution; OS reclaims resources |

### State Transition Diagram
```
                 admit             dispatch
      New ───────────────▶ Ready ───────────▶ Running
                              ▲                  │  │
                              │   preempt/interrupt  │  exit
                              └──────────────────┘  │
                                                     ▼
                          I/O or event wait     Terminated
                              │       ▲
                              ▼       │
                          Blocked/Waiting
                          (event completion → back to Ready)
```

**Golden rule:** A process cycles between Ready ⇄ Running ⇄ Blocked many times, but enters **New** and **Terminated** exactly **once**.

**Interview Q&A:**
- **Why 5 states instead of 2?** Two states can't distinguish "waiting for CPU" (Ready) from "blocked on I/O" (Blocked) — essential for efficient scheduling.
- **Can a process go Running → Terminated directly?** Yes — on finishing or being killed.
- **Can a process go New → Running directly?** No — must go through Ready first.
- **What causes Running → Ready?** Preemption — time quantum expiry, or higher-priority process arriving.

---

## 5. Process Management (Overview) ⭐

**Key tasks:**
1. Process creation & termination
2. CPU Scheduling
3. Deadlock handling
4. Inter-Process Communication (IPC) — shared memory, message passing
5. Process Synchronization — coordinating access to shared resources

**CPU-bound vs I/O-bound**
| Type | Behavior |
|---|---|
| CPU-bound | Spends most time computing |
| I/O-bound | Spends more time waiting on I/O (disk, network, input) |

> A good scheduler mixes both so CPU isn't idle while I/O-bound processes wait, and I/O devices aren't idle while CPU-bound processes run.

**Context Switching:** Saving the state of the currently running process (into its PCB) and loading the next process's state. Triggered by interrupts, I/O requests, time-slice expiry, or higher-priority arrival. **Pure overhead** — no useful work happens during the switch.

**Interview Q&A:**
- **What is context switching and why is it "overhead"?** Saving/restoring CPU state between processes; no user-facing work happens during the switch.
- **Why mix CPU-bound and I/O-bound in the ready queue?** Maximizes both CPU and I/O device utilization.
- **Process synchronization vs IPC?** IPC = exchanging data (pipes, shared memory). Synchronization = controlling order/timing of shared resource access to avoid race conditions.

---

## 6. Schedulers & Dispatcher ⭐⭐

### Types of Schedulers
| Scheduler | Function | Speed | Frequency |
|---|---|---|---|
| **Long-Term** (Job Scheduler) | Job pool (disk) → Ready queue; controls **degree of multiprogramming** | Slowest | Rare |
| **Short-Term** (CPU Scheduler) | Picks a process from Ready → allocates CPU | Fastest | Every few ms |
| **Medium-Term** | Swaps processes out/in between memory and disk | Medium | Moderate |

```
  Disk (Job Pool) ──Long-Term──▶ Ready Queue (Memory) ──Short-Term──▶ Running (CPU)
                                        ▲                                  │
                                        └────────── Medium-Term ◀──── swapped out
```

### Dispatcher
> The module that hands the CPU to the process **selected by the scheduler**. Doesn't decide which process runs — just executes that decision (context switch, mode switch, jump to PC).

**Dispatch latency** = time taken to stop one process and start another.

### Scheduler vs Dispatcher
| Scheduler | Dispatcher |
|---|---|
| Decides *which* process runs next | Transfers CPU control to the chosen process |
| Uses algorithms (FCFS, SJF, RR, Priority...) | No scheduling algorithm, just executes |
| Less frequent | Frequent, handles switching overhead |

**Interview Q&A:**
- **Scheduler vs dispatcher?** Scheduler decides which process runs next; dispatcher performs the actual context switch.
- **Which scheduler controls degree of multiprogramming?** Long-Term.
- **Why is Short-Term the fastest?** Runs every few ms — whenever CPU frees up or time slice expires.

---

## 7. CPU Scheduling Algorithms ⭐⭐⭐ (Very frequently asked)

### 7.1 Terminology & Formulas (memorize)
| Term | Meaning |
|---|---|
| Arrival Time (AT) | When process enters ready queue |
| Burst Time (BT) | CPU time the process needs |
| Completion Time (CT) | When process finishes |
| Turnaround Time (TAT) | `CT − AT` |
| Waiting Time (WT) | `TAT − BT` |
| Response Time | Time from submission to *first* CPU response |

### 7.2 Scheduling Criteria
CPU Utilization, Throughput, Turnaround Time, Waiting Time, Response Time, Fairness (no starvation).

> No single "best" algorithm — every algorithm trades off one criterion against another (e.g., SJF minimizes waiting time but sacrifices fairness).

---

### 📌 Example Process Set (used across every algorithm below)

| Process | Arrival Time (AT) | Burst Time (BT) |
|---|---|---|
| P1 | 0 | 5 |
| P2 | 1 | 3 |
| P3 | 2 | 8 |
| P4 | 3 | 6 |

---

### 7.3 FCFS — First Come First Serve
**Non-preemptive.** Run processes strictly in arrival order.

**Example:** One counter at a government office — whoever came first gets served first, no matter how long their paperwork takes.

```
| P1 | P2  | P3      | P4     |
0    5     8         16       22
```

**Avg WT = 5.75 ms** | **Avg TAT = 11.25 ms**

**Pros:** Trivial, no starvation. **Cons:** Suffers **Convoy Effect** (§7.9).

---

### 7.4 SJF — Shortest Job First (Non-Preemptive)
Among arrived processes, always run the one with smallest total burst time.

**Example:** Express checkout lane — "10 items or less" served before a full cart, but once started, runs to completion.

```
| P1 | P2  | P4      | P3          |
0    5     8         14            22
```

**Avg WT = 5.25 ms** — *better than FCFS.*

**Pros:** Minimum avg WT among non-preemptive algorithms. **Cons:** Needs burst time known in advance; **can starve** long jobs.

---

### 7.5 SRTF — Shortest Remaining Time First (Preemptive SJF)
At every new arrival, compare it against the *remaining* time of the running process — preempt if shorter.

**Example:** A support agent who drops the current task whenever a quicker request comes in, returning to the interrupted task later.

```
| P1 | P2      | P1     | P4          | P3              |
0    1         4        8             14                22
```

**Avg WT = 5.0 ms** — *best possible.*

**Pros:** Lowest avg WT overall. **Cons:** Frequent context switches; still starves long jobs.

---

### 7.6 Round Robin (RR)
**Preemptive.** Every process gets a fixed **time quantum** in cyclic order; unfinished → back of queue.

**Example:** A board game where each player gets exactly 60 seconds per turn, then play passes on regardless of whether they finished.

**Using quantum = 2 ms → Avg WT = 9.75 ms** — *worse average than SJF/SRTF, but much fairer.*

**Pros:** No starvation; great response time for interactive systems (basis of most modern OS schedulers). **Cons:** Higher avg WT; very sensitive to quantum size.

**Quantum size trade-off (frequently asked):**
- Too small → excessive context-switch overhead.
- Too large → behaves like FCFS.
- Rule of thumb: ~80% of CPU bursts should be shorter than the quantum.

---

### 7.7 Priority Scheduling
Each process has a priority (convention: **lower number = higher priority**). Highest-priority ready process runs next.

**Example:** ER (emergency room) — heart attack patient (high priority) seen before minor cut, regardless of arrival order — *unless* the minor-cut patient has waited so long they get bumped up (aging, §9).

**Non-preemptive example (priorities: P1=2, P2=4, P3=1, P4=3):**

**Avg WT = 7.75 ms**

**Notice:** P2 has the *shortest* burst (3 ms) but *worst* priority — ends up waiting 18 ms! Classic starvation of quick-but-low-priority jobs — exactly why **aging** exists.

**Preemptive variant:** newly-arrived higher-priority process immediately preempts the running one — lower avg WT possible, but more starvation-prone.

**Pros:** Urgent tasks get CPU first. **Cons:** Starves low-priority processes; needs aging for fairness.

---

### 7.8 HRRN — Highest Response Ratio Next
**Non-preemptive.** Starvation-resistant SJF. At each decision point:

```
Response Ratio = (Waiting Time + Burst Time) / Burst Time
```

- Short burst → naturally high ratio (like SJF).
- Long-waiting process → ratio climbs over time even with a long burst → eventually gets picked. **This is what prevents starvation.**

**Same order as SJF here → Avg WT = 5.25 ms**, but the *mechanism* differs — HRRN would pick a long-waiting process once its ratio overtakes others, which is exactly why it doesn't starve long jobs the way plain SJF does.

**Pros:** Balances "short job first" with "don't starve old jobs." **Cons:** More computation per decision; still needs burst time estimate.

---

### 7.9 Comparison Table — All Algorithms

| Algorithm | Avg WT (ms) | Preemptive? | Starvation risk? | Key trait |
|---|---|---|---|---|
| FCFS | 5.75 | No | No | Simple, but Convoy Effect |
| SJF | 5.25 | No | Yes | Optimal *non-preemptive* avg WT |
| SRTF | **5.0** (best) | Yes | Yes | Optimal *overall* avg WT |
| Round Robin (q=2) | 9.75 | Yes | No | Fair, but highest avg WT here |
| Priority | 7.75 | No | Yes | Can starve short-but-low-priority jobs |
| HRRN | 5.25 | No | No (self-correcting) | SJF-like speed, starvation-resistant |

> **Interview gold nugget:** SRTF gives the theoretical minimum avg waiting time — but real systems rarely use pure SJF/SRTF because burst time isn't known in advance and long jobs would starve. This is exactly why **Round Robin** (fairness) is what real OSes (Linux, Windows) actually use for interactive scheduling.

### Convoy Effect (FCFS-specific)
> A **long CPU-bound process** occupies the CPU first, forcing all shorter processes behind it to wait — even though they could've finished quickly if scheduled earlier. Named after a convoy of short vehicles stuck behind one slow truck on a single-lane road.

**Example (from FCFS table above):** P3 (burst 8 ms) ran ahead of P4 (burst 6 ms) purely because it arrived first, pushing P4's waiting time to 13 ms.

**Fix:** SJF, SRTF, Round Robin, or Priority scheduling — none let one long job block everyone else indefinitely.

**Interview Q&A:**
- **TAT and WT formulas?** TAT = CT − AT; WT = TAT − BT.
- **Which gives minimum avg waiting time overall?** SRTF — theoretically optimal but starves long jobs. Among non-preemptive, SJF is optimal.
- **SJF vs SRTF?** SJF is non-preemptive (runs to completion once started); SRTF is preemptive (a newly-arrived shorter-remaining-time process interrupts the running one).
- **What problem does HRRN solve, and how?** Prevents SJF's starvation via Response Ratio — the longer a process waits, the higher its ratio climbs, guaranteeing it eventually runs.
- **Why does RR have higher avg WT than SJF/SRTF?** Every process gets a fixed quantum regardless of length, so short jobs may wait behind several turns instead of finishing outright.
- **What determines RR's performance?** Time quantum size.
- **What is the Convoy Effect and which algorithm suffers it?** Long process hogs CPU in FCFS, forcing shorter ones to wait — specific to FCFS.
- **Can a process with a short burst still wait the longest? Example?** Yes — in Priority Scheduling, priority (not burst time) decides order. P2 (burst=3) had the lowest priority and ended up waiting 18 ms.

---

## 8. Preemptive vs Non-Preemptive Scheduling ⭐⭐

| Preemptive | Non-Preemptive |
|---|---|
| CPU allocated for limited time | Process holds CPU until completion or it waits |
| Can be interrupted mid-execution | Cannot be interrupted until it terminates/blocks |
| High-priority arrivals can starve low-priority | Long-burst process can starve short ones behind it |
| Overhead from frequent context switching | Minimal scheduling overhead |
| Lower avg response time | Higher avg response time |
| Examples: Round Robin, SRTF | Examples: FCFS, SJF |

**Interview Q&A:**
- **Advantage/disadvantage of preemptive scheduling?** Advantage: better response time, prevents CPU monopolization. Disadvantage: overhead from context switches, can starve low-priority processes.
- **Which is used in Linux/Windows?** Preemptive — supports responsive multitasking.

---

## 9. Starvation and Aging ⭐⭐

> **Starvation:** A process waits indefinitely for the CPU because other (typically higher-priority) processes keep getting selected first.
> **Aging:** Fix — gradually increase the priority of processes the longer they wait, guaranteeing eventual execution.

**Example:** Priorities range 0 (highest) to 127 (lowest). A waiting process's priority is bumped up (e.g., by 1 every 15 min) until it becomes highest priority and runs.

**Tie-in to §7.7:** P2 waited 18 ms despite having the shortest burst, purely due to low priority. With aging, P2's priority would have incremented the longer it waited, until it outranked P3/P4 and got scheduled.

**Interview Q&A:**
- **What is starvation and which algorithm is most prone to it?** Indefinite waiting; Priority Scheduling and SJF, since low-priority/long jobs can be perpetually skipped.
- **How does aging prevent starvation?** Incrementally raises a waiting process's priority until it reaches the highest and gets scheduled.
- **Does Round Robin suffer starvation?** No — every process gets a guaranteed turn.

---

## 10. What Happens When You Run a Program (Synthesis Question) ⭐⭐⭐

Classic "walk me through it end-to-end" question:

1. **You type the command / double-click the executable.**
   The shell makes a `fork()` call, creating a child process that's a duplicate of the shell.
2. **The child calls `exec()`.**
   Replaces the child's memory image with the new program — same PID, entirely new program.
3. **The OS loader sets up the process.**
   New **PCB** created, PID assigned, state = **New**. Memory layout prepared (text/data/heap/stack), PC set to entry point.
4. **Admission.**
   Process is admitted into memory, moves to **Ready**, PCB placed in the ready queue.
5. **Scheduling.**
   The **Short-Term Scheduler** eventually picks this process based on the active algorithm.
6. **Dispatch.**
   The **Dispatcher** performs a context switch — saves outgoing process's state, loads this process's state, switches to user mode, jumps to PC. State → **Running**.
7. **Execution proceeds.**
   - Needs I/O → issues syscall → moves to **Blocked/Waiting**; scheduler picks another ready process.
   - Time quantum expires or higher-priority process arrives → preempted back to **Ready**.
   - Memory tight → **Medium-Term Scheduler** may swap it to disk (**Suspended**).
8. **Termination.**
   Enters **Terminated**. OS reclaims memory, closes file descriptors, leaves a **zombie** entry until parent calls `wait()`.

**Interview Q&A:**
- **Walk me through running `./a.out` in Linux.** Shell forks a child, child execs the new program, OS sets up fresh PCB/memory, process admitted to Ready, Short-Term Scheduler dispatches via context switch, cycles through Ready/Running/Blocked, then terminates → zombie until parent reaps it.
- **At what point does the process actually start running on the CPU?** Only after the Dispatcher performs a context switch and the scheduler has selected it — not immediately after exec().

---

## 🎯 Rapid-Fire One-Liners (Last-Minute Revision)

- Process = program in execution (active vs passive entity).
- `fork()` creates a new process (duplicate, COW-optimized); `exec()` replaces the current process's image — same PID, new program.
- Zombie = finished but not reaped by parent's `wait()`; Orphan = still running, parent died first (re-parented to init/systemd).
- PCB = data structure holding process metadata; Process Table = array of PCBs.
- 5-state model: New → Ready → Running → (Blocked ⇄ Ready) → Terminated.
- New & Terminated occur exactly once; Ready/Running/Blocked can repeat.
- Long-Term Scheduler controls degree of multiprogramming; Short-Term is fastest; Medium-Term handles swapping.
- Dispatcher ≠ Scheduler: dispatcher executes the scheduler's decision.
- TAT = CT − AT; WT = TAT − BT.
- **FCFS**: simplest, suffers Convoy Effect, no starvation.
- **SJF**: optimal non-preemptive avg WT, can starve long jobs.
- **SRTF**: optimal overall avg WT, more overhead, still starves long jobs.
- **Round Robin**: fairest, no starvation, usually highest avg WT; sensitive to quantum size.
- **Priority**: respects urgency, can starve low-priority jobs regardless of burst time.
- **HRRN**: SJF-like speed without starvation, via Response Ratio.
- Convoy Effect → FCFS-specific: one long process blocks all short ones behind it.
- Preemptive: CPU can be taken away (RR, SRTF). Non-preemptive: runs till completion/block (FCFS, SJF).
- Starvation = indefinite wait; Aging = fix by increasing priority over time.
- Running a program = fork → exec → PCB/memory setup → Ready → dispatched (context switch) → Running → (Blocked/Ready cycles) → Terminated → zombie → reaped.
