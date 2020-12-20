# Async/Await - The challenges besides syntax

[4 years after](https://blog.rust-lang.org/2019/05/15/4-Years-Of-Rust.html)
after the release of Rust 1.0, it seems like Rust is now finally getting close
to getting support for `async/await` - a language feature which aims to make it
easier to write programs in an asynchronous fashion (where multiple logical
tasks get multiplexed on a lower number of OS threads).

One of the last steps before the feature is stabilized is choosing the best
possible syntax. The discussions around syntax have triggered an enormous
amount of feedback and follow-up proposals. Resolving the syntax question
is certainly important - however there also exist a couple of other interesting
design decisions which will have a big impact on how we will be able to utilize
`async/await` in practice. Those partially received a lot less attention.

This article series describes a few of those, with the goal of highlighting
some areas which deserve further thoughts and work besides finalizing syntax.

## Interfaces/Traits

Most examples and discussions around `async/await` focus on "concrete"
implementations: Freestanding `async` functions and `async` methods on `struct`s.

However in order to make our code more generic, reusable and testable we also
need abstractions. Rust uses `trait`s for this purpose - other languages utilize
`interfaces`. This chapter describes the current state of how `async/await`
interacts with `trait`s  based on some examples:

Let's say we want to model an asynchronous calculator API. The calculator is
somewhat weird - it might be connected via some very high latency RPC system or
just has a very-very-very slow CPU - therefore the model contains a timer in
order to make it respond slower. Since we can not block the thread in
`async/await` code, we utilize an `await`able timer system.

Based on this description, the concrete implementation might be:

```rust
struct Calculator {
}

impl Calculator {
    async fn add(a: i32, b: i32) -> i32 {
        globalAsyncTimer.delay(1000).await
        a + b
    }
}
```

Now we want to derive an abstraction for it, which potential users of the
`Calculator` can depend upon. The first assumption for someone new to Rusts
`async/await` system might be that it is possible to write something along
those lines:

```rust
trait CalculatorLike {
    async fn add(a: i32, b: i32) -> i32;
}
```

Unfortunately this is not [yet] possible. Async functions are not supported
in traits. The workaround is to desugar the async function signature manually:
An async function is equivalent to to a synchronous function that returns
a `Future` type. Therefore the trait can be modelled in the following fashion:

```rust
trait CalculatorLike {
    type AddResult: Future<Output=i32>;
    fn add(a: i32, b: i32) -> Self::AddResult;
}
```

After we have a created this trait, we want to implement it for the existing
calculator. The good news is that the Rust compiler understands that `async`
blocks are equivalent to blocks with return a `Future`. Therefore we can use an
`async` block in the implementation of this method. The block mimics the original
implementation, and thereby allows us to implement the method body using
`async/await` syntax.

However we will encounter another challenge during implementation:  
`async` blocks and functions both return unnameable `Future` types, which means
we can't specify `AddResult`. That leads us to the following code snippet:

```rust
impl CalculatorLike for Calculator {
    type AddResult = ?; // What belongs here?
    fn add(a: i32, b: i32) -> Self::AddResult {
        async move {
            globalAsyncTimer.delay(1000).await
            a + b
        }
    }
}
```

Unfortunately this issue will prevent us from implementing async interfaces
using `async/await` with the current state of Rusts async ecosystem. In order to
perform this task, we require the ability to name the generated `Future`s.
This is something that the
[existential type](https://github.com/rust-lang/rfcs/blob/master/text/2071-impl-trait-existential-types.md)
feature aims to resolve.

Replacing the line with the question mark for
```rust
existential type AddResult: Future<Output=i32>;
````
will actually allow us to compile the given example if the experimental
existential_type feature is enabled using the respective feature flag.
However the existential type is not yet on the road to stabilization, and the
current implementation exhibits a few bugs that will lead to compiler errors
when used with more complex APIs.

#### Workarounds

In order to mitigate the situation, there exist a couple of workarounds:

##### Manual implementation

We can implement the `Future` manually using combinators or a hand-written state
machine. Unfortunately that removes the benefit of `async/await` and does not
permit to call any `async` function inside the path.

##### Type erasure

We can utilize type erasure via boxing and dynamic dispatch in order to obtain a
nameable future type:

```rust
impl CalculatorLike for Calculator {
    type AddResult = BoxFuture<'static, i32>
    fn add(a: i32, b: i32) -> Self::AddResult {
        async move {
            globalAsyncTimer.delay(1000).await
            a + b
        }.boxed()
    }
}
```

This approach allows to retain the benefit of the `async/await` syntax.  
However in this case a mandatory allocation is introduced, which means we are
leaving the *zero-cost abstractions* road.

### Async interfaces with lifetimes

The calculator example represented up to now a very simple API:
- The async method actually represented a freestanding function, due to
  the lack of a `self` parameter.
- There were no lifetimes involved.

Let's say we want to model a more sophisticated calculator, which also stores
the result of the last calculation in order to make it retrievable via other
methods.

The concrete implementation would be:

```rust
impl Calculator {
    async fn add(&mut self, a: i32, b: i32) -> i32 {
        globalAsyncTimer.delay(1000).await
        self.last_result = a + b;
        self.last_result
    }
}
```

While this implementation is easy to understand and does not introduce a lot more
complexity compared to the stateless implementation, representing the method in
a `trait` introduces a new challenge:

The new method returns a `Future` which borrows `mut self`. It thereby
obtains a lifetime which associates it to the `Calculator` struct. We now also
need to model this aspect in the interface. In order to do this we need yet
another feature which is only on the horizon: [Generic Associated Types (GATs)](https://github.com/rust-lang/rfcs/blob/master/text/1598-generic_associated_types.md).

If GATs get implemented as described we will be able to define the trait as:

```rust
trait CalculatorLike {
    type AddResult<'a>: Future<Output=i32> + 'a;
    fn add<'a>(&'a mut self, a: i32, b: i32) -> Self::AddResult<'a>;
}
```

The zero-cost implementation would again also require the existential-type
feature and follow the previous example.


#### Workarounds

The best workaround for the moment seems to be to avoid interfaces and `Future`s
with an associated lifetime. This can for example be achieved by implementing
types that store their actual state in a reference-counted object (`Arc`),
which then can be shared with the returned `Future`s. Since the `Future` stores
an `Arc` instead of a direct reference, it doesn't require a lifetime in this case.

```rust
struct CalculatorInner {
}

struct Calculator {
    // The generated Future can clone and capture the following Arc
    inner: Arc<CalculatorInner>,
}
```

### Potential next steps

As described in the previous sections, the most viable way to improve this
situation seems to be stabilizing the `existential type` and
`generic associated types` features in order to regain the ability to
use abstractions within async code.

However it seems like there might also exist the alternative of directly
supporting `async fn` in traits - without an exact definition what those desugar
too. This solution might have the downside that it doesn't yet fit into the
general type/trait strategy and might not support trait objects. However it
seems like it could also have the upside that it would be a lot easier to
understand and use newcomers to `async/await`, since those are then no longer
required to understand what `async fn`s desugar too.


## Streams/Sinks/ByteStreams and other interfaces

There exist a couple of very common interfaces in software:
- Streams - which provide the ability to read values of a specific type (and to
  wait for them if not yet available)
- Sinks - which provide the ability to write values of a specific type (and
  which will wait/block if the no values can be written at this point of time
  since the transport channel might be congested)
- Readable and Writable ByteStreams - which provide the ability to send and
  receive bytes in a more optimzed fashion than trying to handle each `byte`
  individually. These are e.g. often used for represented TCP streams, TLS
  streams, file access, etc.

A number of synchronous traits already exists to describe those concepts, e.g.
the `Iterator`, `Read` and `Write` traits. However in order to utilize the
concepts in asynchronous code, we also need asynchronous versions of the interfaces.

The `futures-rs` crate provides traits which aim to model these concepts in the
asynchronous world:
- [Stream](https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.16/futures/stream/trait.Stream.html)
- [Sink](https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.16/futures/sink/trait.Sink.html)
- [AsyncRead](https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.16/futures/io/trait.AsyncRead.html)
- [AsyncWrite](https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.16/futures/io/trait.AsyncWrite.html)

As can be observed from the linked documentation, those types all define one or
multiple `poll()` methods, but none of them refers to `Future`. This means those
types are quite similar to the raw `Future` trait, but they are not exactly
`Future`s.

This property unfortunately makes them only oneway interoperable
with `async/await` (which is based on the concrete `Future` type):

`await` can await the result of operations on these interfaces [*correction later*].
This is possible through adaptor `Future`s that sit on top of the trait (e.g. created via
[StreamExt::next()](https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.16/futures/stream/trait.StreamExt.html#method.next)).
Those adaptors handle the polling underneath.
Or otherwise said: The methods on those interfaces can be
naturally integrated into the `async/await` world. Users of the interface don't
have to manually `poll()` and retry if no result is available, but will just
utilize calls like `stream.next().await;`

However there exists no interoperability in the other direction:
We can't implement those types on top of `async` functions or `Future`s, since
there is no place to perform the `.await` operation - the `_poll()` methods are
synchronous methods and don't return `Future`s.
This hurts the composition aspect of those types:
They can't be constructed using `async/await` operations right now.

As an example, we might think about a `Tee` implementation of the `Sink` trait,
which needs to write all data to two internal `Sink`s.

It would be great if we could write the implementation for this type as:

```rust
struct Tee<S, S2, Item> {
    sink1: S1,
    sink2: S2,
}

impl<S1, S2, Item> Sink<Item> for Tee<S1, S2, Item>
where S1: Sink<Item>, S2: Sink<Item> {
    async fn send(&mut self, item: Item) -> Result<(), Self::Error> {
        self.sink1.send(item.clone()).await?;
        // Wait a bit - just because we can
        global_timer.delay(Duration::from_millis(100)).await;
        self.sink2.send(item).await?;
        Ok(())
    }
}
```

This looks pretty clean, and adding more layers (e.g. for wrapping each `Sink`
in a `Sink` which asynchronously logs the sent item) is very easy.

However this approach is at the current point of time not possible, since `send`
produces a stateful `Future` which needs to get stored outside of
the `Tee` type - whereas for `Sink` only a `poll` method is expected which does
not store any further temporary object outside of the `Sink/Tee` type.

This means those traits can only be implemented "manually" by implementing the
required `poll()` functions in a state-machine like form.

This might currently be one of the most important and most unique challenges for
Rusts `async/await` support at this point of time:
- The interfaces are super important for interoperable software.
- There has not yet been a clean and universally good solution proposed which
  improves on the current gap between `Stream`s and `Future`s.
- Other languages (like JS, C# and Kotlin) naturally compose `Stream`-like types
  on top of `Future`s. However those also typically use boxed and type-erased
  types for interfaces and `async/await` in general.
  Therefore they experience different tradeoffs by
  building stream-like types from individual `Future`s than Rust does.

If Rust sticks to the current implementation of those highly prominent data
types, we might experience an ecosystem fragmentation:
- People who do not want to manually implement those types might introduce their
  own abstractions, which are not compatible with the ones from the
  de-facto-standard library `futures-rs`.
- If another mechanism gets designed to implement those types in an `async/await`
  like fashion (e.g. on top of Generators), that mechanism might still be
  incompatible with normal `Future`s and thereby
  introduce an ecosystem split into 3 different worlds of types:
  - Synchronous types
  - Asynchronous (`Future`-compatible) types
  - Asynchronous (`Stream`-compatible) types

  This is would certainly not be a desirable outcome.

**Edit/correction:**

This section mentioned that `Stream`s and Co would only be oneway compatible with
Futures, in the sense that `async fn`s can read from a `Stream`, but not the
other way around. However this also applies only if the `Stream` implements the
`Unpin` trait, as indicated by [StreamExt::next()](https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.16/futures/stream/trait.StreamExt.html#method.next).
In the more general case `Stream`s can be only manually polled.

### Potential next steps

Before the mentioned traits get stabilized there should be a clear path forward
in order to guarantee bi-directional compatibility with `Future`s and `async fn`s.
In order to achieve that compatibility, a variety of paths could be explored:

One idea had been to to
[redefine the relevant traits on top of `Future`s](https://github.com/rust-lang-nursery/futures-rs/issues/1365).
However it was pointed out that this would introduce another downside
of no longer being able to use those types as trait objects. In addition to that,
this redefinition also introduces a dependency on `generic associated types` being
available. Therefore this also isn't an ideal solution at this point of time.

Another idea might be getting to a common understanding on how those traits can
be implemented on top of `Future`s and `async fn`s. This could be possible
by storing the relevant futures inside the `Stream/...` `struct`, polling it inline there,
and by providing more seamless conversion methods from `async fn`s to `Stream`s.
However at this point of time it has not been proven out that something like
this will work.

There also exist a few efforts that try to define `Stream`s directly on top of
[unstable] generators, e.g.
[here as a futures-rs pull-request](https://github.com/rust-lang-nursery/futures-rs/pull/1548).
However those yet also don't point out how the Stream implementations can `.await`
futures. If this path is further persued care must also be taken that generators
stay an implementation detail, and that the fact that `async fn` is implemented
on top of generators doesn't accidentally get exposed.


