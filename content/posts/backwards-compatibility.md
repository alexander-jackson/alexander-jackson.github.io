---
title: "Backwards Compatibility"
date: 2026-02-20T14:15:04Z
showtoc: true
---

When we're discussing APIs in software engineering, you'll hear the term "backwards compatibility" thrown around a lot. What does this actually mean?

## Our initial endpoint

Let's imagine we have an endpoint that looks as follows:

```java
// PaymentRequest.java
public record PaymentRequest(
  @NotNull PayeeAccountUid payeeAccountUid,
  @NotNull BigDecimal amount
) {}

// PaymentMobileResource.java
@Path("api/v1/accounts/{accountUid}/payments")
public interface PaymentMobileResource {
  @PUT
  @Consumes(APPLICATION_JSON)
  void createPayment(
    @PathParam("accountUid") AccountUid accountUid,
    @Valid @NotNull PaymentRequest paymentRequest
  );
}
```

Users can call this endpoint with a body that looks like this:

```json
{
  "payeeAccountUid": "23dd167a-9dea-4484-a562-c45ecfaf8a7f",
  "amount": "5.00"
}
```

## Breaking changes

Suppose we want to make a change to the endpoint to allow the user to specify a reference. Our new request object might look like this:

```java
// PaymentRequest.java
public record PaymentRequest(
  // existing fields...
  @NotBlank String reference
) {}
```

This change isn't backwards compatible, since the request we sent earlier wouldn't work anymore. It doesn't include a `reference` field, so would fail the `@NotBlank` validation and thus not be processed. We would typically call this a "breaking change" where the contract of the API has been modified in a way that means old clients would no longer work.

## Making it safely

In order to make this change safely, we first need to introduce it as an optional parameter. We'll need to make sure the handler code can deal with it not being there (perhaps using a default value), but it'll allow new clients to begin supplying it:

```java
// PaymentRequest.java
public record PaymentRequest(
  // existing fields...
  @Nullable String reference
) {}

// PaymentMobileService.java
void createPayment(
  AccountUid accountUid,
  PaymentRequest paymentRequest
) {
  String reference = paymentRequest.reference() == null ?
    "Default reference" :
    paymentRequest.reference();

  // continue using `reference`...
}
```

Since the parameter is nullable, both of the following requests will work:

```json
{
  "payeeAccountUid": "80ca9809-bf96-45b4-8c41-da2d1ee2b04e",
  "amount": "1.23"
}

{
  "payeeAccountUid": "e66d24f1-a468-4526-9944-93f3d6a9fbf2",
  "amount": "4.55",
  "reference": "Cheap pint"
}
```

The first request will end up with a reference of `Default reference` and the second will use the `Cheap pint` passed in the body.

## Making it required

Eventually, all of the clients will have been updated to send the `reference` field and we'll want to come back and make it required. This provides better API safety in terms of validation and allows us to remove our default handling. For that, we can just update the definition in the object:

```java
// PaymentRequest.java
public record PaymentRequest(
  // existing fields...
  // previously `@Nullable`, now enforced
  @NotBlank String reference
) {}
```

## Ignoring fields

While our change in the `Making it safely` section immediately began using the `reference` field (and using a default if it wasn't provided), we also have the option to add it into the API and do nothing with it:

```java
// PaymentRequest.java
public record PaymentRequest(
  // existing fields...
  @Nullable String reference
) {}

// PaymentMobileService.java
void createPayment(
  AccountUid accountUid,
  PaymentRequest paymentRequest
) {
  // don't even use the `reference` field
}
```

This allows clients to begin supplying the new `reference` field without us working out how to handle it yet. Once we've got everything migrated, we can make the same change as before to make it required and then begin using it without any of the `null` handling.
