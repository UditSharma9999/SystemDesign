# ACID — The Four Guarantees

Relational databases don't just store data — they make promises about how they handle it.  
These guarantees are called **ACID**.

| Property        | What It Means                                                                       | Real-World Analogy                                                         |
| --------------- | ----------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| **Atomicity**   | A transaction either fully completes or fully rolls back. No halfway states.        | Transferring money: both accounts update or neither does.                  |
| **Consistency** | The database moves from one valid state to another. Constraints are never violated. | Your bank balance can't go negative if there's a constraint preventing it. |
| **Isolation**   | Concurrent transactions don't interfere with each other.                            | Two people buying the last concert ticket — only one succeeds.             |
| **Durability**  | Once a transaction commits, the data survives crashes, power outages, everything.   | Your deposit is safe even if the bank's servers reboot.                    |

These guarantees are why banks, payment systems, and e-commerce platforms rely on relational databases.  
When money is involved, “it probably worked” is not good enough.

---

# Chapter 1 How Data Lives on Disk — Pages, Indexes, and I/O

## The Heap File — Where Your Data Actually Lives

When you insert a row into a table, the database doesn't keep it in memory forever. It writes it to a heap file on disk — essentially a big, unordered collection of rows.

The heap file is divided into pages — fixed-size blocks. Each page holds as many rows as will fit.

# Heap File (users table)

```text
┌─────────────────┐
│ Page 0          │  ← rows 1-50
│ [alice, bob...] │
├─────────────────┤
│ Page 1          │  ← rows 51-100
│ [dave, eve...]  │
├─────────────────┤
│ Page 2          │  ← rows 101-150
│ [grace, hank...]│
├─────────────────┤
│ ...             │
│ Page 9999       │  ← rows 499,951-500,000
└─────────────────┘
```

A table with 500,000 users might span 10,000 pages. Each page is one trip to disk.

## The Sequential Scan — Brute Force

What happens when you run this query?

```sql
SELECT * FROM users WHERE email = 'carol@example.com';
```

Without an index, the database has no idea which page contains Carol's row. So it does the only thing it can: read every single page, checking each row for a matching email.

This is called a sequential scan (or full table scan). For our 500,000-user table:

- 10,000 pages to read from disk
- Each page requires a disk I/O operation
- Even with modern SSDs (~0.1ms per read), that's ~1 second of pure I/O

One second to find one email. Now imagine 1,000 users searching simultaneously. That's the problem.

## Indexes — The Table of Contents

It's a separate data structure that maps column values to the pages where those rows live.

When you create an index. The database builds a sorted lookup structure (typically a B-tree) Now when you search for Carol's email:

1. Consult the index → "carol@example.com is on page 847, row 3"
2. Read page 847
3. Done

**Two page reads** instead of 10,000. That's the difference between 0.2ms and 1 second.

## The B-Tree — A Quick Preview

A B-tree is like a multi-level table of contents. Imagine a phonebook:

- **Level 1 (root)**: "A-M is in the left half, N-Z is in the right half"
- **Level 2**: "A-F is in section 1, G-M is in section 2"
- **Level 3 (leaf)**: "Carol is on page 847"

<br/>

**The Primary Key Index — You Already Have One**  
When you define a primary key, the database automatically creates an index on it. That's why `SELECT * FROM users WHERE id = 42` is always fast — there's already a B-tree index on the `id` column.

## When Indexes Hurt

Indexes aren't free. Every index you add has a cost:

- **Write overhead**: When you insert, update, or delete a row, every index on that table must also be updated. Five indexes on a table means every write does roughly 6x the work

- **Storage**: An index is a separate data structure on disk. On a large table, indexes can consume as much space as the data itself.

- **Not always used**: The database's query planner decides whether to use an index. For very small tables, a sequential scan is actually faster

## Buffer Pool — The Cache You Didn't Know About

Databases don't actually read from disk on every query. They keep frequently-accessed pages in memory in a structure called the **buffer pool**.

When you request a page:

1. **Is it in the buffer pool?** → Return it instantly (memory speed)
2. **Not in the buffer pool?** → Read from disk, put it in the buffer pool, return it

## The buffer pool acts as an automatic LRU cache. Hot pages stay in memory. Cold pages get evicted to make room.

# Chapter 2 B-Trees, LSM Trees, and Hash Indexes — How Indexes Work

## B-Trees — The Default Index Everywhere

The B-tree (short for "balanced tree," though the inventor never confirmed this) is the most widely used index structure in databases.

### The Structure

A B-tree is a self-balancing tree where every node is a page — a fixed-size block of data. Each page holds sorted keys and pointers to child pages.

# B+ Tree Structure

```text
                    [35 | 70]                         ← Root page (Level 0)
                   /    |    \
          [10|20|30] [45|55|60] [80|90|95]            ← Internal pages (Level 1)
          / | | \    / | | \    / | | \
        [leaves]   [leaves]   [leaves]                ← Leaf pages (Level 2)
```

- **Root node**: The single entry point. Always in memory (buffer pool).
- **Internal nodes**: Contain keys and pointers that guide the search left or right.
- **Leaf nodes**: Contain the actual indexed values and pointers to the heap page where the full row lives.

### Why 3-4 Levels Covers Millions of Rows

Here's the math that makes B-trees magical. Each page is 8 KB. A typical B-tree node can hold around 500 keys (depending on key size). So:

| Tree Depth          | Max Rows Indexed |
| ------------------- | ---------------- |
| 1 level (root only) | ~500             |
| 2 levels            | ~250,000         |
| 3 levels            | ~125,000,000     |
| 4 levels            | ~62,500,000,000  |

Three levels covers 125 million rows. Four levels covers 62 billion. Since the root page is always cached in memory, a lookup in a table with 100 million rows requires just 2-3 disk reads. That's why B-tree lookups are effectively O(log N) but with a very large branching factor that keeps the constant tiny.

### How a B-Tree Lookup Works — Step by Step

Let's trace finding `user_id = 847` in a B-tree index:

        Step 1: Read root page (cached in memory)
                [200 | 500 | 800]
                847 > 800 → go right

        Step 2: Read internal page (1 disk read)
                [810 | 830 | 860 | 890]
                847 > 830 and 847 < 860 → go to child between 830 and 860

        Step 3: Read leaf page (1 disk read)
                [..., 845, 846, 847, 848, ...]
                Found! 847 → heap page 1204, slot 7

        Step 4: Read heap page 1204 (1 disk read)
                Return the full row

**Total: 3 disk reads**. Compare that to scanning 10,000+ pages sequentially. This is why every database uses B-trees as the default.

### In-Place Updates and Write Behavior

When you insert a new key, the B-tree finds the correct leaf page and adds the key in sorted order. If the page is full, it splits into two pages and pushes the middle key up to the parent. This splitting is what keeps the tree balanced — no branch ever gets much deeper than another.

![text14](/assets/14.png)

Updates modify the leaf page in place. Deletes mark the key as removed (and the space gets reclaimed later). All of this involves random I/O — jumping to specific pages on disk. That's fine for reads, but it becomes a `bottleneck for write-heavy workloads`.
