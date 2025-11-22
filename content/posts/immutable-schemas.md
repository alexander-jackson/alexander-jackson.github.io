---
title: "Immutable Schemas"
date: 2025-11-22T18:24:08+00:00
draft: false
---

Representing data in any application is a difficult problem to solve involving
many different tradeoffs. Let's imagine we want to store information about a
purchase:

```sql
CREATE TABLE purchase (
    id BIGINT GENERATED ALWAYS AS IDENTITY,
    purchase_uid UUID NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL,

    CONSTRAINT pk_purchase PRIMARY KEY (id),
    CONSTRAINT uk_purchase_purchase_uid UNIQUE (purchase_uid)
);
```

At a given point in time, a purchase might be in one of a few states:

* Pending (the customer has requested the item)
* Settled (the payment has transacted)
* Dispatched (the item has been sent for delivery)
* Delivered (the customer has the item)

Our first intuition might storing a `status` column on the table:

```sql
CREATE TYPE purchase_status AS ENUM (
    'PENDING',
    'SETTLED',
    'DISPATCHED',
    'DELIVERED'
);

ALTER TABLE purchase
ADD COLUMN status purchase_status NOT NULL;
```

We can then modify the status whenever an action occurs:

```sql
UPDATE purchase
SET status = 'DELIVERED'
WHERE purchase_uid = '...';
```

However, if we have a purchase in the `DELIVERED` status, we have no idea when
it was actually delivered. Let's add another column on the table to store when
the status was changed:

```sql
ALTER TABLE purchase
ADD COLUMN status_changed_at TIMESTAMP WITH TIME ZONE NOT NULL;
```

Every time we modify the status, we update the new field as well:

```sql
UPDATE purchase
SET
    status = 'DELIVERED'
    status_changed_at = now()::timestamp
WHERE purchase_uid = '...';
```

## Historical Data

One issue with this approach is that every time we transition the status, we
lose information we had before. When we move from `DISPATCHED` to `DELIVERED`
and update the timestamp, we no longer know when the purchase was dispatched.

Instead, let's store a timestamp for each state:

```sql
ALTER TABLE purchase
DROP COLUMN status, DROP COLUMN status_changed_at;

ALTER TABLE purchase
ADD COLUMN pending_at TIMESTAMP WITH TIME ZONE NOT NULL,
ADD COLUMN settled_at TIMESTAMP WITH TIME ZONE,
ADD COLUMN dispatched_at TIMESTAMP WITH TIME ZONE,
ADD COLUMN delivered_at TIMESTAMP WITH TIME ZONE;
```

We can make `purchase.pending_at` non-null since rows will have this from the
beginning, but everything else must be nullable. We can then derive the status
from these columns:

```sql
INSERT INTO purchase (purchase_uid, created_at, pending_at)
VALUES (
    'd3b1ca3e-83b0-432b-81ea-330facdf7f56',
    '2025-05-04 09:13:54',
    '2025-05-04 09:13:57'
);

INSERT INTO purchase (purchase_uid, created_at, pending_at, settled_at, dispatched_at)
VALUES (
    'decff56d-60cb-4368-9995-91768c3081dd',
    '2025-05-04 10:28:12',
    '2025-05-04 10:28:14',
    '2025-05-04 10:30:02',
    '2025-05-04 21:01:55'
);

SELECT
    purchase_uid,
    created_at,
    CASE
        WHEN delivered_at IS NOT NULL THEN 'DELIVERED'
        WHEN dispatched_at IS NOT NULL THEN 'DISPATCHED'
        WHEN settled_at IS NOT NULL THEN 'SETTLED'
    ELSE
        'PENDING'
    END AS status
FROM purchase
ORDER BY created_at ASC;
```

This returns the following output:
```
             purchase_uid             |       created_at       |   status
--------------------------------------+------------------------+------------
 d3b1ca3e-83b0-432b-81ea-330facdf7f56 | 2025-05-04 09:13:54+01 | PENDING
 decff56d-60cb-4368-9995-91768c3081dd | 2025-05-04 10:28:12+01 | DISPATCHED
 ```

While this is correct, it's quite a cumbersome approach:

* We're storing lots of `null` values if purchases don't progress
* Since PostgreSQL implements an update as a delete and insert, we might be
  creating lots of dead rows
* Our `SELECT` statement doesn't even return the `status_changed_at` equivalent
  at the moment, we'd need a more complex query for that
* The table will become quite wide over time
* Any queries on status will need to use the same logic, leading to duplication

## Event-based Approach

Let's return to our `purchase` table definition from the beginning:

```sql
DROP TABLE purchase;

CREATE TABLE purchase (
    id BIGINT GENERATED ALWAYS AS IDENTITY,
    purchase_uid UUID NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL,

    CONSTRAINT pk_purchase PRIMARY KEY (id),
    CONSTRAINT uk_purchase_purchase_uid UNIQUE (purchase_uid)
);
```

A nicer model might be to represent each of these changes in state as an event, related to a purchase. We can insert these whenever something happens, such as the item being dispatched or delivered. We'll need a separate table with a foreign key:

```sql
CREATE TYPE purchase_event_type AS ENUM (
    'PENDING',
    'SETTLED',
    'DISPATCHED',
    'DELIVERED'
);

CREATE TABLE purchase_event (
    id BIGINT GENERATED ALWAYS AS IDENTITY,
    purchase_event_uid UUID NOT NULL,
    purchase_id BIGINT NOT NULL,
    event_type purchase_event_type NOT NULL,
    occurred_at TIMESTAMP WITH TIME ZONE NOT NULL,

    CONSTRAINT pk_purchase_event PRIMARY KEY (id),
    CONSTRAINT uk_purchase_event_purchase_event_uid UNIQUE (purchase_event_uid),
    CONSTRAINT fk_purchase_event_purchase_id FOREIGN KEY (purchase_id) REFERENCES purchase (id)
);
```

This gives us a way to represent each change in state as an immutable event. When a
purchase is created, we insert a `PENDING` event:

```sql
INSERT INTO purchase (purchase_uid, created_at)
VALUES (
    'd3b1ca3e-83b0-432b-81ea-330facdf7f56',
    '2025-05-04 09:13:54'
);

INSERT INTO purchase_event (purchase_event_uid, purchase_id, event_type, occurred_at)
VALUES (
    'a1f5e6d2-4c3b-4e29-9f7c-8b6d5e4f3a2b',
    (SELECT id FROM purchase WHERE purchase_uid = 'd3b1ca3e-83b0-432b-81ea-330facdf7f56'),
    'PENDING',
    '2025-05-04 09:13:57'
);
```

When the purchase is settled, we insert a `SETTLED` event:

```sql
INSERT INTO purchase_event (purchase_event_uid, purchase_id, event_type, occurred_at)
VALUES (
    'b2c6d7e3-5d4c-5f3a-0g8h-9c7d6e5f4b3c',
    (SELECT id FROM purchase WHERE purchase_uid = 'd3b1ca3e-83b0-432b-81ea-330facdf7f56'),
    'SETTLED',
    '2025-05-04 10:00:12'
);
```

We can continue this for each event in the purchase lifecycle. To get the current
status of a purchase, we can query the latest event:

```sql
SELECT
    p.purchase_uid,
    p.created_at,
    pe.event_type AS status,
    pe.occurred_at AS status_changed_at
FROM purchase p
JOIN LATERAL (
    SELECT event_type, occurred_at
    FROM purchase_event
    WHERE purchase_id = p.id
    ORDER BY occurred_at DESC
    LIMIT 1
) pe ON true
ORDER BY p.created_at ASC;
```

The benefit of this approach is that we maintain a full history of all the
changes to a purchase, without losing any information. We can easily measure
attributes like how long purchases are taking to be delivered, and we can add
new event types in the future without modifying existing data.
