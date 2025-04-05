---
title: "Storing Enums in PostgreSQL"
date: 2025-04-05T06:41:01Z
showtoc: true
---

Enumerations (or enums) are a wonderful concept that are supported in the
majority of languages these days in some form or another. While a `String` can
store arbitrary text of any length, an `enum` represents a finite set of values
or variants.

These are extremely useful when considering database schema design, as it
allows you to restrict the possible set of states your system can be in.

Take for example, an `account` table with a `state` field:

```sql
postgres=# CREATE TABLE account (
    id BIGINT GENERATED ALWAYS AS IDENTITY,
    state TEXT NOT NULL
);
CREATE TABLE
```

Here, we're storing our state using the `TEXT` datatype in PostgreSQL which
means it could accept any string value. We can store all the ones we might
expect:

* `OPENED`
* `CLOSED`
* `SUSPENDED`

But also plenty of values we might not expect:

* `OPEN`
* `CLOSING`
* `BANANAS`

### Enum Types

This sort of usage (where we know all the possible options) is much better
suited to an enum-based representation. PostgreSQL does support enums:

```sql
postgres=# CREATE TYPE account_state AS ENUM ('OPEN', 'CLOSED', 'SUSPENDED');
CREATE TYPE

postgres=# CREATE TABLE account (
    id BIGINT GENERATED ALWAYS AS IDENTITY,
    state account_state NOT NULL
);
CREATE TABLE
```

This forces the database to check the values on insertion and ensure that they
match the enum:

```sql
postgres=# INSERT INTO account (state) VALUES ('OPENED');
INSERT 0 1

postgres=# INSERT INTO account (state) VALUES ('OPEN');
ERROR:  invalid input value for enum account_state: "OPEN"

postgres=# SELECT * FROM account;
 id | state
----+-------
  1 | OPENED
(1 row)
```

We can easily add new variants to the definition if we decide that we want to
represent the state of an account before it gets approved:

```sql
postgres=# ALTER TYPE account_state ADD VALUE 'PENDING';
```

However, if our data model changes and we decide that the `SUSPENDED` state
isn't needed anymore, it becomes a little more difficult. There's no `ALTER
TYPE <name> REMOVE VALUE <variant>` command, as this would require working out
whether the variant was used anywhere in the database. Even with an index, that
could be a very expensive operation requiring locks on many different tables.

We can try and get around this by creating a new type with our reduced variant
set and altering the type of the column in-place, although this does require
some interesting casting from `account_state` to `TEXT` to `account_state_new`:

```sql
-- create a new type with our variants
postgres=# CREATE TYPE account_state_new AS ENUM ('OPENED', 'CLOSED', 'PENDING');

-- cast the existing column to the new type
postgres=# ALTER TABLE account ALTER COLUMN state TYPE account_state_new USING state::TEXT::account_state_new;

-- drop the old type
postgres=# DROP TYPE account_state;

-- rename the new one as part of tidying up
postgres=# ALTER TYPE account_state_new RENAME TO account_state;
```

This approach probably won't work so well on a large table, since it will take
an access exclusive lock on it. This will block all reads and writes while the
database is either checking that the cast is valid (requiring reading every
row) or rewriting everything. Neither of these methods are going to be fast.

We can actually do this in a much more dangerous way if we like. `pg_enum`
stores metadata about defined enums and their variants, so we can look up the
definition for the one we'd like to change:

```sql
postgres=# SELECT pe.oid, pe.enumlabel, pt.typname
FROM pg_enum pe
JOIN pg_type pt ON pe.enumtypid = pt.oid;

  oid   | enumlabel |    typname
--------+-----------+---------------
 804946 | PENDING   | account_state
 804940 | SUSPENDED | account_state
 804938 | CLOSED    | account_state
 804936 | OPENED    | account_state
(4 rows)
```

From here, we can just delete the value we no longer want in our type:

```sql
postgres=# DELETE FROM pg_enum WHERE oid = 804940;
DELETE 1
```

Note: do not do this. If the enum variant is still used anywhere in the
database it is now corrupted.

### Check Constraints

Another option for constraining the data allowed in a column or table is a
check constraint. If we return to our `TEXT` based `state` representation, we
can add one of these onto the table:

```sql
postgres=# CREATE TABLE account (
    id BIGINT GENERATED ALWAYS AS IDENTITY,
    state TEXT NOT NULL,

    -- new, defining our check constraint
    CONSTRAINT chk_account_state CHECK (state = 'OPENED' OR state = 'CLOSED')
);
CREATE TABLE
```

This works largely the same as before, we can happily insert values if they're
valid and we get errors if not:

```sql
postgres=# INSERT INTO account (state) VALUES ('OPENED');
INSERT 0 1

postgres=# INSERT INTO account (state) VALUES ('OPEN');
ERROR:  new row for relation "account" violates check constraint "chk_account_state"
DETAIL:  Failing row contains (2, OPEN).

postgres=# SELECT * FROM account;
 id | state
----+--------
  1 | OPENED
(1 row)
```

Adding new values becomes a bit more complicated though. Instead of just adding
a new variant to the type, we'll need to redefine the check constraint itself,
drop the old one and then rename it:

```sql
-- add the new constraint
postgres=# ALTER TABLE account
ADD CONSTRAINT chk_account_state_new
CHECK (state = 'OPENED' OR state = 'CLOSED' OR state = 'PENDING');

-- drop the old constraint
postgres=# ALTER TABLE account DROP CONSTRAINT chk_account_state;

-- rename the new constraint
postgres=# ALTER TABLE account
RENAME CONSTRAINT chk_account_state_new
TO chk_account_state;
```

This is more fiddly than the enum-based approach, as well as being harder to
review. We need to check that the old constraint is a subset of the new one in
order to allow it to apply trivially.

Even then, PostgreSQL is not clever enough to notice this and will still need
to check the table data. This can be done asynchronously by adding the
constraint as not valid and then validating it, but it's definitely more of a
headache than the enum version.

Removing a variant is the exact same process but in reverse:

```sql
-- update any existing data
postgres=# UPDATE account SET state = 'CLOSED' WHERE state = 'PENDING';

-- add the new constraint
postgres=# ALTER TABLE account
ADD CONSTRAINT chk_account_state_new
CHECK (state = 'OPENED' OR state = 'CLOSED');

-- drop the old constraint
postgres=# ALTER TABLE account DROP CONSTRAINT chk_account_state;

-- rename the new constraint
postgres=# ALTER TABLE account
RENAME CONSTRAINT chk_account_state_new
TO chk_account_state;
```

Generally speaking, working with the constraints is pretty straightforward.
While adding a variant is more difficult, it's the same difficulty as removing
a variant, and some automation it's fairly safe to add and remove values.

### Reference Tables

Another possibility is a reference table. These store a mapping of an `id`
column and a `name` which represents the enum variant.

```sql
CREATE TABLE account_state (
    id BIGINT GENERATED ALWAYS AS IDENTITY,
    name TEXT NOT NULL,

    CONSTRAINT pk_account_state PRIMARY KEY (id),
    CONSTRAINT uk_account_state_name UNIQUE (name)
);
```

We can then use a foreign key in our source table:

```sql
CREATE TABLE account (
    id BIGINT GENERATED ALWAYS AS IDENTITY,
    account_state_id BIGINT NOT NULL,

    -- new, defining our foreign key constraint
    CONSTRAINT fk_account_account_state_id
    FOREIGN KEY (account_state_id)
    REFERENCES account_state (id)
);
```

Before we can create any data, we need to insert some values into our reference
table:

```sql
INSERT INTO account_state (name) VALUES ('OPENED'), ('CLOSED');
```

The syntax for insertions becomes a little more complicated, since we need to
lookup the right `id` value from `account_state`:

```sql
INSERT INTO account (account_state_id)
VALUES (
    (SELECT id FROM account_state WHERE name = 'OPENED')
);
```

Querying the data is also a little more complicated as we need to join the
tables together and rename the `name` column to make more sense:

```sql
SELECT a.id, s.name AS account_state
FROM account a
JOIN account_state s ON a.account_state_id = s.id;

 id | account_state
----+---------------
  1 | OPENED
(1 row)
```

Adding values becomes very easy however, requiring a simple insert into the
table:

```sql
INSERT INTO account_state (name) VALUES ('PENDING');
```

Removing a value is a little more difficult, depending on the size of the
tables using the enum table. If they're small and you are certain the value is
no longer being used, you can just delete it from the table:

```sql
DELETE FROM account_state WHERE name = 'PENDING';
```

This will require PostgreSQL to validate that the `id` value corresponding to
that name is no longer present in any of the tables with foreign keys. If these
are large or do not have appropriate indexes then this has the potential to
cause a lot of performance issues in the database.

You can also easily rename variants:

```sql
UPDATE account_state SET name = 'PROCESSING' WHERE name = 'PENDING';
```

### Conclusion

PostgreSQL's `enum` type is a great option by default, but the lack of ability
to easily remove values from it makes it more difficult to use. It's likely
that as the schema evolves you will want to tweak these and it doesn't provide
as much flexibility as it could.

Check constraints are a little more complex to work with, but are symmetric in
their difficulty to add or remove variants. With some automation they can be
quite straightforward to manipulate, however they are less obvious to the
developer and thus can cause mistakes to be caught later.

Reference tables make queries more complicated, but they provide easy
ergonomics for adding and removing values in smaller databases. With support
from indexes, you can still trivially remove values without taking long-lived
database locks or causing performance issues.
