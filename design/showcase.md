## Networking

Async reading from a socket

## Generator

Take while / take until syntax

## Parsers

Mixing Lexer and Parser via coroutines

For example "4 abc0 abc1 abc2 abc3"

## Other Producer-Consumer workflow

Single-pass conversion of one-format to another

## Async generator


## Streams

## Hiding memory latency

## Multithreading

- Multithreaded async stream

## Cryptographic protocol

Require no allocation
Require complex state machines, for example TLS handshake.

## Kernel-mode driver

Require no allocation
Require complex state machines

## Saving/Suspending workflows

For example user authentication, registration, etc on a webpage
Requires the suspendable function to be serializable.

## Asynchronous garbage collection

(speculative)

The GC should be able to attach an async `=destroy` or finalizers
at the leaves of the "continuation graph"
