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

```
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
