---
title: "Terraforming on the Fly"
date: 2022-10-02T18:54:59+01:00
draft: true
showtoc: true
---

[Terraform][terraform-homepage] is an Infrastructure as Code tool provided by
Hashicorp which allows you to provision or change your infrastructure through
the use of a declarative language called [HCL][hcl-syntax]. This allows you to
declare that you would like a server and run `terraform apply`, which will
communicate with the cloud provider to create the machine for you.

[`fly.io`][fly-io] is an cloud-based infrastructure provider that makes it
super simple to get an application up and running, with a global traffic proxy
and some nice metrics built-in. It also has a [Terraform
provider][terraform-provider-fly] in its early alpha stages.

See where we are going?

## Writing a simple program

To get started with an application on `fly.io`, we're first going to need
something to run. Let's start with a basic HTTP server just to check things are
working. I'll be using Rust, but any language that has an HTTP library should
work fine as we will package this into a Docker image later.

Let's create a new application and enter the directory:

```bash
cargo new --bin basic-fly-app
cd basic-fly-app
```

Then add some dependencies:

```bash
# HTTP library of choice
cargo add axum

# Asynchronous runtime for `axum`
cargo add tokio --features "macros rt-multi-thread"
```

Then write our basic HTTP server, which listens on port `8000` and has a single
route at the base (`/`) that will just return a `Hello World` style message.

```rust
// src/main.rs
use axum::{Router, Server};
use axum::routing::get;

#[tokio::main]
async fn main() {
	let app = Router::new().route("/", get(index));
	let addr = "[::]:8000".parse().unwrap();

	Server::bind(&addr)
		.serve(app.into_make_service())
		.await
		.unwrap();
}

async fn index() -> String {
	String::from("Hello from fly.io!")
}
```

We can use `cargo run` to build and run the binary before verifying it works as
expected:

```bash
alexander@cinnamon ~/D/U/P/W/a/s/basic-fly-app> cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.08s
    Running `target/debug/basic-fly-app`
Listening for requests on [::]:8000

# In another terminal
alexander@cinnamon ~/D/U/P/W/pages> curl localhost:8000
Hello from fly.io!⏎
```

## Packaging our application

The easiest way to get started with `fly.io` is through Docker images and the
use of Docker Hub. From there, we can simply define our application and Fly
will spin up a machine for us, pull the image from the registry and run it as a
container.

So, we will need a `Dockerfile` to define how to package it up. Since we don't
want to include the entire `/target` directory with all the build files we
produced earlier, let's also add a quick `.dockerignore`:

```
/target
```

Then our `Dockerfile` is as simple as follows, just build the binary and then
copy it into a smaller image:

```dockerfile
# Use something with Rust pre-installed to build the binary
FROM rust:latest AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

# Copy the binary to a smaller image for the runtime
FROM gcr.io/distroless/static AS runtime
COPY --from=builder /app/target/release/basic-fly-app .
ENTRYPOINT ["./basic-fly-app"]
```

[terraform-homepage]: https://www.terraform.io/
[hcl-syntax]: https://www.terraform.io/language/syntax/configuration
[fly-io]: https://fly.io
[terraform-provider-fly]: https://registry.terraform.io/providers/fly-apps/fly
