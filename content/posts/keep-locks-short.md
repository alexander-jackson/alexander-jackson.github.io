---
title: "Keep Locks Short"
date: 2025-12-01T14:27:13Z
draft: false
---

When dealing with a PostgreSQL database, locks are both a blessing and a curse.
Most of the time they get out of your way, coordinating transactions to make
sure they happen in the order you would expect and preventing concurrent data
updates.

However, locks can also be difficult to work with.

## Needless Acquisitions

Let's consider the following migrations:

```sql
-- first migration
ALTER TABLE account
ADD COLUMN closed_at TIMESTAMP WITH TIME ZONE;
```

```sql
-- second migration
ALTER TABLE account
DROP COLUMN changed_at;
```

Altering a table requires an access exclusive lock on the table, which will
prevent any reads or writes at the same time, so it seems like a good idea to
acquire it and then release it as soon as possible by doing one operation at a
time.

Unfortunately, this just means we release the lock and then immediately begin
asking for it. Dropping a column on even a very large table should be a fast
operation so it's usually a better idea to do both once you have the lock:

```sql
-- single migration
ALTER TABLE account
ADD COLUMN closed_at TIMESTAMP WITH TIME ZONE;

ALTER TABLE account
DROP COLUMN changed_at;
```

Since the first statement acquires the access exclusive lock, the second
statement will run immediately instead of blocking and waiting for it. The
transaction holds the lock, not the statement.

## Accidental Overextension

When attempting to make changes to busy tables, we might use lock polling to
avoid blocking other transactions for long periods of time:

```sql
DO $$
  BEGIN
  SET LOCAL LOCK_TIMEOUT TO '50ms';
  LOOP
    BEGIN
      ALTER TABLE something_busy
      ADD COLUMN ...;
      RETURN;
    EXCEPTION WHEN LOCK_NOT_AVAILABLE THEN
      RAISE NOTICE 'Unable to obtain locks to alter public.something_busy, sleeping 2 seconds';
      PERFORM PG_SLEEP(2);
    END;
  END LOOP;
END; $$;
```

This tells the database to wait at most 50ms to acquire the access exclusive
lock we need to alter the table, and to wait at least 2 seconds after that
times out. This allows other transactions to proceed in the meantime and
prevents us causing a pile-up while we wait for the lock.

However, this can have unexpected behaviour if you have multiple lock polling
statements in the same migration:

```sql
DO $$
  BEGIN
  SET LOCAL LOCK_TIMEOUT TO '50ms';
  LOOP
    BEGIN
      ALTER TABLE something_busy
      ADD COLUMN ...;
      RETURN;
    EXCEPTION WHEN LOCK_NOT_AVAILABLE THEN
      RAISE NOTICE 'Unable to obtain locks to alter public.something_busy, sleeping 2 seconds';
      PERFORM PG_SLEEP(2);
    END;
  END LOOP;
END; $$;

DO $$
  BEGIN
  SET LOCAL LOCK_TIMEOUT TO '50ms';
  LOOP
    BEGIN
      ALTER TABLE something_also_really_busy
      ADD COLUMN ...;
      RETURN;
    EXCEPTION WHEN LOCK_NOT_AVAILABLE THEN
      RAISE NOTICE 'Unable to obtain locks to alter public.something_also_really_busy, sleeping 2 seconds';
      PERFORM PG_SLEEP(2);
    END;
  END LOOP;
END; $$;
```

Here, we poll for an access exclusive lock on `something_busy` and then add a
column to the table. Once done, we continue on an poll for an access exclusive
lock on `something_also_really_busy`. Naively this seems sensible, we want to
avoid blocking other transactions.

This goes badly wrong if there's a long running transaction on
`something_also_really_busy` though. If we don't acquire the lock immediately,
we'll keep holding the access exclusive lock on `something_busy` while we
continue to poll for the second one, blocking anything that wants to read from
or write to the table.
