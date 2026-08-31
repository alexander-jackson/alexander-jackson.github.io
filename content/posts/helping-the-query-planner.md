---
title: "Helping The Query Planner"
date: 2026-08-31T12:49:00+01:00
draft: false
showtoc: true
---

PostgreSQL's query planner is generally very smart, and you can usually trust
it to optimise your queries as you might expect.

However sometimes you have more information than it does, which can allow you
to encourage it towards more optimal executions.

### Sample table

Let's imagine we have the following table:

```sql
CREATE TABLE transfer (
    id BIGINT GENERATED ALWAYS AS IDENTITY,
    transfer_uid UUID NOT NULL,
    amount DECIMAL(19, 2) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL,

    CONSTRAINT pk_transfer PRIMARY KEY (id),
    CONSTRAINT uk_transfer_transfer_uid UNIQUE (transfer_uid)
);
```

It stores transfers that have been made in a financial system, and thus has
quite a lot of data in it. To simulate this, we'll insert 5m rows of data into
the table with random `created_at` timestamps within the last 5 years:

```sql
WITH params AS (
    SELECT
        NOW() - INTERVAL '5 years' AS start_ts,
        INTERVAL '5 years' / 5000000 AS step
)
INSERT INTO transfer (transfer_uid, amount, created_at)
SELECT
    gen_random_uuid(),
    round((random() * 10000)::numeric, 2),
    params.start_ts + (i * params.step) + (random() * params.step)
FROM generate_series(1, 5000000) AS i, params
```

We want to find the minimum and maximum `id` values between two values of
`created_at`, let's say the 1st and 3rd of July 2026. Our query might look
something like this:

```sql
SELECT MIN(id)
FROM transfer
WHERE created_at BETWEEN '2026-07-01' AND '2026-07-03'
```

Given that we're filtering on the `created_at` column, we might decide we want
an index on it to ensure this can be searched efficiently:

```sql
CREATE INDEX idx_transfer_created_at
ON transfer (created_at)
```

In theory, this should allow the database to quickly find the rows that it is
interested in, and then find the minimum `id` value for us.

### In practice

Unfortunately, reality doesn't always play out as you might expect. It turns
out, PostgreSQL prefers the index `pk_transfer` and instead searches the table
in order of `id` value until it finds something in this range.

This is because the planner has a special path for `MIN` and `MAX` operations
which rewrites the query to be similar to the following:

```sql
SELECT id
FROM transfer
WHERE created_at BETWEEN '2026-07-01' AND '2026-07-03'
ORDER BY id
LIMIT 1
```

If the search window is recent, this means it will essentially perform a
sequential scan on all the rows of the table up until the lower bound of the
range.

We can see this from the output of `EXPLAIN ANALYZE`:

```sql
 Result (actual time=489.282..489.282 rows=1 loops=1)
   InitPlan 1
     ->  Limit (actual time=489.277..489.277 rows=1 loops=1)
           ->  Index Scan using pk_transfer on transfer (actual time=489.275..489.275 rows=1 loops=1)
                 Filter: ((created_at >= '2026-07-01 00:00:00+00'::timestamp with time zone) AND (created_at <= '2026-07-03 00:00:00+00'::timestamp with time zone))
                 Rows Removed by Filter: 4901569
 Planning Time: 1.249 ms
 Execution Time: 489.331 ms
(8 rows)
```

We can see that it performed an `Index Scan using pk_transfer` on the
`transfer` table, and then applied a filter for the `created_at` value to it.
This filter removed 4.9m rows, implying it had to scan all those rows and
remove them before it found enough data in the range to satisfy the query.

Instead, we can rewrite the query to steer the planner toward our index. With
the following approach, we use a common table expression (CTE) to first find
all the rows in the time range and then another query to filter for the minimum
`id` value:

```sql
WITH identifiers AS (
    SELECT id
    FROM transfer
    WHERE created_at BETWEEN '2026-07-01' AND '2026-07-03'
)
SELECT MIN(id)
FROM identifiers
```

With this approach, it uses our `idx_transfer_created_at` index instead to find
a smaller set of rows, then performs the `MIN` operation on those. The output
of `EXPLAIN ANALYZE` is a lot more healthy here:

```sql
 Aggregate (actual time=1.488..1.489 rows=1 loops=1)
   CTE identifiers
     ->  Index Scan using idx_transfer_created_at on transfer (actual time=0.014..0.677 rows=5555 loops=1)
           Index Cond: ((created_at >= '2026-07-01 00:00:00+00'::timestamp with time zone) AND (created_at <= '2026-07-03 00:00:00+00'::timestamp with time zone))
   ->  CTE Scan on identifiers (actual time=0.015..1.279 rows=5555 loops=1)
 Planning Time: 0.095 ms
 Execution Time: 1.522 ms
(7 rows)
```

Not only is the query ~300x faster than before, we can see that it's doing a
lot less work. The `Index Scan using idx_transfer_created_at` took ~0.7ms to
complete and resulted in 5555 rows, after which it just needed to do a `CTE
Scan on identifiers` (a quick sequential scan) to find the minimum `id` value.

Because we knew more about the structure of the table and the query we were
making, this allowed us to guide the planner into making a more optimal
decision by rewriting our query slightly.
