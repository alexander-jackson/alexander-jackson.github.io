---
title: "Inserts with selects can be confusing"
date: 2026-10-30T11:48:05Z
draft: false
---

Let's imagine we want to store accounts, and the events associated with them.
We'll start with a basic `account` table:

```sql
CREATE TABLE account (
    id BIGINT GENERATED ALWAYS AS IDENTITY,
    account_uid UUID NOT NULL,
    name TEXT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL,

    CONSTRAINT pk_account PRIMARY KEY (id),
    CONSTRAINT uk_account_account_uid UNIQUE (account_uid)
);
```

From here, we can insert our first account:

```sql
INSERT INTO account (account_uid, name, created_at)
VALUES ('a68f8460-cbaf-4b11-a085-3a5d98ea3106', 'Primary', now()::timestamp);
```

Now, we want to start storing events for our account, perhaps for it being
funded or closed, so let's add another table to store those:

```sql
CREATE TABLE account_event (
    id BIGINT GENERATED ALWAYS AS IDENTITY,
    account_event_uid UUID NOT NULL,
    account_id BIGINT NOT NULL,
    event_type TEXT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL,

    CONSTRAINT pk_account_event PRIMARY KEY (id),
    CONSTRAINT uk_account_event_account_event_uid UNIQUE (account_event_uid),
    CONSTRAINT fk_account_event_account_id
        FOREIGN KEY (account_id)
        REFERENCES account (id)
);
```

Once this has been created, we can start writing events into it. There are two
main ways to write this, one using the `INSERT INTO ... SELECT FROM` approach
and the other using a sub-select to fetch the `account_id` field:

```sql
-- approach one, inserting from the select
INSERT INTO account_event (
    account_event_uid,
    account_id,
    event_type,
    created_at
)
SELECT
    '31affc39-1363-471d-9e0b-1d18ca344b98',
    account.id,
    'Funded',
    now()::timestamp
FROM account
WHERE account_uid = 'a68f8460-cbaf-4b11-a085-3a5d98ea3106';

-- approach two, using a sub-select
INSERT INTO account_event (
    account_event_uid,
    account_id,
    event_type,
    created_at
)
VALUES (
    '37a446c6-faf2-4618-b91e-d5128b16c031',
    (
        SELECT id
        FROM account
        WHERE account_uid = 'a68f8460-cbaf-4b11-a085-3a5d98ea3106'
    ),
    'Closed',
    now()::timestamp
);
```

These both work perfectly fine, and we end up with the expected data in the `account_event` table:

```sql
> SELECT ae.event_type, ae.created_at
  FROM account a
  JOIN account_event ae ON a.id = ae.account_id
  WHERE a.account_uid = 'a68f8460-cbaf-4b11-a085-3a5d98ea3106'
  ORDER BY ae.created_at;

 event_type |          created_at
------------+-------------------------------
 Funded     | 2026-10-30 12:03:35.080798+00
 Closed     | 2026-10-30 12:03:56.048017+00
(2 rows)
```

However, what happens if we accidentally use the wrong identifier here and try
to specify an event for an account that doesn't exist?

```sql
-- approach one, inserting from the select
> INSERT INTO account_event (
      account_event_uid,
      account_id,
      event_type,
      created_at
  )
  SELECT
      '62cdacad-80c3-4fbc-a15b-22bb65085418',
      account.id,
      'Funded',
      now()::timestamp
  FROM account
  WHERE account_uid = '11e07ec1-ce46-4201-90cb-83cf20b7283d';

INSERT 0 0

-- approach two, using a sub-select
> INSERT INTO account_event (
      account_event_uid,
      account_id,
      event_type,
      created_at
  )
  VALUES (
      'd4095a77-05ff-4878-8faf-2068ff87461a',
      (
          SELECT id
          FROM account
          WHERE account_uid = '11e07ec1-ce46-4201-90cb-83cf20b7283d'
      ),
      'Closed',
      now()::timestamp
  );

ERROR:  null value in column "account_id" of relation "account_event" violates not-null constraint
DETAIL:  Failing row contains (4, d4095a77-05ff-4878-8faf-2068ff87461a, null, Closed, 2026-10-30 12:08:16.715504+00).
```

Our first query, inserting directly from the select clause, found no rows to
insert and silently did nothing. If we're not checking what this returns on the
application side, we might never notice this and just drop requests without
persisting them.

The second query behaves as we might expect, returning an error and forcing the
application to either fail the request overall or handle it properly.
