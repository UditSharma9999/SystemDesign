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

# Chapter 5 The Relational Model — The Default That Scales

![text22](/assets/22.png)

#### Why SQL Is the Default

The relational model gives you something no other model does out of the box: ACID

## How Modern SQL Actually Scales

### Read Replicas

**read replicas**. Most apps do far more reading than writing, so one main database handles all writes, while multiple copies of that database handle read requests. For example, when you open a dashboard or analytics page, those reads can go to replicas instead of overloading the main database

The only downside is a tiny delay called **replication lag**. A write to the primary might take 10-100ms to propagate to replicas. For most features this is invisible. For critical reads (like checking a balance right after a transfer), you read from the primary.

### Connection Pooling (PgBouncer)

onnection pooling using tools like PgBouncer. Databases struggle when thousands of applications connect directly at the same time. PgBouncer acts like a traffic manager: applications connect to PgBouncer, and it reuses a smaller number of real database connections efficiently. This reduces CPU overhead and allows many more users to access the database smoothly.

### Table Partitioning

As databases grow to billions of rows, searching through one huge table becomes slow. That’s where **table partitioning** helps. Instead of storing everything in one giant table, data is split into smaller sections, often by date. For example, orders from January go into one partition and February orders into another. If you search only January data, the database scans just that section instead of the entire table, making queries much faster.

### Vitess — Horizontal Sharding for MySQL

For even larger systems, companies use horizontal sharding, where data is split across multiple database servers. Vitess helps automate this for MySQL. Instead of developers manually deciding which server stores which user’s data, Vitess automatically routes queries to the correct shard. It also supports online schema changes and distributed queries, making large-scale MySQL systems easier to manage.

### NewSQL — Distributed SQL That Just Works

NewSQL databases like CockroachDB and Google Cloud Spanner. These databases combine traditional SQL features with automatic horizontal scaling. They spread data across many servers while still supporting ACID transactions and standard SQL. To keep data consistent across machines, they use distributed consensus systems, which adds a little extra latency but allows them to scale almost endlessly while remaining reliable.

## When Relational Is NOT the Right Choice

| Scenario                                              | Why SQL Struggles                          | Better Option                      |
| ----------------------------------------------------- | ------------------------------------------ | ---------------------------------- |
| Very high write throughput (100K+ writes/sec)         | Single-primary bottleneck                  | Wide-column (Cassandra, ScyllaDB)  |
| Rapidly evolving schema                               | `ALTER TABLE` on large tables is expensive | Document store (MongoDB, DynamoDB) |
| Simple key-value lookups with sub-millisecond latency | SQL parsing overhead is unnecessary        | Key-value (Redis, DynamoDB)        |
| Full-text search with ranking                         | SQL `LIKE` is slow, no relevance scoring   | Search engine (Elasticsearch)      |
| Deeply nested, hierarchical data                      | Joins get expensive at many levels         | Document store or graph database   |

## Blob and Object Storage — Where Large Files Go

Images, videos, PDFs, and other large blobs belong in object storage (Amazon S3, Google Cloud Storage, Azure Blob Storage), not in PostgreSQL. Your relational database stores the metadata — filename, owner, upload timestamp, content type — with a URL pointing to the object in S3.

---

# Chapter 6 Document and Key-Value Models — Access-Pattern-First Design

In document databases, all related data is usually stored together in one big document instead of being split across many tables. So this makes reads very fast and simple.

But the downside is data duplication, also called denormalization. If Alice’s email is stored inside every order document, and she changes her email address, you now have to update that email in hundreds of separate documents.

## The Core Principle: Access Pattern First

In the relational world, you model your data first and figure out queries later. In the document world, you flip it: **start with how you'll query the data, then model it to serve those queries.**

This is the fundamental mindset shift. A relational schema asks "what are the entities and relationships?" A document schema asks "what will the application screen look like?"

| Design Approach | Starting Question            | Optimized For                      |
| --------------- | ---------------------------- | ---------------------------------- |
| Relational      | What are my entities?        | Flexibility, ad-hoc queries        |
| Document        | What are my access patterns? | Read performance, specific queries |

If your application has a product detail page that shows the product, its reviews summary, and the seller info — a document database lets you store all of that in one document and serve it in one read.

## Embedding vs. Referencing

With **embedding**, related data is stored directly inside the main document. For example, an order document may contain all its line items inside it. This is useful when the data is usually read together, the relationship is small, and the embedded data rarely changes on its own. Embedding also allows atomic updates, meaning the whole document can be updated safely in a single operation. The advantage is speed and simplicity because one database read returns everything you need.

With **referencing**, instead of storing the full related data inside the document, you store only a reference or ID pointing to another document. This is similar to foreign keys in SQL databases. Referencing is better when related data is large, changes frequently, or needs to be accessed separately. For example, a user may have thousands of orders, so storing all orders inside one user document would make it huge and inefficient. In that case, each order is stored separately and linked using references.

## DynamoDB — Single-Table Design and Its Evolution

Amazon DynamoDB is a database designed for massive scale and very fast lookups. Every piece of data is stored using a partition key (which decides where the data lives) and optionally a sort key (which helps organize related data together). Instead of traditional SQL tables with joins, DynamoDB is optimized around known access patterns — you design the database based on how the application will query data.

For a long time, developers promoted something called **single-table design**. Instead of creating separate tables for users, orders, and products, everything was stored in one big table. Different entity types were identified using patterns like `USER#alice` or `ORDER#1234`. This allowed related data to be fetched very efficiently in a single query and reduced the need for joins. Developers also used overloaded indexes (GSIs) to support many query patterns from the same table.

But it came with drawbacks. The schema became hard to understand because generic column names like PK and SK didn’t clearly describe the data. Adding new query patterns could require creating new indexes, and changing the table structure later could mean rewriting millions of records. Many teams found these designs difficult to maintain.

## Key-Value Stores — Redis and the Speed Layer

A key-value store is the simplest possible data model: you have a key, you have a value, you do GET and SET. That's it.

Redis is the most widely used key-value store, and it's everywhere.  
Redis is not a primary database — it's an acceleration layer. Your source of truth lives in PostgreSQL or DynamoDB. Redis sits in front for hot data that needs sub-millisecond latency.

**The pattern**: check Redis first. On a cache hit, return immediately. On a cache miss, query the primary database, populate the cache with a TTL, then return.

## When Document Makes Sense (And When It Doesn't)

| Document Wins                                                    | Document Loses                           |
| ---------------------------------------------------------------- | ---------------------------------------- |
| Data is naturally hierarchical (product catalogs, user profiles) | Complex joins across entities            |
| Access patterns are well-defined and key-based                   | Ad-hoc reporting and analytics           |
| Schema varies between items (CMS content, event data)            | Referential integrity is critical        |
| Read-heavy with predictable query patterns                       | Many-to-many relationships               |
| Rapid iteration on schema (startups, prototyping)                | Transactions spanning multiple documents |

---

# Chapter 7 Wide-Column and Time-Series Models — When Writes Dominate

## Think of It Like a Filing Cabinet With a Twist

Imagine a filing cabinet where each drawer is labeled with a customer name (the partition key), and inside each drawer, documents are sorted by date (the clustering key). You can quickly pull all documents for one customer in date order — but asking "show me all documents from March across all customers" means opening every single drawer.

That's the **wide-column model**. It's blazing fast for the queries it's designed for, and painful for everything else. You design the schema around your queries, not your entities.

## Cassandra — The Wide-Column Workhorse

Its data model revolves around two concepts:

- **Partition key**: Determines which node stores the data. All rows with the same partition key live on the same node.

- **Clustering key**: Determines the sort order within a partition. Rows with the same partition key are sorted by clustering key on disk.

```sql
CREATE TABLE messages (
    channel_id UUID,
    message_id TIMEUUID,
    author_id UUID,
    content TEXT,
    created_at TIMESTAMP,
    PRIMARY KEY (channel_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);
```

Here, `channel_id` is the partition key and `message_id` (a time-based UUID) is the clustering key. This means:

- All messages for one channel are stored together on the same node
- Within a channel, messages are sorted by time (newest first)

### Query-First Design

In relational databases, you model entities first and figure out queries later. In Cassandra, you do the opposite: **start with your queries, then design tables to serve them.**

Need messages by channel? One table. Need messages by user? A second table with the same data, partitioned differently:

```SQL
-- Optimized for "messages in a channel"
CREATE TABLE messages_by_channel (
    channel_id UUID,
    message_id TIMEUUID,
    content TEXT,
    PRIMARY KEY (channel_id, message_id)
);

-- Optimized for "messages by a user"
CREATE TABLE messages_by_user (
    user_id UUID,
    message_id TIMEUUID,
    content TEXT,
    PRIMARY KEY (user_id, message_id)
);
```

Yes, you store the data twice. That's the trade-off: **storage duplication for query performance**. There are no joins in Cassandra, so every query pattern needs its own table.

### Time-Series Bucketing

For time-series data, unbounded partitions are dangerous. If a sensor sends data every second for a year, that's 31 million rows in one partition — too large.

The solution is **bucketing**: add a time component to the partition key.

```sql
CREATE TABLE sensor_readings (
    sensor_id UUID,
    day DATE,
    reading_time TIMESTAMP,
    value DOUBLE,
    PRIMARY KEY ((sensor_id, day), reading_time)
);
```

Now each partition holds one day's data for one sensor. Predictable size, easy to query ("readings for sensor X on March 15"), and old data can be dropped by TTL or partition deletion.

**Google Bigtable** inspired the entire wide-column category. It powers Google's internal systems — search indexing, Maps, Gmail, Analytics. Not available as a standalone product, but Cloud Bigtable is the managed version.

**HBase** is the open-source Bigtable clone, running on top of HDFS (Hadoop's file system). It's common in analytics and batch-processing pipelines where you need to scan large ranges of data.

## TimescaleDB — Time-Series on PostgreSQL

TimescaleDB is a PostgreSQL extension that adds time-series superpowers:

- **Hypertables**: Automatic time-based partitioning. You create a regular table, call create_hypertable(), and TimescaleDB handles partition management.
- **Compression**: 10-20x compression on older data. Recent data stays uncompressed for fast writes.
- **Continuous aggregates**: Materialized views that automatically update as new data arrives. Pre-compute hourly/daily rollups without batch jobs.
- **Retention policies**: Automatically drop data older than N days.

The beauty: you get time-series performance with full SQL, joins, transactions, and your existing PostgreSQL tooling.

> 💡 Interview Tip  
> Only bring up Cassandra or a wide-column store when the problem explicitly involves massive write throughput or time-series data (messaging, metrics, activity feeds). If you mention Cassandra for a simple CRUD app, it signals you're choosing buzzwords over pragmatism.

---

# Chapter 8 Specialized Models — Graph, Search, and Polyglot Persistence

## Graph Databases — When Relationships Are the Data

### How They Work

Graph databases store data as nodes (entities) and edges (relationships between them).

![Text23](/assets/23.png)

### When Graphs Actually Help

| Use Case               | Why Graph Wins                                                      |
| ---------------------- | ------------------------------------------------------------------- |
| Fraud detection        | Detect rings of accounts transferring money in circles              |
| Recommendation engines | "Users who liked X also liked Y" — traverse preference edges        |
| Knowledge graphs       | Wikipedia-style "related concepts" with arbitrary depth             |
| Network topology       | "Which switches are between server A and server B?"                 |
| Access control         | "Does user X have permission Y through any group membership chain?" |

### When Graphs Don't Help (And This Is Important)

most social media queries don't need deep graph traversals. "Show me Alice's friends" is a simple one-hop query that a relational database with a follows table handles perfectly. "Show me friends-of-friends-of-friends" (3+ hops) is where graph databases shine — and most products never need that.

> 💡 Interview Tip  
> Don't mention graph databases in a system design interview unless the problem explicitly involves multi-hop relationship traversal (fraud detection, recommendation engines, access control). For social media apps, a relational follows table with proper indexes is the correct answer.

## Search Engines — When You Need Full-Text and Fuzzy Matching

### The Problem Relational Databases Can't Solve Well

```sql
-- This is slow and limited
SELECT * FROM products WHERE name LIKE '%wireless headphone%';
```

`LIKE '%term%'` can't use a B-tree index. It scans every row. And it can't handle typos ("wireles headphone"), synonyms ("earbuds"), or relevance ranking.

### How Elasticsearch Works

Elasticsearch builds an **inverted index** — the same structure used by search engines like Google.

Before indexing data, Elasticsearch **breaks text into tokens** using something called an analyzer. For example, “Wireless Headphones Pro” might be split into tokens like “wireless,” “headphones,” and “pro.” These tokens are what get stored in the inverted index. This is why it can match individual words inside long text fields.

Normal index: document → words in that document Inverted index: word → documents containing that word

        Inverted Index:
        "wireless"   → [doc_1, doc_5, doc_12]
        "headphone"  → [doc_1, doc_3, doc_5]
        "bluetooth"  → [doc_1, doc_5, doc_8]

Searching for "wireless headphone" → intersect the two posting lists → `[doc_1, doc_5]`. Fast, regardless of how many documents exist.

Elasticsearch also handles:

- **Fuzzy matching**: "wireles" still finds "wireless"
- **Tokenization**: Breaking text into searchable terms
- **Relevance scoring**: Results ranked by TF-IDF or BM25
- **Autocomplete**: Prefix matching with edge-ngrams

### The Pattern: Search Alongside Primary Database

Elasticsearch is not a primary database — it doesn't provide ACID transactions or referential integrity. The standard pattern:

![Text24](/assets/24.png)

Elasticsearch is usually not used as the main database. Instead, applications keep a primary database like PostgreSQL as the “source of truth,” and then copy data into Elasticsearch for search. This sync happens asynchronously, so search results might lag slightly behind real data, but that trade-off is acceptable for most applications.

### When to Add Search

| Signal                                    | Example                                 |
| ----------------------------------------- | --------------------------------------- |
| Full-text search across large text fields | Product descriptions, articles, reviews |
| Autocomplete / type-ahead                 | Search bars, address lookup             |
| Fuzzy matching needed                     | Handling typos in user queries          |
| Complex filtering + sorting + relevance   | E-commerce product search with facets   |
| Log analysis                              | Centralized logging (ELK stack)         |

## NewSQL — Distributed SQL That Actually Scales

Traditional SQL: strong consistency + transactions, but scaling means manual sharding. NoSQL: horizontal scaling, but you give up joins, transactions, and SQL. NewSQL: **horizontal scaling with full SQL, ACID transactions, and automatic sharding**.

**Need SQL + horizontal scaling + ACID**

### Google Spanner ..........

---

**Geospatial Indexes** : If your system design involves "find nearby restaurants" or "drivers within 5km," you need geospatial indexing. Relational databases handle this with extensions.

Most real-world systems use more than one database. This is called **polyglot persistence** — choosing the right database for each specific workload.

## (IMP) The Decision Framework

        1. Start with PostgreSQL (or MySQL) as your primary database
        2. For each access pattern that struggles, ask:
                a. Can I solve it with an index or query optimization? → Do that
                b. Can I solve it with a cache (Redis)? → Add a cache
                c. Do I need full-text search? → Add Elasticsearch
                d. Do I need massive write throughput for time-series? → Add Cassandra or TimescaleDB
                e. Do I need multi-hop graph traversals? → Add Neo4j
                f. Do I need horizontal SQL scaling? → Consider CockroachDB/Spanner
        3. Every additional database adds operational complexity — justify it

---

# Chapter 9 Replication Models — Leader, Multi-Leader, and Leaderless

## Leader-Follower Replication (The Default)

The most common replication model, used by PostgreSQL, MySQL, MongoDB, and Redis.

![Text25](/assets/25.png)

### How it works:

1. One node is the leader (also called primary or master). All writes go through it.
2. The leader sends every write to all followers (replicas) via a replication log.
3. Clients can read from any follower (for read scaling) or from the leader (for consistency).

### Synchronous vs asynchronous replication:

- In **synchronous replication**, the leader waits until at least one (or all) follower replicas confirm they received the write before telling the client “success.” This guarantees all replicas are fully up to date at that moment, so reads from any replica are consistent. The downside is speed: if even one replica is slow or temporarily lagging, every write gets delayed.

- In **asynchronous replication**, the leader immediately confirms the write to the client and then sends the update to followers in the background. This makes writes very fast, but it introduces replication lag — followers might still show old data for a short time. So a user might write data and then immediately not see it on a replica.

- In **semi-synchronous replication**, the system combines both approaches. The leader waits for at least one follower to confirm the write, but not all of them. Other replicas can catch up later asynchronously. This gives a balance: better safety than pure async, but better performance than full sync. Most real-world databases use some variation of this because it provides a practical trade-off between consistency and latency.

In practice, fully synchronous replication is too slow. Most systems use semi-synchronous (one sync follower for durability guarantee, rest async for speed).

#### Replication Lag

With async replication, followers lag behind the leader. A write that just committed on the leader might not be visible on a follower for milliseconds to seconds.

This creates three problems:

- **Read-after-write inconsistency**: Alice updates her profile on the leader, then reads from a follower that hasn't caught up yet — she sees her old profile.  
  **Fix**: Route a user's reads to the leader for a few seconds after they write. Or track the user's latest write timestamp and only read from followers that are caught up past that point.

- **Monotonic read violations**: Bob reads his feed from Follower A (sees 10 posts), then reads from Follower B (which is further behind — sees only 8 posts). Posts disappeared!  
  **Fix**: Pin each user's reads to the same follower (using a hash of user ID).

- **Causal ordering violations**: Carol posts "Is anyone hiring?", Dave replies "Yes, we are!" — but a follower shows Dave's reply before Carol's post because the writes arrived out of order.  
  **Fix**: Use causal consistency mechanisms or ensure causally related writes go through the same replication path.

#### Failover

When the leader dies, a follower must be promoted. This is called failover.

1. Detect that the leader has failed (usually via timeout)
2. Elect a new leader (typically the most up-to-date follower)
3. Reconfigure clients to send writes to the new leader

**Dangers**

1. The new leader may not have all the old leader's latest writes (data loss)
2. If the old leader comes back, it thinks it's still the leader (split brain — two nodes accepting writes simultaneously)
3. Misconfigured timeouts can trigger unnecessary failovers under load

## Multi-Leader Replication

Each datacenter has its own leader. Used when you need writes in multiple geographic regions.

![Text26](/assets/26.png)

When it makes sense:

- Multi-region deployments where users need low-latency writes from their local region
- Offline-first apps (each device acts as a leader, syncs when reconnected)
- Collaborative editing

**The problem**: write conflicts. If User A in the US edits a document and User B in the EU edits the same document simultaneously, both leaders accept the write. Now the two leaders have conflicting versions.

**Conflict resolution strategies:**

- **Last write wins (LWW)**: Use timestamps, higher timestamp wins. Simple but loses data.
- **Merge values**: Keep both versions and let the application merge them (like Git merge)
- **Custom resolution**: Application-specific logic

## Leaderless Replication (Dynamo-Style)

No designated leader. Any node can accept reads and writes. Used by DynamoDB, Cassandra, Riak, and Voldemort.

### 1. Quorum Reads and Writes

The client writes to W nodes and reads from R nodes out of a total of N replicas. As long as:

`W + R > N`

...at least one node in the read set will have the latest write, guaranteeing the client sees up-to-date data.

Example with N=3:

| W   | R   | Guarantee                    | Trade-off                              |
| --- | --- | ---------------------------- | -------------------------------------- |
| 2   | 2   | W+R=4 > 3 — consistent reads | Balanced                               |
| 3   | 1   | W+R=4 > 3 — consistent reads | Writes slow (wait for all), reads fast |
| 1   | 3   | W+R=4 > 3 — consistent reads | Writes fast, reads slow                |
| 1   | 1   | W+R=2 < 3 — may read stale   | Fast but no consistency guarantee      |

### 2. Sloppy Quorums and Hinted Handoff

A key improvement in real systems is **sloppy quorums**. Normally, a write must go to the “correct” replica nodes. But if one is down, the system temporarily writes to another healthy node instead. That node stores a hint saying “this data belongs to node X,” and later forwards it when the original node comes back. This is called **hinted handoff**, and it keeps the system available even during failures.

### 3. Anti-Entropy and Read Repair

Leaderless systems use two mechanisms to keep replicas in sync:

**Read repair**: When a client reads from R nodes and notices one returned stale data (outdated), it writes the newer value back to the stale node.

**Anti-entropy process** : where nodes periodically compare data in the background and sync missing or outdated pieces.. Unlike leader-based replication, this doesn't guarantee any particular order.

---

# Chapter 10 Consistency Models and CAP in Practice

## Three Librarians, One Book

Imagine three librarians working at different desks in a large library. When someone checks out a book at desk A, how quickly do desks B and C learn about it?

- **Strong consistency**: They sync after every single checkout. If you walk to desk B immediately after someone checked out a book at desk A, desk B already knows. Slow but always correct.

- **Eventual consistency**: They sync at the end of the day. For most patrons this is fine -- they browse the catalog, pick a book, and it's on the shelf. But if two people want the same rare book, one of them might walk to the shelf and find it gone despite the catalog saying it was available.

## The CAP Theorem

- **Consistency (C):** Every read receives the most recent write or an error.
- **Availability (A)**: Every request receives a non-error response (no guarantee it's the most recent).
- **Partition tolerance (P)**: The system continues operating despite network partitions between nodes.

**Partition tolerance isn't optional**. Networks fail. Packets get dropped. Switches die. You can't build a distributed system that simply ignores partitions. So the real choice is:

- **CP** -- During a partition, reject requests rather than serve stale data. The system is consistent but not available.

- **AP** -- During a partition, serve whatever data you have. The system is available but might return stale reads.

```text
                 C (Consistency)
                /\
               /  \
              /    \
          CP /      \ CA (only works if
            /        \   no partitions)
           /          \
          /____________\
     P (Partition       A (Availability)
      Tolerance)
            AP

  During a partition, choose CP or AP.
  CA only exists in a single-node system.
```

## The Consistency Spectrum

CAP frames consistency as binary, but real systems live on a spectrum:

### 1. Strong Consistency

All reads reflect the most recent write. If you write balance = `\$500`, every subsequent read from any node returns `\$500`.

**How it works** : Typically requires consensus protocols (Raft, Paxos) or synchronous replication. The write isn't acknowledged until a majority of nodes confirm it.

**Used by** : Banking systems, ticket booking, inventory management. Anywhere double-selling or double-spending is unacceptable.

**Cost** : Higher latency (must wait for quorum). Lower throughput. Cross-region replication adds 50-200ms per write.

### 2. Causal Consistency

Related events appear in the correct order. Unrelated events can be seen in any order.

- **Example**: On a social platform, if Alice posts "I got the job!" and then Bob replies "Congrats!", causal consistency guarantees that no one ever sees Bob's reply without Alice's original post. But two unrelated posts by different users might appear in different orders on different replicas.

- **How it works** : The system tracks causal dependencies between operations (often using vector clocks or logical timestamps).

- **Used by** : Collaboration tools, social media comment threads, chat applications.

### 3. Read-Your-Writes Consistency

You always see your own updates immediately. Other users might see stale data for a short window.

- **Example** : You update your profile bio on Instagram. When you refresh, you see the new bio instantly. Your friend across the country might see the old bio for a few seconds until replication catches up.

- **How it works** : Route the user's reads to the same node that handled their writes (session affinity), or track the user's last write timestamp and ensure reads wait for replication to catch up to that point.

- **Used by** : Social media profiles, user settings, shopping carts.

### 4. Eventual Consistency

All replicas converge to the same value over time, but there's no guarantee about how long that takes.

**Example** : You update a DNS record. It might take minutes to hours for all DNS servers worldwide to reflect the change. But eventually they all will.

**How it works** : Writes propagate asynchronously to all replicas. No synchronization required at write time.

**Used by** : DNS, CDN caches, social media feeds, product catalogs.

**Cost** : Cheapest in terms of latency and throughput. You pay with temporary inconsistency.

```text
Strong ←————————————————————————————→ Eventual
  |          |              |              |
  |     Causal      Read-your-writes       |
  |                                        |
Slowest,                            Fastest,
most correct                     temporarily stale
```

## PACELC: The Full Picture

CAP only tells half the story -- it's about what happens during partitions. But what about normal operations when everything is fine?

**The PACELC theorem extends CAP:**

> If there is a Partition, choose Availability or Consistency. Else (no partition), choose Latency or Consistency.

Even when the network is healthy, there's still a trade-off. Synchronous replication gives you consistency but adds latency. Asynchronous replication gives you low latency but risks stale reads.

## ACID vs BASE

These two acronyms represent the two ends of the consistency spectrum applied to database transactions:

| ACID                                                       | BASE                                                          |
| ---------------------------------------------------------- | ------------------------------------------------------------- |
| Stands for Atomicity, Consistency, Isolation, Durability   | Basically Available, Soft state, Eventually consistent        |
| Consistency: Strong — transactions are all-or-nothing      | Consistency: Eventual — replicas converge over time           |
| Availability: May reject writes to preserve consistency    | Availability: Prioritizes availability, tolerates stale reads |
| Scale model: Vertical (bigger machine) or careful sharding | Scale model: Horizontal (add more nodes)                      |
| Best for: Financial transactions, inventory, bookings      | Best for: Social feeds, product catalogs, analytics           |
| Examples: PostgreSQL, MySQL, Spanner                       | Examples: DynamoDB, Cassandra, CouchDB                        |

## Mixed Consistency: Design Per Feature, Not Per System

Here's what experienced engineers know: **you don't pick one consistency model for your entire system**. You pick different models for different features based on their requirements.

Take Ticketmaster:

- **Seat availability for browsing**: Eventual consistency is fine. If the page shows 50 seats available and the true number is 48, nobody cares. The user hasn't committed to anything yet.
- **Seat booking**: Strong consistency is mandatory. Two users cannot book the same seat. This path uses distributed locks or serializable transactions.
- **Order confirmation emails**: Eventual consistency. The email can arrive a few seconds after the booking. No one notices.

---

# Chapter 11 Concurrency Control and Isolation Levels

## Why Concurrency Is Hard

When transactions run one at a time (serially), everything is simple. Transaction A finishes, then Transaction B starts. No conflicts possible.

But serial execution is slow. Modern databases run **transactions concurrently** — overlapping in time — to maximize throughput. The database's job is to make this concurrent execution appear as if transactions ran serially. How well it achieves this is defined by the isolation level.

## The Four Isolation Levels

SQL defines four isolation levels, from weakest to strongest. Each prevents more types of concurrency bugs, but costs more performance.

### 1. Read Uncommitted (Weakest — Rarely Used)

A transaction can read another transaction's uncommitted writes (dirty reads). Almost never used because it's unsafe — if the other transaction rolls back, you read data that never existed.

### 2. Read Committed (The Practical Default)

A transaction only sees data that has been committed. No dirty reads.

Most databases default to this level (PostgreSQL, Oracle, SQL Server). It prevents dirty reads and dirty writes, but not all race conditions.

#### What it doesn't prevent:

```text
Time     Transaction A              Transaction B
─────────────────────────────────────────────────
T1       SELECT balance → $1000
T2                                  SELECT balance → $1000
T3       UPDATE balance = $900
T4       COMMIT
T5                                  UPDATE balance = $900  ← lost A's update!
T6                                  COMMIT
```

Both transactions `read $1000`, both `subtract $100`, both write $900. The result should be $800 but it's $900. Transaction A's update was `lost`. This is the lost `update problem`.

### 3. Repeatable Read / Snapshot Isolation

A transaction sees a consistent snapshot of the database from the moment it started. Even if other transactions commit changes, this transaction keeps seeing the old data.

PostgreSQL's "Repeatable Read" is actually snapshot isolation (implemented with MVCC — see below).

This prevents lost updates in most cases, and prevents read skew (seeing inconsistent data across multiple reads within one transaction).

### 4. Serializable (Strongest)

Transactions behave as if they executed one at a time. No concurrency anomalies possible. The database achieves this through actual serial execution, two-phase locking, or serializable snapshot isolation (SSI).

**The cost**: Significantly slower. Transactions may **be aborted and retried if the database detects conflicts**.

### Comparison

| Level            | Dirty Reads | Lost Updates     | Read Skew | Write Skew | Performance |
| ---------------- | ----------- | ---------------- | --------- | ---------- | ----------- |
| Read Uncommitted | Possible    | Possible         | Possible  | Possible   | Fastest     |
| Read Committed   | Prevented   | Possible         | Possible  | Possible   | Fast        |
| Repeatable Read  | Prevented   | Mostly prevented | Prevented | Possible   | Medium      |
| Serializable     | Prevented   | Prevented        | Prevented | Prevented  | Slowest     |

## MVCC — Multi-Version Concurrency Control

MVCC is the mechanism that makes snapshot isolation possible. Instead of locking rows when reading, the database keeps **multiple versions** of each row.

```Text
Row versions for user_id = 42:
┌──────────┬──────────┬────────────┬────────────┐
│ version  │ balance  │ created_by │ visible_to │
├──────────┼──────────┼────────────┼────────────┤
│    1     │  $1000   │  txn_100   │ txn < 200  │
│    2     │   $900   │  txn_150   │ txn < 300  │
│    3     │   $800   │  txn_250   │ txn ≥ 250  │  ← current
└──────────┴──────────┴────────────┴────────────┘
```

When Transaction 200 reads this row, it sees version 2 ($900) — the latest version visible to it. Transaction 300 sees version 3 ($800). Neither blocks the other. Readers never block writers, writers never block readers.

This is why PostgreSQL can handle thousands of concurrent reads without locking. Old versions are cleaned up by the VACUUM process after all transactions that might need them have finished.

**Key insight from DDIA**: "Snapshot isolation is a boon for long-running, read-only queries such as backups and analytics. It is very hard to reason about data if it keeps changing while a query is running."

## Solving the Concert Ticket Problem

### Pessimistic vs Optimistic Locking

**pessimistic locking**, the database assumes conflicts are likely, so it prevents other users from changing the data while one transaction is working on it. It does this by placing a lock on the row as soon as someone reads it for an update.

In the concert ticket example, User A starts buying the last ticket and runs a query like SELECT ... FOR UPDATE. This locks that ticket row. While User A is completing the purchase, User B also tries to buy the same ticket, but now User B must wait because the row is locked. Once User A finishes and commits the transaction, the lock is released. Then User B can continue, but by that time the ticket status is already “sold,” so User B cannot buy it.

The main **advantage** of pessimistic locking is safety: it avoids conflicts and prevents two users from updating the same data at the same time. It is useful when contention is high, such as ticket booking, banking, or inventory systems where conflicts are common and correctness is critical.

The **downside** is that locking can reduce performance.

With **optimistic locking**, the database does not immediately block anyone. Instead, both users are allowed to read the ticket as “available” and continue with the purchase process. Along with the ticket data, the database keeps a small number called a version (for example, version 5). When User A tries to buy the ticket, the database updates the row only if the version is still 5. Since no one changed it yet, the update succeeds, the ticket becomes “sold,” and the version changes to 6.

Now User B also tries to buy the same ticket, but their update still expects version 5. The database checks and sees the version is already 6 because User A bought it first. So User B’s update affects 0 rows, which tells the application: “Someone else changed this ticket already.” The app can then **show an error or retry** the process.

This approach is called optimistic because it assumes conflicts are rare. It works well when many users are reading data but only a few are changing it. The **advantage** is that nobody has to wait or get blocked, so the system is faster under normal conditions. The **downside** is that if many people try to update the same data at once, lots of transactions fail and must retry, which wastes some work.

## Write Skew — The Subtle Race Condition

Write skew occurs when two transactions read the same data, make decisions based on it, and write to different rows — but the combined result violates a constraint.

**Example**: A hospital requires at least one doctor on call. Two doctors are on call. Both check "is there more than one doctor on call?" — both see "yes" — both remove themselves. Now zero doctors are on call.

### Solutions:

1. **Serializable isolation**: The database detects the conflict and aborts one transaction.
2. **Materialized conflict**: Add a lock row that both transactions must acquire.
3. **Application-level check**: Verify the constraint after the write and roll back if violated

---

# Chapter 12 Composite Indexes, Covering Indexes, and When NOT to Index

A **composite index** is simply an index built on multiple columns, in a specific order. For example, an index on (department, last_name, date) is sorted first by department, then by last name inside each department, and finally by date inside each last name. Because of this order, the database can efficiently answer queries that start from the left side of the index. So it works for queries using (department), (department, last_name), or (department, last_name, date). But it does not help much for queries on just (last_name) or (date) because the data is not organized starting from those columns. This is called the leftmost prefix rule.

A **covering index** goes one step further. Normally, when the database finds matching rows in an index, it still has to go back to the main table (called the heap) to fetch the remaining columns. But if the index already contains every column the query needs, the database can answer the query directly from the index itself. That saves extra disk reads and makes queries faster. For example, if your query only needs name and email, and both are already stored in the index, the database never touches the main table.

A **partial index** indexes only some rows instead of the entire table. For example, if most users are inactive but your application mostly searches active users, you could create an index only on rows where status = 'active'. This keeps the index much smaller and faster because it ignores data you rarely query.

## Composite Indexes

### The Leftmost Prefix Rule

The index can be used for queries that filter on a leftmost prefix of the index columns:

- Put equality conditions (=) first
- Put range conditions (>, <, BETWEEN) last
- Among equality columns, put the most selective/high-cardinality column first.

## Covering Indexes — Never Touch the Heap

A covering index is an index that contains everything a query needs, so the database never has to go back to the main table to fetch extra data. This matters because a normal index lookup usually happens in two steps:

1. Use the index to find matching rows
2. Go to the actual table (heap/clustered storage) to fetch the remaining columns.

That second step is expensive because it often means extra random disk reads.

For example, imagine an index on (user_id, status). If you run a query asking only for user_id and status, the database can answer directly from the index because those columns are already stored there. But if the query also asks for total_amount, the database must jump back to the table for every matching row since total_amount is missing from the index. That extra step is called a **heap fetch** in PostgreSQL or a **key lookup** in SQL Server.

### (IMP) Same Index, Different Queries — The Key Insight

A covering index can feel confusing because it sounds like the database already “knows the answer” before the query runs. But that is not what happens.

The database is not storing query results. It is only storing data in a smarter structure so queries can read directly from the index instead of repeatedly visiting the main table.

Think of a table as a big book containing complete information.

Example table:
| id | user_id | status | total_amount |
| -- | ------- | -------- | ------------ |
| 1 | 5 | active | 100 |
| 2 | 5 | active | 200 |
| 3 | 8 | inactive | 300 |

Now suppose we create this index:

```sql
CREATE INDEX idx_orders
ON orders(user_id, status, total_amount);
```

This index is like a smaller, organized copy containing only these columns.  
Now imagine this query:

```sql
SELECT total_amount
FROM orders
WHERE user_id = 5
AND status = 'active';
```

What does the database need?

        user_id → for filtering
        status → for filtering
        total_amount → for output

All three columns already exist inside the index.

So the database can:

1. Search the index
2. Find matching rows
3. Read total_amount
4. Return results

It never opens the main table.

Now let’s change the query:

```sql
SELECT id
FROM orders
WHERE user_id = 5
AND status = 'active';
```

`id` is NOT inside the index.

So the database:

1. Searches the index
2. Finds matching rows
3. Uses pointers to go back to the main table
4. Reads id

Now the index is NOT covering.

### Making an Index Covering with INCLUDE

`(total_amount)` These are NOT used for searching or sorting.

They are only stored at the bottom (leaf pages) of the index as extra attached data.

Think of it like:  
`(user_id, status)  ---> extra payload: total_amount`

So when query runs:

```sql
SELECT total_amount
FROM orders WHERE user_id = 5 AND status = 'active';
```

The database:

- Uses (user_id, status) to quickly find rows
- Reads total_amount directly from the index leaf
- Never touches the main table

Why not make `total_amount` a normal key column?  
tree becomes larger, sorting becomes more expensive, traversal slower,maintenance cost higher

![Text27](/assets/27.png)

**Downside** is that indexes become larger because they store more data, which increases storage and maintenance costs.

There are also some important caveats. Covering indexes are fragile: if a query later adds one more column that is not present in the index, the database immediately falls back to heap lookups and performance can drop.

## Partial Indexes — Index Only What Matters

A partial index is an index that stores only the rows you actually care about searching. Normally, a database index contains entries for every row in a table, even if most of those rows are rarely queried. This can make the index large and slower to search. A partial index solves this by indexing only rows that match a specific condition. For example, if an orders table mostly contains `completed` orders but your application frequently searches only for `pending` orders, you can create an index only for rows where `status = 'pending'`. This makes the index much smaller because it ignores all completed orders. Smaller indexes use less memory, require fewer disk reads, and are faster to search.

The database can use this index only for queries that match the same condition. So a query searching for pending orders can use the partial index, but a query searching for completed orders cannot because those rows were never added to the index. Partial indexes are especially useful for things like active users, unprocessed jobs, soft-deleted records, or recent important data where only a small portion of the table is queried often.

One important limitation is that the condition must remain stable. You cannot create a partial index using something dynamic like `WHERE created_at > now()` because the meaning changes constantly as time passes. Databases require the condition to be fixed so the index remains valid. That is why people often use a hardcoded date or periodically recreate the index if they need a rolling “recent data” window.

## Some extra topics

### GIN Indexes — For JSON and Arrays

### GiST Indexes — For Geospatial Queries

### BRIN Indexes — Tiny Indexes for Huge, Ordered Tables

### Bloom Filters — Probabilistic "Maybe" Checks

A Bloom filter isn't an index in the traditional sense. It's a space-efficient probabilistic data structure that answers one question: "Is this element possibly in the set?"

A Bloom filter can tell you:

- **Definitely not here** — 100% certain, skip this file/page
- **Possibly here** — might be a false positive, need to check

It can never give a false negative. If the Bloom filter says "not here," the element is guaranteed absent.

Bloom filters are used heavily inside LSM tree databases (Cassandra, RocksDB) to avoid reading SSTables that definitely don't contain the key. They're also used in PostgreSQL (via the bloom extension) for multi-column approximate matching.

## When NOT to Index

Every index you add has costs: slower writes, more disk space, more memory pressure on the buffer pool. Here's when adding an index is the wrong move:

- **Write-heavy tables**: If a table has 5 indexes, every INSERT does roughly 6 writes — one to the heap and one to each index. For a logging table receiving 100,000 inserts per second, those extra writes add up fast. UPDATEs can be worse.

- **Small tables**: A table with 1,000 rows fits in a handful of pages. A sequential scan of 5 pages is faster than traversing a B-tree.

- **Low cardinality columns**: An index on a gender column with 3 distinct values is nearly useless. The index points to ~33% of the table for each value — the database would rather just scan the whole table.

- **Rarely queried columns**: An index that nobody queries is pure waste — it slows down writes and consumes disk space for zero benefit.

## The Interview Index Selection Strategy

When an interviewer asks about indexes for a table, follow this framework:

1. **Identify the hot queries** — what does the application query most?
2. **Look at WHERE and JOIN columns** — those are index candidates
3. **Check ORDER BY** — can an index provide the sort order?
4. **Consider composite indexes** — one index serving multiple queries beats multiple single-column indexes
5. **Check for covering potential** — can you add INCLUDE columns to avoid heap lookups?
6. **Consider the write load** — every index costs on writes

Then verify: run the real query under EXPLAIN ANALYZE and confirm the planner actually picks your index. An index the optimizer ignores is just write overhead.

> Need to revisit this topic.

---

# Chapter 13 Query Patterns That Break at Scale

## The N+1 Query Problem

**What It Is** : You query a list of posts, then for each post, make a separate query to get the author. 1 query for posts + N queries for authors = N+1 queries.

51 database round trips to load one page. Each round trip has network latency .

**Four Solutions**
**1. Eager loading (JOIN):**

```sql
SELECT posts.*, users.username, users.avatar_url
FROM posts
JOIN users ON posts.user_id = users.id
LIMIT 50;
```

One query. One round trip. The database handles the join.

**2. Batch loading (WHERE IN):**

```sql
-- First query: get posts
SELECT * FROM posts LIMIT 50;
-- Second query: get all authors at once
SELECT * FROM users WHERE id IN (1, 2, 3, 5, 8, 13, ...);
```

Two queries instead of 51. Your application code matches them up.

**3. DataLoader pattern**:  
Used in GraphQL and modern ORMs. Batches individual lookups within a single tick/request cycle into one query automatically.

**4. Subquery:**

```sql
SELECT * FROM users WHERE id IN (
    SELECT DISTINCT user_id FROM posts ORDER BY created_at DESC LIMIT 50
);
```

## Offset Pagination — The Silent Killer

### How Offset Works

```sql
-- Page 1 (fast)
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 0;

-- Page 100 (slow!)
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 1980;
```

Offset pagination is the traditional way websites load pages of data.

When you use `OFFSET 1980`, the database reads and sorts all those earlier rows (`2000`), throws most of them away, and finally returns only 20 results. This wasted work becomes very expensive as the page number grows larger. That’s why deep pagination becomes slow.

### Cursor-Based Pagination (The Fix)

Instead of "skip N rows," use a cursor — the last value you saw:  
"it says “start after this specific item.” The database remembers the last item shown and continues from there.

```sql
-- First page
SELECT * FROM posts
ORDER BY created_at DESC
LIMIT 20;

-- Next page (cursor = timestamp of last item)
SELECT * FROM posts
WHERE created_at < '2024-03-15T10:30:00Z'
ORDER BY created_at DESC
LIMIT 20;
```

**The trade-off**: You can't jump to "page 500" directly — you can only go forward/backward from a cursor.

### Keyset Pagination for Ties

If multiple rows share the same created_at, the cursor is ambiguous. Fix with a composite cursor:

```sql
SELECT * FROM posts
WHERE (created_at, id) < ('2024-03-15T10:30:00Z', 12345)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

This requires an index on (created_at, id) and guarantees uniqueness.

### Real-Time Insertions and Duplicate Items

Cursor pagination is also much better for real-time applications like social media feeds, chat apps, or notifications. With offset pagination, if new posts are inserted while a user is scrolling, the positions of rows shift. This can cause duplicate items or missing items between pages. Cursor pagination avoids this because the cursor points to a fixed item in the dataset, so new insertions above it do not affect the next query.

## Aggregation at Scale

### COUNT(\*) Is Expensive

```sql
SELECT COUNT(*) FROM likes WHERE post_id = 42;
```

On a post with 3 million likes, this scans 3 million rows (even with an index, it must count each qualifying entry). Doing this on every page load is a disaster.

#### Solutions:

1. **Pre-computed counters (most common)**: Store `like_count` on the posts table. Increment/decrement atomically on each like/unlike. Accept minor drift, reconcile periodically.

2. **Approximate counts (HyperLogLog)**: For very large counts where exact numbers don't matter (unique visitors, distinct values), HyperLogLog gives ~2% accuracy with a fixed 12KB of memory. Redis supports this natively with `PFADD` and `PFCOUNT`.

3. **Cached counts**: Compute the count periodically (every minute, every hour) and cache the result. Show "~3.2M likes" instead of "3,247,891 likes."

### Aggregation Across Partitions

If your data is partitioned or sharded, aggregations become scatter-gather operations: query each partition, sum the results. This is inherently expensive. Pre-compute when possible.

## Connection Exhaustion

### The Problem

Every database connection consumes resources.

PostgreSQL typically supports 100-500 max connections. With 50 microservice instances, each wanting 20 connections: 1,000 connections. You've exceeded the limit.

## Connection Pooling — The Solution

**Client-side pooling (HikariCP, SQLAlchemy pool)**: Each application instance maintains a small pool of reusable connections. Instead of opening/closing a connection per query, you borrow one from the pool and return it when done. Avoids the overhead of TCP handshake + authentication per query.

**Server-side pooling (PgBouncer)**: Sits between all application instances and the database. Multiplexes thousands of client connections into a small number of real database connections.

![Text28](/assets/28.png)

PgBouncer's **transaction mode** is the most efficient: a real connection is allocated only for the duration of a transaction, then returned to the pool. Between queries, no connection is held.

**Best practice**: Use both. HikariCP for fast local reuse within each application instance. PgBouncer for protecting the database from connection storms across all instances.

## Read Replicas

When read load overwhelms a single server, add read replicas — copies of the database that handle read queries:

![Text29](/assets/29.png)

**Replication lag**: Replicas are eventually consistent — there's a delay (milliseconds to seconds) between a write on the primary and its appearance on replicas.

**Read-your-writes consistency**: After a user updates their profile, route their reads to the primary for a few seconds to ensure they see their own changes. Other users can read from replicas (slight staleness is fine).

---

# Chapter 14 Partitioning — Splitting Data Within One Database

Imagine a database table that stores millions of records over many years, like invoices, logs, or user activity. If all that data sits in one giant table, every query has to search through a huge amount of information, even when you only need a small part of it. That slows things down.

## What Partitioning Actually Is

**Key point**: `no additional servers`. Partitioning is a single-machine optimization. It improves query speed, simplifies maintenance, and can even help with concurrency -- but it doesn't increase your total storage or write throughput beyond what that one machine offers.

### Horizontal vs Vertical Partitioning

**Horizontal partitioning** splits a table by rows. Each partition contains a subset of the records, but still keeps all the columns. Imagine a giant orders table where each row is an order. You could store orders from 2023 in one partition and orders from 2024 in another. When someone queries only recent orders, the database can ignore older partitions completely. This is the most common form of partitioning because large systems usually grow by accumulating more rows over time.

**Vertical partitioning** works differently. Instead of splitting rows, it splits columns. One table might store frequently accessed columns like user_id, name, and email, while another stores less commonly used data like bio, avatar_url, or large profile settings. Both tables still refer to the same users, but the data is separated based on how often it is accessed.

## PostgreSQL PARTITION BY -- Range, List, Hash

**Range Partitioning**:
Split rows based on a continuous range of values. The classic use case is time-series data.

**List Partitioning** :
Split rows based on a discrete set of values. Great for geographic or categorical data.

**Hash Partitioning**
Split rows by hashing a column value. Useful when there's no natural range or list, but you still want even distribution.

## Partition Pruning -- The Real Performance Win

The real power of partitioning is not just splitting a table into smaller pieces — it’s something called partition pruning. This is the database’s ability to completely ignore partitions that cannot contain the requested data.

Suppose a huge events table is partitioned by date. One partition stores Q1 data, another Q2, another Q3, and so on. When a query includes the partition key in the WHERE clause — such as created_at = '2024-08-15' — PostgreSQL can determine that only the Q3 partition could possibly contain that date. The query `planner` then skips all other partitions entirely.

## Maintenance -- DROP Instead of DELETE

Here's a benefit that doesn't get enough attention: dropping old data becomes trivial.

Suppose you have a data retention policy -- events older than 2 years get deleted. Without partitioning:

```sql
-- Without partitioning: slow, generates massive WAL, locks the table
DELETE FROM events WHERE created_at < '2022-01-01';
-- Could take hours on a billion-row table
```

With partitioning:

```sql
-- Without partitioning: slow, generates massive WAL, locks the table
DELETE FROM events WHERE created_at < '2022-01-01';
-- Could take hours on a billion-row table
```

`DROP TABLE` is a metadata operation. It takes milliseconds regardless of how many rows are in the partition. No dead tuples, no bloat, no VACUUM needed. This alone makes partitioning worth it for time-series data.

Similarly, adding new partitions for upcoming time periods is cheap:

```sql
CREATE TABLE events_2025_q1 PARTITION OF events
    FOR VALUES FROM ('2025-01-01') TO ('2025-04-01');
```

### Sub-Partitioning -- Going Deeper

Sometimes a single level of partitioning is not enough for very large datasets. In those cases, databases can use sub-partitioning, which means partitioning an already partitioned table

## When to Partition

Partitioning is not free. It adds complexity to schema management and can hurt queries that don't include the partition key. Here's a practical decision guide:

| Situation                                                 | Partition? | Why                                      |
| --------------------------------------------------------- | ---------- | ---------------------------------------- |
| Table has 10M+ rows and growing                           | Yes        | Pruning gives measurable speedup         |
| Queries almost always filter by one column (date, region) | Yes        | Natural partition key exists             |
| Time-series data with retention policy                    | Yes        | `DROP PARTITION` enables instant cleanup |
| Small table (under 1M rows)                               | No         | Full scans are already fast              |
| Queries touch random rows across all ranges               | No         | Pruning can't help and adds overhead     |
| Table has many indexes that are getting slow to maintain  | Yes        | Smaller per-partition indexes are faster |

> **You have a 2-billion-row events table partitioned by RANGE on created_at (one partition per month). A query runs: SELECT \* FROM events WHERE user_id = 42. What happens?**  
> The planner scans every partition because the WHERE clause doesn't include the partition key

---

# Chapter 15 Sharding — Splitting Data Across Machines

Instead of storing all data in one massive database server, the data is split across multiple independent databases called **shards**

## Why Shard at All?

Partitioning (from the previous lesson) splits data on a single machine. Sharding splits it across machines. You reach for sharding when a single machine hits hard limits.

## Three Sharding Strategies

### 1. Hash-Based Sharding (Most Common)

Apply a hash function to the shard key and use modulo to pick the shard:

`shard_number = hash(shard_key) % number_of_shards`

A shard-aware query router is just a few extra lines.

That's the entire sharding proxy concept — hash the key, pick the connection, forward the query.

**Pros**: Distributes data evenly regardless of key patterns. Simple to implement.

**Cons**: Adding or removing shards changes the modulo, causing massive data migration. Range queries across shards are expensive.

### 2. Range-Based Sharding

Assign contiguous ranges of the shard key to each shard:

        Shard 0: user_id     1 - 1,000,000
        Shard 1: user_id 1,000,001 - 2,000,000
        Shard 2: user_id 2,000,001 - 3,000,000
        Shard 3: user_id 3,000,001 - 4,000,000

**Pros**: Range queries on the shard key are efficient (only hit relevant shards). Easy to understand and debug.

**Cons**: Uneven distribution if the key isn't uniformly distributed. New users all land on the newest shard, creating a hot spot.

### 3. Directory-Based Sharding

Maintain a lookup table that maps each shard key to its shard:

        ┌────────────┬───────────┐
        │ tenant_id  │ shard     │
        ├────────────┼───────────┤
        │ acme-corp  │ shard-2   │
        │ globex     │ shard-1   │
        │ initech    │ shard-3   │
        │ umbrella   │ shard-2   │
        └────────────┴───────────┘

**Pros**: Maximum flexibility. You can move individual tenants between shards without any algorithmic change. Great for multi-tenant SaaS.

**Cons**: The directory is a single point of failure and a potential bottleneck. Every query requires a lookup before it can be routed.

## Strategy Comparison

| Strategy        | Distribution | Range Queries      | Adding Shards       | Flexibility | SPOF Risk        |
| --------------- | ------------ | ------------------ | ------------------- | ----------- | ---------------- |
| Hash-based      | Excellent    | Poor (scatter)     | Expensive (rehash)  | Low         | None             |
| Range-based     | Risky (skew) | Excellent          | Moderate            | Medium      | None             |
| Directory-based | Controllable | Depends on mapping | Easy (update table) | High        | Directory itself |

> In a system design interview, default to hash-based sharding unless the problem specifically involves range queries or multi-tenant isolation. Say something like: "I'll use hash-based sharding on user_id because it gives even distribution and our primary access pattern is per-user lookups." Then mention the resharding challenge and that you'd use consistent hashing (next lesson) to mitigate it.

## Choosing the Right Shard Key

The shard key is the most important decision in a sharding architecture. Get it wrong, and you'll spend months fixing it. Here's what makes a good shard key:

**High cardinality**. The key must have enough distinct values to distribute across all shards. A boolean column (`is_active`) has cardinality 2 -- useless for 16 shards.

**Even distribution**. Values should spread roughly equally. If 80% of your data has `country = 'US'`, sharding by country puts 80% of data on one shard.

**Query alignment**. Your most common queries should include the shard key, so they can be routed to a single shard. If you shard by `user_id` but most queries filter by `order_date`, every query becomes a cross-shard scatter.

## The Time-Range Sharding Anti-Pattern

> 🔴 Don't Shard by Timestamp  
> Sharding by time range makes the newest shard a write hot spot — 100% of new writes hit one machine while historical shards sit idle. This turns your distributed system back into a single-writer bottleneck.

This one catches people. It seems logical: shard by month or year, since time-series data is often queried by time range.

The problem: all writes go to the newest shard.

        January shard:  0 new writes, 30M existing rows (cold)
        February shard: 0 new writes, 28M existing rows (cold)
        March shard:    ALL new writes (hot!)

The March shard gets hammered while January and February sit idle. You've turned a distributed system into a single-writer bottleneck -- exactly what sharding was supposed to fix.

If you need time-based access patterns, shard by something else (`user_id`) and **partition** within each shard by time. You get the write distribution from sharding and the time-range pruning from partitioning.

---

# Chapter 16 Consistent Hashing and Resharding

The Problem With Simple Hashing

In the previous lesson, we used `shard = hash(key) % N` to distribute data. It works beautifully — until you need more shards.

Imagine you have 4 shards and a key with hash value 14:

- **14 % 4 = 2** → goes to Shard 2

Now you add a 5th shard:

- **14 % 5 = 4** → goes to Shard 4

That key just moved. And it's not alone — roughly 75% of all keys remap when you go from 4 to 5 shards. That means moving 75% of your data across the network. During a resharding event, your system is effectively doing a massive, expensive data migration.

![Text30](/assets/30.png)

## Consistent Hashing — The Ring

Consistent hashing can be thought of as placing both your data (keys) and your servers (shards) on a big circular ring. Every shard is assigned a position on the ring by hashing its name, and every key is also assigned a position by hashing the key value. To find where a key belongs, you start at the key's position and move clockwise around the ring until you encounter the first shard. That shard stores the key.

![Text31](/assets/31.png)

**How it works:**

1. Hash each shard's identifier to a position on the ring
2. Hash each key to a position on the ring
3. Each key is assigned to the next shard clockwise on the ring

Adding a shard: When Shard E is added between Shard B and Shard C, only the keys between B and E need to move to E. All other keys stay where they are.

```py
import bisect, hashlib

class ConsistentHash:
    def __init__(self):
        self.ring = []          # sorted list of (hash_position, shard_name)
        self.nodes = {}

    def add_shard(self, name, vnodes=150):
        for i in range(vnodes):
            h = int(hashlib.md5(f"{name}:{i}".encode()).hexdigest(), 16)
            bisect.insort(self.ring, (h, name))

    def get_shard(self, key):
        h = int(hashlib.md5(key.encode()).hexdigest(), 16)
        idx = bisect.bisect_right(self.ring, (h,))
        if idx == len(self.ring):
            idx = 0              # wrap around the ring
        return self.ring[idx][1]
```

adding a server might force almost all data to be redistributed. With consistent hashing, only the data that falls near the new server needs to move. Everything else stays where it is. For example, if you have 4 servers and add a 5th one, only about 20% of the data moves instead of most of it.

## Virtual Nodes — Evening Things Out

Basic **consistent hashing has a problem**: with only a few physical nodes, the ring can be unevenly divided. One shard might own 40% of the ring while another owns 10%.

Virtual nodes (vnodes) fix this by giving each physical shard multiple positions on the ring.

- Instead of placing each physical shard on the ring once, we place it many times.

```
Physical Shard A:
A1, A2, A3, A4

Physical Shard B:
B1, B2, B3, B4
```

The ring now looks like:

```
A1 -> B1 -> A2 -> B2 -> A3 -> B3 -> A4 -> B4
```

When a key lands between: `A2 ----- B2` it belongs to Shard B.
When it lands between: `B2 ----- A3` it belongs to Shard A.

Since each physical shard owns many small sections scattered around the ring, the total amount of data per shard becomes much more balanced.

**What happens when a new shard is added?**
Suppose each shard has many virtual nodes:

```text
A1 A2 A3 ...
B1 B2 B3 ...
C1 C2 C3 ...
```

Now add Shard D:

```text
D1 D2 D3 ...
```

These new virtual nodes are spread all over the ring.

Instead of taking one large chunk from a single shard, D takes many small chunks from A, B, and C:

```text
A loses a little
B loses a little
C loses a little
```

With hundreds of virtual nodes per physical shard, the distribution becomes statistically even. When a physical shard is added or removed, its virtual nodes are scattered around the ring, so the data movement is evenly distributed across all other shards.

| Approach                    | Keys moved on resize  | Distribution evenness |
| --------------------------- | --------------------- | --------------------- |
| Modulo hashing              | ~(N−1)/N (nearly all) | Perfect               |
| Consistent hashing (basic)  | ~1/N                  | Uneven with few nodes |
| Consistent hashing + vnodes | ~1/N                  | Even                  |

## Resharding in Production

> "What do you do when one shard becomes too big and you need more shards, but your system is serving millions of requests per second?"

Theory is clean. Production is messy. Here's how real systems handle resharding:

### 1. Vitess (YouTube, Slack, GitHub)

Used by companies like YouTube, Slack, and GitHub.

Imagine you have:

```text
Shard 1
 ├── User 1-100M
```

The shard becomes too large.

Vitess can split it into:

```text
Shard 1A
 ├── User 1-50M

Shard 1B
 ├── User 50M-100M
```

While the application is still running, Vitess:

1. Creates new shards.
2. Copies data in the background.
3. Keeps old and new shards synchronized.
4. Switches traffic to the new shards.
5. Removes old shards.

Users never notice the migration.

Think of it as replacing a highway while cars are still driving on it.

### 2. CockroachDB / Spanner Approach

These databases don't expose shards to you.

Instead, data is stored in small chunks called:

```
CockroachDB -> Ranges
Spanner -> Splits

Example:

Range A
Users 1-1M
```

As data grows:

```text
Range A
Users 1-500k

Range B
Users 500k-1M
```

The database automatically splits the range.

If traffic changes, it automatically moves ranges to different machines.

You don't think about: `shard keys`, `shard counts`, `resharding scripts`.
The database continuously balances everything.

### 3. Stripe's Live Migration Approach

Suppose you currently have: `Old Shard` and want to move to: `New Shard`.

You can't simply flip a switch because mistakes happen.

Instead, companies like Stripe use a cautious migration process.

- **Double-write**: Write to both old and new shards simultaneously
- **Backfill**: Copy historical data from old to new
- **Verify**: Compare old and new to ensure completeness
- **Shadow read**: Read from both and compare results
- **Cutover**: Switch reads to new shards
- **Cleanup**: Remove old shards

## Composite Sharding

Sometimes a single shard key isn't enough. Composite sharding combines multiple factors.

### Discord's Approach

Discord shards messages by (channel_id, bucket):

- **channel_id**: All messages for one channel on the same shard (query locality)
- **bucket**: Time-based bucket within the channel (prevents unbounded partition growth)

This means "get messages in channel X from the last hour" hits one shard and one bucket — optimal for their primary access pattern.

### Common Composite Patterns

| Pattern         | Primary Key     | Secondary     | Use Case                           |
| --------------- | --------------- | ------------- | ---------------------------------- |
| user + time     | hash(user_id)   | time_bucket   | Per-user activity with time bounds |
| tenant + entity | hash(tenant_id) | entity_id     | Multi-tenant SaaS                  |
| region + user   | region          | hash(user_id) | Geo-distributed with data locality |

> 💡 Interview Tip  
> If an interviewer asks "how would you handle resharding?" — mention consistent hashing with virtual nodes for the theory, and Vitess or CockroachDB **for the practice**. Then say "Stripe uses a double-write → backfill → verify → cutover approach for zero-downtime migrations." That covers theory, tools, and real-world practice.

---

# Chapter 17 Cross-Shard Queries, Hot Spots, and When NOT to Shard

## The Cost Nobody Mentions Upfront

Sharding tutorials focus on the upside: horizontal scale, distributed writes, theoretically unlimited storage. But they rarely talk about what breaks:

- Queries that used to be a single-table scan now fan out to every shard
- That convenient JOIN between users and orders? Impossible if they're on different shards
- One viral user's data melts a single shard while others sit idle
- Every migration, backup, and monitoring task now happens N times
  Sharding is a permanent architectural commitment with permanent

trade-offs. Let's understand them so you can make informed decisions.

## Cross-Shard Queries

### The Problem

With data sharded by `user_id`, the query "get all posts by user 42" is fast — it hits one shard. But "get the top 100 most-liked posts across all users" must:

- Send the query to every shard
- Wait for the slowest shard to respond (tail latency)
- Merge and re-sort results from all shards

This is called **scatter-gather**, and it gets worse with more shards. 10 shards = 10 parallel queries. 100 shards = 100 parallel queries. The P99 latency is bounded by the slowest shard.

### Cross-Shard Joins

Joins across shards are even harder. If `users` is sharded by `user_id` and `orders` is sharded by `order_id`, joining them requires fetching data from multiple shards on both tables, then joining in the application layer.

Most sharded systems don't support cross-shard joins at all. You work around them.

### Strategies to Avoid Cross-Shard Queries

1. **Co-locate related data** : Shard `users` and `orders` by the same key (`user_id`). All of a user's orders live on the same shard as the user. Joins within a user's data stay on one shard.

2. **Reference table replication** : Small, rarely-changing tables can be replicated to every shard. Joins against reference tables stay local.

3. **Denormalization** : Instead of joining orders with products, store product_name and product_price directly on the order row. Eliminates the join entirely. Instagram stores author usernames on posts for this reason.

4. **Application-level joins**: Fetch data from multiple shards, then join in application code.

5. **Search index for global queries** : Sync data to Elasticsearch for queries that need to span all shards."Top 100 most-liked posts" becomes an Elasticsearch query, not a scatter-gather across shards.

## Hot Spots

## The Celebrity Problem

Even with a perfect hash function and even distribution, some shards get disproportionate traffic.

This isn't a distribution problem. The key itself is hot.

## Time-Based Hot Spots

If you shard by time range (messages from March on Shard 3, April on Shard 4), all new writes hit the current month's shard. The newest shard gets 100% of write traffic while historical shards get only reads.

## Solutions

**Salting the shard key**: Append a random suffix to hot keys. `user:12345` becomes `user:12345:0`, `user:12345:1`, ..., `user:12345:9`. Reads scatter across 10 shards and merge results. Writes distribute evenly.

The trade-off: reads for that key are now 10x more expensive (scatter-gather). Only worth it for the hottest keys.

**Request coalescing (Discord's approach)**: When 1,000 users request the same channel's messages simultaneously, the data services layer coalesces them into a single database read and fans the result out to all 1,000 requestors. The database sees 1 query instead of 1,000.

This is an application-layer solution, not a database solution — and it was one of the most impactful changes in Discord's architecture.

**Dedicated shards**: Move the hottest keys to their own dedicated shard with more resources. A directory-based sharding approach makes this possible — update the lookup table to route celebrity user IDs to beefier hardware.

**Sharded counters**: Instead of one `like_count` field that every like contends for, split it into N counter shards. Each like increments a random shard. Total count = sum of all shards. Eliminates write contention.

## The Alternatives Ladder

Before sharding, exhaust every simpler option. Each step solves a class of problems with less complexity than sharding:

```text
Level 1: Indexing
    └─ Query slow? Add the right index. Solves 80% of performance problems.

Level 2: Read Replicas
    └─ Read-heavy? Add replicas. Split reads to replicas, writes to primary.

Level 3: Partitioning (single machine)
    └─ Table too large? Use PostgreSQL PARTITION BY. Faster scans, easier maintenance.

Level 4: Caching
    └─ Hot data? Add Redis. Cache computed results, session data, rate limiting.

Level 5: Connection Pooling
    └─ Too many connections? Add PgBouncer. Multiplex client connections.

Level 6: Sharding
    └─ Exhausted everything above? Now consider sharding.

Level 7: NewSQL
    └─ Need sharding without the pain? Consider CockroachDB or Spanner.
```

## When NOT to Shard

| Signal                                    | Why You Shouldn't Shard                            |
| ----------------------------------------- | -------------------------------------------------- |
| Data fits on one machine                  | Sharding adds complexity with zero benefit         |
| Read replicas handle the load             | Reads are the bottleneck, not writes or storage    |
| Caching solves the hot path               | Redis is simpler than resharding your database     |
| You need complex cross-shard transactions | Distributed transactions across shards are painful |
| Your team doesn't have the expertise      | Operational complexity of sharding is significant  |
| A NewSQL database would work              | CockroachDB/Spanner auto-shard for you             |

> 💡 Interview Tip  
> The most impressive thing you can say in an interview about sharding is "I'd try these alternatives first." Walk through the ladder: "First I'd optimize indexes, add read replicas for read load, add caching for hot paths, and partition the largest tables. If we're still hitting limits — then I'd shard by user_id using hash-based sharding with consistent hashing." That shows you understand sharding is a last resort, not a first instinct.

---

# Chapter 18 Distributed Transactions — Sagas, Outbox, and 2PC

## The Dinner Table Problem

Two-Phase Commit is like getting everyone at a dinner table to agree on a restaurant before anyone moves. The coordinator asks each person: "Can you commit to Italian?" Everyone says yes. Then the coordinator says "Go." If even one person says no, nobody moves. It works -- but if the coordinator's phone dies mid-conversation, everyone is stuck standing in the hallway, unable to proceed.

Sagas are like everyone ordering their own meal separately, with a refund policy. If the appetizer arrives but the kitchen runs out of your entree, you get a refund for the appetizer. Each step is independent, and each has a plan for undoing itself.

## The Problem: One Operation, Many Databases

In a monolith, a single database transaction can atomically update the orders table, the inventory table, and the payments table. One COMMIT, everything is consistent.

In microservices, each service owns its own database:

![Text32](/assets/32.png)

There's no single transaction that spans all three databases. If the payment fails after inventory was already reserved, you need a way to undo the reservation. This is the distributed transaction problem.

## Two-Phase Commit (2PC)

The oldest solution. A coordinator manages the transaction across all participants.

**Phase 1: Prepare** :
The coordinator asks each participant: "Can you commit this?" Each participant acquires locks, validates the operation, and responds YES or NO.

**Phase 2: Commit (or Abort)** :If all participants said YES, the coordinator sends COMMIT. If any said NO, the coordinator sends ABORT and everyone rolls back.

![Text33](/assets/33.png)

### Why 2PC Is Problematic

## Problems with 2-Phase Commit (2PC)

| Problem                 | Explanation                                                                                                                                   |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| Blocking                | If the coordinator crashes after PREPARE but before COMMIT, all participants hold locks indefinitely, waiting for a decision that never comes |
| Latency                 | Two network round-trips minimum. Cross-region, this adds 100–400ms                                                                            |
| Tight coupling          | All participants must implement the 2PC protocol. Adding a new service means integrating it into the coordination layer                       |
| Single point of failure | The coordinator is a bottleneck. If it dies, the entire transaction is in limbo                                                               |

2PC is still used within a single database (that's how PostgreSQL handles multi-table transactions internally). But across services over a network, it's generally avoided.

![Txet34](/assets/34.png)

## The Saga Pattern

A Saga breaks a distributed operation into a sequence of local transactions. Each local transaction updates one service's database and triggers the next step. If any step fails, previously completed steps are undone by compensating transactions.

### E-Commerce Order Example

```text
Happy path:
  1. Order Service    → Create order (status: PENDING)
  2. Inventory Service → Reserve items
  3. Payment Service   → Charge credit card
  4. Order Service    → Update order (status: CONFIRMED)
  5. Shipping Service  → Schedule shipment

Failure path (payment fails at step 3):
  3. Payment Service   → Charge fails
  2. Inventory Service → COMPENSATE: Release reservation
  1. Order Service    → COMPENSATE: Cancel order (status: CANCELLED)
```

Each step is a local ACID transaction within a single service. The Saga coordinates the sequence. The key insight: instead of one big atomic operation, you get a series of small atomic operations with explicit undo logic.

#### Choreography vs Orchestration

There are two ways to coordinate a Saga:

**Choreography**: Each service publishes events, and other services react. No central coordinator.

![Text35](/assets/35.png)

- **Pros**: Simple for small flows. No single point of failure. Services are loosely coupled.

- **Cons**: Hard to track the overall flow. Debugging is painful -- you're chasing events across 5 services. Adding a step means modifying multiple services.

**Orchestration**: A central Saga orchestrator directs each step explicitly.
![Text36](/assets/36.png)

- **Pros**: Easy to understand the flow. Easy to add or reorder steps. Centralized error handling and retry logic.

- **Cons**: The orchestrator is a single point of failure (mitigated by making it stateful and persistent). Can become a bottleneck.

In practice, most teams at scale use orchestration. Netflix built Conductor, Uber built Cadence (now Temporal) -- both are Saga orchestration frameworks.

## The Dual-Write Problem

It occurs when a service needs to update two different systems as part of a single business operation, but there is no way to make those updates happen atomically. A common example is an Order Service that, after creating an order, must both save the order in its database and publish an OrderCreated event to Kafka so that other services, such as Inventory or Shipping, can react. Since the database and Kafka are separate systems, a single ACID transaction cannot cover both operations.

## The Outbox Pattern

The Outbox pattern solves dual-writes by using the database itself as the message queue.

Instead of writing the order to the database and immediately publishing an event to Kafka, the application writes the order and an event record into a special **outbox table** in one atomic transaction.

For example, when a customer places an order, the Order Service inserts a row into the orders table and also inserts an `OrderCreated` event into the `outbox` table. Because both inserts happen within the same database transaction, they either both succeed or both fail. If the transaction commits, the order exists and the event is safely stored. If the transaction rolls back, neither the order nor the event exists. This eliminates the possibility of having an order without an event or an event without an order.

After the transaction commits, a separate component called the **outbox relay** is responsible for publishing events from the outbox table to Kafka. The relay reads unprocessed rows, publishes the events, and then marks them as processed . If Kafka is temporarily unavailable or the relay crashes, nothing is lost because the event remains stored in the database. The relay can simply retry later. This gives the system reliable event delivery without requiring a distributed transaction between the database and Kafka.

There are two common ways to implement the outbox relay. The simpler approach is **polling** . where a background job periodically queries the outbox table for new events. This is easy to implement but introduces some delay because events are only published during the next polling cycle, and it creates additional database load. The more scalable approach is **Change Data Capture (CDC)**.

> 💡 Interview Tip  
> If asked "How would you handle a transaction that spans multiple microservices?", don't jump straight to 2PC. Explain that 2PC is blocking and fragile across services, then propose the Saga pattern. Mention choreography vs orchestration, lean toward orchestration for complex flows, and bring up the outbox pattern for reliable event publishing. If you mention Temporal or a similar framework, you'll signal that you know how production systems actually solve this.

## Idempotency Keys — Making Retries Safe

Distributed transactions often require retries — networks fail, services timeout, messages get duplicated. But retrying a payment charge without protection might charge the customer twice.

Idempotency keys solve this. The client generates a unique key for each operation and includes it with every request and retry.

---

# Chapter 19 CQRS, Event Sourcing, and Materialized Views

## The Problem: Reads and Writes Want Different Things

Imagine a banking application. For **writes**, you need strict ACID transactions — "transfer $100 from account A to account B" must be atomic, consistent, and isolated. A normalized schema with proper constraints is essential.

For **reads**, the same bank needs dashboards showing "total transactions this month," "average balance by region," "suspicious activity patterns." These queries need denormalized data with pre-computed aggregates — the opposite of what writes want.

Trying to serve both through the same data model is like using the same vehicle for Formula 1 racing and cross-country hauling. You end up with a compromised design that's mediocre at both.

## From Denormalization to CQRS — A Spectrum

If you've read the Schema Design chapter, you already know the basics of this pattern. we denormalized a `like_count` onto the posts table to avoid a COUNT(\*) join on every read. That was a simple, manual optimization: one denormalized field, updated atomically on writes.

CQRS is the same idea taken to its logical conclusion. Instead of sprinkling denormalized fields across your existing tables, you build an entirely separate read model — purpose-built for your query patterns — alongside the write model. Think of it as a progression:

```text
Level 1: Denormalized column (like_count on posts)        ← Chapter 4
Level 2: Materialized view (pre-computed query result)     ← This lesson
Level 3: Separate read database (Elasticsearch, Redis)     ← Full CQRS
Level 4: Event-sourced writes + derived read models        ← CQRS + Event Sourcing
```

Each level adds complexity but solves a wider gap between read and write requirements.

![Text37](/assets/37.png)

## CQRS — Command Query Responsibility Segregation

CQRS splits your system into two sides:

**Command side (writes)**: Handles mutations. Normalized, optimized for consistency. Uses ACID transactions.

**Query side (reads)**: Handles reads. Denormalized, optimized for specific query patterns. Can use different storage entirely.

![text38](/assets/38.png)

The write model publishes events when data changes. The read model consumes those events and updates its denormalized views. The two models are eventually consistent — there's a small delay between a write and its appearance in the read model.

### When CQRS Is Worth It

| Signal                                               | Why CQRS Helps                               |
| ---------------------------------------------------- | -------------------------------------------- |
| Read/write ratio is wildly skewed (1000:1)           | Scale reads independently of writes          |
| Read and write schemas are very different            | Each model is optimized for its purpose      |
| Need to serve different read patterns from same data | Multiple read models for different consumers |
| Need an audit trail                                  | Events provide a natural changelog           |

### When CQRS Is Overkill

For a simple CRUD application where reads and writes use the same schema, CQRS adds complexity with no benefit. A single PostgreSQL database with read replicas serves most applications perfectly.

## Event Sourcing — Store Every Change

### Current State vs Event History

**Traditional approach**: Store current state. When a user updates their name from "Alice" to "Alicia," you overwrite the row. The old value is gone.

**Event sourcing**: Store every change as an immutable event. Never overwrite.

```text
Event Store:
┌─────┬────────────────┬──────────────────────┬────────────┐
│ seq │ event_type     │ payload              │ timestamp  │
├─────┼────────────────┼──────────────────────┼────────────┤
│  1  │ AccountCreated │ {id: 1, name: Alice} │ 2024-01-15 │
│  2  │ NameChanged    │ {id: 1, name: Alicia}│ 2024-03-20 │
│  3  │ EmailChanged   │ {id: 1, email: new}  │ 2024-03-22 │
│  4  │ AccountDeleted │ {id: 1}              │ 2024-06-01 │
└─────┴────────────────┴──────────────────────┴────────────┘

Current state of account 1 = replay events 1→2→3→4 = deleted
```

### Why This Matters

- **Complete audit trail**: Every change is recorded. Perfect for financial systems, compliance, and debugging ("what happened to this user's account between March 15 and March 20?").

- **Time travel**: Rebuild the state of any entity at any point in time by replaying events up to that timestamp.

- **Rebuild read models**: If you realize your read model needs a different structure, replay all events to build a new one from scratch. No data migration needed.

- **Decoupled systems**: Multiple downstream services can consume the same event stream and build their own views of the data.

### The Costs

- **Complexity**: Simple CRUD becomes event design.

- **Storage**: Events accumulate forever. Snapshotting (periodically saving current state) helps — replay from the latest snapshot, not from the beginning of time.

- **Eventual consistency**: Read models lag behind the event stream. A user changes their name and might see the old name for a second.

- **Event versioning**: When the schema of an event changes, old events in the store still have the old format. You need upcasting or versioned event handlers.

## Materialized Views — Pre-Computed Query Results

A **materialized view** is a query result stored as a table. Instead of computing the result on every read, you compute it once and serve the pre-built result.

### PostgreSQL Materialized Views

```sql
-- Create a materialized view of daily order totals
CREATE MATERIALIZED VIEW daily_order_totals AS
SELECT
    DATE(created_at) AS order_date,
    COUNT(*) AS total_orders,
    SUM(amount) AS total_revenue
FROM orders
GROUP BY DATE(created_at);

-- Refresh when data changes (or on a schedule)
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_order_totals;
```

The view is a real table on disk. Queries against it are fast — no aggregation at query time.

**CONCURRENTLY** refreshes the view without locking reads. Readers see the old data until the refresh completes, then atomically see the new data.

### Star Schema and Snowflake Schema

For analytics workloads, materialized views connect to a broader pattern: dimensional modeling.

Star Schema: A central fact table (measurements like order amounts, click counts) surrounded by dimension tables (descriptive data like products, dates, customers). Dimensions are denormalized.

```text
         [dim_product]
              |
[dim_date]──[fact_orders]──[dim_customer]
              |
         [dim_store]
```

**Snowflake Schema**: Like star, but dimensions are further normalized into sub-tables (product → category → department). Less redundancy, more joins.

**OLTP vs OLAP**: Your main application database (OLTP) is optimized for transactions. Analytics databases (OLAP) are optimized for aggregation queries. Star schema lives in the OLAP world — data warehouses like BigQuery, Redshift, Snowflake.

| Aspect     | OLTP                   | OLAP                          |
| ---------- | ---------------------- | ----------------------------- |
| Purpose    | Transactions           | Analytics                     |
| Operations | INSERT, UPDATE, DELETE | SELECT + aggregation          |
| Schema     | Normalized (3NF)       | Denormalized (Star/Snowflake) |
| Example    | PostgreSQL, MySQL      | BigQuery, Redshift            |

## Putting It All Together

CQRS, event sourcing, and materialized views are often combined:

![Text39](/assets/39.png)

Events are the glue: the write model emits events, and multiple read models consume them to build different views of the same data. Each consumer is independent and can use the storage technology that best fits its query pattern.

---

# Chapter 20 Schema Evolution and Zero-Downtime Migrations
## The Kitchen Renovation Analogy
Your application is serving live traffic. You need to change the schema without breaking anything. Every change must be backward-compatible with the currently running application code.

## The Expand-Contract Pattern
This is the fundamental pattern for zero-downtime schema changes. Three phases:

### Phase 1: Expand
Add new structures alongside old ones. Don't remove or rename anything.

```sql
-- Want to rename "name" to "display_name"?
-- Step 1: Add the new column (expand)
ALTER TABLE users ADD COLUMN display_name VARCHAR(100);
```
The old `name` column still exists. The old application code still works.

### Phase 2: Migrate
Update application code to write to both columns. Backfill historical data.

```sql
-- Backfill existing data
UPDATE users SET display_name = name WHERE display_name IS NULL;
```

### Phase 3: Contract
Once all code reads from the new column and all data is migrated, remove the old column.

```sql
-- Step 3: Drop the old column (contract)
ALTER TABLE users DROP COLUMN name;
```
This final step only happens after the new application code is deployed everywhere and verified.

## Safe vs Dangerous Operations
### Safe — Do These Without Worry
| Operation | Why It's Safe |
|-----------|---------------|
| Add a new table | No existing code references it |
| Add a nullable column | Existing rows get `NULL`, old code ignores it |
| Add an index `CONCURRENTLY` | PostgreSQL builds the index without locking the table |
| Add a new view | No impact on existing tables |
| Increase column size | `VARCHAR(50)` → `VARCHAR(100)` is backward-compatible |
### Dangerous — Handle With Care
| Operation | Risk | Safe Approach |
|-----------|------|---------------|
| Rename a column | All queries referencing old name break instantly | Expand-contract: add new column, copy data, update code, then drop old column |
| Add NOT NULL constraint | Old code inserting without the column fails | Add column as nullable first, backfill data, then add the constraint |
| Change column type | Existing data might not convert cleanly | Add a new column with the new type, migrate data, then switch code over |
| Drop a column | Any code still referencing it crashes | Verify zero references in code and queries before dropping |
| Rename a table | Same as renaming a column, but with broader impact | Create a new table, double-write, migrate traffic/data, then drop the old table |

## A Real Walkthrough: Adding a Required Column
Your users table needs a `timezone` column, and it should be NOT NULL with a default.

**Wrong way (causes downtime):**
```sql
-- This locks the table for the entire rewrite in older PostgreSQL versions
ALTER TABLE users ADD COLUMN timezone VARCHAR(50) NOT NULL DEFAULT 'UTC';
```

**Safe way (expand-contract):**
```sql
-- Step 1: Add nullable column (instant, no lock)
ALTER TABLE users ADD COLUMN timezone VARCHAR(50);

-- Step 2: Backfill in batches (not all at once!)
UPDATE users SET timezone = 'UTC' WHERE timezone IS NULL AND id BETWEEN 1 AND 100000;
UPDATE users SET timezone = 'UTC' WHERE timezone IS NULL AND id BETWEEN 100001 AND 200000;
-- ... continue in batches to avoid locking the table

-- Step 3: Update app code to always write timezone

-- Step 4: Add NOT NULL constraint after all rows have values
ALTER TABLE users ALTER COLUMN timezone SET NOT NULL;
ALTER TABLE users ALTER COLUMN timezone SET DEFAULT 'UTC';
```
Batching the backfill prevents long-running transactions that lock the table and block writes.


## Migration Tools
| Tool | Database | Approach |
|------|----------|----------|
| Flyway | Any (SQL-based) | Versioned SQL migration files. Developer-friendly. |
| Liquibase | Any (XML/YAML/SQL) | Enterprise-focused. Supports rollbacks. |
| gh-ost | MySQL | GitHub's online schema migration tool. Copies table, applies changes, and swaps atomically with zero locks. |
| pgroll | PostgreSQL | Schema migrations with automatic expand-contract support. |
| Alembic | PostgreSQL (Python) | SQLAlchemy's migration tool. Auto-generates schema diffs. |

**gh-ost** deserves special mention: it creates a shadow copy of the table, applies the schema change to the copy, replays any writes that happened during the copy via binlog, then atomically renames the tables. The original table is never locked.


## Soft Deletes vs Hard Deletes
### Soft Deletes
Instead of `DELETE FROM users WHERE id = 42`, set a flag:

```sql
UPDATE users SET is_deleted = true, deleted_at = NOW() WHERE id = 42;
```
**Pros:**

- Undo capability ("oops, restore that user")
- Audit trail (who was deleted and when)
- Referential integrity preserved (no cascading deletes)

**Cons:**

- Every query needs WHERE is_deleted = false — forget it once and you leak deleted data
- Indexes include deleted rows (bloat)
- Storage grows indefinitely
- Privacy concerns (GDPR "right to be forgotten" may require actual deletion)


### A Better Alternative: Audit Log Table
Instead of soft deletes on every table, maintain a separate audit log:

```sql
CREATE TABLE audit_log (
    id BIGINT PRIMARY KEY,
    table_name VARCHAR(50),
    record_id BIGINT,
    action VARCHAR(10),  -- INSERT, UPDATE, DELETE
    old_data JSONB,
    new_data JSONB,
    changed_by BIGINT,
    changed_at TIMESTAMP DEFAULT NOW()
);
```

Hard delete the original row (clean data, no `WHERE is_deleted` everywhere), but capture the full state in the audit log. Best of both worlds: clean operational data + complete history.

## Temporal Data — SCD Type 2
Slowly Changing Dimensions (SCD Type 2) tracks history with `valid_from` and `valid_to` timestamps:
```sql
CREATE TABLE product_prices (
    product_id BIGINT,
    price DECIMAL(10,2),
    valid_from TIMESTAMP NOT NULL,
    valid_to TIMESTAMP DEFAULT '9999-12-31',
    PRIMARY KEY (product_id, valid_from)
);
```

```text
product_prices
┌────────────┬────────┬────────────┬────────────┐
│ product_id │ price  │ valid_from │ valid_to   │
├────────────┼────────┼────────────┼────────────┤
│     1      │  9.99  │ 2024-01-01 │ 2024-06-30 │
│     1      │ 12.99  │ 2024-07-01 │ 9999-12-31 │  ← current
└────────────┴────────┴────────────┴────────────┘
```

Current price: `WHERE product_id = 1 AND valid_to = '9999-12-31'` Price on March 15: `WHERE product_id = 1 AND valid_from <= '2024-03-15' AND valid_to >= '2024-03-15'`

This pattern is critical for financial reporting, compliance, and any system where you need to answer "what was the value at time X?"

> 💡 Interview Tip  
> If you need to add a column or change a schema in a system design discussion, say "I'd use the expand-contract pattern — add the new column alongside the old one, backfill data, update the application code, then drop the old column. This avoids downtime." That's a senior-level answer.


---