---
title: "Postgres Lock Tricks"
date: 2024-08-24T17:25:00+01:00
showtoc: true
---

Postgres is a much loved relational database system that provides solid
performance and reliability. However, the concurrency model that it provides
(MVCC - multi-version concurrency control) can lead to some interesting
observations.

## How does MVCC work?

Multi-version concurrency control is not a simple system to explain. It's worth
reading the [Postgres documentation][postgres-mvcc] if you're interested in
going deeper with it, but this section aims to provide a small overview.

### Read Committed Isolation Level

This is the default value for transaction isolation in Postgres and the only
one we'll be discussing in this article. The documentation states:

> When a transaction uses this isolation level, a `SELECT` query (without a
> `FOR UPDATE/SHARE` clause) sees only data committed before the query began;
> it never sees either uncommitted data or changes committed by concurrent
> transactions during the query's execution.

The implication here is that if the following 2 commands are run:

```sql
-- Thread 1
SELECT name FROM account WHERE id = 1

-- Thread 2
UPDATE account SET name = 'John' WHERE id = 1
```

then the first thread will only see the value `John` if the second thread has
already committed its transaction when it starts its own. If the first thread
has already started the transaction, it essentially sees a snapshot of the
data. Even if there is work to be done before this `SELECT` command and thread
2 has committed by the time it is reached, it will still read `John`.

This is somewhat confused by the following:

```sql
-- Thread 1
UPDATE account SET name = 'Mark' WHERE id = 1

SELECT name FROM account WHERE id = 1
```

In this sequence of events we only have a single thread. Despite the
transaction not being committed when the `SELECT` command is executed,
uncommitted writes are still viewable if is was that transaction that wrote
them. This means the query will always return `Mark` in this case.

### Table-Level Locks

Different commands and transactions require different levels of access to
various database structures. For example, a command like `ALTER TABLE account
DROP COLUMN name` requires exclusive access to the `account` table in order to
prevent other transactions trying to read the data while it makes schema
changes.

`SELECT id FROM account` on the other hand needs very weak privileges, almost
all other commands can be run at the same time due to the read-only behaviour.

Postgres implements this access control using table-level locks. Each command
requires a set of locks on one or more tables to be able to run, which allows
for commands like `ALTER TABLE` to constrain themselves to exclusive access.

Some commands may require locks on multiple tables or indexes. For example, the
following command needs exclusive access on both the constrained
(`account_email`) and referenced (`account`) tables in order to drop the
foreign key:

```sql
ALTER TABLE account_email
DROP CONSTRAINT fk_account_account_email
```

This makes sense if you think about it, we need to update the table metadata on
both sides to account for the resulting lack of the foreign key.

### Confusing Access Requirements

However, not all locks are quite as obvious as you might expect. Let's imagine
we have some large tables in our application. We usually perform our schema
migrations on application startup (using [Flyway][flyway] in Java for example)
but we'd like to create an index concurrently to avoid blocking other database
transactions.

Our migration provider doesn't support this, since you cannot create indexes
concurrently within a transaction. Thus, we're going to run this separately to
application startup and add a migration script to create the index later if it
doesn't already exist.

The obvious way of doing this is:

```sql
CREATE INDEX IF NOT EXISTS <index_name>
ON <table> (<columns>)
```

This does exactly what we want it to. If the index already exists, our
migration will do nothing. If it doesn't exist then we proceed and create it
(non-concurrently, so likely only sensible for new databases or local
development). We get our pull request merged and before we know it we're off to
production.

Unfortunately, this doesn't behave in the way we might expect.

### Footguns

Let's consider the following migration, presuming we have already created the
index in a separate change, either on table creation or through an asynchronous
process:

```sql
CREATE INDEX IF NOT EXISTS idx_account_account_uid
ON account (account_uid)
```

We might expect that because the index already exists, this is a "free"
operation. However, if we run another command in a separate transaction and
don't commit it:

```sql
BEGIN;

UPDATE account
SET created_at = now()::timestamp
WHERE id = ...
```

Then we start to see some cracks. Our initial migration will just hang,
seemingly doing nothing. This is because the `UPDATE` command we're running is
taking a `RowExclusiveLock` on the `account` table, and our `CREATE INDEX IF
NOT EXISTS` command needs a `ShareLock` on the same table. These lock types
conflict, so we're essentially blocked until the `UPDATE` completes.

This is pretty surprising. The index already exists, so Postgres should be able
to elide the creation and just do nothing, right? Unfortunately this isn't how
the implementation works, and it still needs to get the lock in order to check
whether it needs to do anything.

### Lock-Free Solutions

Luckily, we can avoid locks in this scenario. Postgres exposes much of the
database metadata in the [catalog tables][postgres-catalogs] which we can use to query index data.

For example, the `pg_index` table contains all of the indexes in the system, so
we can write something like:

```sql
SELECT EXISTS (
    SELECT
    FROM pg_index pi
    JOIN pg_class pc ON pc.oid = pi.indexrelid
    WHERE pc.relname = 'idx_account_account_uid'
)
```

This will check the catalog tables and tell us if `idx_account_account_uid`
already exists in the database. From here we can simply wrap our `CREATE INDEX`
command in this check, which will allow us to avoid the creation if it does
exist without taking any locks (outside of the catalog tables at least, which
very rarely have locks on them).

[postgres-mvcc]: https://www.postgresql.org/docs/current/mvcc.html
[postgres-catalogs]: https://www.postgresql.org/docs/current/catalogs.html
[flyway]: https://flywaydb.org
