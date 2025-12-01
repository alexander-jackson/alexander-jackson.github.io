---
title: "Overriding Kubernetes HPAs with Rust"
date: 2023-09-26
showtoc: true
---

[Horizontal Pod Autoscalers][kube-hpas] (HPAs) are a common concept in the
world of Kubernetes. They allow you to specify that your workloads should scale
based on various factors, such as their CPU or memory consumption. You can also
extend them with custom plugins such as the
[prometheus-adapter][prom-adapter-github] to scale based on Prometheus metrics.

However, sometimes you just want to tell Kubernetes "give me more pods".

Perhaps you've built up a backlog somewhere in the system, such as a processing
queue. The database looks relatively calm and the pods are chugging away, but
not fast enough. The HPA likely won't kick in, there's not enough work being
done.

Sure, you could set up metrics for this, but it's unlikely you'll cover every
use case or reason. Sometimes you know better than the cluster.

## General Approach

In an ideal world, we'd just change the HPA to have more pods. This might lead
to other problems though, since:

* You may forget to revert the change, causing drift between your YAML files
  and actual state
* Kubernetes might reconcile the original values and reset them for you
* You may want more validation in place for these overrides

One way to do this is by combining a [custom resource definition][kube-crds]
(CRD) with a controller for that resource. This allows you to:

* Create an instance of the CRD
* Allow the controller to validate it
* Merge it with the HPA using the controller

This allows us to modify the state of the cluster on an adhoc basis, while
having more control over what happens. We can also add custom logic, such as
automatically expiring the overrides after a certain period of time.

## Setting Up

Luckily for us, Rust has a [library][docs-rs-kube] for interacting with
Kubernetes that supports both generation of our own CRDs and writing
controllers for resources.

We'll need 3 main components here:

* A library that provides the CRD
* A binary that outputs the YAML definition for the cluster
* A binary that acts as the controller

`cargo` supports workspaces, so we can write a top-level `Cargo.toml` for the
workspace with our library as follows:

```toml
[workspace]
members = [
    "custom-resource",
]
```

### Writing CRDs

Let's start out by writing the simple CRD to represent our override:

```bash
cargo new --lib custom-resource
cd custom-resource
```

We'll need to add a dependency on the `kube` crate, as well as some related
libraries:

```bash
# Kubernetes types and resources
cargo add kube -F derive --no-default-features
cargo add k8s-openapi -F v1_26

# Serialisation and deserialisation primitives
cargo add serde -F derive
cargo add serde_json

# Ability to generate JSON schemas (for OpenAPI specs)
cargo add schemars
```

From here, we can define the underlying specification for our CRD and allow the
`kube-derive` crate to generate the relevant implementation for it:

```rust
use std::num::NonZeroU32;

use kube::CustomResource;
use schemars::JsonSchema;
use serde::{Deserialize, Serialize};

#[derive(Clone, Debug, CustomResource, Serialize, Deserialize, JsonSchema)]
#[kube(
    group = "foo.bar",
    version = "v1",
    kind = "AutoscalerOverride",
    namespaced
)]
pub struct AutoscalerOverrideSpec {
    namespace: String,
    hpa: String,
    min_replicas: NonZeroU32,
    max_replicas: NonZeroU32,
}
```

Our resource has 4 properties, being the namespace of the HPA we'd like to
override, the name of it and the new minimum and maximum values to overlay.

This will generate an `AutoscalerOverride` type that we can use elsewhere in
the code. The snippet above will form the base of our CRD and we will write 2
other components, one for generating the definition to apply to the cluster and
the other to control it from within the cluster.

### Generating the Definition

Now that we have a `custom-resource` library that defines the structure of the
CRD and generates the relevant traits, we can add a small binary that depends
on it and produces the YAML specification for the Kubernetes cluster. Let's add
a new member to our workspace:

```toml
# Cargo.toml
[workspace]
members = [
    "custom-resource",
    "custom-resource-generator",
]
```

Generate our new binary:

```bash
cargo new --bin custom-resource-generator
cd custom-resource-generator
```

Add a dependency on our CRD, as well as the `kube` library:

```bash
cargo add custom-resource --path ../custom-resource
cargo add kube -F derive --no-default-features
```

From here, we can update our `main` function to get the resource definition,
serialize it intoa YAML format and display it:

```rust
use custom_resource::AutoscalerOverride;
use kube::core::CustomResourceExt;

fn main() {
    let crd = AutoscalerOverride::crd();
    let yaml = serde_yaml::to_string(&crd).expect("Failed to serialize CRD");

    println!("{yaml}");
}
```

This should display something such as:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: autoscaleroverrides.foo.bar
spec:
  group: foo.bar
  names:
    categories: []
    kind: AutoscalerOverride
    plural: autoscaleroverrides
    shortNames: []
    singular: autoscaleroverride
  scope: Namespaced
  versions:
  - additionalPrinterColumns: []
    name: v1
    schema:
      openAPIV3Schema:
        description: Auto-generated derived type for AutoscalerOverrideSpec via `CustomResource`
        properties:
          spec:
            properties:
              hpa:
                type: string
              max_replicas:
                format: uint32
                minimum: 1.0
                type: integer
              min_replicas:
                format: uint32
                minimum: 1.0
                type: integer
              namespace:
                type: string
            required:
            - hpa
            - max_replicas
            - min_replicas
            - namespace
            type: object
        required:
        - spec
        title: AutoscalerOverride
        type: object
    served: true
    storage: true
    subresources: {}
```

This is what we will provide to Kubernetes to allow it to understand and handle
our new resource type. It mainly contains details about the schema of the type,
such as that we require at least 1 for the minimum and maximum replicas due to
the usage of `NonZeroU32`.

### Writing the Controller

Now that we have a custom resource defined in Rust and YAML, we'll need to
implement a controller for it which will run inside the cluster. This will be
notified of changes to our overrides and allow it to apply the relevant
modifications to the horizontal pod autoscalers themselves.

As before, let's add another binary to our workspace with a dependency on the
custom resource crate:

```toml
# Cargo.toml
[workspace]
members = [
    "custom-resource",
    "custom-resource-generator",
    "controller",
]
```

```bash
cargo new --bin controller
cd controller

cargo add custom-resource --path ../custom-resource
```

We'll need to make requests to the Kubernetes API server as well as listen for
incoming events, so we can add an asynchronous runtime as well as the `kube`
dependencies we've been using so far. Since this will run inside the cluster as
a proper application, we'll likely also want some more sophisticated logging
and error handling.

```bash
# Asynchronous runtime
cargo add tokio -F "macros rt-multi-thread"

# Kubernetes dependencies
cargo add kube -F "runtime client"
cargo add k8s-openapi -F "v1_26"

# Logging/tracing
cargo add tracing
cargo add tracing-subscriber -F env-filter

# Error handling
cargo add color-eyre

# Serializing our overrides for the API server
cargo add serde -F derive

# Helpers for working with streams of asynchronous data
cargo add futures-util
```

From here, we can begin defining our controller. We'll use the Kubernetes
`patch` method to apply our changes to the relevant horizontal pod autoscaler,
so let's define the type for that:

```rust
#[derive(Debug, Serialize)]
#[serde(rename_all = "camelCase")]
struct HPA {
    min_replicas: NonZeroU32,
    max_replicas: NonZeroU32,
}

#[derive(Debug, Serialize)]
struct PatchDetails {
    spec: HPA,
}
```

We'll be using the [`Controller`][kube-controller] type from the [`kube`][docs-rs-kube] crate to implement this, meaning we'll need to define:

* A function to be run when an object we care about is modified
* An error handler in case the reconciliation above fails

First, let's define some types for these functions to take:

```rust
// The type that we care about changes to
type Override = Arc<AutoscalerOverride>;

// The type of any potential errors during reconciliation
type Error = kube::Error;

// The context the handler gets, we don't need any
type Context = Arc<()>;
```

We can then define what our reconciler function should do:

```rust
/// Receives changes to the CRD from the API server and decides how to apply them.
async fn reconciler(override: Override, _ctx: Context) -> Result<Action, Error> {
    let spec = override.spec.clone();

    let client = Client::try_default().await?;
    let autoscalers: Api<HorizontalPodAutoscaler> = Api::namespaced(client, &spec.namespace);

    tracing::info!("Got an event, parameters are {spec}");

    // Patch the HPA based on the incoming event
    let params = PatchParams::default();
    let patch = Patch::Strategic(PatchDetails {
        spec: HPA {
            min_replicas: spec.min_replicas,
            max_replicas: spec.max_replicas,
        },
    });

    autoscalers.patch(&spec.hpa, &params, &patch).await?;

    Ok(Action::requeue(ONE_HOUR))
}
```

The logic for the reconciler is relatively simple, since all it needs to do is:

* Get the incoming event
* Connect to the API server
* Create a patch request
* Apply it to the specified resource

The error handler is similarly basic:

```rust
/// Handles errors when reconciling, simply by asking to retry a minute later.
fn error_handler(object: Object, error: &Error, _ctx: Context) -> Action {
    tracing::error!("Got an error ({error}) on object {object:?}");

    Action::requeue(ONE_MINUTE)
}
```

Since there's not much we can do in the case of an error at the moment (as the
logic is so simple it was likely just a connection error), we just log out the
error for debugging purposes and ask the API server to try again in a minute.

Now that we have defined the reconciler and the error handler, we can create a
new instance of the [`Controller`][kube-controller] and provide our functions
to it:

```rust
/// Entry point for the controller itself.
///
/// Sets up the watching of the CRD along with the reconciler that will overlay onto the HPA.
async fn run_controller() -> Result<()> {
    let client = Client::try_default().await?;

    let overrides: Api<AutoscalerOverride> = Api::all(client.clone());
    let autoscalers: Api<HorizontalPodAutoscaler> = Api::all(client.clone());
    let context = Arc::new(());

    let controller = Controller::new(overrides, Config::default())
        .owns(autoscalers.clone(), Config::default())
        .run(reconciler, error_handler, context)
        .for_each(|res| async move {
            match res {
                Ok(o) => tracing::info!("Reconciled {:?}", o),
                Err(e) => tracing::error!("Reconcile failed: {:?}", e),
            }
        });

    tracing::info!("Running the controller to begin waiting for events");

    controller.await;

    Ok(())
}
```

From here, we just need our `main` function to set up the logging for the
system and start our controller:

```
#[tokio::main]
async fn main() {
    tracing_subscriber::fmt().init();

    run_controller().await.expect("Failed to run controller");
}
```

## Applying Overrides

Now that we've defined our custom resource and matching controller, we can
start it by running `cargo run` in the `controller` directory. We should then
see our start up message (along with no errors) which indicates that it has
successfully connected to the cluster and registered itself, ready to receive
events.

We can then create a small YAML file containing the definition of the HPA we'd
like to override:

```yaml
# hpa-override.yaml
apiVersion: autoscaleroverrides.foo.bar/v1
kind: AutoscalerOverride
spec:
  hpa: baz-service-hpa
  namespace: some-team
  min_replicas: 10
  max_replicas: 20
```

Upon applying that to the cluster:

```bash
kubernetes apply -f hpa-override.yaml
```

we should then see the controller be informed of the change and update the
`baz-service-hpa` in the `some-team` namespace (if it exists) to have a minimum
of 10 replicas and a maximum of 20.

This allows us to quickly and easily update the number of replicas for a
service without needing to change the original files.

[kube-hpas]: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/
[prom-adapter-github]: https://github.com/kubernetes-sigs/prometheus-adapter
[kube-crds]: https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/
[docs-rs-kube]: https://docs.rs/kube
[kube-controller]: https://docs.rs/kube/latest/kube/runtime/struct.Controller.html
