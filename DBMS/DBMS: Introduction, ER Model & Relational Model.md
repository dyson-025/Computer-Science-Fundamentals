# DBMS: Introduction, ER Model & Relational Model — SDE Interview Notes

Covers: DBMS Introduction, Need for DBMS, DBMS Architectures (1/2/3-Tier), ER Model, Generalization/Specialization/Aggregation, Relational Model, Keys, Functional Dependency, Attribute Closure, and Mapping ER Model to Relational Model.

---

## 1. Introduction to DBMS ⭐⭐⭐

**Definition:**
> A DBMS (Database Management System) is software that lets users create, store, manage, and retrieve data in a structured, efficient, and secure way. It sits between the application and the raw data, handling storage, retrieval, concurrent access, and integrity — so applications don't have to manage files directly.

**Example:** Instead of an app manually opening/reading/writing raw files on disk (and dealing with corruption, concurrent access bugs, etc.), it just sends a query like `SELECT * FROM Users WHERE id=5` to the DBMS, which handles everything underneath.

### DBMS vs File System (classic comparison question)
| Aspect | File System | DBMS |
|---|---|---|
| Data redundancy | High (each app stores its own copies) | Low (centralized, shared data) |
| Data integrity | Hard to enforce (no built-in constraints) | Enforced via constraints (PK, FK, etc.) |
| Concurrent access | Prone to conflicts/corruption | Managed via concurrency control (locks, transactions) |
| Security | Basic file permissions only | Fine-grained access control (per-user, per-table) |
| Backup & Recovery | Manual, error-prone | Built-in recovery mechanisms (logs, checkpoints) |
| Querying | Custom code needed for every search | Declarative queries (SQL) |

**Interview Q&A:**
- **What is a DBMS?** Software that manages structured data — storage, retrieval, security, concurrency, and integrity — so applications don't deal with raw files directly.
- **DBMS vs File System — key differences?** DBMS centralizes data, enforces integrity constraints, handles concurrent access safely, and supports declarative querying (SQL); file systems leave all of that to the application.

---

## 2. Need for DBMS ⭐⭐

**Definition:**
> Before DBMS, applications stored data in flat files, leading to redundancy, inconsistency, and poor concurrent access. DBMS was built to solve these exact problems.

### Problems DBMS Solves
| Problem (in file systems) | How DBMS Fixes It |
|---|---|
| **Data Redundancy** — same data duplicated across files | Centralized storage — one copy, shared by all apps |
| **Data Inconsistency** — duplicated data can go out of sync | Normalization + constraints keep data consistent |
| **Difficult Concurrent Access** — two processes writing the same file can corrupt it | Concurrency control (locks, transactions) |
| **No Data Integrity** — nothing stops invalid data from being saved | Constraints (PK, FK, NOT NULL, CHECK) enforce validity |
| **Poor Security** — file-level permissions only | Fine-grained access control down to table/row/column level |
| **No Backup/Recovery** — a crash could permanently lose data | Logging + checkpointing |

**Example:** Imagine a bank storing account balances in plain text files. Two tellers process withdrawals from the same account at the same time — without proper concurrency control, one update could silently overwrite the other, losing money from the record. A DBMS prevents this via locking/transactions.

**Interview Q&A:**
- **Why do we need a DBMS instead of just using files?** Files can't reliably handle concurrent access, don't enforce data integrity, and require every application to write custom code for searching/updating — a DBMS solves all of this centrally.

---

## 3. DBMS Architectures — 1-Tier, 2-Tier, 3-Tier ⭐⭐⭐ (Maps directly to real app architecture)

**Definition:**
> DBMS architecture describes how the database, application logic, and user interface are distributed across layers ("tiers"). More tiers generally mean better separation of concerns, scalability, and security.

### 1-Tier Architecture
The database is directly accessible to the user — client, application, and DB all live on the **same machine**. No network involved.

**Example:** A local SQLite database used by a single desktop app (like a note-taking app storing data on your own laptop).

```
[ User ] ──▶ [ DBMS ]   (same machine)
```

### 2-Tier Architecture
The application (client) talks **directly** to the database server over a network. No middle layer.

**Example:** A desktop banking application installed on a teller's PC that connects straight to the bank's central database server.

```
[ Client App ] ──(network)──▶ [ Database Server ]
```

**Drawback:** Business logic is often embedded in the client — harder to maintain/update across many client machines, and the DB is more exposed to direct client connections.

### 3-Tier Architecture ⭐⭐⭐ (Most important — this is how modern web apps work)
Adds a **middle application/business logic layer** between the client and the database.

```
[ Client (Browser/App) ] ──▶ [ Application Server (business logic/API) ] ──▶ [ Database Server ]
      Presentation Tier              Application/Logic Tier                    Data Tier
```

**Example:** Any typical web app — your browser (client) talks to a backend API server (Node.js/Django/Spring), which talks to the database (PostgreSQL/MySQL). The browser never touches the DB directly.

**Why 3-tier is preferred in practice:**
- **Security** — clients never connect directly to the database.
- **Scalability** — you can scale the app server layer independently of the database.
- **Maintainability** — business logic changes don't require touching the client or the DB.

**Interview Q&A:**
- **Explain 2-tier vs 3-tier architecture.** 2-tier: client talks directly to the DB (business logic often lives in the client). 3-tier: an application/API server sits between the client and DB, handling business logic — this is the standard architecture behind virtually all modern web/mobile apps.
- **Why is 3-tier preferred for web applications?** Better security (DB isn't directly exposed to clients), independent scalability of each layer, and centralized business logic that's easier to maintain and update.

---

## 4. ER (Entity-Relationship) Model ⭐⭐⭐

**Definition:**
> The ER Model is a conceptual way to design a database by representing real-world things as **entities**, their properties as **attributes**, and how entities connect to each other as **relationships** — before ever writing actual tables/SQL.

**Example:** Designing a college database — `Student` and `Course` are entities; `Name`, `RollNo` are attributes of `Student`; `Enrolls In` is a relationship connecting `Student` to `Course`.

### Core Building Blocks
| Term | Meaning | Example |
|---|---|---|
| **Entity** | A real-world object/concept with independent existence | `Student`, `Course`, `Employee` |
| **Attribute** | A property describing an entity | `Name`, `Age`, `Salary` |
| **Relationship** | An association between two or more entities | `Student` **enrolls in** `Course` |
| **Entity Set** | A collection of similar entities | All rows of `Student` |

### Types of Attributes
| Type | Meaning | Example |
|---|---|---|
| **Simple** | Cannot be divided further | `Age` |
| **Composite** | Can be broken into sub-parts | `Name` → `FirstName` + `LastName` |
| **Derived** | Calculated from another attribute | `Age` derived from `DateOfBirth` |
| **Multi-valued** | Can hold more than one value | `PhoneNumbers` (a person can have several) |

### Relationship Cardinality (very commonly asked)
| Type | Meaning | Example |
|---|---|---|
| **One-to-One (1:1)** | One entity relates to exactly one of another | `Person` ↔ `Passport` |
| **One-to-Many (1:N)** | One entity relates to many of another | `Department` → many `Employees` |
| **Many-to-Many (M:N)** | Many entities relate to many others | `Student` ↔ `Course` (many students take many courses) |

```
 Student ──enrolls in──▶ Course
   (M)                     (N)
   Many students can enroll in many courses (M:N relationship)
```

**Interview Q&A:**
- **What is the ER Model?** A conceptual database design technique representing real-world data using entities, attributes, and relationships, before implementation as actual tables.
- **Explain the types of relationship cardinality with examples.** 1:1 (Person↔Passport), 1:N (Department→Employees), M:N (Students↔Courses) — always give a concrete example for each when asked.
- **Difference between simple and composite attributes?** Simple attributes can't be broken down further (e.g., `Age`); composite ones can be split into meaningful sub-parts (e.g., `Name` → `First` + `Last`).

---

## 5. Generalization, Specialization & Aggregation ⭐⭐

**Definition:**
> These are ER modeling techniques for handling entities with shared or hierarchical characteristics — similar in spirit to inheritance in OOP.

### Generalization (Bottom-Up)
Combining multiple entities that share common attributes into a single, more general entity.

**Example:** `Car` and `Truck` entities both have `RegistrationNumber`, `Owner` → generalized into a single `Vehicle` entity.

### Specialization (Top-Down)
Splitting a general entity into more specific sub-entities based on distinguishing features.

**Example:** An `Employee` entity is specialized into `Manager` and `Developer`, each having unique extra attributes (`TeamSize` for Manager, `ProgrammingLanguage` for Developer) on top of the shared `Employee` attributes.

```
             Employee (general)
             /              \
      Manager             Developer
   (+ TeamSize)      (+ ProgrammingLanguage)
```

### Aggregation
Treats a **relationship itself** as a higher-level entity, so it can participate in further relationships. Used when a relationship needs its own relationship with another entity.

**Example:** `Employee` **works on** `Project` is a relationship. If you now want to record which `Manager` **monitors** that specific employee-project assignment, you treat the whole "works on" relationship as an aggregated entity that the `Manager` relationship connects to.

**Interview Q&A:**
- **Generalization vs Specialization?** Generalization is bottom-up (combining similar entities into one general entity); Specialization is top-down (splitting a general entity into specialized sub-entities) — same concept, opposite direction, much like abstraction vs inheritance in OOP.
- **What is Aggregation used for in ER modeling?** When a relationship itself needs to participate in another relationship — you treat the relationship as an entity so it can connect further.
- **Give a real example of specialization.** An `Account` entity specialized into `SavingsAccount` (with `InterestRate`) and `CurrentAccount` (with `OverdraftLimit`).

---

## 6. Relational Model ⭐⭐⭐

**Definition:**
> The Relational Model represents data as a collection of **relations (tables)**, where each table has rows (**tuples**) and columns (**attributes**). It's the mathematical/logical foundation underneath every SQL database.

### Key Terminology
| Term | Meaning |
|---|---|
| **Relation** | A table |
| **Tuple** | A row/record |
| **Attribute** | A column |
| **Domain** | The set of allowed values for an attribute |
| **Degree** | Number of attributes (columns) in a relation |
| **Cardinality** | Number of tuples (rows) in a relation |

**Example:** A `Student(RollNo, Name, Age)` relation — degree = 3 (three columns), cardinality = however many rows currently exist.

**Interview Q&A:**
- **What is the Relational Model?** A way of organizing data into tables (relations) of rows (tuples) and columns (attributes), forming the logical basis of relational databases.
- **Degree vs Cardinality?** Degree = number of columns; Cardinality = number of rows.

---

## 7. Keys in the Relational Model ⭐⭐⭐ (Extremely commonly asked — know all five cold)

**Definition:**
> Keys are attributes (or combinations of attributes) used to uniquely identify tuples in a relation and to establish relationships between tables.

| Key Type | Meaning | Example |
|---|---|---|
| **Super Key** | Any set of attributes that can uniquely identify a tuple (may include extra unnecessary attributes) | `{RollNo}`, `{RollNo, Name}` — both uniquely identify a student |
| **Candidate Key** | A **minimal** super key — no unnecessary extra attributes | `{RollNo}` alone (removing it would break uniqueness) |
| **Primary Key** | The candidate key **chosen** by the designer to uniquely identify tuples in the table | `RollNo` chosen as the official identifier |
| **Alternate Key** | Any candidate key that was **not** chosen as the primary key | If both `RollNo` and `Email` uniquely identify students, and `RollNo` is primary, `Email` is an alternate key |
| **Foreign Key** | An attribute in one table that references the **primary key** of another table, linking the two | `Enrollment.StudentID` referencing `Student.RollNo` |

```
 Student                          Enrollment
 ┌────────┬───────┐               ┌───────────┬──────────┐
 │ RollNo  │ Name   │               │ StudentID  │ CourseID  │
 │ (PK)    │        │  ◀───────────│ (FK)       │           │
 └────────┴───────┘               └───────────┴──────────┘
```

**Interview Q&A:**
- **Difference between Super Key and Candidate Key?** A super key can have extra attributes beyond what's needed for uniqueness; a candidate key is the *minimal* version — no attribute can be removed without breaking uniqueness.
- **Primary Key vs Alternate Key?** Both are candidate keys; the primary key is the one chosen to officially identify rows, and any other candidate key becomes an alternate key.
- **What does a foreign key do?** Links a row in one table to a row in another by referencing that other table's primary key — this is how relationships are actually implemented in the relational model.
- **Can a table have multiple candidate keys but only one primary key?** Yes — e.g., both `RollNo` and `Email` might uniquely identify a student, but only one can be designated the primary key; the other becomes an alternate key.

---

## 8. Functional Dependency ⭐⭐⭐ (Feeds directly into Normalization)

**Definition:**
> A functional dependency `A → B` means: for any two tuples with the same value of A, they must also have the same value of B. In other words, A **determines** B.

**Example:** `RollNo → Name` — if two students have the same RollNo (impossible if RollNo is a key, but conceptually), they'd have to have the same Name. RollNo uniquely determines Name.

### Types of Functional Dependency
| Type | Meaning | Example |
|---|---|---|
| **Trivial FD** | B is a subset of A | `{RollNo, Name} → RollNo` |
| **Non-Trivial FD** | B is NOT a subset of A | `RollNo → Name` |
| **Partial Dependency** | Non-key attribute depends on only *part* of a composite key | Relevant to 2NF |
| **Transitive Dependency** | Non-key attribute depends on another non-key attribute | Relevant to 3NF |

**Interview Q&A:**
- **What is a functional dependency?** A constraint where one attribute (or set) uniquely determines another — `A → B` means the value of A always implies the same value of B.
- **Trivial vs Non-trivial FD?** Trivial: the right side is a subset of the left side (`{A,B} → A`), which is always true and not very informative. Non-trivial: the right side is genuinely determined data not already implied by the left side.

---

## 9. Attribute Closure ⭐⭐

**Definition:**
> The closure of an attribute set A (written `A+`) is the set of **all** attributes that can be functionally determined from A, using the given set of functional dependencies. It's the main tool used to find candidate keys.

**How to compute it (algorithm):**
1. Start with `A+ = A`.
2. Repeatedly: for any FD `X → Y` where `X ⊆ A+`, add `Y` to `A+`.
3. Stop when no more attributes can be added.

**Example:** Given `R(A, B, C, D)` with FDs: `A → B`, `B → C`, `C → D`.
Compute `A+`:
- Start: `A+ = {A}`
- `A → B` applies (A is in A+) → `A+ = {A, B}`
- `B → C` applies → `A+ = {A, B, C}`
- `C → D` applies → `A+ = {A, B, C, D}`
- Result: `A+ = {A, B, C, D}` — since this covers **all** attributes of R, `A` alone is a **candidate key**.

**Interview Q&A:**
- **What is attribute closure, and why is it useful?** The set of all attributes derivable from a given attribute set using the FDs — used to test whether an attribute (or set) is a candidate key (if its closure = all attributes of the relation, it's a key).
- **How do you determine if a set of attributes is a super key?** Compute its closure; if the closure includes every attribute in the relation, it's a super key.

---

## 10. Mapping ER Model to Relational Model ⭐⭐ (Practical — "design this schema" style questions)

**Definition:**
> The process of converting an ER diagram (entities, attributes, relationships) into actual relational tables — the bridge between conceptual design and implementation.

### Mapping Rules (the ones actually worth knowing)

**1. Strong Entity → Table**
Each entity becomes a table; its attributes become columns; a key attribute becomes the primary key.

**2. Weak Entity → Table with a composite key**
A weak entity (one with no key attribute of its own, depends on another entity) becomes a table whose primary key is a composite of its own partial key + the owning entity's primary key (as a foreign key).

**3. 1:1 Relationship**
Add the primary key of one entity as a foreign key in the other table (either direction works).

**4. 1:N Relationship**
Add the primary key of the "1" side as a foreign key in the table on the "N" side.

**Example:** `Department (1) → Employee (N)` — add `DeptID` as a foreign key inside the `Employee` table (not the other way around, since one department has many employees, but each employee has only one department).

**5. M:N Relationship**
Create a **new junction/bridge table** containing the primary keys of both entities as a composite key (plus any relationship-specific attributes).

**Example:** `Student ↔ Course` (M:N) → create `Enrollment(StudentID, CourseID, EnrollmentDate)`, where `(StudentID, CourseID)` together form the composite primary key.

```
 Student(RollNo PK)     Course(CourseID PK)
        \                    /
         \                  /
       Enrollment(StudentID FK, CourseID FK, EnrollDate)
              ↑ composite primary key = (StudentID, CourseID)
```

**Interview Q&A:**
- **How do you map an M:N relationship to relational tables?** Create a separate junction table containing the primary keys of both entities as a composite primary key (and foreign keys pointing back to each original table).
- **How is a 1:N relationship mapped?** The primary key from the "1" side is added as a foreign key in the table on the "N" side — no new table needed.
- **How would you convert a weak entity into a table?** Its primary key becomes a composite of its own partial identifying attribute plus the primary key of the entity it depends on (added as a foreign key).

---

## 🎯 Rapid-Fire One-Liners (Last-Minute Revision)

- **DBMS** = software managing structured data centrally — solves redundancy, inconsistency, poor concurrency, and lack of security that raw file systems suffer from.
- **1-Tier**: client + DB on same machine. **2-Tier**: client talks directly to DB server. **3-Tier**: client → app/API server → DB — standard for modern web apps (security + scalability + maintainability).
- **ER Model**: entities (objects), attributes (properties), relationships (connections) — used for conceptual DB design.
- Attribute types: Simple, Composite, Derived, Multi-valued.
- Relationship cardinality: 1:1, 1:N, M:N.
- **Generalization** = bottom-up (combine similar entities). **Specialization** = top-down (split general entity into specific ones). **Aggregation** = treating a relationship as an entity so it can have its own relationships.
- **Relational Model**: data as tables (relations) of rows (tuples) and columns (attributes). Degree = #columns, Cardinality = #rows.
- **Super Key**: any uniquely-identifying set (can have extra attrs). **Candidate Key**: minimal super key. **Primary Key**: chosen candidate key. **Alternate Key**: non-chosen candidate key. **Foreign Key**: references another table's primary key.
- **Functional Dependency** `A → B`: A uniquely determines B. Trivial (B ⊆ A) vs Non-trivial.
- **Attribute Closure (A+)**: all attributes derivable from A via FDs — if A+ = all attributes of R, then A is a candidate key.
- ER → Relational mapping: Strong entity → table. Weak entity → table with composite key (partial key + owner's PK). 1:1/1:N → foreign key added to one side. M:N → new junction table with composite PK from both entities.
