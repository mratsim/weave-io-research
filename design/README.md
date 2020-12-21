# Design

## Important considerations

### Handling of blocking

### Cancellation

### Control

- Is it possible to manually switch to a specific continuation/stack?
  - Use-case 1 is implementing simulation of various threads interleaving
    to formally verify concurrent data structures
    i.e. that no possible thread interleaving lead to undefined behaviors.
  - Use-case 2 is enabling Lua like coroutines.

### Debugging

- What does debugging look like?
- How does it compare to closure iterator or async stack traces?

### Embedded

- Is there a subset or primitives that can be used in embedded?

### Globals

> Limitation: there is a single, implicit event loop to schedule cooperative threads in CPC
> and every new thread is created attached to this loop. Making the loop explicit would enable
> the creation of several loops; along with the ability to migrate threads from one loop to another,
> it would enable a SEDA style (staged event-driven architecture \[WCB01\]) that is expected to
> make more efficient use of multiple processors. On the other hand, multiple loops involve
> complex synchronisation issues and a clumsier API; we preferred to keep CPC simple and
> understandable. For long running computations and blocking operations, detached threads
> provide an alternative to multiple event loops.

### Iterators composition

- Allow iterator composition
- Speed compared to a manual for loop / state machine

- Can replace D-ranges / C++ ranges:
  - https://dlang.org/phobos/std_range.html
  - https://github.com/ericniebler/range-v3

- use-cases:
  - parsers without creating a state machine manually
  - functional programming composition
  - Probably not a replacement to npeg and zero-functional
    for compile-time state machines
    but allow to define state machines at runtime for example from
    a schema read at runtime.

### Keywords

- {.async.}, is too overloaded, CPS transformation is lower level
  we might need {.async.} for actual async:await written on top of CPS
- {.cps.} means that the function will be CPS-ified
  i.e. it is non-atomic as in it is suspendable
  However this is likely to become "what the hell is this?" moment for users.
- {.suspend.} is likely the better keyword.
  A CPS function can be suspended/resumed at
  specific suspension points.

Also we might not want to tag the function but introduce CPS block instead?

### Limitations

> Fundamental limitations. Not all legal C code is allowable in CPC.The following few lim-
> itations are fundamental to the implementation technique of CPC, unlikely to be lifted in
> a future version: the use of the longjmp library function, and its variants, is not allowed in
> CPC code, and the use of alloca in CPS context yields unpredictable behaviour;3 both of
> these features modify directly the call stack. Although CPC conceals as much as possible to
> the programmer the “stack ripping” phenomenon that occurs when translating threads to
> events, it fails to preserve the illusion of an unaltered stack when these functions are involved.


### Overhead

- Allocations, custom allocator

## Scheduler agnostic

> A CPC thread can be scheduled to be run by a native thread (in a thread pool); intuitively, the
> thread “becomes” a native thread. When this happens, we say that the CPC thread has been
> detached from the CPC scheduler. The opposite operation is known as attaching a detached
> thread back to the CPC scheduler. It is of course possible to migrate a detached thread directly
> from one thread pool to another. Multiple thread pools are convenient to execute tasks of
> various duration: with a single thread pool, long-lasting tasks could end up clogging the
> thread pool and prevent short-lived tasks from executing.


## Disadvantages on non-CPS designs

### Threads

- Resources usage (threads and stack ar limited)

### Event loop (async/await)

- Everything must be non-blocking in the event loop
- Debugging: no call stack, only event-loop -> current function
  > Another weak point of events is the difficulty of debugging. Debugging event-driven code
  > is painful because of the lack of a call stack. Since event handlers return to the event loop
  > whenever they are done, the call stack is reduced to the event loop calling the latest event
  > handler. There is no hint as to which previous event handler registered the current one, which
  > one in turn registered the previous one, and so on. The values of local variables in these
  > past handlers are lost as well, although they could sometimes be very valuable to track a bug
  > of which the effects are visible only after a few turns of the event loop. This phenomenon,
  > sometimes known as “stack ripping” [Ady+02], differs from threaded style where one can
  > inspect the dynamic stack to determine the nested function calls that led to the current point
  > of execution as well as the value of local variables in every intermediary function—except for
  > optimised tail calls, which produce a flat stack similar to event-driven style.3
