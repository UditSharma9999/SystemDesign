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

## LSM Trees — Optimized for Writes

What if your workload is 90% writes? Think of a time-series database ingesting millions of sensor readings per second, or a messaging system like Discord storing billions of messages. Random in-place updates on a B-tree become a bottleneck.

The **Log-Structured Merge Tree (LSM tree)** takes a fundamentally different approach: instead of updating data in place, it **buffers writes in memory** and flushes them to disk as sorted, immutable files.

#### How LSM Trees Work

        1. WRITE arrives
                ↓
        2. Insert into MEMTABLE  (in-memory sorted structure, typically a red-black tree)
                ↓ (when memtable is full, ~64MB)
        3. Flush to disk as an SSTABLE  (Sorted String Table — one sequential write)
                ↓ (over time, SSTables accumulate)
        4. COMPACTION merges multiple SSTables into larger, sorted ones

> **Writes never touch disk until the flush. That's why LSM trees handle millions of writes per second.**

![Text15](/assets/15.png)

**Memtable**: An in-memory sorted data structure (red-black tree or skip list). All writes go here first. This is blazing fast — no disk I/O at all.

- **SSTable (Sorted String Table)**: When the memtable fills up, it gets written to disk as a single sorted file. This is a sequential write — the fastest thing you can do on disk. No random seeks.

- **Compaction**: Over time, you accumulate many SSTables. Background threads merge them together, removing deleted keys and combining updates. This keeps read performance from degrading.

![Text16](/assets/16.png)

### The Read Path (Where LSM Trees Pay)

Reading from an LSM tree is more complex:

1. Check the memtable (in memory — fast)
2. Check the most recent SSTable on disk
3. Check the next SSTable
4. ... keep checking older SSTables until found

This is called **read amplification** — a single read might check multiple files. To mitigate this, LSM trees use Bloom filters at each level: a probabilistic data structure that can tell you "this key is definitely NOT in this SSTable" without reading the file. This skips most unnecessary checks.

![Text17](/assets/17.png)

<br/>

![Text18](/assets/18.png)

### The Big Trade-Off: B-Trees vs LSM Trees

| Amplification Type  | What It Means                          | B-Tree                                 | LSM Tree                                                       |
| ------------------- | -------------------------------------- | -------------------------------------- | -------------------------------------------------------------- |
| Read amplification  | How many disk reads per query          | Low (3–4 reads)                        | Higher (check multiple SSTables)                               |
| Write amplification | How many times data is written to disk | Higher (in-place updates, page splits) | Lower (sequential writes, but compaction re-writes)            |
| Space amplification | How much extra disk space is used      | Low (one copy of data)                 | Higher (multiple SSTables may have same key during compaction) |

<br/>

| Feature          | B-Tree                       | LSM Tree                                       |
| ---------------- | ---------------------------- | ---------------------------------------------- |
| Read latency     | Predictable, fast            | Variable (may check multiple levels)           |
| Write throughput | Good                         | Excellent (sequential writes)                  |
| Space usage      | Efficient                    | Can temporarily use 2× space during compaction |
| Range queries    | Excellent                    | Good (within each SSTable)                     |
| Concurrency      | Page-level locking           | Append-only, minimal lock contention           |
| Used by          | PostgreSQL, MySQL, Oracle    | Cassandra, RocksDB, LevelDB, DynamoDB          |
| Best for         | OLTP with mixed reads/writes | Write-heavy, time-series, logging              |

## Hash Indexes — O(1) But Limited

A hash index is the simplest index type. It hashes the key and maps it directly to a location.

`hash("alice@example.com") → bucket 4,271 → heap page 847, slot 3`

![Text19](/assets/19.png)

- **O(1) equality lookups** — the fastest possible for exact-match queries
- Simple implementation

**Weaknesses:**

- **No range queries** — WHERE age > 25 can't use a hash index because hashing destroys ordering
- **No sorting** — ORDER BY can't leverage a hash index
- **No partial matching** — WHERE name LIKE 'Al%' needs ordered data

---

# Chapter 3 Normalization — Store Each Fact Once

| Anomaly Type   | What Happens                                                                 | Example                                             |
| -------------- | ---------------------------------------------------------------------------- | --------------------------------------------------- |
| Update Anomaly | Changing one piece of information requires updating it in multiple rows.     | Updating Alice's email in every order record.       |
| Insert Anomaly | You cannot add data unless some unrelated data also exists.                  | Cannot add a new user until they place an order.    |
| Delete Anomaly | Removing one piece of data accidentally removes other important information. | Deleting Bob's only order also deletes Bob's email. |

## The Normal Forms

### First Normal Form (1NF) — No Repeating Groups

Every column must contain a single, atomic value. No arrays, no comma-separated lists, no nested structures.

### Second Normal Form (2NF) — No Partial Dependencies

Satisfies 1NF plus: every non-key column depends on the entire primary key, not just part of it. This only matters when you have a composite primary key.

### Third Normal Form (3NF) — No Transitive Dependencies

Satisfies 2NF plus: no non-key column depends on another non-key column.

### BCNF — The Strict Version

Boyce-Codd Normal Form is a stricter version of 3NF. It says: every determinant (column that determines another column's value) must be a candidate key.

In practice, 3NF covers 99% of cases. BCNF only matters in edge cases with overlapping candidate keys. You'll never need it in a system design interview, but knowing it exists shows depth if an interviewer asks.

## How to Apply This in Practice

You don't sit in an interview reciting normal forms. Instead, use this mental checklist:

- **Is any data repeated across rows?** If yes, it probably belongs in its own table with a foreign key reference.
- **If I update this value, do I need to update it in multiple places?** If yes, normalize it.
- **Can I add a new entity without creating dummy data in unrelated columns?** If no, you have an insert anomaly.

## When to Stop Normalizing

Normalization isn't a goal in itself — it's a tool for data integrity. Over-normalizing creates its own problems:

- **Too many joins**: If every query requires joining 7 tables, your reads slow down
- **Complexity**: More tables means more foreign keys, more migration files, more cognitive load
- **Diminishing returns**: The jump from unnormalized to 3NF is huge. The jump from 3NF to BCNF is almost never worth it in application databases.

---

# Chapter 4 Denormalization — Trading Writes for Reads

**Normalization optimizes for writes and consistency. Denormalization optimizes for reads and speed.**

## What Denormalization Actually Means

Denormalization means deliberately storing redundant data to avoid expensive operations at read time.

### Example 1: Embedding the Author's Name

**Normalized**: To display a post with its author's name, you join two tables:

```SQL
SELECT posts.content, users.username
FROM posts
JOIN users ON posts.user_id = users.id
WHERE posts.id = 42;
```

**Denormalized**: Store the author's name directly on the post:

```sql
-- No join needed
SELECT content, author_username
FROM posts
WHERE id = 42;
```

The `author_username` column is redundant — it duplicates data from the `users` table. But it eliminates a join on every post read. For a feed showing 50 posts, that's 50 joins you just avoided.

### Example 2: Pre-computed Counts

Normalized: Count likes by querying the likes table:

```sql
SELECT COUNT(*) FROM likes WHERE post_id = 42;
```

**Denormalized**: Store the count directly on the post:

```sql
SELECT like_count FROM posts WHERE id = 42;
```

Every time someone likes or unlikes a post, you increment or decrement `like_count`. The count is always available without scanning the likes table.

### Example 3: Materialized Feeds

**Normalized**: Compute the home feed at read time by joining posts, follows, and sorting:

```sql
SELECT posts.* FROM posts
JOIN follows ON posts.user_id = follows.following_id
WHERE follows.follower_id = ?
ORDER BY posts.created_at DESC
LIMIT 50;
```

**Denormalized**: Pre-compute the feed into a `feed_items` table:
feed_items

```txt
┌─────────┬─────────┬────────────┬─────────┐
│ user_id │ post_id │ created_at │ author  │
├─────────┼─────────┼────────────┼─────────┤
│    1    │   500   │ 2024-03-15 │ bob     │
│    1    │   499   │ 2024-03-15 │ carol   │
│    1    │   497   │ 2024-03-14 │ bob     │
└─────────┴─────────┴────────────┴─────────┘
```

Reading the feed is now a simple range scan on one table. The complexity shifts to write time

## The Cost of Denormalization

1. **Consistency Becomes Your Responsibility**
2. **Writes Get More Complex**

## When to Denormalize

| Signal                                          | Example                                                  |
| ----------------------------------------------- | -------------------------------------------------------- |
| A hot read path requires joins that add latency | Home feed joining posts + follows + users                |
| An aggregation runs on every page load          | `COUNT(*)` for likes, comments, followers                |
| The same join appears in many queries           | Author name shown on posts, comments, notifications      |
| Read/write ratio is heavily skewed toward reads | Social media (1000:1 reads to writes on popular content) |

### Don't denormalize when:

| Signal                      | Example                                                                   |
| --------------------------- | ------------------------------------------------------------------------- |
| The data changes frequently | User email (changes create mass updates across tables)                    |
| Consistency is critical     | Financial transactions, inventory counts                                  |
| The read pattern is simple  | Direct lookups by primary key (already fast with an index)                |
| A cache would solve it      | If the data fits in Redis, cache the join result instead of denormalizing |

## Cache vs Denormalize — A Key Decision

Before denormalizing, ask: would a cache solve this problem?

![Text20](/assets/20.png)

## A Practical Framework

Here's how to think about denormalization in a system design interview:

![Text21](/assets/21.png)

---
