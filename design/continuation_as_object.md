https://github.com/disruptek/cps/discussions/40
Context: https://github.com/disruptek/cps/commit/6b4cdc4aa82b110b09f1d22b59a4d7450713ea52

@disruptek
---------------------

To be clear, we're talking about return values for continuation invocations *inside* the trampoline.

What we did originally was to return a continuation object.  This was a `ref`.  It held a `fn` pointing to the next call.

The recent rewrite of `experiments/main.nim` looks like this:

```nim
import eventqueue

proc ticker(c: var Cont): pointer =
  inc c.i
  echo "tick ", c.i
  if c.i < 200:
    c.addTimer(ticker, 0.2 * 1000)

run Cont(i:   0): ticker
run Cont(i: 100): ticker

forever()
```
See also https://github.com/disruptek/cps/blob/master/experiments/eventqueue.nim

In this model, calls return a pointer to the next call and the continuation is mutated.  The `fn` doesn't need to be stored in the continuation because it is returned by the call.  This approach naturally lends itself to rewrite as a stack-based state machine.

The deal-breaker, for me, is that my calls cannot return an *arbitrary* continuation type.  This has pretty massive ramifications to composition, limiting what continuations can inject into whatever dispatcher they find themselves in.

@mratsim
---------------------

> The deal-breaker, for me, is that my calls cannot return an arbitrary continuation type. This has pretty massive ramifications to composition, limiting what continuations can inject into whatever dispatcher they find themselves in.

This is the same problem for any dispatcher if you do
```Nim
let a = spawn intToString(1)
let b = spawn sumThenFloat(1, 2)
```
Your dispatcher needs to be able to handle both `proc(int): string` and `proc(int, int): float`

## Type-erasure

The proper solution is type erasure however the basic continuation must not be tied to the GC.
- Suddenly you would have refcounting on every continuation
- Unsuitable in environment with no allocation
- Refcounting is not threadsafe
- Refcounting is unnecessary since continuations are only movable
- There many use cases that want or require no allocation, but would greatly benefits from first-class resumable functions:
  - kernel drivers
  - cryptographic handshake
  - multithreaded schedulers
  - high performance computing. High performance computing is all about hiding memory latency,
    and people spend decades of years of research devising schemes to get data in the proper CPU caches:
    https://github.com/numforge/laser/blob/d1e6ae6/laser/primitives/matrix_multiplication/gemm_tiling.nim#L284-L305
    Involving the GC in continuations would flush the caches and replace useful data with GC pollution.
  - games
  - hiding latencies in database join or dataframes of large size (>1GB with random access)
- It adds RTTI fields
- Generic inheritance is less tested especially generics methods
- It contributes to memory fragmentation
- It does not play well with custom allocation scheme.

Assuming mutant continuations the signatures are always in that vein
```Nim
type Continuation[T] = object
  fn: proc (c: var Continuation[T]) {.nimcall.}
  env: Envs
```

Here are 2 solutions depending on the kind of application.

## 1. Compute and memory-bound dispatcher

A dispatcher for compute application SHOULD NOT ever involve the Nim GC for it's low-level throughput sensitive operations.

Inside the dispatcher you type erase all continuations to
```Nim
type ContinationBase = object
  fn: proc (c: pointer){.nimcall.}
  envs: UncheckedArray[byte] # or array[N, byte] with N a suitable max size
```

Furthermore, the continuation can be made intrusive with other scheduler metadata:
- parent task
- loop index, start, stop bound
- future/flowvar
- barrier
- workstealing or other scheduling scheme metadata

And the scheduler will be able to manage it's own memory management scheme.

## 2. IO-bound scheduler

For an IO-bound scheduler where a GC is acceptable the raw continuation object can be wrapped.

```Nim
type ContinuationWrapper = ref object of RootObj

type ContinuationT = ref object of ContinuationWrapper
  cont: proc (c: var Iterator){.nimcall.}

type ContinuationNS = ref object of ContinuationWrapper
  cont: proc (c: var NetworkStream){.nimcall.}
```

You get the same flexibility as directly using a ref object of RootObj
but continuation themselves can be composed with schedulers and environment with different requirements.

## 3. Advantages

I don't see any inconvenient to have a plain object since plain object can always be wrapped in a ref object of RootObj.
Furthermore this has significant composition advantage:
- An IO scheduler can send compute continuations to a compute scheduler.
- Continuations can be send to C, Python, Rust code as they don't involve Nim runtime either.
- They give developers the choice to choose their memory mangement scheme for their scheduler depending on the performance/convenience tradeoff:
  - stack (plain state machine, generator, iterator)
  - memory pool (compute-bound multithreaded scheduler)
  - Nim GC if overhead is acceptable

This would be a killer Nim feature for the "no GC" constrained environments, especially
because they often involve state machines and continuations make them much easier to write.

### The particular case of multithreading

Lastly, for multithreading runtime using fork-join Parallelism
a `spawn` is a split point where the control flow splits between:
- a child task
- the continuation of the current scope
Either one can be submitted to a threadpool for multithreaded execution.

Submitting a child task is significantly easier as it is just a function call + parameters to serialize.
For the past 25 years, beside the original breakthrough of Cilk and Go, no runtime used continuation stealing.
This is because it was because it's hard to resume a continuation from any thread.
CPS would allow us to do that and it has very practical and a beneficial consequence:
- Very nice explainer: http://www.open-std.org/JTC1/SC22/WG21/docs/papers/2014/n3872.pdf
- With contination stealing the number of pending tasks that use memory at any point in time
  is **bounded**, each thread can use at most the same stack space as serial execution.
- With child stealing, the current Weave approach, the number of pending tasks that require memory at any point
  is **unbounded**.
- You have to see the initial root task as the root of a tree and you create a task graph, each "spawn" is a node.
  In serial execution, with depth-first search the space used is bounded by the tree depth. Serial execution is depth first.
  Continuation stealing is also depth-first hence the same bound per thread.
  Child stealing is breadth-first and causes a lot of task to reserve stack space but not being worked on until all tasks are created.
- As a consequence, the memory overhead with continuation is way less than child stealing, I might even be able to remove all the clever memory management I had to write to give Weave such a high performance:
  - https://github.com/mratsim/weave/tree/master/weave/memory
  Also the problem I have with child stealing has been the subject of numerous papers and whole PhD Thesis and many very complex approaches to solving it have been tried including remapping memory offset with mmap and modular arithmetic magic or forking the Linux Kernel:
  - https://github.com/mratsim/weave/blob/master/weave/memory/multithreaded_memory_management.md
  Intel multithreading library TBB, uses bounded stack space instead but bounding stack space prevents the library from using the full parallelism exposed by the code and there are many instances where TBB was shown not to scale due this.

In short CPS is an tremendous opportunity for multithreading.
Task schedulers would be able to allocate significantly less memory, reduce their overhead,
and still respect the criteria of an asymptotically optimal scheduler which is to be greedy "as long as their is a task and a thread isn't busy, it should work on it", which is not possible if you worry about having a large stack.

Obviously all those memory and overhead advantages disappear if each `spawn` involves Nim GC or even raw malloc.

## Conclusion

A plain object is more flexible and composable than a ref object of RootObj which would have no escape hatch.
It has plain advantages for many domains where a ref object would be a showstopper.

There are easy to implement solutions on plain object to fallback to `ref object of RootObj` for convenience.

Furthermore, core primitives of the language should be as much as possible usable without the GC,
this is the whole point of having arc, to make seq and string usable in no GC environment.

This would also suitably replace closures by something everyone could use instead of having to build their poor man's closure and trying to decipher `liftLocals` and the `owner` macro.
