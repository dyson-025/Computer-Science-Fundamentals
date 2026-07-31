# Process Management in Operating Systems — Interview Notes (SDE-1 / SDE-2 / Intern)

Covers: Process, Process Creation (fork/exec, zombie/orphan), PCB, Process States, Process Management, Process Table, Schedulers & Dispatcher, **CPU Scheduling Algorithms (in depth, with worked examples)**, Preemptive vs Non-Preemptive, Starvation & Aging, and a full "what happens when you run a program" walkthrough.

---

## 1. Process

**Definition (say this in an interview):**
> A process is a program in execution. A program is a passive entity (just code sitting on disk); a process is an active entity — it has a program counter, resources, and a current state.

**Real-life analogy**
A **recipe** is a program — just instructions on paper. The moment a cook actually starts cooking it (using a stove, ingredients, and following steps in order) — that's a process. Two cooks can follow the *same* recipe *at the same time* → two separate processes from one program.

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
Stack and heap grow toward each other so they can share the free space between them efficiently — if they collide, that's a **stack overflow**.

### Attributes of a Process (stored in the PCB)
1. Process ID (PID)
2. Process State
3. Priority / scheduling info
4. I/O status information
5. File descriptors (open files, sockets)
6. Accounting info (CPU time used, time limits)
7. Memory management info (page tables, base/limit registers)

### Quick Interview Q&A
- **Q: Difference between a program and a process?**
  A program is passive instructions on disk; a process is that program loaded into memory and executing, with its own state, PC, and resources.
- **Q: Can one program create multiple processes?**
  Yes — each execution instance gets its own PCB, memory space, and PID, even though the code is identical.
- **Q: Why does the stack grow downward and heap upward?**
  Convention that lets both grow toward each other and use available memory efficiently until they collide (stack overflow).

---

## 2. Process Creation: `fork()`, `exec()`, Zombie & Orphan Processes

### `fork()`
**Definition:**
> `fork()` is a system call (Unix/Linux) that creates a new process (the **child**) by duplicating the calling process (the **parent**). After `fork()`, both processes continue execution from the same point but as separate processes with separate memory spaces.

```
                 fork()
   Parent  ───────────┬───────────▶  Parent  (fork() returns child's PID)
   (PID 100)          │
                       └───────────▶  Child   (fork() returns 0)
                                      (PID 101, near-identical copy)
```

**Key facts**
- `fork()` returns **twice**: the **child's PID** to the parent, and **0** to the child. A negative value means fork failed.
- The child gets a **copy** of the parent's PCB, data, heap, and stack — but has its **own PID**, own memory space, and own copy of the file descriptor table (open files' offsets are shared initially).
- Modern OSes use **Copy-On-Write (COW)**: instead of physically copying all the parent's memory immediately, both processes initially share the same physical pages marked read-only. A page is only actually copied when either process **writes** to it. This makes `fork()` fast and memory-efficient.
- The child is a near-duplicate but not identical: different PID, different parent PID, resource-usage counters reset to zero.

### `exec()` family (`execve`, `execvp`, etc.)
**Definition:**
> `exec()` replaces the current process's memory image (code, data, stack, heap) with a new program's image. It does **not** create a new process — the PID stays the same, but the program running under that PID changes completely.

```
 Before exec()             After exec("ls")
 ┌───────────────┐         ┌───────────────┐
 │ PID 101        │         │ PID 101        │  ← same PID!
 │ Code: shell     │  ───▶   │ Code: ls binary │  ← entirely new program
 │ Data: shell vars│         │ Data: ls vars   │
 └───────────────┘         └───────────────┘
```

**Key facts**
- `exec()` never returns on success (the old program is gone); it returns only if it **fails**.
- Commonly used right after `fork()`: the classic Unix pattern is **"fork + exec"** — fork a child, have the child call exec to run a different program, while the parent continues (often calling `wait()` for the child).
- This is exactly how a shell works: shell forks a child, child execs the command (e.g., `ls`), parent shell waits for it to finish.

### Zombie Process
**Definition:**
> A zombie (defunct) process has **finished execution**, but its entry still exists in the process table because the **parent hasn't yet read its exit status** via `wait()`.

**Real-life analogy:** A finished food order sitting on the "ready" counter — the kitchen (OS) is done cooking, but until the waiter (parent) picks it up (`wait()`), the ticket stays open on the board.

- Consumes a PID and a process table slot, but no memory/CPU — a "corpse" waiting to be cleaned up.
- Fixed when the parent calls `wait()`/`waitpid()`, which reads the exit status and lets the OS remove the entry.
- If the parent never calls `wait()` and never terminates, zombies accumulate — can exhaust the PID table (a real production issue).

### Orphan Process
**Definition:**
> An orphan process is a still-running child process whose **parent has terminated** before it did.

- Automatically **re-parented** — on Unix/Linux, `init`/`systemd` (PID 1) adopts it and calls `wait()` on its behalf when it eventually finishes, preventing permanent zombies.
- Orphan ≠ Zombie: an orphan is still *running* under a new parent; a zombie is already *dead* but not yet reaped.

```
        Zombie                          Orphan
 Child finishes first,           Parent finishes first,
 parent still alive but          child still running
 hasn't called wait()             │
        │                          ▼
        ▼                    Re-parented to init/systemd
 Stuck as a "corpse" entry    (which will reap it later)
 until reaped
```

### Quick Interview Q&A
- **Q: What does `fork()` return, and to whom?**
  Child's PID to the parent; 0 to the child; negative value = failure.
- **Q: What is Copy-On-Write and why does `fork()` use it?**
  Parent and child share physical pages (read-only) until either writes to one — only then is that page copied. Makes fork() fast since most forked processes immediately exec() and never touch most of the copied memory.
- **Q: Difference between `fork()` and `exec()`?**
  `fork()` creates a new process (duplicate); `exec()` replaces the current process's program image without creating a new process (same PID).
- **Q: What is a zombie process, and how do you prevent/clean it?**
  Finished process whose exit status hasn't been collected via `wait()`. Prevented/cleaned by the parent calling `wait()`/`waitpid()` (or handling `SIGCHLD`).
- **Q: What is an orphan process, and who adopts it?**
  A running child whose parent died first; re-parented to `init`/`systemd` (PID 1), which reaps it when it exits.
- **Q: Can a process be both an orphan and a zombie?**
  Not simultaneously — but an orphan *becomes* a zombie briefly after finishing, until init (its new parent) reaps it, which happens near-instantly.

---

## 3. Process Control Block (PCB)

**Definition:**
> The PCB (also called Task Control Block) is a data structure the OS maintains for every process, storing everything needed to manage, schedule, and resume that process.

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

**Important facts**
- Protected — normal user processes cannot access another process's PCB.
- Often stored at the start of the process's **kernel stack** for security.
- The **Process Table** is simply an array/list of PCBs for all active processes — maps PID → PCB.
- On every context switch, the OS **saves** the outgoing process's register values into its PCB and **loads** the incoming process's saved values from its PCB.

### Quick Interview Q&A
- **Q: What is stored in a PCB?**
  PID, process state, program counter, CPU registers, memory management info, open file list, accounting/scheduling data.
- **Q: Why is the PCB often placed on the kernel stack?**
  For protection — user-mode code cannot tamper with it.
- **Q: Difference between the Process Table and a PCB?**
  PCB holds info for one process; the Process Table is the collection (array) of PCBs for all processes currently in the system.
- **Q: What role does the PCB play in context switching?**
  It's where the CPU register state, PC, and stack pointer of a preempted process are saved so execution can resume exactly where it left off.

---

## 4. Process States

### Two-State Model
| State | Meaning |
|---|---|
| Running | Process is executing on CPU |
| Not Running | Waiting for CPU or blocked |

Simplest model; doesn't distinguish "ready to run" from "waiting on I/O."

### Five-State Model (most commonly asked)
| State | Meaning |
|---|---|
| New | Process is being created; PCB being set up |
| Ready | In memory, waiting for CPU allocation |
| Running | Currently executing on CPU (only 1 per core) |
| Blocked/Waiting | Waiting for an event (I/O, resource) |
| Terminated/Exit | Finished execution; OS reclaims resources |

### Seven-State Model
Adds **suspend** states to handle memory shortages: New → Ready → Running → Terminated, plus **Blocked/Waiting**, **Suspend Ready** (a ready process swapped to disk due to low memory; returns to Ready when swapped back in), and (in some versions) **Suspend Blocked**.

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

**Golden rule:** A process cycles between Ready ⇄ Running ⇄ Blocked many times, but enters **New** and **Terminated** exactly **once** in its lifetime.

### Quick Interview Q&A
- **Q: Why 5 states instead of 2?**
  Two states can't distinguish "waiting for CPU" (Ready) from "blocked on an external event like I/O" (Blocked) — essential for efficient scheduling.
- **Q: Can a process go directly from Running to Terminated?**
  Yes — on finishing or being killed.
- **Q: Can a process go directly from New to Running?**
  No — must first enter Ready and be dispatched.
- **Q: What causes a Running → Ready transition?**
  Preemption — time quantum expiry, or a higher-priority process becoming ready.
- **Q: What is the Suspended state and why is it needed?**
  Used by the medium-term scheduler when memory is over-committed — a ready/blocked process is swapped to disk to free RAM, reloaded later.

---

## 5. Process Management (Overview)

**Definition:**
> The OS function responsible for creating, scheduling, synchronizing, and terminating processes to ensure efficient CPU and resource utilization.

**Key tasks**
1. **Process creation & termination** — assign PID, set up PCB, later release resources.
2. **CPU Scheduling** — decide which process runs next.
3. **Deadlock handling** — avoid processes waiting on each other cyclically forever.
4. **Inter-Process Communication (IPC)** — shared memory, message passing.
5. **Process Synchronization** — coordinate access to shared resources (locks, semaphores).

**CPU-bound vs I/O-bound**
| Type | Behavior |
|---|---|
| CPU-bound | Spends most time computing; needs more CPU time |
| I/O-bound | Spends more time waiting on I/O (disk, network, user input) |

> A good scheduler mixes both types so the CPU isn't idle while I/O-bound processes wait, and I/O devices aren't idle while CPU-bound processes run.

**Context Switching**
> Saving the state of the currently running process (into its PCB) and loading the state of the next process to run.
- Triggered by: interrupts, I/O requests, time-slice expiry, or a higher-priority process arriving.
- Pure overhead — no useful work is done during the switch itself.

### Quick Interview Q&A
- **Q: What is context switching and why is it "overhead"?**
  Saving/restoring CPU state between two processes; no useful user-facing work happens during the switch itself.
- **Q: Why mix CPU-bound and I/O-bound processes in the ready queue?**
  Maximizes both CPU and I/O device utilization.
- **Q: Difference between process synchronization and IPC?**
  IPC = exchanging data (shared memory, pipes, queues). Synchronization = controlling *order/timing* of access to shared resources to avoid race conditions.

---

## 6. Process Table

**Definition:**
> A system-wide data structure (essentially an array of PCBs) the OS uses to track every active process, mapping each PID to its PCB.

**Roles**
- Scheduling — supplies scheduler with process details.
- Context switching — stores register/stack-pointer context.
- Interrupt handling, resource allocation tracking, IPC support.
- Security — stores user/group IDs and permissions.
- Debugging tools like `top`/`ps` read from it.
- Backing store for virtual memory (page table pointers).

### Quick Interview Q&A
- **Q: How is the Process Table related to the PCB?**
  It's the collection of PCB entries — one per active process — indexed by PID.
- **Q: Which OS tools rely on the process table?**
  `ps`, `top`, Task Manager.

---

## 7. Schedulers & Dispatcher

### Types of Schedulers
| Scheduler | Also called | Function | Speed | Frequency |
|---|---|---|---|---|
| Long-Term | Job Scheduler | Job pool (disk) → Ready queue (memory); controls **degree of multiprogramming** | Slowest | Rare |
| Short-Term | CPU Scheduler | Picks a process from Ready queue → allocates CPU | Fastest | Every few ms |
| Medium-Term | — | Swaps processes out/in between memory and disk to manage load | Medium | Moderate |

```
  Disk (Job Pool) ──Long-Term──▶ Ready Queue (Memory) ──Short-Term──▶ Running (CPU)
                                        ▲                                  │
                                        └────────── Medium-Term ◀──── swapped out
                                              (swap in/out to manage memory)
```

- **Degree of multiprogramming** = max number of processes that can reside in Ready at once.
- Many modern time-sharing OSes (e.g., Windows) skip a distinct long-term scheduler — processes go straight into memory.

### Dispatcher
**Definition:**
> The module that hands the CPU to the process selected by the Short-Term Scheduler. Unlike the scheduler, it doesn't decide *which* process runs — it just executes that decision.

**Functions:** context switching (save/load state), mode switching (kernel → user), jump to the correct instruction (PC).

**Dispatch latency** = time taken by the dispatcher to stop one process and start another.

### Scheduler vs Dispatcher
| Scheduler | Dispatcher |
|---|---|
| Decides *which* process runs next | Transfers CPU control to the chosen process |
| Long-term, Medium-term, Short-term types | Single module, no types |
| Uses algorithms (FCFS, SJF, RR, Priority...) | Uses no scheduling algorithm |
| Less frequent, lower overhead | Frequent, handles switching overhead |
| Works with the ready queue | Works directly with the CPU |

### Quick Interview Q&A
- **Q: Difference between scheduler and dispatcher?**
  Scheduler decides which process runs next; dispatcher performs the actual context switch/mode switch.
- **Q: Which scheduler controls the degree of multiprogramming?**
  Long-Term.
- **Q: Why is the Short-Term Scheduler the fastest?**
  It runs every few milliseconds — whenever the CPU frees up or a time slice expires.
- **Q: What does the Medium-Term Scheduler do?**
  Swaps processes out of/into main memory to manage the degree of multiprogramming or free memory.
- **Q: What is dispatch latency?**
  Time the dispatcher takes to stop one process and start another — pure overhead, should be minimized.

---

## 8. CPU Scheduling Algorithms — In Depth

**Definition:**
> CPU Scheduling is how the OS decides which ready process gets the CPU at any given moment, aiming to maximize CPU utilization and minimize waiting/response time.

### 8.1 Terminology & Formulas (memorize — asked in almost every interview)
| Term | Meaning |
|---|---|
| Arrival Time (AT) | When the process enters the ready queue |
| Burst Time (BT) | CPU time the process needs |
| Completion Time (CT) | When the process finishes |
| Turnaround Time (TAT) | `CT − AT` — total time in the system |
| Waiting Time (WT) | `TAT − BT` — time spent waiting, not running |
| Response Time | Time from submission to the *first* CPU response (critical for interactive systems) |

### 8.2 Scheduling Criteria (what a "good" algorithm optimizes)
- **CPU Utilization** — keep CPU busy (ideally 40–90% in real systems)
- **Throughput** — processes completed per unit time
- **Turnaround Time** — minimize
- **Waiting Time** — minimize
- **Response Time** — minimize (esp. for interactive/UI systems)
- **Fairness** — no process starves indefinitely

> There is no single "best" algorithm — every algorithm trades off one of these against another (e.g., SJF minimizes waiting time but sacrifices fairness).

---

### 📌 The Example Process Set (used across every algorithm below, for direct comparison)

| Process | Arrival Time (AT) | Burst Time (BT) |
|---|---|---|
| P1 | 0 | 5 |
| P2 | 1 | 3 |
| P3 | 2 | 8 |
| P4 | 3 | 6 |

We'll run this **same set** through FCFS, SJF, SRTF, Round Robin, Priority, and HRRN so you can directly see *why* the average waiting time changes between algorithms — this comparison itself is a favorite interview question.

---

### 8.3 FCFS — First Come First Serve
**Type:** Non-preemptive | **Idea:** Run processes strictly in arrival order, like a single-lane queue at a bank counter.

**Real-life analogy:** One counter at a government office — whoever came first gets served first, no matter how long their paperwork takes.

**Gantt Chart**
```
| P1 | P2  | P3      | P4     |
0    5     8         16       22
```

| Process | AT | BT | CT | TAT (CT−AT) | WT (TAT−BT) |
|---|---|---|---|---|---|
| P1 | 0 | 5 | 5 | 5 | 0 |
| P2 | 1 | 3 | 8 | 7 | 4 |
| P3 | 2 | 8 | 16 | 14 | 6 |
| P4 | 3 | 6 | 22 | 19 | 13 |

**Avg WT = 5.75 ms** | **Avg TAT = 11.25 ms**

**Pros:** Trivial to implement, no starvation (everyone eventually runs).
**Cons:** Suffers the **Convoy Effect** (below) — P4 waited 13 ms mostly because P3 (a long job) ran ahead of it.

---

### 8.4 SJF — Shortest Job First (Non-Preemptive)
**Type:** Non-preemptive | **Idea:** Among the processes that have *arrived*, always run the one with the smallest total burst time next.

**Real-life analogy:** An express checkout lane — "10 items or less" gets served before someone's full grocery cart, but once someone starts checking out, they finish completely before the next person starts.

**Gantt Chart**
```
| P1 | P2  | P4      | P3          |
0    5     8         14            22
```
(At t=5, P2, P3, P4 have all arrived; P2 has the shortest burst (3), so it goes next. At t=8, between P3(8) and P4(6), P4 is shorter.)

| Process | AT | BT | CT | TAT | WT |
|---|---|---|---|---|---|
| P1 | 0 | 5 | 5 | 5 | 0 |
| P2 | 1 | 3 | 8 | 7 | 4 |
| P4 | 3 | 6 | 14 | 11 | 5 |
| P3 | 2 | 8 | 22 | 20 | 12 |

**Avg WT = 5.25 ms** | **Avg TAT = 10.75 ms** — *better than FCFS!*

**Pros:** Minimum average waiting time among **non-preemptive** algorithms (provably optimal, if all burst times were known ahead of time).
**Cons:** Requires knowing burst time in advance (usually estimated); **can starve** long processes (notice P3 waited 12 ms, more than under FCFS).

---

### 8.5 SRTF — Shortest Remaining Time First (Preemptive SJF)
**Type:** Preemptive | **Idea:** Same as SJF, but at *every new arrival*, compare the new process's burst time against the *remaining* time of the currently running process — preempt if the new one is shorter.

**Real-life analogy:** A customer service agent who drops what they're doing whenever a "quicker" request comes in, and comes back to the interrupted task later.

**Gantt Chart**
```
| P1 | P2      | P1     | P4          | P3              |
0    1         4        8             14                22
```
(At t=1, P2 arrives with BT=3 < P1's remaining 4 → preempt to P2. P2 runs to completion at t=4 since no shorter job arrives while it runs. Then P1 resumes with its remaining 4 units.)

| Process | AT | BT | CT | TAT | WT |
|---|---|---|---|---|---|
| P2 | 1 | 3 | 4 | 3 | 0 |
| P1 | 0 | 5 | 8 | 8 | 3 |
| P4 | 3 | 6 | 14 | 11 | 5 |
| P3 | 2 | 8 | 22 | 20 | 12 |

**Avg WT = 5.0 ms** | **Avg TAT = 10.5 ms** — *best so far!*

**Pros:** Even lower average waiting time than SJF (reacts to new short jobs immediately instead of waiting for the current one to finish).
**Cons:** Frequent context switches (overhead); still **starves** long jobs (P3 again waits the most); needs continuous burst-time tracking.

---

### 8.6 Round Robin (RR)
**Type:** Preemptive | **Idea:** Every process gets a fixed **time quantum** in cyclic order; if it doesn't finish, it goes to the back of the ready queue.

**Real-life analogy:** A board game where each player gets exactly 60 seconds per turn, then play passes to the next player regardless of whether they finished their move — you just wait for your next turn to continue.

**Using quantum = 2 ms:**

**Gantt Chart**
```
|P1|P2|P3|P1|P4|P2|P3|P1|P4|P3|P4|P3|
0  2  4  6  8 10 11 13 14 16 18 20 22
```

| Process | AT | BT | CT | TAT | WT |
|---|---|---|---|---|---|
| P1 | 0 | 5 | 14 | 14 | 9 |
| P2 | 1 | 3 | 11 | 10 | 7 |
| P3 | 2 | 8 | 22 | 20 | 12 |
| P4 | 3 | 6 | 20 | 17 | 11 |

**Avg WT = 9.75 ms** | **Avg TAT = 15.25 ms** — *worse average than SJF/SRTF, but much fairer.*

**Pros:** No starvation — every process is guaranteed a turn; great **response time** for interactive systems (the basis of most modern time-sharing OS schedulers).
**Cons:** Higher average waiting/turnaround time than SJF/SRTF; performance is **very sensitive to quantum size**.

**Quantum size trade-off (frequently asked):**
- **Too small** → excessive context-switching overhead (system spends more time switching than doing useful work).
- **Too large** → behaves like FCFS (a process just runs to completion within its huge quantum), losing RR's fairness benefit.
- **Rule of thumb**: ~80% of CPU bursts should be shorter than the time quantum.

---

### 8.7 Priority Scheduling
**Type:** Both preemptive and non-preemptive variants exist | **Idea:** Each process has a priority number; the highest-priority ready process runs next (convention: **lower number = higher priority**).

**Real-life analogy:** An ER (emergency room) — a patient with a heart attack (high priority) is seen before someone with a minor cut, regardless of who arrived first — *unless* the minor-cut patient has now been waiting so long they get bumped up (that's aging, see §10).

**Non-preemptive example** (priorities: P1=2, P2=4, P3=1, P4=3 — 1 is highest):

**Gantt Chart**
```
| P1 | P3      | P4     | P2  |
0    5         13       19    22
```
(At t=0 only P1 has arrived, so it must run first regardless of priority. At t=5, among arrived P2/P3/P4, P3 has the best priority (1) → runs next. Then P4 (priority 3) beats P2 (priority 4).)

| Process | AT | BT | Priority | CT | TAT | WT |
|---|---|---|---|---|---|---|
| P1 | 0 | 5 | 2 | 5 | 5 | 0 |
| P3 | 2 | 8 | 1 | 13 | 11 | 3 |
| P4 | 3 | 6 | 3 | 19 | 16 | 10 |
| P2 | 1 | 3 | 4 | 22 | 21 | 18 |

**Avg WT = 7.75 ms**

**Notice:** P2 has the *shortest* burst (3 ms) but the *worst* priority — it ends up waiting 18 ms! This is exactly how priority scheduling can **starve** low-priority processes even if they're quick — the classic drawback, and the reason **aging** exists (§10).

**Preemptive variant:** works the same way, but a newly-arrived higher-priority process **immediately preempts** the running one (instead of waiting for the current process to finish) — even lower average waiting time is possible, but even more starvation-prone and more context-switch overhead.

**Pros:** Important tasks get CPU time first — matches real-world urgency (e.g., a kernel task vs. a background batch job).
**Cons:** Starvation of low-priority processes; needs aging to guarantee fairness.

---

### 8.8 HRRN — Highest Response Ratio Next
**Type:** Non-preemptive | **Idea:** A smarter, starvation-resistant version of SJF. At each decision point, compute a **Response Ratio** for every waiting process and run the one with the highest ratio:

```
Response Ratio = (Waiting Time + Burst Time) / Burst Time
```

- A process with a **short burst time** naturally has a high ratio (like SJF).
- But a process that has been **waiting a long time** *also* sees its ratio climb over time — even if its burst is long — so it eventually gets picked. This is what prevents starvation.

**Worked calculation (using our example set):**

At **t = 5** (P2, P3, P4 have all arrived):
| Process | Waiting so far | BT | Response Ratio = (W+BT)/BT |
|---|---|---|---|
| P2 | 5−1=4 | 3 | (4+3)/3 = **2.33** ✅ highest |
| P3 | 5−2=3 | 8 | (3+8)/8 = 1.375 |
| P4 | 5−3=2 | 6 | (2+6)/6 = 1.33 |

→ Run **P2** next (5–8).

At **t = 8** (P3, P4 remain):
| Process | Waiting so far | BT | Response Ratio |
|---|---|---|---|
| P3 | 8−2=6 | 8 | (6+8)/8 = 1.75 |
| P4 | 8−3=5 | 6 | (5+6)/6 = **1.83** ✅ highest |

→ Run **P4** next (8–14), then **P3** last (14–22).

**Gantt Chart**
```
| P1 | P2  | P4      | P3          |
0    5     8         14            22
```

**Avg WT = 5.25 ms** (same order as SJF here, but the *mechanism* is different — HRRN would have picked P3 instead if it had been made to wait long enough that its ratio overtook the others, which is exactly why it doesn't starve long jobs the way plain SJF does.)

**Pros:** Balances "short job" preference with "don't starve old jobs" — best of both SJF and aging.
**Cons:** More computation per scheduling decision (must recompute ratios for all waiting processes); needs to know/estimate burst time like SJF.

---

### 8.9 Multilevel Queue (MLQ)
**Idea:** The ready queue is split into **multiple separate queues** based on process type/priority (e.g., System processes, Interactive processes, Batch processes), and each queue can have its **own scheduling algorithm**. Queues themselves are typically scheduled with **fixed priority** (higher queue always preempts lower) or **time-slicing between queues**.

```
 ┌───────────────────────┐
 │ Queue 1: System procs   │ ← Round Robin, highest priority
 ├───────────────────────┤
 │ Queue 2: Interactive     │ ← Round Robin, medium priority
 ├───────────────────────┤
 │ Queue 3: Batch/Background │ ← FCFS, lowest priority
 └───────────────────────┘
        Fixed priority between queues (or time-sliced)
```

**Real-life analogy:** An airport with separate lines for First Class, Business, and Economy — each line has its own process, but First Class boards before Business ever gets called.

**Key property:** A process is **permanently assigned** to one queue (based on its type) — it cannot move between queues.

**Pros:** Simple, good for systems with clearly distinct process categories.
**Cons:** **Starvation across queues** — if Queue 1 always has work, Queue 3 (batch jobs) may never run. A process stuck in a low queue has no way to "prove" it deserves better treatment.

---

### 8.10 Multilevel Feedback Queue (MLFQ)
**Idea:** Like MLQ, but processes **can move between queues** based on their observed behavior — this is what makes it "feedback."

- A new process starts in the **highest-priority queue** (usually with a **small time quantum**).
- If it **doesn't finish** within its quantum (i.e., it's CPU-bound / behaving greedily), it's **demoted** to a lower-priority queue (with a larger quantum).
- If a process keeps **giving up the CPU early** (I/O-bound, quick bursts), it **stays high** or can even be **promoted** back up.
- Lower queues are checked only when higher queues are empty (or time-sliced with them).

```
 Queue 0 (quantum=2)   New process starts here
     │  didn't finish in time → demote
     ▼
 Queue 1 (quantum=4)
     │  didn't finish in time → demote
     ▼
 Queue 2 (quantum=8, or FCFS)
     │
     ▼
   ... lowest queue, may use FCFS
```

**Real-life analogy:** A support ticketing system — quick questions ("what's your refund policy?") get resolved fast and the agent moves to the next customer; a customer with a complex, long issue is naturally routed to a specialist queue with more time allotted, instead of hogging the fast lane.

**Pros:** Most flexible and adaptive algorithm — approximates SJF behavior (favors short/interactive jobs) without needing to know burst times in advance, since it *learns* from observed behavior.
**Cons:** Most complex to implement and tune (number of queues, quantum per queue, promotion/demotion rules); needs careful tuning to avoid starving long jobs in the lowest queue (often solved with **aging** — see §10).

---

### 8.11 Comparison Table — All Algorithms Side-by-Side

Using the same example process set (P1–P4) from above:

| Algorithm | Avg WT (ms) | Preemptive? | Starvation risk? | Key trait |
|---|---|---|---|---|
| FCFS | 5.75 | No | No | Simple, but suffers Convoy Effect |
| SJF (non-preemptive) | 5.25 | No | Yes | Optimal *non-preemptive* avg WT |
| SRTF (preemptive SJF) | **5.0** (best) | Yes | Yes | Optimal *overall* avg WT |
| Round Robin (q=2) | 9.75 | Yes | No | Fair, but highest avg WT here |
| Priority (non-preemptive) | 7.75 | No | Yes | Can starve short-but-low-priority jobs |
| HRRN | 5.25 | No | No (self-correcting) | SJF-like speed, starvation-resistant |
| MLQ | — | No | Yes (across queues) | Fixed category-based queues |
| MLFQ | — | Yes | Possible (mitigated by aging) | Learns behavior, most flexible |

> **Interview gold nugget:** SRTF gives the *theoretical minimum* average waiting time — but real systems rarely use pure SJF/SRTF because burst time isn't known in advance and long jobs would starve. This is *exactly* why Round Robin (fairness) and MLFQ (adaptive, no need to know burst time upfront) are what real OSes (Linux, Windows) actually use.

---

### 8.12 Convoy Effect (specific to FCFS)
**Definition:**
> The Convoy Effect happens in FCFS when a **long CPU-bound process** occupies the CPU first, forcing all shorter processes behind it to wait — even though they could've finished quickly if scheduled earlier. Named after a convoy of short vehicles stuck behind one slow truck on a single-lane road.

- **Cause:** FCFS has no notion of process length or priority — pure arrival order.
- **Effect:** Average waiting time balloons; I/O-bound (typically short) processes suffer especially, since they're stuck behind long CPU-bound ones, leaving I/O devices idle too.
- **Fix:** Use SJF, SRTF, Round Robin, or Priority scheduling — none of them let one long job block everyone else indefinitely.

**Concrete example (from our own FCFS table above):** P3 (burst 8 ms) ran ahead of P4 (burst 6 ms) purely because it arrived first, pushing P4's waiting time up to 13 ms — worse than if the scheduler had picked based on burst time.

### Quick Interview Q&A (Section 8)
- **Q: Formula for Turnaround Time and Waiting Time?**
  TAT = CT − AT; WT = TAT − BT.
- **Q: Which algorithm gives the minimum average waiting time overall?**
  SRTF (preemptive SJF) — theoretically optimal, but starves long jobs and has high context-switch overhead. Among non-preemptive algorithms, plain SJF is optimal.
- **Q: Difference between SJF and SRTF?**
  SJF is non-preemptive (once started, runs to completion); SRTF is preemptive — a newly arrived process with shorter *remaining* time can interrupt the running one.
- **Q: What problem does HRRN solve, and how?**
  Prevents the starvation seen in SJF by factoring in waiting time via Response Ratio = (waiting time + burst time) / burst time — the longer a process waits, the higher its ratio climbs, guaranteeing it eventually gets picked.
- **Q: Why does Round Robin have a higher average waiting time than SJF/SRTF?**
  Every process only gets a fixed quantum per turn regardless of length, so short jobs may wait behind several other processes' turns instead of finishing outright.
- **Q: What determines Round Robin's performance?**
  Time quantum size — too small → excessive context-switch overhead; too large → behaves like FCFS.
- **Q: What is the Convoy Effect and which algorithm suffers from it?**
  A long process hogs the CPU in FCFS, forcing shorter processes to wait unnecessarily — specific to FCFS (non-preemptive, arrival-order-only).
- **Q: How do you fix the Convoy Effect?**
  Switch to burst-aware or preemptive scheduling — SJF, SRTF, or Round Robin.
- **Q: What's the difference between MLQ and MLFQ?**
  MLQ permanently assigns a process to one queue based on its type; MLFQ allows processes to *move between queues* based on observed behavior (demoted if CPU-hungry, kept high if it yields quickly for I/O).
- **Q: Why don't real operating systems just use SJF/SRTF if it's optimal?**
  Because burst time isn't known in advance (only estimable), and it can starve long-running processes indefinitely. Real OSes prefer Round Robin or MLFQ, which don't require foreknowledge of burst time and remain fair.
- **Q: In priority scheduling, can a process with a short burst time still wait the longest? Give an example.**
  Yes — priority, not burst time, decides order. In our worked example, P2 (burst=3, the shortest job) had the lowest priority and ended up with the *highest* waiting time (18 ms) — this is exactly the kind of starvation aging is designed to fix.

---

## 9. Preemptive vs Non-Preemptive Scheduling

**Definitions:**
> **Preemptive scheduling:** The OS can forcibly take the CPU away from a running process (time-slice expiry, higher-priority arrival) and give it to another.
> **Non-preemptive scheduling:** Once a process gets the CPU, it keeps it until it finishes or voluntarily moves to waiting — the OS cannot interrupt it.

### Comparison Table
| Preemptive | Non-Preemptive |
|---|---|
| CPU allocated for a limited time | Process holds CPU until completion or it waits |
| Process can be interrupted mid-execution | Cannot be interrupted until it terminates/blocks itself |
| High-priority arrivals can starve low-priority ones | Long-burst process can starve short ones behind it |
| Overhead from frequent context switching | Minimal scheduling overhead |
| Lower average response time | Higher average response time |
| Scheduler decides based on priority/time slice | Process itself determines when it yields |
| Examples: Round Robin, SRTF | Examples: FCFS, SJF |

### Quick Interview Q&A
- **Q: One advantage and one disadvantage of preemptive scheduling?**
  Advantage: better average response time, prevents CPU monopolization. Disadvantage: higher overhead from frequent context switches, and can starve low-priority processes.
- **Q: Which is used in modern OSes like Linux/Windows?**
  Preemptive — it supports responsive multitasking.
- **Q: Can non-preemptive scheduling suffer starvation?**
  Yes — e.g., under FCFS or non-preemptive priority scheduling (see the Convoy Effect).

---

## 10. Starvation and Aging

**Definition:**
> **Starvation** (indefinite blocking) occurs when a process waits indefinitely for the CPU because other (typically higher-priority) processes keep getting selected first.
> **Aging** is the standard solution: gradually increase the priority of processes the longer they wait, guaranteeing eventual execution.

**Causes of starvation**
- Unfair scheduling (priority scheduling continuously favoring high-priority jobs)
- Limited/contended resources

**How aging works (example)**
- Priorities range 0 (highest) to 127 (lowest).
- A waiting process's priority is bumped up (e.g., by 1 level every 15 minutes) until it eventually becomes highest priority and runs.

**Tie-in to our worked example:** In §8.7, P2 waited 18 ms despite having the shortest burst, purely due to a low priority number. With aging, P2's priority would have been incremented the longer it waited, until it eventually outranked P3/P4 and got scheduled — this is precisely the mechanism that fixes the priority-inversion-style starvation shown in that example.

### Starvation vs Aging
| Starvation | Aging |
|---|---|
| Process waits indefinitely | Technique to gradually raise priority |
| It's a problem | It's the solution |
| Priority of waiting process stays fixed | Priority increases over time |
| Process may never run | Process guaranteed to eventually run |

### Quick Interview Q&A
- **Q: What is starvation and which algorithm is most prone to it?**
  Indefinite waiting for CPU/resources; Priority Scheduling and SJF are classic examples, since low-priority/long jobs can be perpetually skipped.
- **Q: How does aging prevent starvation?**
  By incrementally raising a waiting process's priority until it reaches the highest priority and gets scheduled.
- **Q: Does Round Robin suffer from starvation?**
  No — every process gets a guaranteed turn (time quantum) in cyclic order.

---

## 11. What Happens When You Run a Program (Synthesis Question)

Classic "walk me through it end-to-end" question tying the whole topic together:

1. **You double-click the executable / type the command.**
   The shell (or GUI) makes a `fork()` system call, creating a child process that's a duplicate of the shell.
2. **The child calls `exec()`.**
   Replaces the child's memory image with the new program's code, data, stack — same PID, entirely new program.
3. **The OS loader sets up the process.**
   - A new **PCB** is created with a fresh PID, initial Process State = **New**.
   - Memory layout prepared: text segment loaded (often lazily via demand paging), data/heap/stack initialized.
   - Program Counter set to the entry point.
4. **Admission.**
   The **Long-Term Scheduler** (if present) admits the process into memory, moving it to **Ready**, placing its PCB in the ready queue.
5. **Scheduling.**
   The **Short-Term (CPU) Scheduler** eventually picks this process based on the active algorithm (FCFS/SJF/RR/Priority/MLFQ/etc.).
6. **Dispatch.**
   The **Dispatcher** performs a **context switch**: saves the outgoing process's registers/PC into its PCB, loads this process's saved context, switches to user mode, jumps to its PC. State → **Running**.
7. **Execution proceeds.**
   - If it needs I/O, it issues a system call and moves to **Blocked/Waiting**; the Short-Term Scheduler picks another ready process (keeps CPU busy — the whole point of multiprogramming).
   - If its time quantum expires (preemptive scheduling) or a higher-priority process arrives, it's preempted back to **Ready**.
   - If memory gets tight, the **Medium-Term Scheduler** may swap it out to disk (**Suspended**) and back in later.
8. **Termination.**
   On finishing (or being killed), it enters **Terminated**. The OS reclaims memory, closes file descriptors, and (on Unix) leaves a **zombie** entry until the parent calls `wait()` — after which the PCB/process-table entry is finally removed.

### Quick Interview Q&A
- **Q: Walk me through what happens when you run `./a.out` in a Linux terminal.**
  Shell forks a child (`fork()`), child execs the new program (`exec()`), OS sets up a fresh PCB and memory layout, process is admitted to Ready, Short-Term Scheduler eventually dispatches it via a context switch, it cycles through Ready/Running/Blocked due to I/O or preemption, then terminates — becoming a zombie briefly until the parent reaps it with `wait()`.
- **Q: At what point does the process actually start running on the CPU?**
  Only after the Dispatcher performs a context switch and the Short-Term Scheduler has selected it — not immediately after `exec()`, since it must first sit in the Ready queue.
- **Q: What's the very first state of a newly created process, and the state right before it can execute?**
  New (right after fork/exec sets it up) → Ready (once admitted, waiting for CPU) → Running (once dispatched).

---

## 12. Rapid-Fire One-Liners (Last-Minute Revision)

- Process = program in execution (active vs passive entity).
- `fork()` creates a new process (duplicate, COW-optimized); `exec()` replaces the current process's image — same PID, new program.
- Zombie = finished but not yet reaped by parent's `wait()`; Orphan = still running, but parent died first (re-parented to init/systemd).
- PCB = data structure holding all process metadata; Process Table = array of PCBs.
- 5-state model: New → Ready → Running → (Blocked ⇄ Ready) → Terminated.
- New & Terminated occur exactly once; Ready/Running/Blocked can repeat.
- Long-Term Scheduler controls degree of multiprogramming; Short-Term is fastest; Medium-Term handles swapping.
- Dispatcher ≠ Scheduler: dispatcher executes the scheduler's decision (context switch).
- TAT = CT − AT; WT = TAT − BT.
- **FCFS**: simplest, suffers Convoy Effect, no starvation.
- **SJF**: optimal non-preemptive avg WT, can starve long jobs.
- **SRTF**: optimal overall avg WT (preemptive SJF), more overhead, still starves long jobs.
- **Round Robin**: fairest, no starvation, but usually highest avg WT; sensitive to quantum size.
- **Priority**: respects urgency, but can starve low-priority jobs regardless of their burst time.
- **HRRN**: SJF-like speed without the starvation, via Response Ratio = (WT+BT)/BT.
- **MLQ**: fixed queues by category, can starve lower queues entirely.
- **MLFQ**: adaptive, moves processes between queues based on behavior — most flexible, most complex.
- Convoy Effect → FCFS-specific: one long process blocks all short ones behind it.
- Preemptive: CPU can be taken away (RR, SRTF, preemptive Priority). Non-preemptive: runs till completion/block (FCFS, SJF).
- Starvation = indefinite wait; Aging = fix by increasing priority over time.
- Running a program = fork → exec → PCB/memory setup → Ready → dispatched (context switch) → Running → (Blocked/Ready cycles) → Terminated → zombie → reaped.

---

*Compiled for placement/interview prep — definitions, comparison tables, formulas, mechanics (fork/exec, zombie/orphan), fully worked Gantt-chart examples for every major scheduling algorithm on one shared process set, the convoy effect, an end-to-end program-execution walkthrough, and likely Q&A for each subtopic.*
