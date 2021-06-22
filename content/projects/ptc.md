---
title: "ptc: Experimental Python to C Compiler"
date: 2020-05-09
draft: false
tags:
- rust
- compiler
---

`ptc` is an experimental compiler that converts Python source code into the C
language and was written as my dissertation project.

<!--more-->

Source: https://github.com/alexander-jackson/ptc

## Overview

`ptc` takes code written in a subset of the Python language and converts it to
equivalent C code. Given the following Python code in `basic.py`:

```python
def fibonacci(n: int):
    first = 0
    second = 1
    c = 1
    total = 0
    next = 0

    while c < n:
        if c <= 1:
            next = c
        else:
            next = first + second
            first = second
            second = next

        c += 1
        total += next

    return total
```

The compiler will lex and parse the program into an abstract syntax tree before
performing type analysis and producing the following in `basic.c`:

```c
#include "basic.h"

int fibonacci(int n) {
	int first = 0;
	int second = 1;
	int c = 1;
	int total = 0;
	int next = 0;

	while (c < n) {
		if (c <= 1) {
			next = c;
		} else {
			next = first + second;
			first = second;
			second = next;
		}

		c += 1;
		total += next;
	}

	return total;
}
```

It will also produce a header file called `basic.h` to allow the code to be easily included in other C projects:

```c
#ifndef BASIC_H
#define BASIC_H

int fibonacci(int n);

#endif /* BASIC_H */
```

## Motivation

This can be used for various reasons, such as:

- Increased performance
- Ease of development
- Basic type checking

### Performance

Due to being a typed and compiled language, C is generally more performant than
Python as the compiler can optimise code based on the entire program content
and knowledge of the types used. During the development of the compiler,
speedups of around 60x were achieved on scientific-focused programs after
compilation. Additionally, speedups of 7x were achieved on the same program
even when compiling first through `ptc`, then `gcc` and finally executing.

### Ease of Development

Python is regarded as one of the easiest languages to learn for developers and
is generally much faster to write than C code due to the lack of strict typing
and higher level abstractions. Thus, developers can write code faster while
gaining the benefits of compiled binaries.

### Type Checking

As `ptc` aims to produce valid C code, it must infer the types of variables,
function arguments and returns. It will still produce outputs if it cannot, but
these will have malformed types which when provided to a C compiler will cause
errors. Thus, users can run `ptc` first to perform type inference and then
`gcc` or `clang` to check that these are valid.
