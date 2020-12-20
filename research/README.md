# Research

This collects research on continuation-passing style, async/await, coroutines

## Included in repo

- Kotlin coroutines: https://github.com/Kotlin/KEEP/blob/31bb8af/proposals/coroutines.md
- Rust async/await interface challenge - Matthias247: https://gist.github.com/Matthias247/5e5e7430149bbb04eebf18cf31747fe0
- Rust async/await cancellation challenge - Matthias247: https://gist.github.com/Matthias247/ffc0f189742abf6aa41a226fe07398a8
- Swift async proposal - lattner: https://gist.github.com/lattner/429b9070918248274f25b714dcfc7619
- Swift async proposal - oleganza: https://gist.github.com/oleganza/7342ed829bddd86f740a

- Downloading a billion files with Python: https://ep2019.europython.eu/media/conference/slides/KNhQYeQ-downloading-a-billion-files-in-python.pdf
- Efficient IO with io_uring: http://kernel.dk/io_uring.pdf

## Case studies

- Downloading a billion files with Python: https://ep2019.europython.eu/media/conference/slides/KNhQYeQ-downloading-a-billion-files-in-python.pdf

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
