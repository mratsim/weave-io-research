# Extract from the PhD thesis

## Important considerations

### Debugging

> Another weak point of events is the difficulty of debugging. Debugging event-driven code
> is painful because of the lack of a call stack. Since event handlers return to the event loop
> whenever they are done, the call stack is reduced to the event loop calling the latest event
> handler. There is no hint as to which previous event handler registered the current one, which
> one in turn registered the previous one, and so on. The values of local variables in these
> past handlers are lost as well, although they could sometimes be very valuable to track a bug
> of which the effects are visible only after a few turns of the event loop. This phenomenon,
> sometimes known as “stack ripping” \[Ady+02\], differs from threaded style where one can
> inspect the dynamic stack to determine the nested function calls that led to the current point
> of execution as well as the value of local variables in every intermediary function—except for
> optimised tail calls, which produce a flat stack similar to event-driven style.

### Globals

> Limitation: there is a single, implicit event loop to schedule cooperative threads in CPC
> and every new thread is created attached to this loop. Making the loop explicit would enable
> the creation of several loops; along with the ability to migrate threads from one loop to another,
> it would enable a SEDA style (staged event-driven architecture \[WCB01\]) that is expected to
> make more efficient use of multiple processors. On the other hand, multiple loops involve
> complex synchronisation issues and a clumsier API; we preferred to keep CPC simple and
> understandable. For long running computations and blocking operations, detached threads
> provide an alternative to multiple event loops.

### Coloring

> Native and CPS functions CPC threads might execute two kinds of code: native functions
> and CPS functions (annotated with the cps keyword). Intuitively, CPS functions are inter-
> ruptible and native functions are not: it is possible to interrupt the flow of a block of CPS
> code in order to pass control to another piece of code, to wait for an event to happen or to
> switch to another scheduler (Section 2.1.3). Note that the cps keyword does not mean that
> the function is written in continuation-passing style, but rather that it is to be CPS-converted
> by the CPC translator. Native code, on the other hand, is “atomic”: if a sequence of native
> code is executed in cooperative mode, it must be completed before anything else is allowed to
> run in the same scheduler. As we shall see in Chapter 3, from a more technical point of view,
> CPS functions are compiled by performing a transformation into Continuation-Passing Style
> (CPS), while native functions execute on the native stack.
> There is a global constraint on the call graph of a CPC program: a CPS function may only
> be called by a CPS function; equivalently, a native function can only call native functions—but
> a CPS function may call a native function. This means that at any point in time, the dynamic
> chain consists of a “CPS stack” of cooperating functions, and a “native stack” of regular C
> functions called by the most recent CPS function. Since context switches are forbidden in
> native functions, only the CPS stack needs to be saved and restored when a thread cooperates.

### Limitations

> Fundamental limitations. Not all legal C code is allowable in CPC.The following few lim-
> itations are fundamental to the implementation technique of CPC, unlikely to be lifted in
> a future version: the use of the longjmp library function, and its variants, is not allowed in
> CPC code, and the use of alloca in CPS context yields unpredictable behaviour;3 both of
> these features modify directly the call stack. Although CPC conceals as much as possible to
> the programmer the “stack ripping” phenomenon that occurs when translating threads to
> events, it fails to preserve the illusion of an unaltered stack when these functions are involved.


### Overhead

> Structure of a CPC program Just like a plain C program, a CPC program is a set of functions.
> Functions in a CPC program are partitioned into “CPS” functions and “native” functions;
> a global constraint is that a CPS function can only ever be called by another CPS function,
> never by a native function. The precise set of contexts where a CPS function can be called is
> the set of CPS contexts, defined in Section 2.2.1.
> Intuitively, CPS code is “interruptible”: when in the attached mode, it is possible to
> interrupt the flow of a block of CPS code in order to pass control to another piece of code or
> to wait for an event to happen. Native code, on the other hand, is “atomic”: if a sequence of
> native code is executed in attached mode, it must be completed before anything else on the
> same scheduler is allowed to run.
> Technically, native function calls are executed by using the machine’s native stack. CPS
> function calls, on the other hand, are executed by using a lightweight stack-like structure
> known as a continuation (Section 3.2). This arrangement makes CPC context switches ex-
> tremely fast; the trade-off is that a CPS function call is at least an order of magnitude slower
> than a native call (Section 6.1.2). Thus, computationally expensive code should be imple-
> mented in native code whenever possible.
> Execution of a CPC program starts at a native function called main. This function usually
> starts by registering a number of threads with the CPC runtime (using cpc_spawn), and then
> passes control to the CPC runtime (by calling cpc_main_loop, Section 2.2.2).

> The converted program is then compiled by a standard C compiler and linked to the CPC
> scheduler to produce the final executable.
> All of these passes are well-known compilation techniques for functional programming
> languages, but lambda lifting and CPS conversion are not correct in general for an imperative
> call-by-value language such as C. The problem is that these transformations copy free variables:
> if the original variable is modified, the copy becomes out-of-date and the program yields a
> different result. This is not an issue for the first lambda-lifting pass, because user-defined
> inner functions have a copy semantics for free variables that is compatible with lambda lifting
> (Section 2.2.1), but the second lambda lifting, applied to inner functions introduced by the
> translator during the splitting pass, and the CPS conversion pass might be incorrect.
> Copying is not the only way to handle free variables. When applying lambda lifting to a
> call-by-value language with mutable variables, the common solution is to box variables that
> are actually mutated, and to store them in environments. However, this solution turns out to
> be less efficient:1 adding a layer of indirection hinders cache efficiency and disables a number
> of compiler optimisations. Therefore, CPC strives to avoid boxing as much as possible.
> One key point of the efficiency of CPC is that we do not need to box every mutated variable
> for lambda lifting and CPS conversion to be correct. As we show in Chapter 4, even though C
> is an imperative language, lambda lifting is correct without boxing for most variables, provided
> that the lifted functions are called in tail position. As it turns out, functions issued from the
> splitting pass are always called in tail position, and it is therefore correct to perform lambda
> lifting in CPC while keeping most variables unboxed.
> Only a small number of variables, whose addresses have been retained with the “address
> of ” operator “&”, needs to be boxed: we call them extruded variables. Our experiments with
> Hekate show that 50 % of local variables in CPS functions need to be lifted (ie. duplicated
> by the lambda-lifting pass). Of that number, only 10 % are extruded. In other words, in the
> largest program written in CPC so far, our translator manages to box only 5 % of the local
> variables in CPS functions.

## Scheduler agnostic

> A CPC thread can be scheduled to be run by a native thread (in a thread pool); intuitively, the
> thread “becomes” a native thread. When this happens, we say that the CPC thread has been
> detached from the CPC scheduler. The opposite operation is known as attaching a detached
> thread back to the CPC scheduler. It is of course possible to migrate a detached thread directly
> from one thread pool to another. Multiple thread pools are convenient to execute tasks of
> various duration: with a single thread pool, long-lasting tasks could end up clogging the
> thread pool and prevent short-lived tasks from executing.
