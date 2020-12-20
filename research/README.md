# Research

This collects research on continuation-passing style, async/await, coroutines

## Included in repo

- Continuation Passing for C:
  - https://github.com/kerneis/cpc
  - Chroboczek, A space-efficient implementation of concurrency: https://www.irif.fr/~jch//cpc.pdf
  - Kerneis PhD Thesis: https://tel.archives-ouvertes.fr/tel-00751444/document
  - Continuation-Passing C Compiling threads to events through continuations, https://www.irif.fr/~jch/research/cpc-2012.pdf
  - Discussion: http://lambda-the-ultimate.org/node/4157
  - Manual: https://github.com/kerneis/cpc/tree/develop/doc

- Kotlin coroutines: https://github.com/Kotlin/KEEP/blob/31bb8af/proposals/coroutines.md
- Rust async/await interface challenge - Matthias247: https://gist.github.com/Matthias247/5e5e7430149bbb04eebf18cf31747fe0
- Rust async/await cancellation challenge - Matthias247: https://gist.github.com/Matthias247/ffc0f189742abf6aa41a226fe07398a8
- Swift async proposal - lattner: https://gist.github.com/lattner/429b9070918248274f25b714dcfc7619
- Swift async proposal - oleganza: https://gist.github.com/oleganza/7342ed829bddd86f740a

- Downloading a billion files with Python: https://ep2019.europython.eu/media/conference/slides/KNhQYeQ-downloading-a-billion-files-in-python.pdf
- Efficient IO with io_uring: http://kernel.dk/io_uring.pdf

## Nim RFCs

- next steps for CPS: https://github.com/nim-lang/RFCs/issues/295
- completing the async/await implementation: https://github.com/nim-lang/RFCs/issues/304
- async I/O with structural control flow (a.k.a Enforced awaits): https://github.com/status-im/nim-chronos/issues/2

## Case studies

- Downloading a billion files with Python: https://ep2019.europython.eu/media/conference/slides/KNhQYeQ-downloading-a-billion-files-in-python.pdf

### Structured concurrency / avoid eager execution

C++ Unified Executor proposals
- CppCon2019 talk: https://youtu.be/tF-Nz4aRWAM
- In particular "Would iterators be successful if they all allocate, require synchronization and do type-erasure, removing optimization opportunities from the compiler" at https://youtu.be/tF-Nz4aRWAM?t=1037
  - Instead use lazy future that start suspended and continuations/types are attached to it:
  - https://github.com/CppCon/CppCon2019/blob/master/Presentations/a_unifying_abstraction_for_async_in_cpp/a_unifying_abstraction_for_async_in_cpp__eric_niebler_david_s_hollman__cppcon_2019.pdf
- https://github.com/facebookexperimental/libunifex
- http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p0443r12.html

Python Trio
- https://vorpus.org/blog/notes-on-structured-concurrency-or-go-statement-considered-harmful/

_Note from experience in eager execution vs delayed execution of compute graph in Machine Learning (which are similar to async task graphs), people found eager execution (Pytorch) significantly easier to reason about than lazy (Tensorflow)._

_Note 2: Rust seems to have come to the same conclusion. Rust features are delayed until they are polled_

## Languages write up

### Concurrent ML and Guile

- https://wingolog.org/archives/2017/06/29/a-new-concurrent-ml
- https://github.com/wingo/fibers

### Kotlin coroutines

- https://github.com/Kotlin/KEEP/blob/31bb8af/proposals/coroutines.md

Kotlin builds async/await on top of coroutines abstraction

Talks:
- KotlinConf 2017 - Introduction to Coroutines by Roman Elizarov\
  https://www.youtube.com/watch?v=_hfBv0a09Jc
- Concurrency doesn't have to be hard: Kotlin Coroutines and Channels by Jag Saund\
  https://www.youtube.com/watch?v=3WGM-_MnPQA \
  Excellent example with concurrent baristas a single coffee machine and a waiter taking orders

### Lua coroutines

- https://spin.atomicobject.com/2013/07/01/lua-coroutines/
- Scheduler example: https://gist.github.com/Deco/1818054

### Ocaml Async

- https://dev.realworldocaml.org/concurrent-programming.html

Note: OCaml optimizes chain of Deferred in tail call (i.e. async tail call optimization)

### Python Greenlet, Stacklet, Continulet, Fibers

- https://greenlet.readthedocs.io/en/latest/
- Nim: https://github.com/treeform/greenlet

- Stackless:
  - https://doc.pypy.org/en/latest/stackless.html
  - Continulet: https://doc.pypy.org/en/latest/stackless.html#continulet
  - Composability of continulet vs switchable coroutines: https://doc.pypy.org/en/latest/stackless.html#theory-of-composability
  - https://www.grant-olson.net/files/why_stackless.html
  - http://dalkescientific.com/StacklessPyCon2007-Dalke.pdf

- Fibers: https://github.com/saghul/python-fibers
  - https://www.slideshare.net/saghul/understanding-greenlet

### Rust readiness-based zero-alloc futures

_Caveat: As Windows IOCP and Linux io_uring require being passed an owned buffer to allow "zero-copy" IO, Rust futures will need to allocate anyway, see [async_await_cancellation_rust_matthias247](async_await_cancellation_rust_matthias247)_

- https://aturon.github.io/blog/2016/08/11/futures/
- https://aturon.github.io/blog/2016/09/07/futures-design/
- https://jblog.andbit.net/2019/11/10/rust-async-execution/
- https://tmandry.gitlab.io/blog/posts/optimizing-await-1/
- https://gist.github.com/Matthias247/5e5e7430149bbb04eebf18cf31747fe0
-  https://gist.github.com/Matthias247/ffc0f189742abf6aa41a226fe07398a8

In particular it distinguishes between the traditional completion-based and their own poll-based futures with completion-based requiring a buffer for each future and so requiring more memory allocation (which are problematic because it stresses the GC, and lead to memory fragmentation on long running application). In particular the poll approach is attractive because it eases cancellation (don't poll) and since there is no heap indirection for the future, the compiler can do deep optimizations.

### Rust RFCs

- (merged) Futures: https://github.com/rust-lang/rfcs/blob/master/text/2592-futures.md
- (merged) Async/await: https://github.com/rust-lang/rfcs/blob/master/text/2394-async_await.md

- (open) Stackless coroutines: https://github.com/rust-lang/rfcs/pull/1823

Note: stackless in Rust means on the stack while stackless in Python means on the heap (don't use stack) ¯\\_(ツ)_/¯

### Rust-style futures in C

- https://axelforsman.tk/2020/08/24/rust-style-futures-in-c.html

### Swift Async proposal

- Lattner: https://gist.github.com/lattner/429b9070918248274f25b714dcfc7619
- Andreev: https://gist.github.com/oleganza/7342ed829bddd86f740a

Swift would build async/await on top of coroutines abstraction

## Kernel I/O primitives

### Windows IOCP

- https://docs.microsoft.com/en-us/windows/win32/fileio/i-o-completion-ports
- https://www.microsoftpressstore.com/articles/article.aspx?p=2224047&seqNum=5

### Windows Async Procedure Calls

- https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-queueuserapc

### Linux AIO

- AIO Poll (2018): https://lwn.net/Articles/743714/
- AIO submit: https://blog.cloudflare.com/io_submit-the-epoll-alternative-youve-never-heard-about/

### Linux io_uring

- https://stackoverflow.com/questions/13407542/is-there-really-no-asynchronous-block-i-o-on-linux/57451551#57451551
- https://brandur.org/nanoglyphs/011-shelter
- https://thenewstack.io/how-io_uring-and-ebpf-will-revolutionize-programming-in-linux
- http://kernel.dk/io_uring.pdf
- https://blogs.oracle.com/linux/an-introduction-to-the-io_uring-asynchronous-io-framework

## Academic research

### Coroutines

- Coroutines in Lua\
  Ana Lùcia de Moura, Noemi Rodriguez and Roberto Ierusalimschy, 2004\
  http://www.jucs.org/jucs_10_7/coroutines_in_lua/de_Moura_A_L.pdf
- Revisiting Coroutines\
  Ana Lùcia de Moura and Roberto Ierusalimschy, 2004\
  http://citeseer.ist.psu.edu/viewdoc/download;jsessionid=DE49740527F844892BF3C426FA2E8ED3?doi=10.1.1.58.4017&rep=rep1&type=pdf

## Wikipedia - related concepts

- Protothread: https://en.wikipedia.org/wiki/Protothread
- CallCC / Call with Current Continuation: https://en.wikipedia.org/wiki/Call-with-current-continuation
- Coroutines: https://en.wikipedia.org/wiki/Coroutine
- Continuation: https://en.wikipedia.org/wiki/Continuation
- Continuation Passing Style: https://en.wikipedia.org/wiki/Continuation-passing_style
- Delimited COntinuation: https://en.wikipedia.org/wiki/Delimited_continuation

## CPS for compilers

- Guile VM, control flow constructs are CPS transformed and optimization happens in CPS, then a codegen generates bytecode directly from CPS: https://www.gnu.org/software/guile/manual/html_node/Compiling-CPS.html
- CPS as a functional intermediate representation: https://xavierleroy.org/mpri/2-4/fir.2up.pdf
- Compiling with continuations: https://www.microsoft.com/en-us/research/publication/compiling-with-continuations-continued/
- How to compile with continuations: http://matt.might.net/articles/cps-conversion/
- Compiling with continuations, or without Whatever: https://www.cs.purdue.edu/homes/rompf/papers/cong-icfp19.pdf

## Type Erasure

A scheduler will likely need to enqueue continuations/frames/tasks in a data structure.
For that it will need to be type-erasedat 2 level, the tasks and the environment.
Tpe erasing the task is easy as it is always used by pointer. However the environment as dynmic size.

Some solutions:
- Make the environment another untyped pointer. This implies heap allocation of the task and heap allocation of the environment, which will be very costly when a scheduler deals with billions of them.
- Like Weave, make the environment intrusive but choose a max size. A decent max size can be chosen (144 bytes which leaves 112 bytes of scheduler metadata). This helps using Tasks on a memory pool. AFAIk actor languages like Erlang have a similar restriction.
- Understand what's going on here: https://github.com/CppCon/CppCon2019/blob/master/Presentations/back_to_basics_type_erasure/back_to_basics_type_erasure__arthur_odwyer__cppcon_2019.pdf

## Function colors

Function coloring problem might be exagerated.
`waitFor` solves crossing the async->sync barrier.
And an equivalent construct is needed for sync->async especially:
- to handle blocking file operations on Linux
- to "asyncify" channels/flowvers used to communicate in a multithreaded program.

### Async function color

- http://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/
- https://elizarov.medium.com/how-do-you-color-your-functions-a6bb423d936d

### {.sideeffect.}, IO Monad, Tainted string, Result, error code are all colored

- https://wiki.haskell.org/Introduction_to_IO

Nim {.sideeffect.} are also API infectious, which is usually OK except when we want to introduce logs in a previously side-effect free function.

However in the case of {.sideeffect.}, Result, error codes or {.raises: [].}, the viral nature is actually wanted, just like having proper parameter types as this ensures program correctness.

### Fork/Join parallelism color

In most modern implementations fork/join parallel runtime are reentrant, a caller isn't "infected" by the internals of the called function, parallelism is black-boxed.
However, Cilk, the foundational implementation of fork/join parallelism, that also proved how to schedule optimally on multicore is colored. http://supertech.csail.mit.edu/papers/PPoPP95.pdf
At a spawn point, the current execution is separated into a child call and a continuation:
```C
int fib(int n)
{
    if (n < 2)
        return n;
    int x = cilk_spawn fib(n-1);
    int y = fib(n-2);
    cilk_sync;
    return x + y;
}
```
At this point, the current thread can either:
- execute the child call (work-first) and another thread can swoop in and steal the continuation (continuation-stealing). This is Cilk mode of operation.
- execute the continuation (help-first) and another thread can swoop in and steal the child (child-stealing). This is most other framework mode of operations.

The main difference is that continuation stealing require saving the stack frame of the current function or it can't be resumed from an arbitrary thread. And another consequence is that function including cilk_spawn have a different calling convention than C since they are resumable. This means that Cilk program cannot naively be called from C. The workaround Cilk developers used is to always compile 2 versions of a program, one with the Cilk resumable calling convention and one with the C calling convention.

A side note: continuation stealing ensures that on a single-thread order of execution of the program is the same as it's serial equivalent. Also child stealing suffers from unbounded suspended tasks spawned since they are only worked on late. It's an excellent way to DOS GCC OpenMP implementation.

## Schedulers

It is interesting to have both global schedulers and scoped schedulers.

Global schedulers have significantly more optimization opportunities.
This is the reason why Microsoft forces everyone to use Windows Fibers for multithreading and Apple forces everyone to use Grand Central Dispatch.
On the other hand, developers might want local scheduler either to control their priority/niceness
or ensure reentrancy if used as a library.

### Scheduler research

- Cappriccio: http://capriccio.cs.berkeley.edu/pubs/capriccio-sosp-2003.pdf
- Making Tokio 10x faster: https://tokio.rs/blog/2019-10-scheduler
- FairThread switchable schedulers: http://www-sop.inria.fr/mimosa/rp/FairThreads/FTC/documentation/ft.pdf

## Async I/O frameworks

- C++ Elle by Docker: https://github.com/infinit/elle
  - elle: Utilities including serialization, logs, buffer, formatting, ...
  - reactor: An asynchronous framework using a coroutines scheduler
  - cryptography: Object-oriented cryptography wrapper around OpenSSL
  - protocol: Network communication designed to support RPCs
  - das: Symbol-based introspection
  - athena: Byzantine environment algorithms (Paxos)
  - service/aws: reactorified AWS API wrapper
  - service/dropbox: reactorified Dropbox API wrapper
  - nbd: A Network Block Device implementation.

- C++ Mordor by Mozy: https://github.com/mozy/mordor
  - Streams
  - HTTP server and clients
  - logging, configuration, statistics gathering, and exceptions.
  - unit test

- Rust async-std: https://github.com/async-rs/async-std
