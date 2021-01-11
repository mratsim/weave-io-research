# Implementation of 1st-class continuation and coroutines

> Companion repo: https://github.com/disruptek/cps

We explore here the possible implementations of first-class continuations and coroutines.

We denote continuation and coroutines the stackless variants.
Stackful coroutines (that allocate a custom stack and switch to it) are called fibers (alternatively green threads or user-mode threads).

We choose stackless one-shot continuations and stackless one-shot coroutines
as the basic building block for resumable functions.
The [../design](design) and [../research](research) folders have a rationale behind this decision.
In summary we want something:
1. Portable to targets that don't allow stack pointer manipulation for example Javascript and WASM. Excludes fibers.
2. Low overhead. Excludes fibers which need to manage their stack using complex techniques to deal with split stacks/segmented stacks/cactus stacks.
3. Embedded and kerneldev friendly. Imply to not impose heap usage.
4. Scheduler friendly. Implies type erasure facilities, can be allocated on heap or memory pooled at the scheduler discretion.
5. (Very!) Fast. An optimizing compiler should be able to constant-fold away the code, in particular for generators and parsers. Implies possibility of stack usage if they don't escape their scope.
6. Ergonomic. Continuations come across as intimidating concepts.

## Reminder: Resumable functions & continuation

A reminder from the design document before we diving into the internal tradeoffs.

A traditional function has 2 capabilities regarding control-flow:
1. On function call, suspend the caller, jump into the function body and run it.
2. On reaching "return", terminate the callee, resume the caller.

Resumable functions adds the following 2 key capabilities:
1. The callee can suspend itself, returning control to their last resumer.
2. The current owner (not always the initial caller) of the function can suspend itself and resume the continuation.

In particular, neither traditional function or resumable function as designed can:
1. Change their return type in the middle of computation.
2. Take a new argument that wasn't declared from the start in the middle of computation, for example for cancellation.
3. Have an "instance" be executed twice.

Those designs constraints are rooted in both theory, type theory and continuation research for the first 2, and practice as multishot continuations are controversial in the Scheme language due to being able to "enter the function once and exit twice" and the resource acquisition/management implied.

When a resumable function is suspended, we call the rest of the computation that wasn't yet run the "continuation".

We note that for cancellation, not calling the continuation is equivalent to cancelling it. If there are specific "wind down" to do, for example sending an exit message, the resumable function can accept a "channel" argument that it will check after each suspension point.
Cancellation has to be done cooperatively in this manner as while destructors/finalizers may know how to free a resource, cancellation might be more complex. We expect schedulers and higher-level framework to streamline cancellation.

## API levels

In the library we name
- "Continuation" a continuation of a resumable function that has no return value.
- "Coroutine" a contination of a resumable function that yield values.

Coroutines are an alternate implementation of Nim already existing closure iterators
in terms of high-level capabilities. As such even though their difference is minimal we consider them higher-level than raw continuations.

We envision the following API levels build on resumable functions

1. Raw continuations, interacted through:
  - `{.suspend.}` for functions that will suspend their caller
  - `{.resumable.}` for functions that can be suspended
  - `bindCallerContination` to allow storing the `{.resumable.}` caller within a `{.suspend.}` function. Schedulers will use this to continue a computation at more opportune time (async IO) or on more opportune resources (multithreading).
2. Coroutines, interacted through:
  - `{.coro.}` tags a function as a coroutine. Hopefully `coro` ultimately becomes
    a procedure declarator like `proc` and `func`
  - `yield` to denote suspension points.
3. Schedulers, interacted through:
  - `async`/`await` for IO tasks, building on `bindCallerContination`
  - `spawn`/`sync` for CPU tasks, building on `bindCallerContination`
  - Futures/Flowvar, which may build on coroutines.
4. Stdlib API:
  - iterutils, chainable iterators based on coroutines
    - for sequtils
    - for strutils
  - asyncstreams, suspendable streams based on coroutines
  - non-blocking IO
  Note: Continuations are a boon for ergonomics, flexibility and composability for example to compose
  an [implicit suspendable state machine](https://eli.thegreenplace.net/2009/08/29/co-routines-as-an-alternative-to-state-machines) between streams coming from various threads, IO and iterators. Nonetheless a `var openarray` API is simpler and more efficient for single use of say `toLowerASCII` (unless we can have disappearing coroutines https://godbolt.org/g/26viuZ) and a compile-time iterator API has unique guarantees that can't be matched by a runtime mechanism.
5. High-level concurrency:
  - Ergonomic lambda for anonymous functions, coroutines, channels.
  - collect support for chained coroutines
  - Communicating Sequential Processes (CSP) (= named channel + anonymouse coroutine)
  - Actors (= name coroutine + anonymous channel)

## The innards of resumable functions

Continuations (the "what's next" after suspending a resumable function) require 3 core components. We outline their roles, possible implementations and design constraints.

Note that a significant part of the design space is implemented in https://github.com/disruptek/cps. We refer to it as the reference implementation.

At its core a continuation is very simple
```Nim
type
  ContinuationProc[T] = proc(c: var T) {.nimcall.}
    ## using mutating contination

  Continuation* = concept cont
    cont.fn is ContinuationProc[Continuation]
    cont.frame is (object or ref)

  Coroutine* = concept coro
    type Output = auto
    coro is Continuation
    coro.promise is Option[Output]
    coro.hasFinished is bool
```

### The CPS transform

Behind this scary 3 letters transform hides a very simple concept, all control-flow can be represented as `if cond1: goto continuation1 else: goto continuation2` or can be represented as `if cond1: callback1(env) else: callback1(env)`. The CPS transform scans a function, transforming all control flow into chained gotos/callbacks making those explicit.

We call an "atomic section", as section of the function body that has no suspension point.

We do not discuss the `how` to transform as it has no runtime implication. We focus on `into what`. There are 2 final representations of the resumable function to consider:
- One callback per "atomic" section of the code, the current callback replaces itself with the next function to call before suspending (mutating continuation) or return the next function to call (immutable continuation).
- A state machine which can be implemented with `goto Labels` using Nim hidden `{.goto.}` pragma. Each "atomic" section of the code would be one of the label. At each call the function stores the next label to jump to at resume.

The `reference` implementation was implemented with one callback **per control flow construct**. This is a just a proof-of-concept artifact.

At first glance it might seem that state machines are more efficient than function if you are familiar with traditional iterators or parsers where we prefer a big state machine instead of multiple function calls. However whether we use callback or a state machine for a resumable function we need the same number of function calls. Furthermore compilers are significantly better at optimizing function calls and constant fold across function calls. This is something that the C++ CoroutineTS designers and LLVM compiler developers outlined when implementing the low-level details of C++ coroutines: it's very important to transform the code into a state machine as late as possible so that the LLVM optimizer passes work on functions. For CPS transform this might be even more true because the control-flow is greatly reduced improving fusion and folding capabilities.

### The activation frames storage

In between resumes and suspends the continuation context needs to be stored.
Given that our continations are one-shot we can overwrite our storage with the environment for the next continuation to call.

The storage needs to be type-erased as an environment might save `var a: int, b: int` in a higher scope but in an inner scope have `var a: seq[float64], b: string` due to shadowing.

Furthermore the storage needs to properly handle ref objects, destructors and the Javascript backend.

It should facilitate optimizations by the C compiler where possible, in particular for the case of iterators and generators created and consumed in the same scope. Optimizations are much easier with stack objects (though `{.noalias.}` on ref types might help as well).

Lastly it should give users, in particular schedulers, the flexibility to choose where continuation is allocated. This means continuations must be plain object by default and whether to use the stack, transform them into a `ref` or use them in a memory pool is left at the users discretions.

We address the 2 differing concerns with 2 storage schemes.

#### Portable storage scheme

The portable storage scheme uses inheritance + stack object to stack type-erase the environment.

The target is Javascript, the Nim VM and saving non-`supportsCopyMem()` frames where we cannot use lower-level constructs that rely on memory layout.

This leads to the following code for environment / activation frames

```Nim
type
  ContFrameBase_myFn = object of RootObj

  ContFrameBase_myFn0 = object of ContFrameBase_myFn
    a_atomblock0: int
    b_atomblock0: int

  ContFrameBase_myFn1 = object of ContFrameBase_myFn
    a_atomblock1: seq[float64]
    b_atomblock1: string

  ContFrameBase_myFn2 = object of ContFrameBase_myFn
    a_atomblock2: int
    b_atomblock2: int
    c_atomblock2: int
```

We note that the scheme is partially optimized for size and spaced is reused between calls. Partially as in for starter we will likely have the current continuation storage and the next continuation storage be alive at the same time at one point instead of having one field for possible locals.

On use this would be allocated on the heap and would rely on Nim doing the right thing regarding destruction/finalizers/garbage collection when converting between
  siblings types.

Note: parent->child would retain common fields and is an optimization used in the reference implementation. Furthermore the reference implementation reorganizes the frames so that the biggest object is at the root and all others derive from it. We argue that given the 2 targets of the portable scheme, Javascript and the Nim VM we do not need to optimize to such lengths.
Having deep ancestry instead of having all frames being siblings would lead to this if env2 derives from env0:
```Nim
type
  SomeContinuation = object of RootObj

  ContFrameBase_myFn = object of SomeContinuation

  ContFrameBase_myFn0 = object of ContFrameBase_myFn
    a_atomblock0: int
    b_atomblock0: int

  ContFrameBase_myFn1 = object of ContFrameBase_myFn
    a_atomblock1: seq[float64]
    b_atomblock1: string

  ContFrameBase_myFn2 = object of ContFrameBase_myFn0
    c_atomblock2: int
```
Notice that when debugging, looking at `ContFrameBase_myFn2` we need to unravel the ancestry to see that the environment also as `a_atomblock0` and `b_atomblock0`.
The second optimization, making the biggest object the root, also requires hundreds of lines of code and so is a maintenance burden for targets were performance is secondary.

#### Portable focused storage scheme

On the C and C++ backend, when we have trivial types (no ref or destructors) we can have a storage scheme with maximum space efficiency, that can be allocated on the stack to allow compilers to constant-fold across calls.

```Nim
type
  SomeContinuation = object of RootObj

  ContFrameBase_myFn0 = object
    a_atomblock0: int
    b_atomblock0: int

  ContFrameBase_myFn1 = object
    a_atomblock1: seq[float64]
    b_atomblock1: string

  ContFrameBase_myFn2 = object
    c_atomblock3: int

  ContFrameBase_myFn {.union.} = object of SomeContinuation
    frame0: ContFrameBase_myFn0
    frame1: ContFrameBase_myFn1
    frame2: ContFrameBase_myFn2
```

With `{.union.}` the continuation frames are sized to the max field size, avoiding reallocation and so allowing stack allocation.

The `object of SomeContinuation` is needed for type-erasure at least until we have VTable.

### Type-erasure for schedulers

The type eraser can just transform typed continuation to `SomeContinuation`
which is what the scheduler need to handle.
The callbacks or state machines know how to handle the environment even when type-erased and the hidden runtime type information can handle destruction/finalizers for non-trivial types.
