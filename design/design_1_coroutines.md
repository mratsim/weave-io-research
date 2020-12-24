# Coroutines-based design

> Coroutines are a one-shot delimited continations
> - Author of "Monoids are monads in the category of endofunctors"

This document outlines a "new" primitive for Nim called "coroutine".
For the introduction we use the historical name subroutine to mean procedure or function.

Traditional 'subroutines' have 2 control-flow capabilities:
- When a subroutine is 'called', the caller suspends itself and jump into the subroutine body.
- When a subroutine 'returns', the subroutine returns control back to the caller.

Coroutines have the same capabilities and add the following:
- Coroutines can suspend themselves, at specific suspension point, usually called 'yield'.
- A resumer, which may be different from the initial caller, can resume a coroutine at the previous yield point.

Coroutines come in flavors:
- asymmetric vs symmetric.\
  An asymmetric coroutine can only hand control back to its current owner (caller or resumer).
  They have asymmtric power.
  A symmetric can switch to any other coroutine and not only its caller/resumer.
  Asymmetric and symmetric can be implemented in terms of each other (with overhead).
  The difference is in the ergonomics we want by default.
- stackful vs stackless\
  Beware of some conflicting terminology between languages.
  We define stackful coroutines as coroutines that carry a full independant stack.
  That stack need to be reinstalled on coroutine switch.
  In the rest of the document we call stackful coroutines `fibers`,
  they are also known as green threads or user-mode threads in the litterature.\
  We define stackless coroutines as coroutines that use the stack of their caller for nested function calls.
  `coroutines` will refer to stackless `coroutines` in the future.

The rest of the document will propose:
1. Coroutine ergonomics
2. Use-cases
3. Design constraints and choices
4. Low-level implementation
5. Interesting capabilities enabled
6. Extensibility and Limitations

## Coroutine ergonomics

I choose asymmetric stackless coroutines. They closely extends current closure iterators capabilities.
- The 'asymmetric' part is relevant when we want to control coroutine scheduling.
  We will show an efficient trick with low overhead to avoid switching control to the caller.
- Fibers/green threads have shown time and time that managing userland stacks are very error prone.
  Furthermore they have higher overhead than stackless coroutines as they always require allocating pages on the heap for the stack. They also require assembly for stack switching. They cannot be used for embedded, kernel drivers or Javascript/WASM. See also "Fibers under the magnifying glass "http://www.open-std.org/JTC1/SC22/WG21/docs/papers/2018/p1364r0.pdf".

### Iterators example

As a reminder here is the syntax for closure iterators from https://nim-by-example.github.io/for_iterators/
```Nim
proc countTo(n: int): iterator(): int =
  return iterator(): int =
    var i = 0
    while i <= n:
      yield i
      inc i

let countTo20 = countTo(20)

echo countTo20()
```

And for chained iterators

```Nim
from sugar import `=>`

proc takeWhile*[T](iter: iterator(): T, cond: proc(x: T):bool): iterator(): T =
  result = iterator(): T =
    var r = iter()
    while not finished(iter) and cond(r):
      yield r
      r = iter()

let countTo4 = countTo(20).takeWhile(x => x <= 4)

while not countTo4.finished():
  echo countTo4()
```

Note: there is a bug here this prints "0 1 2 3 4 0"

Here is the corresponding example of a simple coroutine

```Nim
coro countTo(n: int): int =
  var i = 0
  while i <= n:
    yield i
    inc i

let countTo20 = countTo(20)

echo countTo20.resume()
```

And for chained coroutines

```Nim
from sugar import `=>`

coro takeWhile*[T](generator: coro(): T, cond: proc(x: T):bool): T =
  var r = generator.resume()
  while not finished(generator) and cond(r):
    yield r
    r = generator.resume()

let countTo4 = countTo(20).takeWhile(x => x <= 4)

while not countTo4.finished():
  echo countTo4.resume()
```

For generators this play well with `collect`

### Async IO/streams example

We take for example reading from a file by chunks of 4K
and interleave "expensive processing" after each chunk read.
We assume that there is no stream logic underneath and so
we need to keep state of the reads ourselves
(memory-mapped file, TCP connection, parser, ...)

#### Synchronous interleaving via callback

```Nim
proc process(path: string,
             buf: var array[4 * 1024, byte],
             expensiveWork(buf: var array[4 * 1024, byte]){.nimcall.}): int =
  let f = openFile(file)
  var curIdx = 0
  while true:
    let numBytes = f.readBytes(curIdx, buf)
    curIdx += numBytes
    expensiveWork(buf, numBytes)
    if numBytes == 0:
      break
  return result
```

#### Asynchronous stream with coroutines

```Nim
coro asyncReader(path: string, buf {.resume.}: var array[4 * 1024, byte]): int =
  # <---- Implicit suspension
  let f = openFile(path)
  var curIdx = 0
  while true:
    let numBytes = f.readBytes(curIdx, buf)
    curIdx += numBytes
    yield numBytes # <---- Explicit suspension
    if numBytes == 0:
      break

let reader = asyncReader("path/to/greatness")
var buf: array[4 * 1024, byte]
for numBytes in reader.resume(buf):
  expensiveWork(buf, numBytes)
```

#### Comments

As you can see, with coroutines we separate the file reading algorithm from the computation,
without callbacks. This eases separation of concerns.

Contrary to closure iterators, the coroutine can accept arguments on resume.
Those should be tagged by the pragma `{.resume.}`.
Alternative signatures that are already parsable could be:
```Nim
coro read{buf: var array[4 * 1024, byte]}(path: string): int
coro read(path: string): int {.resume: "buf: var array[4 * 1024, byte]".}
```
but they are likely too foreign.

The `{.resume.}` solves cleanly the problem of view parameters.
There is no need to capture them at coroutine initialization,
they only need to live in the consumer.

It makes it also easy to build cancellation via passing a `cancel: bool` parameter

_Note: we can also build a state machine manually with "onOpen(...), tryReceive(...), close(...)"_

### Open question

`yield` within the coroutine highlights suspension points.
`resume` in the caller highlights that the async use. Is this desirable?

## Use-cases

Here are some use cases:

- Networking:
  - suspend while the kernel establishes a connection
  - suspend while the kernel is buffering a TCP stream,
    assuming epoll, only allocate when data is actually ready
    (and so can use a stack array!)

- Generator:
  - First class generator object that are lazy and can be chained efficiently.
    If they don't escape their scope, see the whole code and can optimize them away. ("Disappearing coroutines")

- Parsers:
  - Allow separations of duties between lexing/tokenizing, parsing and processing in the code but interleaved passes on execution. This allows doing one-pass translation while keeping code maintainable. For high performance parsing with memory-mapped files, this also allows to hide latencies by prefetching the next tokens before processing what we have in a buffer.

- Streams:
  - Streams can be thin wrapper around coroutines.
    This would standardize their interface to "resume"

- AsyncStreams:
  - Async streams can be a stream coroutine calling a networking coroutine
  - AsyncStreams can be multiplexed at zero ergonomic or performance cost.
    Read from a stream, suspend, read from the other, suspend, no state machine
    to keep track of manually.

- Sessions and protocols
  - Sessions, for example TLS, require managing a complex test.
    Instead of writing a state machine that handle IO asynchrony
    and the protocol own state, we can compose coroutines.

- Kernel-mode drivers
  - Drivers also require async IO and complex state machine (example bluetooth driver: https://forum.nim-lang.org/t/5509)

- Saving and suspending workflows
  - If an user workflow (long forms for example) on a webpage
    is represented as a coroutine and it is serializable,
    it can be saved and resumed.

- Multithreading
  - Coroutines gives a cheap producer-consumer abstraction.
    1. A producer creates a coroutine that represents a message or task or a handle to resources it.
    2. The coroutine starts suspended.
    3. The consumer receives it and can schedule it as it sees fit.
    4. Furthermore as coroutines are moved, resources like database connection or sockets are guaranteed unique, there is no need for lock or condition variable synchronization.
    5. Lastly compared to plain closures, an async stream can be consumed and the consumer can suspend computation.
    6. Cancellation is easy since coroutines support resume with value
       and the consumer has full ownership.
    7. This is an excellent base to implement higher-level abstraction like the actor model.
    8. For data parallel application this would help scheduling unbounded streams, for example stream video processing.

## Design constraints and choices

- Coroutines are move-only and cannot be copied
- try/catch/finally cannot wrap a suspension point `yield`

## To explain

### Multithreading

- Synchronization without atomics or locks


