﻿---
title: "Strings"
description: >
    Introduction to Strings
---

Strings store sequences of bytes, including text, serialized objects, and binary arrays.
As such, strings are the simplest type of value you can associate with
a Valkey key.
They're often used for caching, but they support additional functionality that lets you implement counters and perform bitwise operations, too.

Since Valkey keys are strings, when we use the string type as a value too,
we are mapping a string to another string. The string data type is useful
for a number of use cases, like caching HTML fragments or pages.

```
127.0.0.1:6379> SET bike:1 Deimos
OK
127.0.0.1:6379> GET bike:1
"Deimos"
```

As you can see using the `SET` and the `GET` commands are the way we set
and retrieve a string value. Note that `SET` will replace any existing value
already stored into the key, in the case that the key already exists, even if
the key is associated with a non-string value. So `SET` performs an assignment.

Values can be strings (including binary data) of every kind, for instance you
can store a jpeg image inside a value. A value can't be bigger than 512 MB.

The `SET` command has interesting options, that are provided as additional
arguments. For example, I may ask `SET` to fail if the key already exists,
or the opposite, that it only succeed if the key already exists:

```
127.0.0.1:6379> set bike:1 bike nx
(nil)
127.0.0.1:6379> set bike:1 bike xx
OK
```

There are a number of other commands for operating on strings. For example
the `GETSET` command sets a key to a new value, returning the old value as the
result. You can use this command, for example, if you have a
system that increments a Valkey key using `INCR`
every time your web site receives a new visitor. You may want to collect this
information once every hour, without losing a single increment.
You can `GETSET` the key, assigning it the new value of "0" and reading the
old value back.

The ability to set or retrieve the value of multiple keys in a single
command is also useful for reduced latency. For this reason there are
the `MSET` and `MGET` commands:

```
127.0.0.1:6379> mset bike:1 "Deimos" bike:2 "Ares" bike:3 "Vanth"
OK
127.0.0.1:6379> mget bike:1 bike:2 bike:3
1) "Deimos"
2) "Ares"
3) "Vanth"
```

When `MGET` is used, Valkey returns an array of values.

### Strings as counters
Even if strings are the basic values of Valkey, there are interesting operations
you can perform with them. For instance, one is atomic increment:

```
127.0.0.1:6379> set total_crashes 0
OK
127.0.0.1:6379> incr total_crashes
(integer) 1
127.0.0.1:6379> incrby total_crashes 10
(integer) 11
```

The `INCR` command parses the string value as an integer,
increments it by one, and finally sets the obtained value as the new value.
There are other similar commands like `INCRBY`,
`DECR` and `DECRBY`. Internally it's
always the same command, acting in a slightly different way.

What does it mean that INCR is atomic?
That even multiple clients issuing INCR against
the same key will never enter into a race condition. For instance, it will never
happen that client 1 reads "10", client 2 reads "10" at the same time, both
increment to 11, and set the new value to 11. The final value will always be
12 and the read-increment-set operation is performed while all the other
clients are not executing a command at the same time.


## Limits

By default, a single String can be a maximum of 512 MB.

## Basic commands

### Getting and setting Strings

* `SET` stores a string value.
* `SETNX` stores a string value only if the key doesn't already exist. Useful for implementing locks.
* `GET` retrieves a string value.
* `MGET` retrieves multiple string values in a single operation.

### Managing counters

* `INCRBY` atomically increments (and decrements when passing a negative number) counters stored at a given key.
* Another command exists for floating point counters: `INCRBYFLOAT`.

### Bitwise operations

To perform bitwise operations on a string, see the [bitmaps data type](bitmaps.md) docs.

See the [complete list of string commands](../commands/#string).

## Performance

Most string operations are O(1), which means they're highly efficient.
However, be careful with the `SUBSTR`, `GETRANGE`, and `SETRANGE` commands, which can be O(n).
These random-access string commands may cause performance issues when dealing with large strings.

## Alternatives

If you're storing structured data as a serialized string, you may also want to consider Valkey [hashes](hashes.md).
