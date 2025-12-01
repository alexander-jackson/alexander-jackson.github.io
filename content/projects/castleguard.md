---
title: "CASTLEGUARD: Anonymised Data Streams with Guaranteed Differential Privacy"
date: 2020-03-18
tags:
- python
- security
- anonymisation
- privacy
---

CASTLEGUARD is an extension of the CASTLE algorithm and provides the same
guarantees as its predecessor with the addition of differential privacy. It was
written as part of a team for CS347 Fault Tolerant Systems and published later
that year.

<!--more-->

Source: https://github.com/hallnath1/CASTLEGUARD/

## Overview

The original CASTLE algorithm is a streaming approach for data privacy, aimed
at big data and scenarios where not all the data can be analysed at once. It
takes a stream of incoming data tuples and inserts each into a specific cluster
based on its distance to their centres. On each insertion, it can then output a
cluster itself, with each tuple inside anonymised based on the contents. This
provides guarantees of `k-anonymity` and `l-diversity`.

For this project, we worked as a team to implement the original CASTLE
algorithm itself based on the research paper it was published in. This involved
understanding academic content and ensuring it met the guarantees described,
with heavy testing on large datasets and with malformed data.

### Extending CASTLE

During the development of the algorithm, the team realised that we could
additionally provide the guarantee of differential privacy for output tuples by
using a method called Bernoulli sampling. Whenever a tuple was inserted to the
container, we could randomly decide to ignore it. This would mean that some
tuples would not be output and thus attackers would not know which ones had
been input.

After presenting our work to our supervisor and achieving a high mark, we went
on to publish the improved algorithm at the IEEE International Conference on
Dependable, Autonomic and Secure Computing 2020 in Calgary, Canada.

## Publication

The full published paper can be read [here][publication].

[publication]: https://ieeexplore.ieee.org/document/9251212/
