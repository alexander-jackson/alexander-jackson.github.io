---
title: "Sealed Interfaces Are Great"
date: 2026-08-25T16:00:03+01:00
draft: false
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

let success = Decision::Success;
let failure = Decision::Failure {
    reason: FailureReason::InsufficientEvidence,
};
```

Above, we've defined a type called `Decision` which has two variants called
`Success` and `Failure`. Unlike a regular `enum` in a lot of languages, the
`Failure` case also carries some data (the `reason` field) alongside it.

### Using Java classes

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

### Adding new variants

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

Having four constructor parameters, where three of them are generally `null`,
makes this approach quite difficult to follow. You can imagine this might grow
in complexity over time, other engineers might add additional states that
require other data to be stored, and we haven't even discussed how to get data
out of this object yet.

### Retrieving stored information

If we've got an instance of our decision, how do we work out which one we have?
Well that's easy, we just look at the decision type:

```java
Decision decision = Decision.failure(FailureReason.INSUFFICIENT_EVIDENCE);

if (decision.getDecisionType() == DecisionType.FAILURE) {
    // what type should this be?
    var failureReason = decision.getFailureReason();
}
```

It becomes more complicated when we try and get other attributes from it. We
now need to know that if the decision type is `FAILURE`, we should expect to
have a `failureReason` property. However, what should `getFailureReason`
return?

Our two options are:

- Return `FailureReason` and return `null` if it's not a failure, which isn't
  obvious from the signature
- Return `Optional<FailureReason>` and force us to always handle the
  `Optional.empty()` case, even if we know it's a failure

Neither of these are particularly fun to work with.

### The sealed interface approach

Regular interfaces allow us to express this type hierarchy more easily, but
sealed interfaces specifically enable some additional language features and
document our options better.

Here's what the code above might look like if we represented it with a sealed
interface instead:

```java
public sealed interface Decision {
    public record Success() implements Decision {}

    public record Failure(FailureReason reason) implements Decision {}

    public record Referral(Team team, ReferralReason reason) implements Decision {}
}
```

Now, we've defined each of our decision types as a separate record, all
implementing the `Decision` marker interface. Each record clearly tells us
which parameters are relevant to it, and we can choose simpler names for the
variables since there's no overlap.

Note, if the variants are not nested inside the interface, you'd need a
`permits` clause as well:

```java
public sealed interface Decision permits Success, Failure, Referral {}

// definitions in separate files
```

The compiler automatically infers this clause if the records are nested though.

Instances of `Decision` are still easy to create, and they compose nicely with
`switch` statements. We can even pull out fields of the states using pattern
matching:


```java
Decision failure = new Decision.Failure(FailureReason.INSUFFICIENT_EVIDENCE);

switch (failure) {
    case Success() -> {
        // handle the success case
    }
    case Failure(FailureReason reason) -> {
        // handle the failure case
    }
    case Referral(Team team, ReferralReason reason) -> {
        // handle the referral case
    }
}
```

If we miss a variant or someone adds a new one without it being handled, we get
a compile failure:

```java
switch (decision) {
    case Success() -> {
        // handle the success case
    }
    case Failure(FailureReason reason) -> {
        // handle the failure case
    }
    // missing handling for `Referral`
}
```

```bash
error: the switch statement does not cover all possible input values
    switch (decision) {
    ^
```

We don't have to handle `null` by default, but we can if that's a valid state:

```java
switch (decision) {
    // others...
    null -> {
        // handle the `null` case
    }
}
```

In general, this leads to code which expresses its intention better, as we can
list all the representations we expect and what the objects should be storing
in each of them.
