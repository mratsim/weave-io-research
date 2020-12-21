Delimited Continuations
=======================

Delimited continuations manipulate the control flow of programs. Similar to control structures like conditionals or loops they allow to deviate from a sequential flow of control.

We use exception handling as another example for control flow manipulation and later show how to implement it using delimited continuations. Finally, we show that nondeterminism can also be expressed using delimited continuations.


Exception Handling
------------------

Many programming languages provide a mechanism to throw and handle exceptions. When an exception is thrown, the normal flow of control is aborted and a surrounding exception handler is invoked. Exception handling is built into many languages using special keywords to `throw` and `catch` exceptions. A (hypothetical) language may provide the following syntax:

```
try A catch (\e -> B)
```

The result of such an expression is the result of `A` or, if `A` throws an exception `e`, the result of `B`.

For example, the result of

```
try (17+4) catch (\e -> 42)
```

is `21` and the result of

```
try (17 + throw 4) catch (\e -> 42 + e)
```

is `46`.

A call to `throw` transfers control to the surrounding `try/catch` block which determines the final result. Because `try/catch` and `throw` manipulate control flow, they may be called *control operators*.


Delimited Continuations
-----------------------

Let's consider a different kind of control operators. Say, our language provides the following syntax:

```
capture A
escape (\continue -> B)
```

The result of `capture A` is the result of `A` unless `escape` is called inside `A`. If `escape` is called inside `A` then the result of `capture A` is the result of `B`. Inside `B` it is possible to continue the computation of `A`: if `B` contains a call `continue C` then the computation of `A` is continued using `C` as result of the call to `escape`.

The way `capture` and `escape` modify control flow is best explained using examples.

The result of

```
capture (2 * escape (\_ -> 42))
```

is `42` because calling `escape` inside `capture` changes the result of the call to `capture` to the result of the function passed to `escape`. In this case, the original computation inside `capture` is discarded and the result is replaced by the result of a new computation.

We can also continue the original computation inside `capture` by using the `continue` argument of the function passed to `escape`. The result of

```
capture (2 * escape (\continue -> 17 + continue 4))
```

is `25` because the `escape`-call replaces the result of the `capture`-call with `17+x` where `x` is the result of the argument of `capture` using `4` as result of the `escape`-call. In the expression `17 + continue 4` the result of `continue 4` is `2 * 4`, so the result of the whole expression is `17 + (2 * 4) = 25`.

These control operators may seem confusing to the uninitiated but they are quite powerful. For example, we can use them to implement exception handling.

```
throw e = escape (\_ -> (\h -> h e))
try a catch h = (capture (let x = a in \_ -> x)) h
```

The implementation of `throw` uses escape to abort the surrounding computation. The passed function ignores the given continuation and yields a function that takes an exception handler and applies it to the thrown exception.

The implementation of catch uses `capture` to evaluate the argument `a` (suppose our hypothetical language is strict and always evaluates `let`-bound variables.) If `throw` is not called inside `a` then the call to `capture` returns a function that ignores the exception handler and yields the result of `a`.

Using the definitions of `throw` and `try/catch` in terms of delimited continuations, the example

```
try (17+4) catch (\e -> 42)
```

from above translates into

```
(capture (let x = 17+4 in (\_ -> x))) (\e -> 42)
```

which results in

```
(\_ -> 21) (\e -> 42)
  = 21
```

The other example

```
try (17 + throw 4) catch (\e -> 42 + e)
```

translates into

```
(capture (let x = 17 + escape (\_ -> (\h -> h 4)) in \_ -> x)) (\e -> 42 + e)
```

which results in

```
(\h -> h 4) (\e -> 42 + e)
  = (\e -> 42 + e) 4
  = 42 + 4
  = 46
```

as intended.


Nondeterminism and Encapsulated Search
--------------------------------------

Nondeterministic programs make use of an operator `?` to nondeterministically choose one of its argument. For example

```
0 ? 1
```

evaluates to `0` or `1` nondeterministically. 

In order to collect all values of a nondeterministic expression a nondeterministic language may provide the following syntax

```
values (\() -> A)
```

The result of such an expression is a set of all possibly nondeterministic results of `A`. for example, the result of

```
values (\() -> 0 ? 1)
```

is the set `{0,1}` because `0` and `1` are all possible results of `0 ? 1`.

We can implement `?` and `values` using delimited continuations:

```
x ? y = escape (\continue -> continue x ∪ continue y)
values f = capture {f ()}
```

The definition of `?` uses the passed continuation twice to execute the surrounding computation with either of the arguments `x` and `y`. It then collects the results of both computations using set union. This requires that the surrounding computation returns a set which is ensured by the implementation of `values` which uses a singleton set as argument of `capture`.

Using these definitions, the example

```
values (\() -> 0 ? 1)
```

from above translates into

```
capture {(\() -> escape (\continue -> continue 0 ∪ continue 1)) ()}
```

which results in

```
capture {escape (\continue -> continue 0 ∪ continue 1))}
  = {0} ∪ {1}
  = {0,1}
```

as intended.

The definition of `?` using delimited continuations above is problematic. Consider the following derivation:

```
values (\() -> 1?(2?3))
  = capture {1?(2?3)}
  = capture {escape (\c1 -> c1 1 ∪ c1 (escape (\c2 -> c2 2 ∪ c2 3)))}
  = capture ({1} ∪ {escape (\c2 -> c2 2 ∪ c2 3)})
  = ({1} ∪ {2}) ∪ ({1} ∪ {3})
  = {1,2,3}
```

While the result is correct (because sets abstract from result miltiplicity), it seems suspicious that the result `1` is included twice. It gets even more problematic if we incorporate failure. Apart from nondeterministic choice, nondeterministic languages also provide an operation `failure` that does not have any results. For example

```
values (\() -> failure)
```

should represent the empty set `∅`. Moreover, if `failure` is part of a computation, as in `1 + failure`, then the result of this computation should itself fail to produce results. On the other hand, if `failure` is an argument of `?` then there might still be results from the other argument of `?`.

We can define `failure` in terms of delimited continuations.

```
failure = escape (\_ -> ∅)
```

With this definition the expression `1 + failure` indeed does not yield any results:

```
values (\() -> 1 + failure)
  = capture {1 + escape (\_ -> ∅)}
  = ∅
```

However, the expression `failure ? 42` also does not yield any results:

```
values (\() -> failure ? 42)
  = capture {failure ? 42}
  = capture {escape (\c -> c failure ∪ c 42)}
  = capture ({failure} ∪ {42})
  = capture ({escape (\_ -> ∅)} ∪ {42})
  = ∅
```

This result is not intended. Instead the result should be `{42}`.

We can fix both the duplication of nested nondeterminism above and the unintended failure observed here by using a different definition of `?`:

```
x ? y = escape (\continue -> capture (continue x) ∪ capture (continue y))
```

Here, we use additional calls to `capture` to delimit the effect of failure or nondeterminism in the arguments of set union.

With this new definition we can proceed as follows to investigate the problematic examples:

```
values (\() -> 1?(2?3))
  = capture {1?(2?3)}
  = capture {escape (\c1 -> capture (c1 1) ∪ capture (c1 (escape (\c2 -> capture (c2 2) ∪ capture (c2 3)))))}
  = capture {1} ∪ capture {escape (\c2 -> capture (c2 2) ∪ capture (c2 3))}
  = {1} ∪ (capture {2} ∪ capture {3})
  = {1,2,3}
```

Note that now the result `1` is not duplicated anymore because `{1}` is not part of the delimited continuation `c2`. Regarding the unintended failure consider the following derivation:

```
values (\() -> failure ? 42)
  = capture {failure ? 42}
  = capture {escape (\c -> capture (c failure) ∪ capture (c 42))}
  = capture {failure} ∪ capture {42}
  = capture {escape (\_ -> ∅)} ∪ {42}
  = ∅ ∪ {42}
  = {42}
```

Now, the result is `{42}` as intended.

Further Reading
---------------

Usually, the control operators `escape` and `capture` are called `shift` and `reset`, respectively. A more detailed introduction to delimited continuations with `shift` and `reset` can be found in the [tutorial notes](http://pllab.is.ocha.ac.jp/~asai/cw2011tutorial/) of a tutorial given at the [Continuation Workshop 2011](http://logic.cs.tsukuba.ac.jp/cw2011/).