---
title: "Sybl: Accessible Machine Learning through Ensemble Learning"
date: 2021-05-04
draft: false
tags:
- rust
- vuejs
- nosql
- websockets
- machine-learning
---

Sybl is an accessible platform that allows users with low technical experience
to gain the benefits of machine learning by providing data. Clients provide
models for training and are reimbursed based on their performance on the
problem.

<!--more-->

Source: https://github.com/Sybl-ml/dodona/

## Overview

Sybl was designed and implemented as a team for the MEng CS407 Group Project at
the University of Warwick. It is a MLaaS (Machine Learning as a Service) system
and provides a streamlined and simple interface for users to get insights on
their structured data.

### Problem Context

Despite businesses and individuals are generating more data than ever, there is
still a high barrier for entry for analysis. Only large scale businesses can
afford to pay highly qualified data science experts for this process, with
smaller businesses and individuals unable to gain the benefits.

While other services in the MLaaS sector provide similar functionality to the
described Sybl system, they typically require users to have some technical
knowledge to get started. Services such as DataDog for example require users to
pick a classifier for their data, which could cause inaccurate predictions if
they did not know which ones would work well.

## Solutions

Instead of requiring users to pick a classifier, the Sybl system instead runs
multiple classifiers on the user's data and combines their results based on
their individual performances. This uses a technique known as ensemble methods
and allows users to simply upload data and the system will choose an
appropriate set of classifiers for them.

## Implementation

The implementation of the system was written in a mixture of [vuejs][vuejs],
[Rust][rust-lang] and [Python][python].

[vuejs]: https://vuejs.org/
[rust-lang]: https://www.rust-lang.org/
[python]: https://www.python.org/
