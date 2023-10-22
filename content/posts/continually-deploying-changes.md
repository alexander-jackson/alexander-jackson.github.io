---
title: "Continually Deploying Changes"
date: 2023-10-07T12:29:20+01:00
draft: true
showtoc: true
---

My fourth year of university took place in the midst of the COVID-19 pandemic,
and as such we needed a way to control the number of members turning up to the
powerlifting sessions the club I was part of organised. We had experimented
with Google Forms, but it wasn't quite flexible enough for what we wanted.

Instead, I wrote a web server called `blackboards`. It was written primarily in
Rust and ran on a single Digital Ocean instance alongside the PostgreSQL
installation. This was likely overkill, but it was good fun and taught me a lot
about software development and delivery.

### Manually Deploying Changes

As expected in the beginning, changes were deployed manually. I'd write some
code, test it locally and commit to `master` before pulling the changes onto
the instance and building the new binary. From there, it was just a case of
stopping the old one and starting the new one (with a little bit of downtime in
the middle).

This worked fine, but it never felt very good running `ssh` to get onto the
production instance. It also added manual steps to the process, increasing the
chance of mistakes and accidental downtime. The ideal world involved pushing
code and having it build and deploy itself without any human intervention.

### Writing a Continuous Deployment Tool

Once I had some free time from adding new features to the server I began
working on another project called `fisherman`, designed to replace my manual
steps. It expected to receive GitHub webhooks from pushes to `master`, at which
point it would:

* Pull the newest changes onto the instance
* Invoke `cargo build --release` to get a new binary
* Ask `supervisord` to restart the project

This worked pretty well. It required some extensions over time to handle
non-Rust projects (such as a Python-based websocket server for another
university project) and support for monorepos (since we had multiple
applications in a single repository for our group project) but it could
generally be relied on to deploy new changes without any other inputs.

It did, however, have some major limitations. My Digital Ocean server was not
particularly big (originally using the smallest droplet size but slowly being
upgraded over time) and thus did not have as much compute power as it might
have required.

Remember, `fisherman` just invoked `cargo build --release` on the production
server, which could cause some serious CPU and memory usage. Some of the
binaries we were running had ~2 minute recompile times due to the size of the
instance, which would cause small outage periods as the CPU struggled.

At times, the compilation would even fail due to out-of-memory issues which
meant we needed to increase the compute capacity just to run these builds. This
didn't make much sense in reality, the instance barely used any compute outside
of these recompilations so we were just wasting money.

Regardless, `fisherman` was a great experience (ignoring some minor outages). I
pushed code and the server would automatically update to the new binaries and
let me know on Discord that it had done so.

### Foraying into Kubernetes

After graduating from university, I had my first taste of proper production
software engineering. Our deployments were a mixture of EC2 instances and
Kubernetes workloads running on EKS with everything being containerised, with
the infrastructure team being hard at working migrating the legacy EC2 services
into Kubernetes.

On our cluster we were running a Kubernetes extension called Flux, which would
regularly poll a GitHub repository that contained our YAML service definitions.
If it detected any changes it would pull them onto the cluster and apply them,
which would deploy the latest state.

This was something that interested me greatly. I'd garnered a few more
applications on the Digital Ocean server I was operating and it was beginning
to become a hassle to maintain `fisherman`, especially as the projects became
larger and required more compute to recompile every time. Deploying containers
felt like a major improvement over the current method, since it meant the build
phase could happen in GitHub Actions.

Kubernetes and Flux provided additional benefits outside of the obvious
differences to `fisherman`. It had support for detecting new images in Docker
repositories and rolling them out, sensible deployments with health checks and
Flux provided notifications for events into Discord.

#### Compute Constraints

Almost every cloud provider these days has a Kubernetes offering. AWS has
Elastic Kubernetes Service (EKS), GCP has Google Kubernetes Engine (GKE) and
Digital Ocean itself had just released their Kubernetes offering when I began
debating the switch.

However, all of these were relatively expensive for what I wanted. AWS began at
around $70/month, Google wasn't much better and even Digital Ocean started
at $20/month for the cheapest 2 node offering with a control plane. I didn't
*need* multiple nodes or massive amounts of fault tolerance for what I was
doing, I just wanted to run some containers.

So, I started with my own self-managed cluster through `kubeadm`. This took a
long while to get set up (since there's lots of functionality that needs to be
separately installed, and it really disagrees with a single instance being the
control plane and the only cluster node), but eventually I had something that
worked and looked at my `infrastructure` repository for changes.

#### Honeymoon Phase

This workflow was great. Much like in the `fisherman` world, I'd just push
changes to the repository and new images would be built. Flux would notice the
tag update and push a change to the `infrastructure` repository which would
then be reconciled onto the cluster.

Unfortunately after a year, my TLS certificates expired. This meant I couldn't
even talk to my own cluster, which was a bit of an issue since the database
currently lived as a pod inside it and I was slightly worried I'd lost all the
data.

Having battled it for a while (and luckily extracting the data from the
PostgreSQL pod) I eventually gave up. By this point I was running a fairly
large instance anyway to cope with all the CPU requests that various Kubernetes
workloads were asking for (even if they didn't need them) and started looking
at managed solutions.

I experimented with the Digital Ocean offering as that was the cheapest of all
the providers, but even the smallest node setup didn't have capacity for Flux
alongside the basic setup (despite me running 2 nodes and a control plane).

### Returning to Manual

At this point, I had a cluster that still worked but I couldn't interact with
which felt like a ticking timebomb. I started a new instance, installed Docker
and booted the containers from the Kubernetes cluster before failing the
traffic over. This didn't have *any* nice features on it, and I was back to
logging into the server myself to stop containers and start new ones.

Thus, I began returning to `fisherman`. Well not entirely, but the same sort of
idea. I wanted what Kubernetes had given me in terms of containers, what Flux
provided in Git-based configuration and automated reconciliation and
`fisherman` had in simplicity and low compute cost.

So, I started working on the second iteration of `fisherman`, named `f2`.

### Writing a Container Orchestrator

From the offset, `f2` needed a couple of components:

* Ability to manage containers
* Support for load balancing requests

In the beginning it was going to be hosted behind `nginx` since I didn't fancy
handling any of the TLS myself, so all the SSL connections would be terminated
there and `f2` would handle the request routing into the containers.

However, once these were done it turned out to be relatively simple to handle
TLS myself (thanks to the `rustls` library) and perform remote configuration
reconcilation. This allowed the `config.yaml` file to be stored in S3, with
`f2` being informed when it had changed so it could fetch the new state.

In the background, I'd also been migrating everything I had into Terraform.
This served both to learn how to use it but also for much easier management of
any changes I wanted to make. The `aws_instance` resource contains a
`user_data` property which provides a script for the instance to run on
startup.

So, I wrote a small script that installed Docker and ran `f2` on a specific
version with a given configuration file location. This allowed me to define
something like:

```tf
module "instance" {
  source = "./modules/f2-instance"

  name          = "instance"
  tag           = "20231018-1716"
  config_arn    = module.config_bucket.arn
  config_bucket = module.config_bucket.name
  config_key    = "f2/config.yaml"
  vpc_id        = aws_vpc.main.id
  subnet_id     = aws_subnet.main.id
  ami           = "ami-0ab14756db2442499"
  instance_type = "t2.nano"
  key_name      = aws_key_pair.main.key_name
}
```

This would then create all the necessary resources (security group and rules,
IAM role and policy) alongside the instance itself. Upon startup it would
install Docker and run `f2`, loading the configuration from an S3 bucket. I
could then point a DNS record at it (once it had started up and everything
looked healthy) and it would begin automatically rolling out new versions of
containers and handling traffic.
