---
title: "Sealed Interfaces Are Great"
date: 2026-08-20T15:39:03+01:00
draft: true
showtoc: true
---

As someone who has written a lot of Rust code, I often find myself looking for
an equivalent to the `enum` keyword in other languages. This lets you nicely
represent data which might take different shapes in some scenarios:

```rust
enum FailureReason {
    InsufficientEvidence,
}

enum Decision {
    Success,
    Failure {
        reason: FailureReason,
    }
}

let success = Decision::Successful;
let failure = Decision::Failure {
    reason: FailureReason::InsufficientEvidence,
};
```

Above, we've defined a type called `Decision` which has two variants called
`Success` and `Failure`. Unlike a regular `enum` in a lot of languages, the
`Failure` case also carries some data (the `reason` field) alongside it.

We could represent this in Java using a class, but it feels more clunky:

```java
public enum FailureReason {
    INSUFFICIENT_EVIDENCE,
}

public static class Decision {
    private final boolean success;
    private final FailureReason reason;

    private Decision(boolean success, FailureReason reason) {
        this.success = success;
        this.reason = reason;
    }

    public static Decision success() {
        return new Decision(true, null);
    }

    public static Decision failure(FailureReason reason) {
        return new Decision(false, reason);
    }
}
```

This allows us to use it in a similar way to the Rust version:

```java
Decision success = Decision.success();
Decision failure = Decision.failure(FailureReason.INSUFFICIENT_EVIDENCE);
```

However, it doesn't hold together as nicely if we want to add more options to
it. Let's imagine we wanted to add a new state, indicating that we decided to
refer this to another team. In Rust, that becomes:

```rust
enum Team {
    Accounts,
    Billing,
}

enum ReferralReason {
    ExpertRequired,
}

enum Decision {
    // existing states...
    Referral {
        team: Team,
        reason: ReferralReason,
    }
}
```

The Java side is a bit more tricky. We already have a `reason` field for
starters which has a type of `FailureReason`, so that's going to need renaming.
Our `success` field isn't well named either, especially since it just stores a
`boolean` at the moment which can't distinguish between 3 variants.

Hammering away at it a bit, we might arrive on:

```java
public enum DecisionType {
    SUCCESS,
    FAILURE,
    REFERRAL,
}

public enum FailureReason {
    INSUFFICIENT_EVIDENCE,
}

public enum Team {
    ACCOUNTS,
    BILLING,
}

public enum ReferralReason {
    EXPERT_REQUIRED,
}

public static class Decision {
    private final DecisionType decisionType;
    private final FailureReason failureReason;
    private final Team team;
    private final ReferralReason referralReason;

    private Decision(
        DecisionType decisionType,
        FailureReason failureReason,
        Team team,
        ReferralReason referralReason
    ) {
        this.decisionType = decisionType;
        this.failureReason = failureReason;
        this.team = team;
        this.referralReason = referralReason;
    }

    public static Decision success() {
        return new Decision(DecisionType.SUCCESS, null, null, null);
    }

    public static Decision failure(FailureReason failureReason) {
        return new Decision(DecisionType.FAILURE, failureReason, null, null);
    }

    public static Decision referral(Team team, ReferralReason referralReason) {
        return new Decision(DecisionType.REFERRAL, null, team, referralReason);
    }
}
```

It's getting a little hard to follow. You can imagine this might grow in
complexity over time, other engineers might add additional states that require
other data to be stored, and we haven't even discussed how to get data out of
this object yet.
