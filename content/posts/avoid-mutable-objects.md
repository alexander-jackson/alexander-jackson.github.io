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
// TransactionRequest.java
public class TransactionRequest {
    private final AccountUid accountUid;
    private Money amount;

    public TransactionRequest(AccountUid accountUid, Money amount) {
        this.accountUid = accountUid;
        this.amount = amount;
    }

    // standard getters...

    public void setAmount(Money amount) {
        this.amount = amount;
    }
}

// TransactionService.java
public void processDefaultDonation(AccountUid accountUid) {
    var request = new TransactionRequest(accountUid, Money.of(GBP, "5.00"));

    validateTransaction(request);

    // do more things here, using `request`
}
```

As a software engineer reading this code, we can see when the transaction gets
created and we know that the amount is going to be £5. We then pass the request
into the validation function.

Without looking into the content of that method, we can't guarantee what the
amount would be afterwards. It could leave the request untouched, it could also
use the `setAmount` method to reduce the amount if it decided it was too high.
Perhaps it could have logic to cap the donation amount based on the amount in
the account, we don't know.

The point is that this makes the code more difficult to reason about. If the
request was immutable, we could avoid reading the implementation of
`validateTransaction` as we'd know the object was exactly the same as when it
was created.

## The Solution

Instead, we should prefer immutable objects. `TransactionRequest` should be
defined as:

```java
// TransactionRequest.java
public class TransactionRequest {
    private final AccountUid accountUid;
    private final Money amount;

    // standard constructor...

    // standard getters...
}
```

This means that once the fields have been set in the constructor, we know they
cannot possibly change. An alteration to the amount would need to create a new
object:

```java
public void processDefaultDonation(AccountUid accountUid) {
    var request = new TransactionRequest(accountUid, Money.of(GBP, "5.00"));
    var validatedRequest = validateTransaction(request);

    // do more things here, using `validatedRequest` instead
}
```

This signals to the developer that if they see usages of `validatedRequest`,
they should probably read the implementation of `validateTransaction` as
there's no guarantee that the details inside might be the same.

## Builders

One place where mutation is acceptable is in builder classes. These are used to
build up a representation of a more complex object, potentially starting with
some default values that can be overridden. These are especially useful in
tests where there may only be one or two fields that are relevant to the
scenario, allowing you to highlight these to the developer.

An example builder for the `TransactionRequest` object might look as follows:

```java
// TransactionRequestBuilder.java
public class TransactionRequestBuilder {
    private AccountUid accountUid;
    private Money amount;

    public static TransactionRequestBuilder withDefaults() {
        return new TransactionRequestBuilder()
            .withAccountUid(new AccountUid("..."))
            .withAmount(Money.of(GBP, "5.00"));
    }

    public withAccountUid(AccountUid accountUid) {
        this.accountUid = accountUid;
        return this;
    }

    public withAmount(Money amount) {
        this.amount = amount;
        return this;
    }

    public TransactionRequest build() {
        return new TransactionRequest(accountUid, amount);
    }
}
```

`TransactionRequestBuilder` is a mutable object, but it allows us to easily
build up test scenarios:

```java
// a transaction request without customisation
var default = TransactionRequestBuilder.withDefaults().build();

// one with the `amount` changed, but the `accountUid` as the default
var changedAmount = TransactionRequestBuilder.withDefaults()
    .withAmount(Money.of(GBP, "3.00"))
    .build();

// one with both values changed
var accountUid = new AccountUid();
var changedBoth = TransactionRequestBuilder.withDefaults()
    .withAccountUid(accountUid)
    .withAmount(Money.of(GBP, "3.00"))
    .build();
```

These typically make tests easier to understand and the mutable objects are
short-lived, so they are considered acceptable and an improvement to the code
in this case.

[immutable-schemas]: /posts/immutable-schemas
