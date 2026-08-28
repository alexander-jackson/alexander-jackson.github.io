---
title: "Helping The Query Planner"
date: 2026-08-26T16:31:32+01:00
draft: true
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
quite a lot of data in it. We want to find the minimum and maximum `id` values
between two values of `created_at`, let's say the 1st and 3rd of July 2026.

Our query might look something like this:

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

## In practice

Unfortunately, reality doesn't always play out as you might expect. It turns
out, PostgreSQL prefers the index `pk_transfer` and instead searches the table
in order of `id` value until it finds something in this range.

If the search window is recent, this means it will essentially perform a
sequential scan on all the rows of the table up until the lower bound of the
range.

Instead, we can rewrite the query to encourage usage of our index and try to
better convey what we are trying to find.

In the following query, we use a common table expression (CTE) to first find
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
a smaller set of rows, then performs the `MIN` operation on those.
