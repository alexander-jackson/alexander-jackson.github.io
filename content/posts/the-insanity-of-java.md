---
title: "The Insanity of Java"
date: 2022-11-12T20:33:46Z
draft: true
showtoc: true
---

Java is a highly infamous language for a multitude of reasons. Once touted as a
successor to C and something we would be writing operating systems in, it's now
one of the most common languages for enterprise systems. Source code compiles
to bytecode which is run on the Java Virtual Machine (JVM) allowing programs to
be highly portable and predictable, alongside preventing memory unsafety
through the use of garbage collection.

However, it does have its quirks.

## Converting Strings to UUIDs

A UUID (universally unique identifier) is comprised of 128 bits (16 bytes),
meaning there are around 340 trillion trillion trillion trillion possible
values. This allows them to be randomly generated and almost guarantee they are
universally unique.

We can also represent a UUID as a 32 character hexadecimal string, usually
split into 5 segments. An example might be
`116b418b-4edb-4250-ae85-6aeb9f01a9ba`, where we have blocks of length `[8, 4,
4, 4, 12]`, with the first digit of the 3rd block telling you the UUID version.
This particular one is version 4, which means it was randomly generated. Other
algorithms might use the time and MAC address.

Java provides a `java.util.UUID` type to represent these objects natively. We
can generate a random one by doing `UUID.randomUUID()`. However, sometimes we
might be receiving these UUIDs from an external source. Maybe a client is
sending us one, or we are storing them in the database and then reading them
later. These will be represented as strings, so Java provides the
`UUID.fromString` method to convert from a `String` to a `UUID`.
