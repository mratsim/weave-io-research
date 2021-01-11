# Research

This collects research on continuation-passing style, async/await, coroutines

## Introduction to suspendable/resumable functions

React 16 uses coroutines, fibers, continuations and so a lot
of Javascript developers have to learn that stuff:
- https://www.yld.io/blog/continuations-coroutines-fibers-effects/

Alternatively this post in the Python mailing list from 1999 is also a good explainer:
- https://mail.python.org/pipermail/python-dev/1999-July/000467.html

Layman's explanation of delimited continuations with examples of using them for exception handling and nondeterministic programming.
- https://gist.github.com/sebfisch/2235780

The Rise of Coroutines, by Kotlin Coroutines lead:
- https://learningactors.com/the-rise-of-the-coroutines/

## Low-level introduction to Continuation-Passing Style and Coroutines

- KotlinConf 2017 - Deep Dive into coroutines on the JVM by Roman Elizarov\
  https://www.youtube.com/watch?v=YrrUCSi72E8
- https://resources.jetbrains.com/storage/products/kotlinconf2017/slides/2017+KotlinConf+-+Deep+dive+into+Coroutines+on+JVM.pdf

## Coroutines can replace state-machines:

- Eli Bendersky - [Implementing First-Class Polymorphic Delimited Continuations by a Type-Directed Selective CPS-Transform](https://eli.thegreenplace.net/2009/08/29/co-routines-as-an-alternative-to-state-machines)

### State machine
```python
class ProtocolWrapper(object):
    def __init__(self,
            header='\x61',
            footer='\x62',
            dle='\xAB',
            after_dle_func=lambda x: x):
        self.header = header
        self.footer = footer
        self.dle = dle
        self.after_dle_func = after_dle_func

        self.state = self.WAIT_HEADER
        self.frame = ''

    # internal state
    (WAIT_HEADER, IN_MSG, AFTER_DLE) = range(3)

    def input(self, byte):
        """ Receive a byte.
            If this byte completes a frame, the
            frame is returned. Otherwise, None
            is returned.
        """
        if self.state == self.WAIT_HEADER:
            if byte == self.header:
                self.state = self.IN_MSG
                self.frame = ''

            return None
        elif self.state == self.IN_MSG:
            if byte == self.footer:
                self.state = self.WAIT_HEADER
                return self.frame
            elif byte == self.dle:
                self.state = self.AFTER_DLE
            else:
                self.frame += byte
            return None
        elif self.state == self.AFTER_DLE:
            self.frame += self.after_dle_func(byte)
            self.state = self.IN_MSG
            return None
        else:
            raise AssertionError()
```

### Coroutines

```python
@coroutine
def unwrap_protocol(header='\x61',
                    footer='\x62',
                    dle='\xAB',
                    after_dle_func=lambda x: x,
                    target=None):
    """ Simplified framing (protocol unwrapping)
        co-routine.
    """
    # Outer loop looking for a frame header
    #
    while True:
        byte = (yield)
        frame = ''

        if byte == header:
            # Capture the full frame
            #
            while True:
                byte = (yield)
                if byte == footer:
                    target.send(frame)
                    break
                elif byte == dle:
                    byte = (yield)
                    frame += after_dle_func(byte)
                else:
                    frame += byte
```

## Included in repo

- General: Layman's explanation of delimited continuations
  - https://gist.github.com/sebfisch/2235780

- Continuation Passing for C:
  - https://github.com/kerneis/cpc
  - Chroboczek, A space-efficient implementation of concurrency: https://www.irif.fr/~jch//cpc.pdf
  - Kerneis PhD Thesis: https://tel.archives-ouvertes.fr/tel-00751444/document
  - Continuation-Passing C Compiling threads to events through continuations, https://www.irif.fr/~jch/research/cpc-2012.pdf
  - Discussion: http://lambda-the-ultimate.org/node/4157
  - Manual: https://github.com/kerneis/cpc/tree/develop/doc
- Continuations in C: https://web.archive.org/web/20100106212446/http://homepage.mac.com/sigfpe/Computing/continuations.html
- Continuations in Cee: https://wiki.c2.com/?ContinuationsInCee
- C++ coroutines:
  - Theory: https://lewissbaker.github.io/2017/09/25/coroutine-theory
  - co_await: https://lewissbaker.github.io/2017/11/17/understanding-operator-co-await

- F#: Implementing coroutines (async/await) with continuations: https://gist.github.com/mrange/147fe94da28fbb237af4e9bd39da4ad3
- Kotlin coroutines: https://github.com/Kotlin/KEEP/blob/31bb8af/proposals/coroutines.md
- Kotlin coroutine deep dive: https://resources.jetbrains.com/storage/products/kotlinconf2017/slides/2017+KotlinConf+-+Deep+dive+into+Coroutines+on+JVM.pdf
- Python mailing list on fake threads (coroutines, continuations, generators): https://mail.python.org/pipermail/python-dev/1999-July/000467.html
- Rust async/await interface challenge - Matthias247: https://gist.github.com/Matthias247/5e5e7430149bbb04eebf18cf31747fe0
- Rust async/await cancellation challenge - Matthias247: https://gist.github.com/Matthias247/ffc0f189742abf6aa41a226fe07398a8
- Swift async proposal - lattner: https://gist.github.com/lattner/429b9070918248274f25b714dcfc7619
- Swift async proposal - oleganza: https://gist.github.com/oleganza/7342ed829bddd86f740a

- Downloading a billion files with Python: https://ep2019.europython.eu/media/conference/slides/KNhQYeQ-downloading-a-billion-files-in-python.pdf
- Efficient IO with io_uring: http://kernel.dk/io_uring.pdf
- A Primer on Scheduling via work-stealing (continuation stealing or child task stealing):\
  http://www.open-std.org/Jtc1/sc22/wg21/docs/papers/2014/n3872.pdf

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

### Continuations in C

- Continuations in C: https://web.archive.org/web/20100106212446/http://homepage.mac.com/sigfpe/Computing/continuations.html
- Continuations in Cee: https://wiki.c2.com/?ContinuationsInCee

### C++ coroutines

- https://lewissbaker.github.io/2017/09/25/coroutine-theory
- https://lewissbaker.github.io/2017/11/17/understanding-operator-co-await

Library impact: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0975r0.html

Disappearing coroutines:
- https://godbolt.org/g/26viuZ

- Await 2.0 Stackless resumable function
  - https://www.youtube.com/watch?v=KUhSjfSbINE

- CppCon 2015
  - C++ Coroutines: A negative overhead abstraction
    - https://www.youtube.com/watch?v=_fu0gx-xseY
    - https://github.com/CppCon/CppCon2015/blob/master/Presentations/C%2B%2B%20Coroutines/C%2B%2B%20Coroutines%20-%20Gor%20Nishanov%20-%20CppCon%202015.pdf
    - https://github.com/GorNishanov/await/tree/master/2015_CppCon/SuperLean
- CppCon 2016
  - C++ coroutines: under the cover
    - https://www.youtube.com/watch?v=8C8NnE1Dg4A
    - https://github.com/GorNishanov/await/tree/master/2016_CppCon
    - https://github.com/CppCon/CppCon2016/tree/master/Presentations/C%2B%2B%20Coroutines%20-%20Under%20The%20Covers

- CppCon 2017
  - Coroutines: what can't they do.
    - https://youtu.be/mlP1MKP8d_Q
    - https://github.com/CppCon/CppCon2017/blob/master/Presentations/Coroutines%20What%20Can't%20They%20Do/Coroutines%20What%20Can't%20They%20Do%20-%20Toby%20Allsopp%20-%20CppCon%202017.pdf
  - Naked Coroutines Live with Networking
    - https://www.youtube.com/watch?v=UL3TtTgt3oU
    - https://github.com/CppCon/CppCon2017/blob/master/Demos/Naked%20Coroutines%20Live/Naked%20Coroutines%20Live%20-%20Gor%20Nishanov%20-%20CppCon%202017.pdf
    - https://github.com/GorNishanov/await/tree/master/2017_CppCon
  - Concurrency, Parallelism and Coroutines
    - https://www.youtube.com/watch?v=JvHZ_OECOFU
    - https://github.com/CppCon/CppCon2017/blob/master/Presentations/Concurrency%2C%20Parallelism%20and%20Coroutines/Concurrency%2C%20Parallelism%20and%20Coroutines%20-%20Anthony%20Williams%20-%20CppCon%202017.pdf

- CppCon 2018
  - NanoCoroutines: sub-nanosecond cost to hide cache latencies in databases
    - https://www.youtube.com/watch?v=j9tlJAqMV7U
    - https://github.com/GorNishanov/await/tree/master/2018_CppCon

For ultra low overhead coroutines usecase:
- CoroBase: https://arxiv.org/pdf/2010.15981.pdf
- Interleaving with Coroutines: A Practical Approach for Robust Index Joins\
  Very Large DataBase conference
  - http://www.vldb.org/pvldb/vol11/p230-psaropoulos.pdf
  - https://infoscience.epfl.ch/record/231318
- Exploiting Coroutines to Attack the “Killer Nanoseconds”\
  http://www.vldb.org/pvldb/vol11/p1702-jonathan.pdf
  > A key requirement for efficient “interleaving” is that a context-switch must take less time than a memory stall.
  > Otherwise, switching contexts adds more overhead than originally imposed by thememory stalls.
  > This requirement renders many existing multi-threading techniques useless,
  > including light-weight, user-mode threads, known as fibers or stackful coroutine
- Bridging the Latency Gap between NVM and DRAM for Latency-bound Operations\
  https://www.semanticscholar.org/paper/Bridging-the-Latency-Gap-between-NVM-and-DRAM-for-Psaropoulos-Oukid/1b3e3dd80c1ae2c02c6a2745e941d8cccb75f6c1

Alternatives and criticism of C++ CoroutineTS
- https://botondballo.wordpress.com/2019/03/20/trip-report-c-standards-meeting-in-kona-february-2019/#coroutines
  - Core Coroutines: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1063r2.pdf
  - Symmetric Coroutines: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1430r0.pdf
- Are stackless coroutines a problem?
  - https://stackoverflow.com/questions/57163510/are-stackless-c20-coroutines-a-problem

### C++ Fibers

Fibers == stackful coroutines == user-mode threads
as in they allocate their own stacks on the heap
and modify the stack base pointer and stack pointer to jump to it

- Fibers under the magnifying glass\
  http://www.open-std.org/JTC1/SC22/WG21/docs/papers/2018/p1364r0.pdf

### C++ Resumable Functions

- https://isocpp.org/files/papers/N4402.pdf

### C# Async/Await

- C# Async/Await compilation in state machine:\
  https://weblogs.asp.net/dixin/understanding-c-sharp-async-await-1-compilation
- https://www.codeproject.com/Articles/535635/Async-Await-and-the-Generated-StateMachine

### Concurrent ML and Guile

- https://wingolog.org/archives/2017/06/29/a-new-concurrent-ml
- https://github.com/wingo/fibers

### F# continuations

- F#: Implementing coroutines (async/await) with continuations:\
  https://gist.github.com/mrange/147fe94da28fbb237af4e9bd39da4ad3

### Kotlin coroutines

- https://github.com/Kotlin/KEEP/blob/31bb8af/proposals/coroutines.md

Kotlin builds async/await on top of coroutines abstraction

Talks:
- KotlinConf 2017 - Introduction to Coroutines by Roman Elizarov\
  https://www.youtube.com/watch?v=_hfBv0a09Jc
- KotlinConf 2017 - Deep Dive into coroutines on the JVM by Roman Elizarov\
  https://www.youtube.com/watch?v=YrrUCSi72E8 \
  - Continuation Passing Style
  - Direct style
  - suspension
  - continuation
  - restoring state
  - state machine vs callbacks
  - dispatch
  - cancellation
  - Communicating Sequential Process
  - Actors
  - https://resources.jetbrains.com/storage/products/kotlinconf2017/slides/2017+KotlinConf+-+Deep+dive+into+Coroutines+on+JVM.pdf
- Concurrency doesn't have to be hard: Kotlin Coroutines and Channels by Jag Saund\
  https://www.youtube.com/watch?v=3WGM-_MnPQA \
  Excellent example with concurrent baristas a single coffee machine and a waiter taking orders

- How Kotlin compiles CPS and coroutines to state machines:\
  https://labs.pedrofelix.org/guides/kotlin/coroutines/coroutines-and-state-machines
- https://jorgecastillo.dev/digging-into-kotlin-continuations
- https://proandroiddev.com/how-do-coroutines-work-under-the-hood-803e6e9da8bb

### LLVM coroutines

- https://llvm.org/docs/Coroutines.html
- https://clang.llvm.org/docs/LanguageExtensions.html#c-coroutines-support-builtins
  - "stable"
    - `void  __builtin_coro_resume(void *addr);;`
    - `void  __builtin_coro_destroy(void *addr);;`
    - `bool  __builtin_coro_done(void *addr);;`
    - `void *__builtin_coro_promise(void *addr, int alignment, bool from_promise);`
  - Internal
    - `size_t __builtin_coro_size()`
    - `void  *__builtin_coro_frame()`
    - `void  *__builtin_coro_free(void *coro_frame)`

    - `void  *__builtin_coro_id(int align, void *promise, void *fnaddr, void *parts)`
    - `bool   __builtin_coro_alloc()`
    - `void  *__builtin_coro_begin(void *memory)`
    - `void   __builtin_coro_end(void *coro_frame, bool unwind)`
    - `char   __builtin_coro_suspend(bool final)`
    - `bool   __builtin_coro_param(void *original, void *copy)`

- LLVM coroutines
  - https://llvm.org/devmtg/2016-11/Slides/Nishanov-LLVMCoroutines.pdf
  - https://www.youtube.com/watch?v=Ztr8QvMhqmQ
- Coroutines representation and ABIs in LLVM
  - https://www.youtube.com/watch?v=wyAbV8AM9PM
  -

### Lua coroutines

- https://spin.atomicobject.com/2013/07/01/lua-coroutines/
- Scheduler example: https://gist.github.com/Deco/1818054

### Javascript React Fiber

- https://www.yld.io/blog/continuations-coroutines-fibers-effects/

### Nim closure iterators

- https://github.com/nim-lang/Nim/blob/v1.4.2/compiler/closureiters.nim

### Ocaml Async

- https://dev.realworldocaml.org/concurrent-programming.html

Note: OCaml optimizes chain of Deferred in tail call (i.e. async tail call optimization)

### Python Greenlet, Stacklet, Continulet, Fibers

- Python mailing list on fake threads (coroutines, continuations, generators): https://mail.python.org/pipermail/python-dev/1999-July/000467.html

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
- https://tokio.rs/tokio/tutorial/async
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

24 Dec announcement:
- Async/Await proposal accepted: https://forums.swift.org/t/accepted-with-modification-se-0296-async-await/43318
-

## Kernel I/O primitives

- The Secret to 10 Million Concurrent Connections -The Kernel is the Problem, Not the Solution\
  http://highscalability.com/blog/2013/5/13/the-secret-to-10-million-concurrent-connections-the-kernel-i.html

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

### Continuations

- The Discoveries of Continuations,
  John Reynolds\
  https://www.cs.tufts.edu/~nr/cs257/archive/john-reynolds/histcont.pdf

- Representing Monads\
  Filinski, 1994\
  http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.43.8213&rep=rep1&type=pdf
  Reddit quote: https://www.reddit.com/r/programming/comments/pz26z/oleg_says_its_time_to_think_about_removing_callcc/c3twxmh/?utm_source=reddit&utm_medium=web2x&context=3
  > A compelling argument for why delimited continuations are elegant is the paper Representing Monads. Two very short one liners are enough to make any monad native to your programming language. For example your language doesn't support exceptions? Define a maybe monad and use monadic reflection to make them feel like native exceptions. Your language doesn't have mutable state? Define a state monad and use monadic reflection to make it feel like native mutable state. Your language doesn't have coroutines and requires you to write callback manually as in Node.js? Define a monad similar to F#'s async workflows (=~ continuation monad), and now it feels like your language has native asynchronicity. The same goes for monadic parsers.

  > Summary: delimited continuations give you a way to write monadic code as if you were writing normal code (you could say it's do-notation on steroids).

- Capturing the Future by Replaying the Past\
  Functional Pearl\
  James Koppel, Gabriel Scherer, Armando Solar-Lezama\
  https://arxiv.org/pdf/1710.10385.pdf\
  > Delimited continuations are the mother of all monads! So goes the slogan inspired by Filinski’s 1994 paper,which showed that delimited continuations can implement any monadic effect, letting the programmer usean effect as easily as if it was built into the language. It’s a shame that not many languages have delimited continuations.


Ergonomics (coroutine == delimited continuations):
- http://okmij.org/ftp/continuations/undelimited.html
- http://okmij.org/ftp/continuations/against-callcc.html
- Scheme callCC: http://scheme-reports.org/mail/scheme-reports/msg02780.html
- Ruby continuations considered harmful: http://www.atdot.net/~ko1/pub/ContinuationFest-ruby.pdf

- A monadic framework for delimited continuations
  - https://legacy.cs.indiana.edu/~dyb/pubs/monadicDC.pdf

### Coroutines

- Coroutines in Lua\
  Ana Lùcia de Moura, Noemi Rodriguez and Roberto Ierusalimschy, 2004\
  http://www.jucs.org/jucs_10_7/coroutines_in_lua/de_Moura_A_L.pdf
- Revisiting Coroutines\
  Ana Lùcia de Moura and Roberto Ierusalimschy, 2004\
  http://citeseer.ist.psu.edu/viewdoc/download;jsessionid=DE49740527F844892BF3C426FA2E8ED3?doi=10.1.1.58.4017&rep=rep1&type=pdf
- Yield: Mainstream Delimited Continuations\
  Roshan P. James, Amr Sabry, 2011\
  http://parametricity.net/dropbox/yield.subc.pdf

The yield paper proved that `yield` as used in programming language
is equivalent to `shift` and `reset` used to build delimited continuations.

#### Use cases

Database optimization
- Interleaving with Coroutines: A Practical Approach for Robust Index Joins\
  Very Large DataBase conference
  - http://www.vldb.org/pvldb/vol11/p230-psaropoulos.pdf
  - https://infoscience.epfl.ch/record/231318
- Exploiting Coroutines to Attack the “Killer Nanoseconds”\
  http://www.vldb.org/pvldb/vol11/p1702-jonathan.pdf
  > A key requirement for efficient “interleaving” is that a context-switch must take less time than a memory stall.
  > Otherwise, switching contexts adds more overhead than originally imposed by thememory stalls.
  > This requirement renders many existing multi-threading techniques useless,
  > including light-weight, user-mode threads, known as fibers or stackful coroutine
- Bridging the Latency Gap between NVM and DRAM for Latency-bound Operations\
  https://www.semanticscholar.org/paper/Bridging-the-Latency-Gap-between-NVM-and-DRAM-for-Psaropoulos-Oukid/1b3e3dd80c1ae2c02c6a2745e941d8cccb75f6c1


## Wikipedia - related concepts

- Protothread: https://en.wikipedia.org/wiki/Protothread
- CallCC / Call with Current Continuation: https://en.wikipedia.org/wiki/Call-with-current-continuation
- Coroutines: https://en.wikipedia.org/wiki/Coroutine
- Continuation: https://en.wikipedia.org/wiki/Continuation
- Continuation Passing Style: https://en.wikipedia.org/wiki/Continuation-passing_style
- Delimited Continuation: https://en.wikipedia.org/wiki/Delimited_continuation

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

Rebuttal of the function color myth:
- https://lukasa.co.uk/2016/07/The_Function_Colour_Myth/

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

### Ergonomics

Adding async call vs not adding it:
- https://glyph.twistedmatrix.com/2014/02/unyielding.html

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
- CppCoro by Lewis Baker: https://github.com/lewissbaker/cppcoro
  - Task, generator
  - Scheduler
  - Event, Barrier
  - Cancellation
  - Threadpool
  - File IO
  - Socket IO, IP
  - awaitable traits
  - awaiter, awaitable, scheduler concepts

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
