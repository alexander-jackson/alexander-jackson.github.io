---
title: "Rust on AWS Lambda"
date: 2022-09-30T20:43:15+01:00
draft: true
showtoc: true
---

Prior to December 2020, AWS Lambda functions were executed on a runtime that
users could pick from, such as `nodejs16.x` for JavaScript, `python3.9` for
Python and `provided.al2` for custom runtimes.

Due to this being a limited set maintained by AWS themselves, this meant that
if you were using a language that didn't have a provided runtime, you weren't
able to use the service without rewriting in a supported language. Rust is an
example of a language without a supported runtime.

However, this changed when AWS [announced][aws-lambda-containers-announcement]
support for container images on the first of the month. Many workloads and
companies were already utilising containers for other deployments, so this
enabled the use of any language that could interpret the inputs from Lambda.

## Getting Started

[aws-lambda-containers-announcement]: https://aws.amazon.com/blogs/aws/new-for-aws-lambda-container-i
