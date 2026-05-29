---
title: "How Indexes Work"
date: 2026-05-21T20:57:36+01:00
draft: true
showtoc: true
---

Whenever you have a slow performing database query, the immediate suggestion is
always to add an index on it. They can take statements which used to take
minutes just milliseconds to complete instead.

How does an index work though? What allows them to provide these performance
benefits?

## Storing people

We'll start by considering a basic table to store people, with no indexes on
it:

```sql
CREATE TABLE person (
    id BIGINT GENERATED ALWAYS AS IDENTITY,
    name TEXT NOT NULL,

    CONSTRAINT pk_person PRIMARY KEY (id)
);
```

If we imagine some data in our table, it might look as follows:

```sql
> INSERT INTO person (name)
  VALUES
    ('Alex'),
    ('James'),
    ('April'),
    ('Matthew'),
    ('Lucy');

> SELECT * FROM person;
 id |  name
----+---------
  1 | Alex
  2 | James
  3 | April
  4 | Matthew
  5 | Lucy
(5 rows)
```

## Searching for some data

Now, we want to find the rows where the name column is equal to `Matthew`. We
would just query that as follows:

```sql
> SELECT id
  FROM person
  WHERE name = 'Matthew';
 id
----
  4
(1 row)
```

It's worth trying to build a mental model of what the database is doing here.
The most naive approach (and what will happen in this case) is to look at each
row and check whether the name is equal to `Matthew.`

We can also ask PostgreSQL how it executes this by requesting a query plan:

```sql
> EXPLAIN (COSTS FALSE) SELECT id
  FROM person
  WHERE name = 'Matthew';
             QUERY PLAN
------------------------------------
 Seq Scan on person
   Filter: (name = 'Matthew'::text)
(2 rows)
```

Here, we're asking the planner to explain what it wants to do (and hiding the
cost outputs, since they're not entirely relevant in this case).

As we can see, the database thinks that the best option it has is to
sequentially scan the table (reading each row in order) and filtering for the
ones where the `name` column is equal to `Matthew`, just as we expected.

This will require the database to look at all 5 of the rows. While this doesn't
seem like much for a small example, if we had millions of people in the
database this might take a while!

## Adding an index

We'll start by making a big assumption, that we'll never have 2 people with the
same name. As unlikely as this is in practice, it'll make this example a bit
simpler by allowing us to use a unique index.

Index creation just requires the name of the index, the name of the table and
the columns we want to create it on:

```sql
> CREATE UNIQUE INDEX uk_person_name
  ON person (name);
CREATE INDEX
```

As we can see, the database now uses our index to satisfy that query:

```sql
> EXPLAIN (COSTS FALSE) SELECT id
  FROM person
  WHERE name = 'Matthew';
             QUERY PLAN
------------------------------------
 Seq Scan on person
   Filter: (name = 'Matthew'::text)
(2 rows)
```

Oh. No it doesn't. Since there's so little data in the table, PostgreSQL still
decides that it's faster to just sequentially scan all the rows.

Indexes aren't always appropriate until you have a certain amount of data in
the table, but having one there won't make your queries slower, it just adds a
little bit of extra work when inserting data.

If we increase the [number of
names](https://github.com/danielmiessler/SecLists/blob/master/Usernames/Names/names.txt)
in the table:

```sql
> SELECT COUNT(*) FROM person;
 count
-------
 10734
(1 row)
```

We can see that the database decides that it is worth using the index now:

```sql
> EXPLAIN (COSTS FALSE) SELECT id
  FROM person
  WHERE name = 'Matthew';
                QUERY PLAN
-------------------------------------------
 Index Scan using uk_person_name on person
   Index Cond: (name = 'Matthew'::text)
(2 rows)
```

We can check whether this has improved the performance of our query by using the `EXPLAIN ANALYZE` command, which will execute the query and provide timings:

```sql
> EXPLAIN (
    ANALYZE TRUE,
    COSTS FALSE,
    TIMING FALSE,
    BUFFERS FALSE
  )
  SELECT id
  FROM person
  WHERE name = 'Matthew';
                              QUERY PLAN
----------------------------------------------------------------------
 Index Scan using uk_person_name on person (actual rows=1.00 loops=1)
   Index Cond: (name = 'Matthew'::text)
   Index Searches: 1
 Planning Time: 0.189 ms
 Execution Time: 0.112 ms
(5 rows)

> DROP INDEX uk_person_name;

> EXPLAIN (
    ANALYZE TRUE,
    COSTS FALSE,
    TIMING FALSE,
    BUFFERS FALSE
  )
  SELECT id
  FROM person
  WHERE name = 'Matthew';
                  QUERY PLAN
-----------------------------------------------
 Seq Scan on person (actual rows=1.00 loops=1)
   Filter: (name = 'Matthew'::text)
   Rows Removed by Filter: 10733
 Planning Time: 0.168 ms
 Execution Time: 1.174 ms
(5 rows)
```

As we can see, the query with the index is around 10 times faster than the one
without, even with a relatively small amount of data in the table. This
difference will only increases as the size of the table grows, which is why
indexes are so important for performance.
