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
| Level              | Dirty Reads | Lost Updates      | Read Skew | Write Skew | Performance |
|--------------------|-------------|-------------------|------------|-------------|-------------|
| Read Uncommitted   | Possible    | Possible          | Possible   | Possible    | Fastest     |
| Read Committed     | Prevented   | Possible          | Possible   | Possible    | Fast        |
| Repeatable Read    | Prevented   | Mostly prevented  | Prevented  | Possible    | Medium      |
| Serializable       | Prevented   | Prevented         | Prevented  | Prevented   | Slowest     |

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

----

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