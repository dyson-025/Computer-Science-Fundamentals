# DBMS: Transactions & Concurrency Control — SDE Interview Notes

Covers: ACID Properties, Why Concurrency Control is Needed, Locks (Shared/Exclusive), Two-Phase Locking (2PL), Deadlocks (Detection & Prevention), Starvation, Log-Based Recovery (Undo/Redo, Checkpoints), and Rapid-Fire Revision.

---

## 1. ACID Properties ⭐⭐⭐

**Definition:**
> ACID is a set of properties that guarantee database transactions are processed reliably, even during crashes or concurrent access.

### The Four Properties
| Property | Meaning | Who Enforces It |
|---|---|---|
| **Atomicity** | All operations in a transaction succeed, or none do | Transaction Manager |
| **Consistency** | DB moves from one valid state to another; all constraints hold | Application Programmer |
| **Isolation** | Concurrent transactions don't see each other's uncommitted changes | Concurrency Control Manager |
| **Durability** | Once committed, changes survive a crash | Recovery Manager |

**Example (Atomicity):** Transfer $100 from A→B = (debit A, credit B). If the system crashes after debiting A but before crediting B, the whole transaction rolls back — A is never left short with no corresponding credit to B.

**Example (Consistency):** Total money in the bank before and after a transfer must stay the same. If $50 leaves A but never reaches B, that's a consistency violation → rollback.

**Example (Isolation):** While T1 is transferring money (not yet committed), T2 calculating total balance should NOT see T1's half-done state.
Isolation prevents:
- **Dirty Read** — reading uncommitted data
- **Non-repeatable Read** — same row gives different values on re-read within one transaction
- **Phantom Read** — new rows appear mid-transaction

**Example (Durability):** After you get "Payment Successful," even if the server crashes a second later, that payment record isn't lost.

**Interview Q&A:**
- **Explain ACID with a bank transfer example.** (Use the four examples above — atomicity/consistency/isolation/durability each map cleanly onto a money transfer.)
- **Which DB component is responsible for each property?** Atomicity → Transaction Manager, Consistency → Application logic, Isolation → Concurrency Control Manager, Durability → Recovery Manager.
- **What's the difference between dirty read, non-repeatable read, and phantom read?** Dirty read = reading uncommitted data; non-repeatable read = same row changes between two reads in one transaction; phantom read = new rows appear mid-transaction.

---

## 2. Why Concurrency Control Is Needed ⭐⭐

**Definition:**
> Multiple transactions run at the same time for performance. Without control, this causes conflicts that corrupt data or produce wrong results.

### Problems Without Concurrency Control
| Problem | What Happens | Example |
|---|---|---|
| **Lost Update** | Two transactions write the same item; one overwrites the other | T1 sets stock=5, T2 sets stock=3 (based on stale read) — T1's update lost |
| **Dirty Read** | Reading data another transaction hasn't committed yet | T2 reads T1's uncommitted balance update; T1 then rolls back → T2 used bad data |
| **Inconsistent Read** | Same row read twice in one transaction gives different results | T1 reads price=100, then reads again mid-transaction and gets price=120 |

**Goal:** Achieve **serializability** — concurrent execution produces the same result as *some* sequential execution of the same transactions.

**Interview Q&A:**
- **Give a real scenario where missing concurrency control breaks something.** Double-booking a seat, overselling inventory in an e-commerce flash sale — both trace back to a lost update or dirty read.
- **What is serializability?** The property that a concurrent schedule's outcome matches some serial (one-at-a-time) execution order.

---

## 3. Locks: Shared vs Exclusive ⭐⭐⭐

**Definition:**
> A lock restricts access to a data item so only permitted transactions can read/write it at a time.

- **Shared Lock (S)** — for reading only. Multiple transactions can hold S-locks on the same item at once.
- **Exclusive Lock (X)** — for reading + writing. Only one transaction can hold it; blocks all other locks on that item.

### Compatibility Matrix
| Held \ Requested | S | X |
|---|---|---|
| **S** | ✅ Allowed | ❌ Blocked |
| **X** | ❌ Blocked | ❌ Blocked |

**Example:** Two users viewing (S-lock) the same product listing — fine, both proceed. One user updating stock (X-lock) while another reads — the reader waits until the write commits.

**Interview Q&A:**
- **Shared vs Exclusive lock — give a real scenario for each.** Shared: multiple users reading a product page simultaneously. Exclusive: one user checking out and decrementing stock — no one else can read/write that row until it commits.

---

## 4. Two-Phase Locking (2PL) ⭐⭐⭐ (Most-asked topic in this area)

**Definition:**
> A protocol where every transaction acquires and releases locks in two distinct phases, guaranteeing conflict serializability.

1. **Growing Phase** — can acquire locks, cannot release any.
2. **Shrinking Phase** — can release locks, cannot acquire any.

```
 Locks held
    ▲
    │        ╱╲
    │       ╱  ╲
    │      ╱    ╲
    │     ╱      ╲
    └────╱────────╲────▶ time
      Growing   Shrinking
       Phase      Phase
```

> **Guarantees conflict serializability.** This is the single most important fact to remember.

### Variants (know the difference)
| Variant | Rule | Practical Use |
|---|---|---|
| **Strict 2PL** | Hold all **X-locks** until commit/abort | Prevents cascading rollbacks — widely used |
| **Rigorous 2PL** | Hold **all locks (S+X)** until commit/abort | Simplest to reason about; used by most commercial DBs |

**Common misconception to correct in interviews:**
> "Does 2PL prevent deadlocks?" → **No.** 2PL only guarantees serializability. Deadlocks can still happen and must be handled separately (detection or prevention).

**Interview Q&A:**
- **What is 2PL? Does it prevent deadlocks?** Growing + shrinking phase locking protocol that guarantees conflict serializability — but does NOT prevent deadlocks.
- **Strict 2PL vs Rigorous 2PL?** Strict releases S-locks any time but holds X-locks till commit (avoids cascading rollback). Rigorous holds everything (S+X) till commit — simplest to reason about, most commonly used in practice.

---

## 5. Deadlocks ⭐⭐⭐

**Definition:**
> A deadlock is a circular wait — T1 waits for a lock held by T2, T2 waits for a lock held by T1. Neither can proceed.

**Example:** T1 holds a lock on Row A, wants Row B. T2 holds a lock on Row B, wants Row A. Classic deadlock — a very common whiteboard question.

```
   T1 ──waits for──▶ Row B (held by T2)
   ▲                         │
   │                         ▼
   Row A (held by T1) ◀──waits for── T2
```

### Detection: Wait-For Graph
- Node = transaction. Edge Ti → Tj means "Ti is waiting on a resource held by Tj."
- **A cycle in the graph = deadlock.** DBMS picks a victim transaction to abort and break the cycle.

### Prevention (Timestamp-based)
| Scheme | Rule | Mnemonic |
|---|---|---|
| **Wait-Die** | Older transaction may wait for younger; younger requesting older's lock is aborted | "Old **waits**, young **dies**" |
| **Wound-Wait** | Older transaction preempts (aborts) younger holding a lock it needs; younger waits for older | "Old **wounds** young, young **waits**" |

Both avoid deadlocks entirely because waiting only ever flows in one direction (old→young or young→old), so cycles can't form.

**Interview Q&A:**
- **How do you detect a deadlock? How do you prevent one?** Detection: build a wait-for graph, look for cycles, abort a victim. Prevention: Wait-Die or Wound-Wait schemes based on transaction timestamps.
- **Explain Wait-Die vs Wound-Wait with an example.** Wait-Die: T1(old) requests a lock held by T2(young) → T1 waits. T2(young) requests a lock held by T1(old) → T2 is aborted. Wound-Wait: reverses this — the older transaction wounds (aborts) the younger one instead of waiting.
- **How would you debug a production deadlock?** Check DB logs/wait-for graph, identify the two conflicting queries, and fix by enforcing a consistent lock acquisition order across the app (a very common practical fix).

---

## 6. Starvation ⭐

**Definition:**
> A transaction waits indefinitely because the system keeps favoring other transactions for the same lock.

**Example:** T1 wants an X-lock, but a continuous stream of other transactions keep getting S-locks first, so T1 never gets its turn.

**Fix:** FCFS-style lock queuing — grant locks in the order requests arrived, not just whichever is compatible first.

**Interview Q&A:**
- **What is starvation and how do you avoid it?** A transaction waits indefinitely because other transactions keep jumping ahead of it. Avoided using fair (FCFS) lock queuing.

---

## 7. Recovery: Undo / Redo & Logging ⭐⭐

**Definition:**
> Every operation is logged **before** it's applied to the DB — a principle called Write-Ahead Logging (WAL) — so the system can always recover to a consistent state after a crash.

### Log Record Formats
```
<Tn, Start>
<Tn, Attribute, Old_Value, New_Value>
<Tn, Commit>
```

### Undo vs Redo
| Log shows | Action | Why |
|---|---|---|
| `<Ti, Start>` ... no `<Ti, Commit>` | **UNDO** | Transaction never finished — revert using old values |
| `<Ti, Start>` ... `<Ti, Commit>` | **REDO** | Transaction finished but changes may not have hit disk yet — reapply new values |

**Example (Undo):** T1 updates balance 500→600 but crashes before commit → **undo**, restore 500.
**Example (Redo):** T2 updates balance 300→400 and commits, but crash happens before the write is flushed to disk → **redo**, reapply 400.

### Checkpoints
Periodically save DB state + active transaction list (`<checkpoint L>`). On recovery, only scan the log back to the oldest active transaction at the last checkpoint — not the entire log history. This is what makes recovery fast in real systems.

```
 log start ───────[checkpoint]───────[crash]
                        │
              only scan from here
              back to oldest active txn
```

**Interview Q&A:**
- **Walk me through how a database recovers after a crash.** WAL ensures every change is logged first → scan the log → classify transactions as committed or uncommitted → undo uncommitted, redo committed → checkpoints limit how far back you need to scan, making recovery fast.
- **Why log before writing to the DB?** So there's always a durable record to recover from, even if the crash interrupts the actual data write.

---

## 🎯 Rapid-Fire One-Liners (Last-Minute Revision)

- ACID = Atomicity, Consistency, Isolation, Durability — reliability guarantees for transactions.
- Atomicity → Transaction Manager; Consistency → App logic; Isolation → Concurrency Control Manager; Durability → Recovery Manager.
- Dirty read = uncommitted data read; non-repeatable read = value changes between two reads; phantom read = new rows appear mid-transaction.
- Concurrency control exists to prevent lost updates, dirty reads, inconsistent reads — goal is serializability.
- Shared (S) lock = read-only, multiple holders OK. Exclusive (X) lock = read+write, only one holder, blocks everything else.
- **2PL** = Growing phase (acquire only) + Shrinking phase (release only) → guarantees conflict serializability, but does **NOT** prevent deadlocks.
- **Strict 2PL** holds X-locks till commit (avoids cascading rollback); **Rigorous 2PL** holds all locks till commit (simplest, most used).
- Deadlock = circular wait between transactions. Detect via **Wait-For Graph** (cycle = deadlock); break by aborting a victim.
- **Wait-Die**: old waits, young dies (aborted). **Wound-Wait**: old wounds (aborts) young, young waits.
- Starvation = indefinite wait for a lock; fixed via FCFS-style fair queuing.
- Log-based recovery uses **Write-Ahead Logging**: `<Tn, Start>`, `<Tn, Attr, Old, New>`, `<Tn, Commit>`.
- No commit in log → **Undo**. Commit in log → **Redo**.
- Checkpoints limit how far back recovery needs to scan → faster crash recovery.
