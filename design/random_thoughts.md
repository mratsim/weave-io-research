# Random thoughts

## Awaitable concept

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

##### Work-stealing

> @Zevv, @disruptek, btw if CPS work you can publish a paper on it, because it would make it very easy to implement a "continuation-stealing" scheduler. To understand how big of a deal it is http://www.open-std.org/Jtc1/sc22/wg21/docs/papers/2014/n3872.pdf
> In practice: that would make Weave have less tasks in flight at any point in time, bounded by the number of processors instead of unbounded.
> so way less stress on the memory manager
> and maybe I can remove the memory pool from Weave altogether
> I have to think about it.
> I.e. instead of a memory pool, I can just have 1 scratchspace on each thread and tail-call continuations.

##### Blocked threads

> Oh, another realization for @dom96 , and @Zevv and @disruptek , if you have first continuations that you execute on a work-stealing threadpool you don't have to worry about blocking reads unless ALL your threads end up blocked.
> you make progress on your events even if one is blocked.
