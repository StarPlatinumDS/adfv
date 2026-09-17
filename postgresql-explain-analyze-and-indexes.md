# PostgreSQL `EXPLAIN ANALYZE` and Indexes

## Goal

The purpose of this exercise was not to become a PostgreSQL performance expert.

The goal was to:

- use `EXPLAIN ANALYZE` for the first time,
- understand what a PostgreSQL query plan roughly represents,
- see what happens when a query has no useful index,
- add an index,
- observe PostgreSQL choosing a different execution plan,
- compare the amount of work before and after indexing.

This was enough for the current roadmap stage. Deeper PostgreSQL work can be revisited later when the roadmap reaches transactions, locking, deadlocks, or more serious performance work.

---

## Environment

PostgreSQL was kept inside Docker.

The Go application and pgAdmin connected to the exposed PostgreSQL port on the host.

Conceptually:

```text
Go / pgAdmin
    |
localhost:5432
    |
Docker port mapping
    |
PostgreSQL container
```

For pgAdmin, the connection used:

```text
Host: localhost
Port: 5432
Database: explain_lab
User: postgres
Password: postgres
```

The Go application used `pgxpool` and verified the connection with `Ping()`.

---

## Test Table

The experiment used a simple `orders` table:

```sql
CREATE TABLE orders (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id BIGINT NOT NULL,
    status TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    amount NUMERIC(10, 2) NOT NULL
);
```

About 200,000 fake rows were generated with PostgreSQL's `generate_series()` and `random()` functions.

After populating the table:

```sql
ANALYZE orders;
```

was run so PostgreSQL could update statistics used by the query planner.

The `PRIMARY KEY` on `id` automatically created an index for `id`.

No useful index initially existed for `user_id` or `created_at`.

---

# The Query

The query used throughout the experiment was:

```sql
SELECT *
FROM orders
WHERE user_id = 42
ORDER BY created_at DESC
LIMIT 20;
```

The query has two important requirements:

1. Find rows belonging to one user.
2. Return that user's newest orders first.

---

# What `EXPLAIN ANALYZE` Does

The query was wrapped in:

```sql
EXPLAIN ANALYZE
SELECT ...
```

`EXPLAIN ANALYZE` does two useful things:

- shows the execution plan PostgreSQL selected,
- actually runs the query and reports real execution statistics.

A query plan is a tree.

For understanding execution flow, it is often easiest to read the important operations from the bottom upward.

Example:

```text
Limit
└── Sort
    └── Seq Scan
```

means approximately:

```text
scan rows
→ sort them
→ return only the requested number
```

---

# Baseline — No Useful Index

Initial query plan:

```text
Limit
└── Gather Merge
    └── Sort
        └── Parallel Seq Scan on orders
```

Execution time:

```text
12.023 ms
```

Important part:

```text
Parallel Seq Scan on orders
Filter: (user_id = 42)
Rows Removed by Filter: 99980
loops=2
```

There was one additional worker, so the table scan happened in parallel.

Because there was no index for `user_id`, PostgreSQL effectively had to inspect almost the entire table to find the matching rows.

Conceptually:

```text
row 1      -> user_id == 42?
row 2      -> user_id == 42?
row 3      -> user_id == 42?
...
row 200000 -> user_id == 42?
```

The scan produced about 41 matching rows in total.

Those rows then had to be sorted by:

```sql
created_at DESC
```

before PostgreSQL could return the first 20.

Approximate flow:

```text
~200,000 rows
      |
      | scan/filter
      v
~41 matching rows
      |
      | sort by created_at DESC
      v
~41 sorted rows
      |
      | LIMIT 20
      v
20 returned rows
```

---

# Important Query Plan Terms

## Sequential Scan

A sequential scan means PostgreSQL reads through table rows looking for matches.

Without a useful index, this may require checking most or all of the table.

---

## Parallel Sequential Scan

PostgreSQL may divide a table scan between multiple execution processes when it estimates that parallel work is worthwhile.

The original plan had:

```text
Workers Planned: 1
Workers Launched: 1
```

The existence of parallel execution did not change the fundamental issue:

> PostgreSQL still had to inspect nearly the entire table.

---

## Sort

The original rows were not already available in the requested:

```sql
ORDER BY created_at DESC
```

order.

PostgreSQL therefore sorted the matching rows after finding them.

---

## Gather Merge

Parallel workers can produce separate sorted result streams.

`Gather Merge` combines those streams while preserving the required ordering.

---

## Planning Time vs Execution Time

The plan reports both:

```text
Planning Time
Execution Time
```

Planning time is how long PostgreSQL spent deciding how to execute the query.

Execution time is how long it spent actually running the selected plan.

---

## Planner `cost`

Values such as:

```text
cost=4208.15..4210.43
```

are **not milliseconds**.

They are planner cost estimates used internally to compare possible plans.

---

## Buffers

The original plan showed:

```text
Buffers: shared hit=1775
```

A shared-buffer `hit` means PostgreSQL found the requested database page in its shared memory cache.

This means execution time alone should not be treated as a perfect benchmark.

The more useful question is often:

> How much work did the new plan avoid?

---

# First Index — `user_id`

The first index created was:

```sql
CREATE INDEX idx_orders_user_id
ON orders(user_id);
```

An index is a separate data structure maintained by PostgreSQL.

Very simplified:

```text
user_id
   1  -> row locations
   2  -> row locations
   ...
  42  -> row locations
```

Instead of inspecting every row to discover where `user_id = 42` exists, PostgreSQL can use the index to locate matching rows.

Indexes improve some reads, but they also:

- consume storage,
- must be maintained during `INSERT`,
- may require work during `UPDATE`,
- may require work during `DELETE`.

Therefore the goal is not to index every column.

PostgreSQL's normal/default index type is a B-tree.

---

# Plan After `user_id` Index

The same query was run again without changing it.

New plan:

```text
Limit
└── Sort
    └── Bitmap Heap Scan on orders
        └── Bitmap Index Scan on idx_orders_user_id
```

Execution time:

```text
0.267 ms
```

The important difference:

```text
Parallel Seq Scan
```

disappeared.

In its place PostgreSQL used:

```text
Bitmap Index Scan on idx_orders_user_id
Index Cond: (user_id = 42)
```

The index found about 41 matching row locations.

Then:

```text
Bitmap Heap Scan on orders
```

loaded the corresponding complete table rows.

Conceptually:

```text
index
  |
  | find user_id = 42
  v
matching row locations
  |
  v
orders table
  |
  | fetch actual rows
  v
~41 rows
```

The index did not contain everything required by `SELECT *`, so PostgreSQL still needed to fetch the actual table rows.

---

# Why "Bitmap"?

Very roughly, PostgreSQL can first collect the relevant table locations from the index and then visit the required table pages efficiently.

Conceptually:

```text
index matches
   |
   v
bitmap / collection of useful locations
   |
   v
visit the relevant table pages
```

The exact planner rules were intentionally not explored further in this exercise.

---

# First Major Result

Before:

```text
Parallel Seq Scan
~200,000 rows effectively examined
Execution Time: 12.023 ms
```

After indexing `user_id`:

```text
Bitmap Index Scan
~41 matching rows located directly
Execution Time: 0.267 ms
```

The exact execution-time ratio should not be treated as a universal benchmark, because timings vary and caching matters.

The important result was structural:

> PostgreSQL stopped scanning almost the entire table.

---

# PostgreSQL Does Not Cache Arbitrary Query Results

A possible explanation for the speedup was that the previous query result might have been cached.

The important correction:

> PostgreSQL commonly caches database pages, but it does not simply cache arbitrary `SELECT` results and return them unchanged.

Caching affected both runs.

The index was the primary reason the execution strategy changed so dramatically.

---

# Remaining Problem — Sorting

After adding:

```sql
INDEX ON user_id
```

the plan still contained:

```text
Sort
Sort Key: created_at DESC
```

The index solved:

```sql
WHERE user_id = 42
```

but it did not encode the requested ordering by:

```sql
created_at DESC
```

So PostgreSQL still had to:

```text
find ~41 matching rows
→ fetch them
→ sort by created_at
→ return 20
```

---

# Composite Index

The next index used more than one column:

```sql
CREATE INDEX idx_orders_user_created
ON orders(user_id, created_at DESC);
```

This is a **composite index**.

Its key is effectively:

```text
(user_id, created_at)
```

The ordering is hierarchical:

```text
first by user_id
then, within each user_id, by created_at DESC
```

Conceptually:

```text
user 1, newest order
user 1, older order
user 1, older order
...

user 2, newest order
user 2, older order
...

user 42, newest order
user 42, older order
user 42, older order
...
```

The order of the columns matters.

The query is:

```sql
WHERE user_id = 42
ORDER BY created_at DESC
LIMIT 20
```

So placing `user_id` first matches the equality filter, and ordering the second key by `created_at DESC` matches the requested order inside that user's range.

`user_id` itself is **not unique**.

Many orders can belong to the same user.

The unique column in this schema is `id`.

---

# Plan After Composite Index

The same query was run again.

New plan:

```text
Limit
└── Index Scan using idx_orders_user_created on orders
```

Execution time:

```text
0.208 ms
```

The important part was not the small timing change from `0.267 ms` to `0.208 ms`.

At sub-millisecond times, normal variation can easily affect measurements.

The important structural change was:

```text
Sort
```

disappeared completely.

PostgreSQL could now do approximately:

```text
find user_id = 42 in composite index
↓
walk rows already ordered newest → oldest
↓
fetch first 20
↓
stop
```

The `LIMIT 20` became particularly useful because PostgreSQL did not need to collect every order for that user and sort them before deciding which 20 to return.

---

# Three Stages Observed

## 1. No useful index

```text
Parallel Seq Scan
→ filter
→ sort
→ limit
```

Execution time observed:

```text
12.023 ms
```

---

## 2. Index on `user_id`

```text
Bitmap Index Scan
→ Bitmap Heap Scan
→ sort
→ limit
```

Execution time observed:

```text
0.267 ms
```

This solved the expensive row search.

---

## 3. Composite index on `(user_id, created_at DESC)`

```text
Index Scan
→ limit
```

Execution time observed:

```text
0.208 ms
```

This additionally removed the explicit sorting step and allowed PostgreSQL to stop after the first 20 matching index entries.

---

# Indexes Are Separate Database Objects

An index is not a new table column.

Creating:

```sql
CREATE INDEX idx_orders_user_created
ON orders(user_id, created_at DESC);
```

does not modify the logical row structure:

```text
id
user_id
status
created_at
amount
```

It creates an additional data structure maintained by PostgreSQL.

---

# PostgreSQL Chooses Whether to Use an Index

Creating an index does not force PostgreSQL to use it.

The query planner compares possible execution strategies and selects the one it estimates will be cheaper.

Therefore:

```text
index exists
```

does **not** imply:

```text
index will always be used
```

This is one reason `EXPLAIN ANALYZE` is useful.

It shows what PostgreSQL actually chose.

---

# Main Lessons

## 1. An index is a lookup structure

Without one, PostgreSQL may need to inspect a large part of the table.

With one, PostgreSQL may be able to locate matching rows directly.

---

## 2. Query shape matters

For:

```sql
WHERE user_id = 42
ORDER BY created_at DESC
```

an index beginning with:

```text
user_id
```

is naturally useful because that is how the query first narrows the data.

---

## 3. Composite indexes can solve more than one part of a query

The composite index:

```text
(user_id, created_at DESC)
```

helped both:

```text
WHERE user_id = 42
```

and:

```text
ORDER BY created_at DESC
```

---

## 4. Index column order matters

These are not equivalent:

```text
(user_id, created_at)
```

```text
(created_at, user_id)
```

The leading column determines the primary organization of the index.

---

## 5. Execution time is not the only useful measurement

A more useful question is often:

> What work disappeared?

In this experiment:

```text
full/parallel table scan
```

disappeared after the first index.

Then:

```text
explicit sort
```

disappeared after the composite index.

---

## 6. Defaults can be perfectly adequate

The user has used PostgreSQL with Go for a long time without needing explicit performance tuning because previous applications did not involve particularly large datasets or expensive queries.

That is a valid engineering outcome.

Optimization is useful when there is an actual reason for it.

This exercise was about learning how to investigate database behavior when that reason eventually appears, not about adding indexes and tuning queries everywhere by default.

---

# Current Learning Result

The user now has introductory practical experience with:

- PostgreSQL `EXPLAIN ANALYZE`
- query-plan trees
- sequential scans
- parallel sequential scans
- index scans
- bitmap index scans
- bitmap heap scans
- sorts
- gather merge
- planner cost vs execution time
- planning time vs execution time
- shared buffer hits
- B-tree indexes at a conceptual level
- single-column indexes
- composite indexes
- index column ordering
- query planner choice
- measuring structural improvements rather than only timing changes

This is enough for the current assignment.

The deeper internals of PostgreSQL indexing and query optimization are intentionally left for later.

---

# Result

Assignment status:

**Completed.**

The central progression was:

```text
no index
↓
observe expensive scan
↓
add user_id index
↓
observe index-based lookup
↓
add composite index
↓
observe sort disappear
```

This demonstrated directly that indexes can change not only execution time, but the entire strategy PostgreSQL uses to answer a query.

PostgreSQL will be revisited later in the roadmap for deeper database-engineering topics such as transactions, locking, isolation, and deadlocks.
