# DBMS: File Organization, Indexing, B-Tree & B+ Tree — SDE Interview Notes

Covers: File Organization (Sequential, Heap), Indexing fundamentals, Dense vs Sparse Index, Clustered vs Non-Clustered Indexing (with SQL), Multilevel Indexing, B-Tree, and B+ Tree.

---

## 1. File Organization ⭐⭐

**Definition:**
> File organization is the method of storing data records in a file so they can be accessed efficiently. It determines how data is arranged, stored, and retrieved from physical storage.

**Why it matters:** Faster record selection, faster insert/update/delete, prevents duplicate records, and keeps storage cost low.

### Sequential File Organization
Records are stored one after another. Two variants:

**1. Pile File Method** — new records are simply appended at the end, in the order they're inserted.

**Example:** Think of an inbox that never sorts emails — every new email just gets added at the bottom, regardless of sender or date.

**2. Sorted File Method** — new records are inserted in sorted (ascending/descending) order based on a key.

**Example:** A library card catalog that keeps every new card inserted in the correct alphabetical position, rather than just tacked on at the end.

```
Pile:    R1 → R3 → R5 → R4 → (new R2 goes to the END)
Sorted:  R1 → R3 → R5 → R7 → R8 → (new R2 inserted in ITS SORTED position, then re-sorted)
```

**Pros:** Fast/efficient for huge sequential data, simple design, cheap storage (works well on tape).
**Cons:** No random access — must move sequentially to find a record; sorted method is slow to insert (needs re-sorting).

### Heap File Organization
Records are inserted into any data block with free space — no sorting or ordering required. DBMS decides which block a new record goes into.

**Example:** Think of it like a warehouse where new deliveries go into whatever empty shelf-space is available, not a fixed spot — fast for dropping stuff off, but slow to look something up later because you must search shelf by shelf.

**Pros:** Fast inserts, great for loading huge volumes of data quickly.
**Cons:** Unused/wasted memory blocks; searching/deleting/updating is slow since you must scan the file from the beginning — inefficient for large databases.

**Interview Q&A:**
- **Heap vs Sequential File Organization?** Heap has no ordering — fast inserts, slow search (must scan from the start). Sequential keeps records ordered — faster search (especially sorted variant), but slower/costlier inserts since the file may need re-sorting.
- **When would Heap File Organization be preferred?** When you need to bulk-load a huge number of records quickly and don't care (yet) about search speed — e.g., an initial data dump before indexes are built.

---

## 2. Indexing in Databases ⭐⭐⭐

**Definition:**
> Indexing is a data structure technique used to speed up data retrieval by minimizing the number of disk accesses needed to locate records. It stores a copy of selected column values (the search key) along with pointers to the actual data rows — so the DB doesn't need to scan the entire table.

**Example:** Exactly like the index at the back of a textbook — instead of flipping through every page to find "mitochondria," you check the index, see it's on page 142, and jump straight there.

### Key Attributes of an Index
| Attribute | Meaning |
|---|---|
| Access Time | Time to locate a data element |
| Insertion Time | Time to find space + insert new data |
| Deletion Time | Time to find + delete + update the index structure |
| Space Overhead | Extra storage the index itself consumes |

### Dense vs Sparse Index
| Type | Rule | Trade-off |
|---|---|---|
| **Dense Index** | One index entry for **every** search key value in the data file | More storage, but direct lookup |
| **Sparse Index** | One index entry only for **some** records (e.g., per block) | Less storage, but requires finding the nearest entry ≤ key, then scanning sequentially from there |

**Example (Sparse):** Like chapter headings in a book instead of a full page-by-page index — you jump to the nearest chapter start, then flip forward a few pages to find the exact spot.

**Interview Q&A:**
- **What is indexing and why does it help?** A structure that stores column values + pointers to rows, so the DB can jump directly to matching rows instead of scanning the whole table — turns an O(n) scan into something closer to O(log n).
- **Dense vs Sparse index — trade-off?** Dense = faster direct access but more storage (entry per record). Sparse = less storage but slightly slower (entry per block, requires a short sequential scan after locating the block).

---

## 3. Types of Indexing ⭐⭐⭐

### Primary Index
Index built on the **primary key**; data is physically stored in sorted order by that key. Since the key is unique, each index entry usually points to a block of records rather than every single record.

### Clustered Index
Data rows are **physically stored in the same order** as the index. Only **one** clustered index is possible per table, since data can only be physically sorted one way.

**Example:** A dictionary — the words themselves are physically arranged alphabetically. There's no separate "index page"; the data *is* the index.

### Non-Clustered (Secondary) Index
A **separate structure** that just tells you *where* the data is — it doesn't reorder the actual table.

**Example:** A book's contents/index page — it lists topics with page numbers, but the pages themselves aren't rearranged to match that list. You always take one extra hop: look up the index, then follow the pointer to the actual data.

```
 Clustered Index                    Non-Clustered Index
 ┌─────────────┐                    ┌───────────┐      ┌──────────────┐
 │ Data IS      │                    │ Index      │ ──▶  │ Actual Data   │
 │ physically    │                    │ (pointers) │      │ (unsorted)    │
 │ sorted        │                    └───────────┘      └──────────────┘
 └─────────────┘
```

### Multilevel Index
When a single-level index itself grows too large to fit efficiently in memory, it's split into a hierarchy — an "index of the index." Outer blocks point to inner blocks, which point to the actual data blocks.

**Example:** Similar to how a large company directory might be organized as Building → Floor → Office, rather than one giant flat employee list — you navigate through a few small levels instead of scanning one huge index.

**Interview Q&A:**
- **Clustered vs Non-Clustered index — core difference?** Clustered physically sorts the table's data to match the index (only one per table); non-clustered is a separate lookup structure with pointers back to the (unsorted) data, and a table can have several.
- **Why can a table have only one clustered index?** Because it dictates the *physical* storage order of the rows — data can only be laid out on disk in one order at a time.
- **What problem does multilevel indexing solve?** When a single-level index becomes too big to search efficiently (or fit in memory), multilevel indexing breaks it into a hierarchy, reducing the number of comparisons/disk accesses needed.

---

## 4. Clustered vs Non-Clustered Index — SQL Perspective ⭐⭐⭐ (Very frequently asked)

| Aspect | Clustered Index | Non-Clustered Index |
|---|---|---|
| Data Storage Order | Determines the physical order of rows in the table | Does not affect physical row order |
| Count per Table | Only **one** | Multiple allowed |
| Structure | The index *is* the table | Separate structure with pointers to data rows |
| Default | Primary key becomes the clustered index automatically (if none specified) | Created explicitly on any column |
| Best Use Case | Range queries, ordered retrieval | Fast lookups on non-primary-key columns |

**Example — Clustered (via primary key):**
```sql
CREATE TABLE Student_info (
    ROLL_NO INT PRIMARY KEY,
    NAME VARCHAR(20),
    DEPARTMENT VARCHAR(20)
);
-- ROLL_NO becomes the clustered index automatically;
-- rows are physically stored in ascending ROLL_NO order.
```

**Example — Non-Clustered (explicit):**
```sql
CREATE NONCLUSTERED INDEX IX_Student_info_NAME
ON Student_info (NAME ASC);
-- Creates a separate lookup structure on NAME,
-- pointing back to the actual (ROLL_NO-ordered) rows.
```

**Practical impact on queries:**
- **SELECT with WHERE on the clustered column** (e.g., `WHERE ROLL_NO = ...`) → DB jumps straight to the row, minimal disk I/O.
- **SELECT with WHERE on a non-clustered column** (e.g., `WHERE NAME = ...`) → DB uses the non-clustered index to find the pointer, then follows it to fetch the actual row — one extra hop compared to clustered lookups.
- **JOIN queries** — joining on the clustered index column is fastest; joining on other columns benefits from a non-clustered index on those columns, but still costs more than a clustered lookup.

**Interview Q&A:**
- **When would you add a non-clustered index?** On columns frequently used in `WHERE`/`JOIN`/`ORDER BY` that aren't the primary key — e.g., indexing `NAME` for name-based searches, or an `email` column for login lookups.
- **Why is a non-clustered index generally slower than a clustered one?** Because it requires an extra step — first find the entry in the index structure, then follow a pointer to the actual row, versus the clustered index where the data is already sitting right there in sorted order.
- **What happens to a table's ordering when you drop the clustered index?** The table reverts to an unordered heap structure until a new clustered index is created.

---

## 5. B-Tree ⭐⭐⭐

**Definition:**
> A B-Tree is a self-balancing m-way search tree designed for disk-based storage systems. In a B-Tree of order m, each node can have up to m children and m−1 keys — allowing it to store many keys per node and keep the tree height very small.

**Why it matters for databases:** Every level of a tree you have to traverse usually means a disk access. A B-Tree packs many keys into each node, so even a huge dataset only needs a few levels — drastically cutting down disk I/O compared to a plain Binary Search Tree.

### Properties of a B-Tree
- All leaf nodes are at the **same level** (height).
- Keys within each node are stored in **ascending order**.
- Every non-leaf node (except root) has at least `⌈m/2⌉` children.
- A node with `n` keys has exactly `n+1` children.
- The tree stays **balanced at all times** — self-adjusts on insert/delete.

```
                [ 30 | 60 ]
               /     |      \
      [10|20]   [40|50]    [70|80]
```

### Operations & Time Complexity
| Operation | Time Complexity |
|---|---|
| Search | O(log n) |
| Insert | O(log n) |
| Delete | O(log n) |
| Traverse | O(n) |

**Search process:** Start at the root, compare the key against the node's keys, follow the appropriate child pointer, repeat until found or a leaf is reached with no match (return NULL).

**Interview Q&A:**
- **Why use a B-Tree instead of a plain BST for disk-based storage?** A BST node holds only 1 key, so large datasets need many levels — meaning many disk accesses. A B-Tree node holds many keys, keeping the tree shallow and drastically reducing the number of disk reads needed per search.
- **What guarantees does a B-Tree give you?** O(log n) search/insert/delete, and the tree is always balanced — no skewed/degenerate cases like an unbalanced BST.
- **Where are B-Trees used in practice?** Database indexing, file systems, and any system needing fast lookups over data stored on disk.

---

## 6. B+ Tree ⭐⭐⭐ (Most important — what real DB indexes use)

**Definition:**
> A B+ Tree is an extended version of the B-Tree where all actual data is stored **only in the leaf nodes**; internal nodes store just keys, used purely for navigation. All leaf nodes are linked together, forming an ordered chain across the bottom of the tree.

**Why this matters:** Because internal nodes hold *only* keys (no data), more keys fit per node than in a B-Tree — making the tree even shorter. And because leaves are linked like a list, range queries (`BETWEEN`, `ORDER BY`) become very fast — you find the starting leaf once, then just walk the linked list forward.

### Structure
```
                [ 30 | 60 ]                ← internal nodes: keys only, for navigation
               /     |      \
      [10|20] → [30|40|50] → [60|70|80]    ← leaf nodes: hold actual data, linked together
```

- **Internal node:** `<P1, K1, P2, K2, ..., Pc-1, Kc-1, Pc>` — pointers and keys alternating, guiding the search.
- **Leaf node:** `<<K1,D1>, <K2,D2>, ..., Pnext>` — each key paired with a data pointer, plus a pointer to the *next* leaf node.

**Example (searching):** To find key 58 — start at root, compare against internal node keys to pick the right branch, descend to the correct leaf node, and scan it for 58.

### Insertion (high level)
1. Navigate to the correct leaf node.
2. Insert the new key in sorted order.
3. If the node overflows: split it, push the middle key up to the parent (may propagate up to the root).

### Deletion (high level)
1. Remove the key from the appropriate leaf.
2. Rebalance if needed — borrow a key from a sibling, or merge nodes.

### B-Tree vs B+ Tree
| Aspect | B-Tree | B+ Tree |
|---|---|---|
| Data location | Stored in **any** node (leaf or internal) | Stored **only** in leaf nodes |
| Leaf linking | Not linked | Leaves linked like a list — fast range queries |
| Order/Fan-out | Lower (fewer keys per node, since nodes also store data) | Higher (internal nodes hold only keys → more fit per node) |
| Best for | General search/insert/delete balance | Range queries, sequential scans, DB indexing |
| Real-world use | In-memory structures, general-purpose trees | Database indexes (e.g., MySQL InnoDB), file systems |

**Interview Q&A:**
- **B-Tree vs B+ Tree — the one fact to remember.** B-Tree can store data in any node; B+ Tree stores data ONLY in leaf nodes, and those leaves are linked together — making B+ Trees far better for range queries and sequential access.
- **Why do real databases use B+ Trees instead of B-Trees for indexing?** The linked leaf structure makes range scans fast (walk the linked list instead of re-traversing the tree), and since internal nodes carry no data, more keys fit per node — shorter tree, fewer disk reads.
- **What happens on a B+ Tree leaf split during insertion?** The leaf splits into two, and the **middle key is copied up** to the parent as a separator (unlike a B-Tree, where the middle key would be moved up, not copied, since B-Trees don't duplicate data between levels).

---

## 🎯 Rapid-Fire One-Liners (Last-Minute Revision)

- File Organization = how records are physically stored. **Sequential**: ordered, faster search, costlier insert. **Heap**: unordered, fast insert, slow search — best for bulk loads.
- **Indexing** = stores column values + pointers to rows, avoiding full table scans.
- **Dense index**: one entry per record (more storage, direct access). **Sparse index**: one entry per block (less storage, needs a short scan after locating the block).
- **Primary index**: on the primary key, data physically sorted by it.
- **Clustered index**: data physically sorted to match the index — only **one** per table. **Non-clustered index**: separate pointer structure — **multiple** allowed per table.
- **Multilevel index**: index-of-index, used when a single-level index grows too large.
- Clustered = like a dictionary (data itself is sorted). Non-clustered = like a book's contents page (separate pointer list).
- **B-Tree**: self-balancing m-way tree; data can live in any node; O(log n) search/insert/delete; all leaves at same depth.
- **B+ Tree**: extension of B-Tree; data lives ONLY in leaf nodes; leaves linked together → fast range queries; industry standard for DB indexes (MySQL InnoDB etc.).
- B+ Tree insertion overflow → split leaf, push (copy) middle key up to parent, may propagate to root.
- B+ Tree deletion → remove key from leaf, then rebalance via borrowing from siblings or merging nodes.
