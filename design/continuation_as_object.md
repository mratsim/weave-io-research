https://github.com/disruptek/cps/discussions/40
Context: https://github.com/disruptek/cps/commit/6b4cdc4aa82b110b09f1d22b59a4d7450713ea52

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

---------------------
