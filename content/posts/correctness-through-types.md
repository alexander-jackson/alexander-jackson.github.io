---
title: "Correctness Through Types"
date: 2023-01-14T10:57:33Z
draft: true
showtoc: true
---

## Writing a Server

Let's imagine we are writing an HTTP server in Java. We want to have 2
endpoints for our frontend to consume, one to create a user and another to
fetch a user. We might start by defining our API as follows:

```java
@Path("api/v1/users")
public interface UserResource {
    @PUT
    void createUser(User user);

    @GET
    @Path("{userUid}")
    User fetchUser(UUID userUid);
}
```

We'll also want to define what `User` looks like, so let's go with a basic
implementation:


```java
public class User {
    private final UUID userUid;
    private final String email;

    public User(UUID userUid, String email) {
        this.userUid = userUid;
        this.email = email;
    }
}
```

This seems sensible, our `User` has a unique identifier as well as an email
address. The frontend can make a `PUT` request to `api/v1/users` with the user
they'd like to create, and they can fetch a user by making a `GET` request to
`api/v1/users/{userUid}` with the unique identifier.

## But hold on, this doesn't work?

Ignoring the lack of an actual implementation of our resource, you're entirely
correct. The frontend won't be able to hit either of these endpoints, or if it
can, it won't get anything useful back from it.

We've made a lot of mistakes already in this relatively simple API definition,
so let's have a look through them.

### Content Type Annotations

Neither of our endpoints actually define what format they consume or produce.
The web server doesn't know whether it should expect the `User` as a JSON
payload in the body, query parameters in the URI or form encoded. It also has
no idea what format to return the `User` in when we fetch it.

The way we do this is through the `@Consumes` and `@Produces` annotations.
These allow us to specify that, yes, we expect this in a JSON format and any
other format is an invalid request. We can also specify that we'd like to
serialize the `User` into JSON when fetching it.

We just need to make the following simple change:

```java
@PUT
@Consumes(MediaType.APPLICATION_JSON) // new
void createUser(User user);

@GET
@Path("{userUid}")
@Produces(MediaType.APPLICATION_JSON) // new
User fetchUser(UUID userUid);
```

Nice! Now the server should understand how to receive and return our `User`
when communicating with the frontend. Sort of.

### JSON Property Annotations

We're also missing some annotations on our `User` class. Presuming we're using
the `jackson` library for parsing JSON, we'll need to tell it which fields in
the payload correspond to the fields of our `User`.

That's as simple as adding these:

```java
// new: adding some annotations
public User(
    @JsonProperty("userUid") UUID userUid,
    @JsonProperty("email") String email
) {
    this.userUid = userUid;
    this.email = email;
}
```

Once we've done this, we should be able to hit the `createUser` endpoint
successfully! Assuming we've implemented some kind of storage (like a
database), we can then call the `fetchUser` endpoint and get it back. Except
that doesn't work.

### Object Getters

See, when we try and return the `User` from `fetchUser`, we also need to tell
`jackson` what fields are exposed and thus need to be serialized. Since our
`User` currently has no `get*` methods, it will just serialize to the empty
object `{}`. Not much use for our frontend.

All we need to do is add some basic `getUserUid` and `getEmail` methods:

```java
// in the `User` class

public UUID getUserUid() {
    return userUid;
}

public String getEmail() {
    return email;
}
```

Okay, finally. The frontend can successfully hit our two endpoints and can now
create and display users to our customers. Job well done.

## Bug Report

It's been a couple days since we deployed our web server and the frontend has
been happily creating and fetching users. But then we get a message, telling us
that someone has created a `User` with a `null` email address. This should
probably be rejected by the web server, so we go about finding a solution.

We could just check in the `createUser` handler that the `email` field is not
null, but we'd like to get the server to handle it automatically by validating
the request body a bit more. We find the `@NotNull` annotation and slap it on
the `email` field. While we are here, we might as well make sure no one submits
a `userUid` that is `null`, so let's add that too:

```java
public class User {
    @NotNull
    private final UUID userUid;

    @NotNull
    private final String email;

    // same as before
}
```

Happily, we ship this off to production and bask in our success. No one can
ever create a user with a `null` email address again.

Until it happens again the next day.

### Validation Annotations

See, we added these nice annotations to our object, but we never told the
server to actually validate them. They're just there mocking us. We do a bit of
searching and find out we need to add the `@Valid` annotation to our
`createUser` definition:

```java
// as before
void createUser(@Valid User user);
```

We actually test this one ourselves locally, and are happy to see that the
annotations are working as we expect now. Off to production.

The next day, we get a message that someone has seen a `NullPointerException`
in production. That seems odd, so we pull out the stack trace and dive into the
code. It seems like instead of submitting a `User` with a null identifier or
email, they've just submitted the entire user as `null` instead. When we go to
pull out the email, we're calling `getEmail` on `null`.

Fine, we can just add a `@NotNull` to the whole payload:

```java
// as before
void createUser(@Valid @NotNull User user);
```

While we are here, we make sure that the identifier provided to `fetchUser` is
not null as well, and we might want to make sure the response we provide to the
client is validated just in case we try and return one of those pesky users
from before. This also prevents us from making a mistake and setting one of the
fields to `null` somewhere before returning to the client.

```java
@Valid User fetchUser(@NotNull UUID userUid);
```

### Full Listing

At this point, it's worth looking over what we have so far:

```
// UserResource.java

@Path("api/v1/users")
public interface UserResource {
    @PUT
    @Consumes(MediaType.APPLICATION_JSON)
    void createUser(@Valid @NotNull User user);

    @GET
    @Path("{userUid}")
    @Produces(MediaType.APPLICATION_JSON)
    @Valid User fetchUser(@NotNull UUID userUid);
}

// User.java

public class User {
    @NotNull
    private final UUID userUid;

    @NotNull
    private final String email;

    public User(
        @JsonProperty("userUid") UUID userUid,
        @JsonProperty("email") String email
    ) {
        this.userUid = userUid;
        this.email = email;
    }

    public UUID getUserUid() {
        return userUid;
    }

    public String getEmail() {
        return email;
    }
}
```

## Something Here

I'm not sure about you, but it feels like there's a lot of ways to go wrong in
Java. You need to make sure you're annotating everything correctly and in the
right places, as soon as you miss something it can cause cascading issues. How
do other languages solve this sort of problem?

Let's have a look at how this works in Rust. We'll start by creating a new
project using the `tokio` runtime and the `axum` HTTP server library.

```bash
# create the project and `cd` into it
cargo new --bin axum-example
cd axum-example

# add our dependencies
cargo add axum
# macros for `tokio::main`, rt-multi-thread for the multithreaded runtime
cargo add tokio --features "macros rt-multi-thread"
# for the `Uuid` Type
cargo add uuid
```

We can then define our API as we did before:

```rust
use axum::routing::{get, put};
use axum::{Router, Server};
use uuid::Uuid;

struct User {
    user_uid: Uuid,
    email: String,
}

async fn create_user(user: User) {
    todo!()
}

async fn fetch_user(user_uid: Uuid) -> User {
    todo!()
}

#[tokio::main]
async fn main() {
    // create our application, setting up the routing
    let app = Router::new()
        .route("/api/v1/users", put(create_user))
        .route("/api/v1/users/:user_uid", fetch_user)
        .into_make_service();

    // bind it on a specific port and create the server
    let addr = "0.0.0.0:5000".parse().unwrap();
    let server = Server::bind(&addr).serve(app);

    // `.await` the server to begin handling connections
    server.await.expect("Failed to run server");
}
```

This is the outlines of a very basic server in Rust. We're copying most of this
from Java, we have the same 2 endpoints on the same paths and the same `User`
struct. We're also doing some of the server setup itself here, by the end of
the example we will have a working server that we can hit with requests.

### Handler Errors

Only problem is, this doesn't even compile yet. `axum` spits out a very gnarly
error for this:

```bash
alexander@cinnamon ~/D/U/P/W/a/s/axum-example> cargo c
    Checking axum-example v0.1.0 (/Users/alexander/Documents/University/Programming/Web/alexander-jackson.github.io/scratchpad/axum-example)
error[E0277]: the trait bound `fn(User) -> impl std::future::Future<Output = ()> {create_user}: Handler<_, _, _>` is not satisfied
   --> src/main.rs:13:37
    |
13  |         .route("/api/v1/users", put(create_user))
    |                                 --- ^^^^^^^^^^^ the trait `Handler<_, _, _>` is not implemented for fn item `fn(User) -> impl std::future::Future<Output = ()> {create_user}`
    |                                 |
    |                                 required by a bound introduced by this call
    |
    = help: the trait `Handler<T, S, B2>` is implemented for `Layered<L, H, T, S, B, B2>`
note: required by a bound in `put`
   --> /Users/alexander/.cargo/registry/src/github.com-1ecc6299db9ec823/axum-0.6.4/src/routing/method_routing.rs:408:1
    |
408 | top_level_handler_fn!(put, PUT);
    | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ required by this bound in `put`
    = note: this error originates in the macro `top_level_handler_fn` (in Nightly builds, run with -Z macro-backtrace for more info)

error[E0277]: the trait bound `fn(Uuid) -> impl std::future::Future<Output = User> {fetch_user}: Handler<_, _, _>` is not satisfied
   --> src/main.rs:14:46
    |
14  |         .route("/api/v1/users/:userUid", get(fetch_user))
    |                                          --- ^^^^^^^^^^ the trait `Handler<_, _, _>` is not implemented for fn item `fn(Uuid) -> impl std::future::Future<Output = User> {fetch_user}`
    |                                          |
    |                                          required by a bound introduced by this call
    |
    = help: the trait `Handler<T, S, B2>` is implemented for `Layered<L, H, T, S, B, B2>`
note: required by a bound in `axum::routing::get`
   --> /Users/alexander/.cargo/registry/src/github.com-1ecc6299db9ec823/axum-0.6.4/src/routing/method_routing.rs:403:1
    |
403 | top_level_handler_fn!(get, GET);
    | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ required by this bound in `axum::routing::get`
    = note: this error originates in the macro `top_level_handler_fn` (in Nightly builds, run with -Z macro-backtrace for more info)

For more information about this error, try `rustc --explain E0277`.
error: could not compile `axum-example` due to 2 previous errors
```

There's a lot going on here, but we can use the `axum-macros` crate and the `debug_handler` macro to get a better error message:

```bash
cargo add axum-macros
```

```rust
#[axum_macros::debug_handler] // new
async fn create_user(user: User) {
    todo!()
}

#[axum_macros::debug_handler] // new
async fn fetch_user(user_uid: Uuid) -> User {
    todo!()
}
```

This produces some even longer error messages, so let's just have a look at
part of one of them:

```bash
alexander@cinnamon ~/D/U/P/W/a/s/axum-example> cargo check
error[E0277]: the trait bound `User: FromRequestParts<()>` is not satisfied
  --> src/main.rs:24:28
   |
24 | async fn create_user(user: User) {
   |                            ^^^^ the trait `FromRequestParts<()>` is not implemented for `User`
   |
   = help: the following other types implement trait `FromRequestParts<S>`:
             <() as FromRequestParts<S>>
             // more suggestions following
```

Huh, `axum` seems to be confused about how to convert a request into our `User`
struct. This is essentially the equivalent of missing the
`@Consumes(MediaType.APPLICATION_JSON)` we had earlier in Java, except we're
seeing this when we run `cargo check` instead of testing the server.

So, what's the equivalent of `@Consumes` in the `axum` world? Well, there's a
type for that. See, instead of adding annotations to tell `axum` how to convert
a request into our custom struct, there's a type (`Json<T>`) that understands
how to convert from a request body.
