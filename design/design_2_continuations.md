# First-class continuations

The promise of continuations:
- Solving the non-blocking producer->consumer flow without intermediate buffers, without atomics or locks synchronization, without state machine, can be paused, interleaved, resumed, composed.

## Introduction

We propose to introduce first-class delimited continuations to the Nim language.

Continuations are low-level control-flow primitives that are able to model
all other control flow primitives. They are actually used as the privileged intermediate representation of optimizing compilers for functional language just like SSA (Single Static Assignment) is used for imperative language.

It has been proven in 1994 that delimited continuations are able to model any monads including exceptions, mutable state, coroutines, asynchrony. In particular just like the macro system at compile-time, they would allow Nim developers to integrate most (any?) breakthrough of concepts and programming practices from any language, at run-time.

Furthermore, delimited continuations have been heavily studied in theory, have been successfully implemented in other programming languages (Scheme, undelimited continuations), and have a restricted set of primitives on which we can build address several higher-level constructs.

### Introduction to continuation

Let's take the following pseudocode example:

```Nim
proc main() =
  let conn = tcpConnect("127.0.0.1", 1234)
  var buf: array[4*1024, byte]
  conn.read(buf)
  echo buf
```

Assuming we are executing line 2 `let conn = tcpConnect("127.0.0.1", 1234)`,
that line is called the current instruction.
And what is after is called a `continuation` so:
```Nim
# The continuation of `let conn = tcpConnect("127.0.0.1", 1234)`
  var buf: array[4*1024, byte]
  conn.read(buf)
  echo buf
```
with the same logic, the continuation of `conn.read(buf)` is
```Nim
# The continuation of `conn.read(buf)`
  echo buf
```

Another way to look at continuation is to remark that they represent "What comes next?" in your program. They solve the same problem as callback but while keeping the code straight line.

A continuation is called `undelimited` if it represents the rest of the whole program. It is called `delimited` if it only represent a subset of the program.

It has been shown that delimited continuations are both more powerful and easier to manipulate as they are typed and can return values, though technically undelimited continuation can return an exit code.
We are interested in delimited continuations.

Criticism:
> With great power comes great responsibility
1. Undelimited continuations have been compared to (named) gotos.
   We argue that we only provide delimited continuations and that
   it will be as dangerous as using callbacks.
2. Continuations are equivalent to setjmp/longjmp. This is true
   nonetheless, the type system and clear names will allow us
   reduce the potential bug surface.

### Terminology

We distinguish between suspending a blocking.
`Suspending` means stopping execution of the current context at a statement and returning to the caller, without running the statement continuation.
`Blocking` means stopping execution of the current context at a statement, waiting for it to complete before continuing.

### Introduction to resumable functions

Continuations will be used to implement resumable functions.
Traditional functions have two effects on control flow:
1. On function call, suspend the caller, jump into the function body and run it.
2. On reaching "return", terminate the callee, resume the caller.

Resumable functions adds the following 2 key capabilities:
3. The callee can suspend itself, returning control to "something".
4. The current owner (not always the initial caller) of the function can suspend itself and resume the continuation.

Resumable functions are called coroutines if the continuation can only be called once and it is delimited. "Coroutines are one-shot delimited continuations".

Coroutines are very similar to closure iterators, however raw continuations help solve problems in a very natural way that require extra boilerplate or manipulating the type system hence we want to provide both access to delimited continuations and coroutines to Nim.

## Motivation

Coroutines can be applied to:

- Networking:
  - suspend while the kernel establishes a connection
  - suspend while the kernel is buffering a TCP stream,
    assuming epoll, only allocate when data is actually ready
    (and so can use a stack array!)

- Iterators & Generators:
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

- Higher-level concurrency:
  - Communicating Sequential Processes (named channels, anonymous coroutines)
  - Actors (named coroutines, anonymous channels)

Furthermore first-class continuations add the following capabilities:
- Async IO: composable "zero-cost" async IO that can easily be integrated by any scheduler or event-loop
- Enabling multithreading runtime to use "continuation-stealing" scheduling (distributing our continuation at spawn point) which is both CPU optimal and work in bounded task space,
  contrary to the usual "child-stealing" (distributing the spawned task) which is CPU optimal but has unbounded task space as it needs to spawn all recursive tasks before actually working on them. Short explainer: http://www.open-std.org/JTC1/SC22/WG21/docs/papers/2014/n3872.pdf. This significantly reduces parallel task scheduling overhead and memory usage and might mitigate the need to use a memory pool to work around allocators, even malloc, limitations.

As you notice, a lot of those domain problems can be unified under a single keyword `resume`. With resumable functions every workflow can be efficiently composed in a "zero-copy" manner without intermediate buffers, for example:
- Wait for a socket connection
- Distribute on a threadpool, suspend and wait for the next chunk
- The worker thread now owns the AsyncStream from the socket
- Read a chunk from a socket
- Decrypt that chunk
- Uncompress that chunk
- Pass to the parser
- Process()
- sendResponse

The key idea is that **continuations are a perfect fit for describing producer->consumer flow**.
- `suspend` is the producer side
- `resume` is the consumer side.

This is an example with 2 schedulers integration.
Having continuations as the building blocks allows
Network IO, AsyncStreams, async/await and multithreading
compose seamlessly in an efficient manner.
Furthermore scheduler development becomes extremely simple,
and cross-thread synchronization can even be done
without

```Nim
type Awaitable[T] = concept aw
  ## An awaitable concept, for example a Future or Flowvar.
  ## Implementation left at the scheduler discretion.
  aw.get is T
  aw.isReady is bool

var ioScheduler{.threadvar.}: IOScheduler
var computeScheduler{.threadvar.}: CPUScheduler
var isRootThread{.threadvar.}: bool

proc saveContinuationAndSuspend() {.suspend.} =
  ioScheduler.enqueue bindCallerContinuation()

proc await[T](e: Awaitable[T]): T {.resumable.} =
  if not e.isReady():
    saveContinuationAndSuspend()
  return e.get()

proc continueOnThreadpool(pool: CPUScheduler) {.suspend.} =
  pool.spawn bindCallerContinuation()

proc serve(socket: Socket) =
  while true:
    let conn = await socket.connect()
    if isRootThread:
      suspendWith pool.continueOnThreadpool()
    # --- the rest is on another thread on the CPU threadpool.
    #     and the root thread can handle IO again.
    # Stream processing
    var finished = false
    while not finished:
      let size = await conn.hasData()
      var chunk = newSeq[byte](size)
      conn.read(chunk) # non-blocking read
      finished = process(chunk)
    # Now that thread still owns the socket,
    # we can return it to the main thread via channel
    # or keep its ownership.
```

## APIs

### Raw continuations

Continuations will had only 3 ways of interacting with them:
- `bindCallerContinuation`: store the caller in a first class object (reification) that can be used to call the continuation later, just like calling a callback.
  We allow `bindCallerContinuation` only once and only in a procedure tagged `{.suspend.}`.
- `resumeContinuation`: jumps to a continuation, execute it and return to the current context.
- `suspendWith`: call a `{.suspend.}` function that will suspend us afterward.

`bindCallerContinuation` and `resumeContinuation` are intentionally made verbose and easy to grep for a maintenance and debugging perspective.
They represent jump points to non-local control flow.
Besides higher-level primitives like coroutines, closure iterators and async/await,
we expect that only low-level library developers for IO, streams, lexers, parsers and schedulers will use continuations directly.

#### Research and other languages

`suspendWith` is similar to `callCC` from Scheme except that:
- It passes a delimited continuation (only until the end of the function)
- The continuation can only be run once

### Coroutines

Coroutines will build on top of continuations and provide `yield` and `resume` primitives.
We choose `object.resume()` syntax over `object()` to make control flow changes and intentions explicit, also to ease maintenance and debugging.

## Design constraints and choice

Here are some design choices:
- Continuations are delimited by `{.resumable.}` and `{.suspend.}` pragmas.
  Furthermore, a corresponding tag will be added and propagated to detect resumable functions.
- We add a little friction (color) to resumable functions to ease maintenance, debugging and highlight non-linear control flow.
- Continuations are `Isolated` types, they cannot be copied to ensure that a cancellation cannot be run twice or more. Continuations in scheme are multishot ("you can enter the room once and leave twice")
  but no practical use case was found, while it caused serious garbage collection constraints.
  Upon a call (`resumeContinuation`), the continuation replaces itself with the next instructions of the program or `nil` if it reached the end of a `{.resumable.}` scope.

### TODO

- try/except over a resume point?
- On the stack and reuse only if they
  involve trivially destructible types `supportsCopyMem`
  - Can we extend for non-GC type?
    - non-ref types with non-trivial destructors
      should be OK since they are moved.
