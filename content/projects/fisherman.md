---
title: "Fisherman: Continuous Deployment"
date: 2021-04-24
description: "Something"
draft: false
tags:
- rust
- continuous-deployment
---

Fisherman is a continuous deployment tool mainly targeted at Rust projects
that listens for GitHub webhooks to rebuild and deploy changes.

<!--more-->

Source: https://github.com/alexander-jackson/fisherman/

## Overview

`fisherman` listens for [webhook messages][webhooks] sent by GitHub whenever a
project is changed. For `fisherman`, the main focus is the [push
event][push-event] which occurs whenever a branch is updated, either through a
`git push` command or merging of a pull request. The program will [validate the
webhook message][validation] before checking whether it is following the pushed
branch for that repository. If it is, it will pull the changes locally, rebuild
all the associated binaries and restart them within `supervisord`.

It was initially written to deploy [`blackboards`][blackboards] as it had
become cumbersome pushing updates to GitHub and then manually connecting to the
server and rebuilding the binaries. Usage of `fisherman` allowed this process
to be automated, reducing the chance of forgetting steps or changes not being
deployed for a period of time.

### Discord Integration

`fisherman` also supports notifying of rebuilds through Discord messages. This
allows developers to be made aware of successful deployments and errors quicker
than checking the server for updates.

## Success Stories

`fisherman` is currently used to continually deploy several of my other
projects, most notably [`blackboards`][blackboards] but also [Pythia][pythia]
(Discord bot for project management) and `fisherman` itself across various
servers.

It was also utilised during our MEng group project to deploy the [`Sybl`][sybl]
system when changes were made to the `develop` branch. This required numerous
updates to `fisherman` itself, as it needed the ability to build multiple
binaries for the distributed system, following a branch other than the default
and running arbitrary commands to build and deploy the frontend written in
[Vue.js][vuejs].

[webhooks]: https://docs.github.com/en/developers/webhooks-and-events/webhooks/about-webhooks
[push-event]: https://docs.github.com/en/developers/webhooks-and-events/webhooks/webhook-events-and-payloads#push
[validation]: https://docs.github.com/en/developers/webhooks-and-events/webhooks/securing-your-webhooks
[blackboards]: https://alexander-jackson.github.io/projects/blackboards/
[pythia]: https://github.com/Sybl-ml/pythia
[sybl]: https://github.com/Sybl-ml/dodona
[vuejs]: https://vuejs.org/
