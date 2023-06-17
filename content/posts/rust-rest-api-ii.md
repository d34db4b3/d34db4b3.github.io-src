+++
title = "A modern REST web service, written in Rust, Part II"
date = 2023-06-11 19:14:18
updated = 2023-06-11 19:14:26
authors = ["d34db4b3"]
description =  "We will dive into web back-end design with a simple REST API, using the reliable and efficient programming language Rust"
draft = true
[taxonomies]
tags = ["rust", "web", "backend", "REST"]
+++

# TLDR

> TODO

# What could go wrong ?

Before trying to handle errors, we need to know what could cause them. There is a good [chapter](https://doc.rust-lang.org/book/ch09-00-error-handling.html) about errors in the Rust book.

## Admit defeat

Some errors that happen in our process lifetime are not detectable and/or we cannot recover from them. This includes for example some signals:
- SIGKILL: cannot be caught/ignored by the process, terminates it
- SIGSTOP: cannot be caught/ignored by the process, pause it until SIGCONT
- SIGSEGV: illegal memory access, continuing execution would be undefined behavior

Generally, an out-of-memory situation leads to a process to be killed.

Some external events could also impact our poor process:
- Power failure
- System failure
- Hardware fault
- Being swallowed by a blackhole

We unfortunately cannot recover from those and it will lead to a service interruption (there are some option to avoid an impact on the end user but it will not be discussed here).

## "Hard to recover from" errors

Some errors are detectable and can be handled by our process but are not trivial to recover from.
For example, our system can:
- run out of disk space: what it we need to store data/logs ?
- get disconnected from the network: what if we need access to other services ?
- have corrupted data: how do we access the data ?
- some service we depend on fails/is not accessible (database, etc.): how do we store/get our data ? 
<!-- - get overloaded (cpu/io/network): what if we have some timing constraints ? (this one is not really an error but you may then encounter some timeouts) -->

The easier way to handle those is to return an internal error (at least temporarily) to the users of aur process.
It is not necessary/desired to expose the actual reason for the error to the user as he probably can't do anything about it.

It is also possible to handle some of them depending on the situation. For example, non critical logs could be rotated/truncated forcefully on disk exhaustion (and preferably before complete exhaustion) and switch to a less aggressive logging strategy (only logging warnings/errors).

We can either panic or represent the error as a `Result::Err` or equivalent, knowing that explicitly handling errors is generally beneficial for robustness and clarity.

## The bugs

Even the best&#2122; language cannot prevent the even better programers from every bug. We have some guaranties about memory and thread safety and if the tooling helps avoiding logic mistakes, someone's "bug" is someone else's feature.

The symptoms of these errors can be:
- an invalid/unexpected result (worst case)
- a panic (unexpected failure in the program)

Recovery from the first case is not really possible as there is *no* actual error. Just an unexpected erroneous result, but from a human/client perspective only. The program did what it was told to, as it does most of the time (see "cosmic ray bitflip" to have some bad dreams).

Panicking in rust *can* (but not always - see panic abort) be caught and handled. This is generally handled the same as for internal errors. You could provide some information about the issue to the client because panics can "return" some context but as it was not expected, it not a very good idea and could lead to some undesired data leak (exposing internal/implementation details, internal data, etc.).

## The easy ones

The rest of the errors now consist of "expected" errors, generally represented by a `Result::Err` or equivalent.
Of course a programer can always chose to "ignore/not handle" or thinks that it cannot happen by `unwrap`ing an error but in this case, we panic in there is indeed an error and we fall in the previous category.

Handling those is quite trivial and we can either:
- bubble them up (see `?`)
- change their "content" (se `map`)

In the context of a REST API, we will necessarily convert the error into some kind of HTTP body at some point.

# Describing our error types
