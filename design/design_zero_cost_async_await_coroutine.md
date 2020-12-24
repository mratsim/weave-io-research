# Async/await abstraction over coroutines

This explores the design space of zero-cost async/await abstractions,
built on top of a coroutine

As long as we have suspendable function-like primitives, we can model asynchrony
by switching between them as long as they did not complete.
The only question is how to inject  primitives that are scheduler agnostic into the scheduler

Here are potential solutions.

### 1. Naked coroutines

Solution 1 would be wrapping processing coroutines into void coroutines that handle the whole workflow.

Applications would be able to define at the scheduler level
something in that spirit
```Nim
proc eventLoop(scheduler: var Scheduler) =
  while true:
    if scheduler.queue.len == 0:
      scheduler.park()
    else:
      var cur = move scheduler.queue.popFirst()
      cur.resume()
      if cur.finished():
        `=destroy`(cur)
      else:
        scheduler.queue.addLast(sink cur)
```

This however prevents the scheduler from scheduling the coroutine besides the one as the top of the chain, for example:
1. . remoteCsvReader
2. .. parseInt/Float/String, splitOnSeparator
3. ... eatChar (AsyncStringStream)
4. .... asyncTCPRead

#### 2. Pragma rewrite

Solution 2 would be similar to what is actually implemented in asyncdispatch and chronos,
an {.async.} pragma rewrite the code to return a suspendable wrapped into a Future or an FutureStream on `await`.

#### 3. Awaitable concept (C++)

Solution 3 is presented is an approximate translation from C++.
It seems to require some type system or type erasure fixes and
so is provided as food for thought.

We define a concept `Awaitable` with `isReady`, `onSuspend`, `onResume`.
See: https://github.com/GorNishanov/await/blob/5dc5d0a/2018_CppCon/src/coro_infra.h#L36-L51

```Nim
type Awaitable[T] = concept aw
  aw.completed is bool
  aw.onSuspend(sink CoroObject) is Option[CoroObject]
  aw.onResume() is T

proc await[T](producer: Awaitable[T]): T =
  let p {.byaddr.} = producer # don't trigger side-effect twice, works with move only types.
  if not p.completed():
    producer.onSuspend(coroHandle)
    # <----- suspended and returns only when ready
  return producer.onResume()
```

The awaitable interacts with the scheduler in `onSuspend`
```Nim
# my_scheduler.nim
var scheduler: MyScheduler # AsyncDispatch, Chronos, a thread-local scheduler or your own tailor made scheduler

type MyFuture[T] = object
  fut_value: T

proc `=`(dst: MyFuture, src MyFuture){.error: "A future cannot be copied"}

proc completed(producer: MyFuture): bool =
  producer.coroHandle.finished()

proc isReady(producer: MyFuture) = ...

proc onSuspend(producer: var MyFuture, coroHandle: sink CoroHandle): Option[CoroHandle] =
  # if the result is not available, enqueue ourself last and dequeue the first coroutine in the scheduler
  # the scheduler is a global, hence Awaitable are scheduler specific (but asyncdispatch and Chronos can provide their own).
  scheduler.addLast(move coroHandle)
  result = move scheduler.popFirst()
  # ^ This switcheroo bypasses asymmetric coroutine limitation of only returning to their caller.

proc onResume(producer: sink MyFuture[T]): T =
  result = move producer.fut_value
  `=destroy`(producer)
```

This would allow give us a common abstraction for the low-level and high-level for all schedulers,
improving interop between asyncdispatch and Chronos code. They would only need to define their Awaitable type that tie in with their scheduler. Similarly this abstraction is sufficient to represent fork/join parallelism.

You might have caught a couple of uncertainties in the pseudocode.
1. What is the "CoroHandle", it's a handle to one of the coroutines that the scheduler manage.
   On suspend, we return control to the scheduler with a new coroutine it can schedule instead.
   It can cycle through them all until one is ready.

#### 4. Refining the concept

We can borrow ideas from Rust futures to refine the C++ Awaitable concept.

To be finished ...


## Random thoughts

### Awaitable concept

> @disruptek, @Zevv, Ok I have found a compelling argument on why continuations are probably the better primitives over coroutines, at least asymmetric coroutines.
>
> 1. Asymmetric coroutines only yield to their current owner and symmetric coroutines need a handle of the coroutine to switch to.
> Assume we are in a latency sensitive context (say webserver) and we have a simple fair scheduler that does exec a coroutine for a client then switch to the next one (it's a queue with popFirst() scheduling). Now we are currently parsing some data, we might have some nested coroutines for a client (for example a lexer coroutine then a parser coroutine).
>
> If we have symmetric coroutines, we can switchTo(scheduler.popFirst()) "easy peasy".
> If we have asymmetric coroutines, we can yield scheduler.popFirst() as a workaround to having to unwrap the whole coroutine nest.
>
> However both solutions need the coroutine to be made aware of the scheduler, the solution adopted by C++ (which chose asymmetric coroutines because their symmetric coroutine competing proposal was deemed "unknown if possible to implement") is an "awaitable" concept that allowed overriding "yield", and so a scheduler could override the default yield with one that was aware of itself.
>
> The main issue of this awaitable concept imo, is that it's quite the ceremony and as my unpublished post on IO tasks vs CPU tasks show, scheduler adapted to the application are very important. So we likely want to make that as easy as possible. Also isn't awaitable a "continuation"?
>
> So it seems to me like C++ had to introduce more flexibility in their coroutines by defining them on the base of "continuations" without referring to it as such. (and then in their unified executors proposal they prove something about awaitable being equivalent to Senders and so the Receiver-Sender model fit perfectly with coroutine). So an unified executor in Nim would likely just be "Producer of continuations -> Consumer of continuations".
>
> There is no 2.
