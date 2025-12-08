---
title: "Avoid Mutable Objects"
date: 2025-12-05T21:25:28Z
draft: false
showtoc: true
---

In software engineering, you'll hear a lot about immutability of data and the
benefits this can provide. I've already written about this at [the database
level][immutable-schemas], but it's also worth discussing it in terms of plain
Java code.

## The Problem

Consider the following piece of code:

```java
// DonationRequest.java
public class DonationRequest {
    private final AccountUid accountUid;
    private BigDecimal amount;

    public DonationRequest(AccountUid accountUid, BigDecimal amount) {
        this.accountUid = accountUid;
        this.amount = amount;
    }

    // standard getters...

    public void setAmount(BigDecimal amount) {
        this.amount = amount;
    }
}

// DonationService.java
public void processDefaultDonation(AccountUid accountUid) {
    var request = new DonationRequest(accountUid, new BigDecimal("5.00"));

    validateDonation(request);

    // do more things here, using `request`
}
```

As a software engineer reading this code, we can see when the donation gets
created and we know that the amount is going to be £5. We then pass the request
into the validation function.

Without looking into the content of that method, we can't guarantee what the
amount would be afterwards. It could leave the request untouched, it could also
use the `setAmount` method to reduce the amount if it decided it was too high.
Perhaps it could have logic to cap the donation amount based on the amount in
the account, we don't know. Here are two possible definitions:

```java
// performs no mutations
public void validateDonation(DonationRequest request) {
    if (request.getAmount().isZero()) {
        throw new IllegalArgumentException("donations cannot have a zero amount");
    }
}

// caps the amount
public void validateDonation(DonationRequest request) {
    BigDecimal amount = request.getAmount();
    BigDecimal limit = new BigDecimal("3.00");

    if (amount.isGreaterThan(limit)) {
        request.setAmount(limit);
    }
}
```

This makes the code more difficult to reason about. If the request was
immutable, we could avoid reading the implementation of `validateDonation` as
we'd know the object was exactly the same as when it was created.

Additionally, this problem propagates further into the code. If
`validateDonation` called another method, we'd have to read further into the
usages to understand if the object could be mutated. As a result, we have to
read every line of code instead of being able to ignore large parts of it.

## The Solution

Instead, we should prefer immutable objects. `DonationRequest` should be
defined as:

```java
// DonationRequest.java
public class DonationRequest {
    private final AccountUid accountUid;
    private final BigDecimal amount;

    // standard constructor...

    // standard getters...
}
```

This means that once the fields have been set in the constructor, we know they
cannot possibly change. An alteration to the amount would need to create a new
object:

```java
public void processDefaultDonation(AccountUid accountUid) {
    var request = new DonationRequest(accountUid, new BigDecimal("5.00"));
    var validatedDonation = validateDonation(request);

    // do more things here, using `validatedDonation` instead
}
```

This signals to the developer that if they see usages of `validatedDonation`,
they should probably read the implementation of `validateDonation` as there's
no guarantee that the details inside might be the same.

## Builders

One place where mutation is acceptable is in builder classes. These are used to
build up a representation of a more complex object, potentially starting with
some default values that can be overridden. These are especially useful in
tests where there may only be one or two fields that are relevant to the
scenario, allowing you to highlight these to the developer.

An example builder for the `DonationRequest` object might look as follows:

```java
// DonationRequestBuilder.java
public class DonationRequestBuilder {
    private AccountUid accountUid;
    private BigDecimal amount;

    public static DonationRequestBuilder withDefaults() {
        return new DonationRequestBuilder()
            .withAccountUid(new AccountUid("..."))
            .withAmount(new BigDecimal("5.00"));
    }

    public withAccountUid(AccountUid accountUid) {
        this.accountUid = accountUid;
        return this;
    }

    public withAmount(BigDecimal amount) {
        this.amount = amount;
        return this;
    }

    public DonationRequest build() {
        return new DonationRequest(accountUid, amount);
    }
}
```

`DonationRequestBuilder` is a mutable object, but it allows us to easily build
up test scenarios:

```java
// a transaction request without customisation
var default = DonationRequestBuilder.withDefaults().build();

// one with the `amount` changed, but the `accountUid` as the default
var changedAmount = DonationRequestBuilder.withDefaults()
    .withAmount(new BigDecimal("3.00"))
    .build();

// one with both values changed
var accountUid = new AccountUid();
var changedBoth = DonationRequestBuilder.withDefaults()
    .withAccountUid(accountUid)
    .withAmount(new BigDecimal("3.00"))
    .build();
```

These typically make tests easier to understand and the mutable objects are
short-lived, so they are considered acceptable and an improvement to the code
in this case.

[immutable-schemas]: /posts/immutable-schemas
