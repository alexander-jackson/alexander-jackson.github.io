---
title: "Write Skew Anomalies"
date: 2026-03-07T17:46:04Z
showtoc: true
---

Ever since I started working with PostgreSQL back in my second year of
university, I've developed some assumptions about the way that it works.

Every so often, one of those is proven wrong and I have to start re-thinking
how the layer storing all my data actually operates.

## The assumption

A core assumption I've made over the years is that once a transaction starts,
it essentially sees a snapshot of the database at that time. This means that it
doesn't matter what changes are going on at the same time or how long it takes,
you get a consistent view of the data.

## Where it works

Imagine a schema which stores account information:

```sql
CREATE TABLE account (
    id BIGINT GENERATED ALWAYS AS IDENTITY,
    name TEXT NOT NULL,

    CONSTRAINT pk_account PRIMARY KEY (id),
    CONSTRAINT uk_account_name UNIQUE (name)
);
```

If we start two transactions and make changes in one of them without
committing, the other won't see the changes yet:

```bash
first> BEGIN;

second> BEGIN;

second> INSERT INTO account (name) VALUES ('Alex');

first> SELECT * FROM account;
 id | name
----+------
(0 rows)
```

This makes a lot of sense, the `second` transaction might end up rolling back
its changes so it doesn't make sense for `first` to see them as they're not
guaranteed to be consistent with the rest of the database.

## Where it breaks down

My presumption was that this continued to hold even if `second` committed the
transaction, since `first` started before the changes were written and thus it
wouldn't have that data in its snapshot.

However, that's not the case:

```bash
first> BEGIN;

second> BEGIN;
second> INSERT INTO account (name) VALUES ('Alex');
second> COMMIT;

first> SELECT * FROM account;
 id | name
----+------
  1 | Alex
(1 row)
```

This is because PostgreSQL [defaults to][transaction-isolation-default] the
read committed isolation level, which means that queries (not transactions)
only see data committed before the query itself began.

The result of this is that you can run a query twice in succession and get two
different results if another transaction has been committed in the meantime.

## Repeatable reads

PostgreSQL also implements a [repeatable read][repeatable-read-isolation]
isolation level that does provide the guarantee that transactions only see data
committed before they started, which resolves this issue.

Due to the extra work needed to be done by the database however, this typically
results in lower performance as certain guarantees have to be upheld.

[transaction-isolation-default]: https://www.postgresql.org/docs/current/transaction-iso.html#XACT-READ-COMMITTED
[repeatable-read-isolation]: https://www.postgresql.org/docs/current/transaction-iso.html#XACT-REPEATABLE-READ
