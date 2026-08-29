# DBMS: Normalization — SDE Interview Notes

Covers: Introduction to Normalization, The Problem of Redundancy, Normal Forms (1NF–BCNF), Lossless Join Decomposition, Denormalization, and Data Replication.

---

## 1. Introduction to Normalization ⭐⭐⭐

**Definition:**
> Normalization is the process of organizing data in a database to reduce redundancy and improve data integrity. It involves decomposing large tables into smaller, well-structured tables and defining relationships between them using foreign keys — without losing any information.

**Why it matters:** A poorly designed table (everything crammed into one big table) leads to wasted storage, inconsistent data, and error-prone updates. Normalization fixes this by applying a series of rules called **Normal Forms**.

**Example:** Imagine one giant `Orders` table with columns `OrderID, CustomerName, CustomerAddress, ProductName, ProductPrice`. If the same customer places 5 orders, their name and address get repeated 5 times — that's redundancy waiting to cause bugs (e.g., update the address in 4 rows but forget the 5th).

**Interview Q&A:**
- **What is normalization and why do we need it?** The process of structuring tables to minimize redundancy and avoid data anomalies, by breaking large tables into smaller related ones connected via foreign keys.

---

## 2. The Problem of Redundancy ⭐⭐⭐

**Definition:**
> Redundancy means the same piece of data is stored in multiple places unnecessarily. It's the root cause normalization is designed to eliminate, because it leads to three types of anomalies.

### The Three Anomalies (very commonly asked — know all three cold)

**1. Insertion Anomaly** — You can't insert certain data without also having unrelated data available.
> Example: In a combined `Student_Course` table, you can't add a new course to the catalog until at least one student enrolls in it — because `CourseName` only exists alongside a `StudentID` row.

**2. Update Anomaly** — The same fact is stored in multiple rows, so updating it means changing *every* occurrence — miss one, and the data becomes inconsistent.
> Example: If a professor's department changes, and that professor's name+department appears in 50 rows (one per student they teach), you'd have to update all 50 rows. Miss even one, and now the database has conflicting information about the same professor.

**3. Deletion Anomaly** — Deleting one piece of information unintentionally wipes out other, unrelated information.
> Example: If the last student enrolled in a course drops it, and that row is deleted, you might accidentally lose all record that the course itself ever existed.

```
 Unnormalized Table (redundancy)
 ┌──────────┬───────────┬──────────────┐
 │ StudentID │ CourseName │ ProfName      │  ← ProfName repeated for every student in that course
 ├──────────┼───────────┼──────────────┤
 │ S1        │ DBMS       │ Dr. Rao       │
 │ S2        │ DBMS       │ Dr. Rao       │  ← redundant! update anomaly risk
 │ S3        │ OS         │ Dr. Iyer      │
 └──────────┴───────────┴──────────────┘
```

**Interview Q&A:**
- **Explain the three anomalies with an example.** Insertion: can't add a course without a student. Update: changing a repeated fact requires updating every row it appears in. Deletion: removing one record accidentally destroys unrelated information.
- **How does normalization fix these anomalies?** By splitting the table so each fact is stored in exactly one place — e.g., a separate `Courses` table and `Students` table linked by `CourseID`, so a course can exist independently of any enrolled student.

---

## 3. Normal Forms ⭐⭐⭐ (Most-asked topic in this section)

Normal Forms are a series of **progressively stricter rules** — each one builds on the previous. A table in 3NF is automatically in 2NF and 1NF too.

### Functional Dependency (quick refresher — needed to understand every normal form)
> `A → B` means: given a value of A, the value of B is uniquely determined.
> Example: `StudentID → StudentName` — knowing the StudentID tells you exactly one name.

---

### 1NF — First Normal Form
**Rule:** Every column must hold **atomic (indivisible)** values — no repeating groups, no multi-valued/composite cells.

**Example (violates 1NF):**
| StudentID | Subjects |
|---|---|
| S1 | Math, Physics |

**Fixed (1NF):**
| StudentID | Subject |
|---|---|
| S1 | Math |
| S1 | Physics |

---

### 2NF — Second Normal Form
**Rule:** Must be in 1NF, **and** every non-key attribute must depend on the **whole** primary key — not just part of it. (Only relevant when you have a **composite** primary key.)

**Example (violates 2NF):** Table with composite key `(StudentID, CourseID)`:
| StudentID | CourseID | CourseName | Marks |
|---|---|---|---|
| S1 | C1 | DBMS | 90 |

Here `CourseName` depends only on `CourseID` (part of the key), not on the full `(StudentID, CourseID)` — this is a **partial dependency**, which violates 2NF.

**Fixed (2NF):** Split into two tables:
- `Enrollment(StudentID, CourseID, Marks)`
- `Course(CourseID, CourseName)`

---

### 3NF — Third Normal Form
**Rule:** Must be in 2NF, **and** no **transitive dependency** — a non-key attribute must not depend on another non-key attribute.

**Example (violates 3NF):**
| StudentID | DeptID | DeptName |
|---|---|---|

Here `DeptName` depends on `DeptID`, and `DeptID` depends on `StudentID` — so `DeptName` depends on `StudentID` only *transitively* (through `DeptID`). This violates 3NF.

**Fixed (3NF):** Split into:
- `Student(StudentID, DeptID)`
- `Department(DeptID, DeptName)`

---

### BCNF — Boyce-Codd Normal Form (stricter version of 3NF)
**Rule:** For every functional dependency `A → B`, `A` must be a **super key**. (Handles edge cases 3NF misses, usually involving multiple overlapping candidate keys.)

**Example (violates BCNF but satisfies 3NF):** A table where `(Student, Subject) → Teacher` and also `Teacher → Subject` — here `Teacher` is not a super key, but it determines `Subject`, violating BCNF even though the table might pass 3NF's rules.

### Normal Forms Comparison Table
| Normal Form | Rule | Fixes |
|---|---|---|
| **1NF** | Atomic values only | Repeating groups / multi-valued cells |
| **2NF** | 1NF + no partial dependency | Non-key attributes depending on only part of a composite key |
| **3NF** | 2NF + no transitive dependency | Non-key attributes depending on other non-key attributes |
| **BCNF** | Every determinant must be a super key | Edge cases 3NF misses (overlapping candidate keys) |

**Interview Q&A:**
- **Explain 1NF, 2NF, 3NF, BCNF with examples.** (Use the four examples above — this is almost guaranteed to be asked, know each one's rule + a quick example.)
- **Difference between 3NF and BCNF?** 3NF allows a non-key attribute to determine another attribute if the determining attribute is part of a candidate key; BCNF is stricter — it requires every determinant (left side of a functional dependency) to be a super key, with no exceptions.
- **What is a transitive dependency?** When a non-key attribute depends on another non-key attribute, rather than directly on the primary key (A→B→C, so C transitively depends on A).
- **Is every 3NF table also in BCNF?** Not necessarily — BCNF is stricter. A table can satisfy 3NF but still violate BCNF in cases with overlapping composite candidate keys.

---

## 4. Lossless Join Decomposition ⭐⭐

**Definition:**
> When a table is decomposed into smaller tables (during normalization), the decomposition is "lossless" if you can always reconstruct the exact original table by joining the smaller tables back together — no extra or missing rows.

**Why it matters:** Normalization always involves splitting tables — but a *bad* split can lose information or create spurious rows when joined back. Lossless decomposition guarantees this never happens.

**Rule of thumb:** A decomposition of R into R1 and R2 is lossless if the common attribute(s) between R1 and R2 form a **candidate key** for at least one of them (i.e., `R1 ∩ R2 → R1` or `R1 ∩ R2 → R2`).

**Example:**
- `Employee(EmpID, Name, DeptID)` split into `Emp(EmpID, Name)` and `Dept(EmpID, DeptID)` — since `EmpID` is a key in both, joining them back on `EmpID` perfectly reconstructs the original table → **lossless**.
- A bad split (e.g., splitting on a non-key column) could introduce duplicate/incorrect rows on rejoin → **lossy**.

**Interview Q&A:**
- **What is lossless join decomposition, and why does it matter?** A decomposition where joining the smaller tables back together exactly reproduces the original table with no extra/missing rows — critical because normalization always splits tables, and a lossy split would silently corrupt query results.
- **How do you check if a decomposition is lossless?** Verify that the common attribute(s) between the two resulting tables form a candidate key in at least one of them.

---

## 5. Denormalization ⭐⭐⭐ (Very practical, frequently asked)

**Definition:**
> Denormalization is the deliberate process of introducing some redundancy back into a normalized database — usually by merging tables or duplicating data — to improve **read performance**, at the cost of extra storage and more complex writes.

**Why it's needed:** Highly normalized schemas require many `JOIN`s to answer a query. In read-heavy systems (analytics dashboards, reporting, high-traffic APIs), those joins become a performance bottleneck. Denormalization trades some redundancy for fewer joins and faster reads.

**Example:** Instead of `Orders(OrderID, CustomerID)` joined with `Customers(CustomerID, CustomerName)` every time you display an order, you might store `CustomerName` directly inside the `Orders` table too — one extra copy of data, but no join needed to show an order with the customer's name.

### Normalization vs Denormalization
| Aspect | Normalization | Denormalization |
|---|---|---|
| Goal | Reduce redundancy, ensure consistency | Improve read performance |
| Storage | Less | More (duplicated data) |
| Writes | Simple, single source of truth | More complex (must keep duplicates in sync) |
| Reads | Slower (needs joins) | Faster (fewer/no joins) |
| Best for | OLTP (transactional systems — banking, orders) | OLAP / reporting / read-heavy systems |

**Interview Q&A:**
- **When would you denormalize a database?** In read-heavy systems where join performance becomes a bottleneck — e.g., analytics/reporting dashboards, or caching frequently-joined data directly into a single table for a hot API endpoint.
- **What's the trade-off of denormalization?** Faster reads, but higher storage use and the risk of data inconsistency if duplicated fields aren't updated everywhere they appear.
- **Give a real scenario for denormalization.** An e-commerce order history page — instead of joining `Orders` with `Products` and `Customers` on every page load, you might snapshot the product name/price and customer name directly into the order record at the time of purchase (also useful since product prices might change later, but the order should reflect the price *at the time*).

---

## 6. Data Replication ⭐⭐ (More of a system-design angle)

**Definition:**
> Data replication is the process of storing copies of the same data on multiple database servers (nodes), so the data is available from more than one location — improving availability, fault tolerance, and read performance.

### Common Replication Models
| Model | How it works | Trade-off |
|---|---|---|
| **Master-Slave (Primary-Replica)** | All writes go to the master; slaves replicate from it and serve reads | Simple, scales reads well, but slave data can lag behind (replication lag) |
| **Master-Master (Multi-Primary)** | Multiple nodes accept writes and sync with each other | Higher write availability, but risk of write conflicts needing resolution |

**Example:** A high-traffic e-commerce site might have one master DB handling all checkouts (writes), with 3 read replicas serving product listing pages — spreading out read load and keeping the site fast even under heavy traffic.

**Replication Lag:** The delay between a write hitting the master and that write becoming visible on a replica. Can cause a user to write data and then immediately not see it if their next request is routed to a lagging replica (a classic real-world consistency bug).

**Interview Q&A:**
- **What is data replication and why is it used?** Keeping copies of data across multiple servers to improve availability, fault tolerance, and distribute read load.
- **What is replication lag, and what problem can it cause?** The delay before a write on the master is reflected on a replica; can cause a user to not see their own just-written data if reads are routed to a lagging replica — a classic read-your-writes consistency issue.
- **Master-Slave vs Master-Master replication?** Master-Slave: single write source, simpler, no conflict resolution needed, but slaves can lag. Master-Master: multiple write sources, higher availability, but requires conflict resolution when the same data is written differently on two masters.

---

## 🎯 Rapid-Fire One-Liners (Last-Minute Revision)

- Normalization = organizing tables to reduce redundancy and prevent anomalies.
- Three anomalies: **Insertion** (can't add data without unrelated data), **Update** (must change many rows for one fact), **Deletion** (removing one thing accidentally removes another).
- **1NF**: atomic values only, no repeating groups.
- **2NF**: 1NF + no partial dependency (non-key attr depends on the *whole* composite key).
- **3NF**: 2NF + no transitive dependency (non-key attr shouldn't depend on another non-key attr).
- **BCNF**: every determinant (left side of a functional dependency) must be a super key — stricter than 3NF.
- **Lossless Join Decomposition**: splitting a table such that joining the pieces back reconstructs the original exactly — check via common attribute(s) forming a candidate key.
- **Denormalization**: intentionally reintroducing redundancy to speed up reads — trades storage/write complexity for fewer joins. Best for OLAP/reporting; normalization is best for OLTP.
- **Data Replication**: copies of data across multiple servers for availability + read scaling. Master-Slave (simple, can lag) vs Master-Master (higher availability, conflict risk).
- Replication lag = delay before a replica reflects the master's latest write — classic source of read-your-writes bugs.
