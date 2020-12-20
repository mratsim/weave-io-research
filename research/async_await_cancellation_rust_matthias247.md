# Async/Await - The challenges besides syntax - Cancellation

This is the second article in a series of articles around Rusts new `async/await`
feature. The first article about interfaces can be found
[here](https://gist.github.com/Matthias247/5e5e7430149bbb04eebf18cf31747fe0).

In this part of the series we want to a look at a mechanism which behaves very
different in Rust than in all other languages which feature `async/await`
support. This mechanism is **Cancellation**.

## Cancellation - an overview

For some applications it can be very useful to cancel ongoing operations before
they have finished - mostly for the reason that their results are no longer out
of any interest and that continuing the operation would needlessly tie up
resources.

Examples for those are:
- In graphical applications (like an IDE) users might start long-runnning
  sub-operations and then cancel and restart them before they are finished (e.g.
  because they changed a program or parameters for the operation).
- A webservice might create a requests for a variety of other services that need
  to be answered before a response for the actual user can be generated.
  However from time to time one of those requests might take an exorbitant amount
  of time (e.g. due to network failures). Since the user won't wait multiple
  minutes for the request to finish anyway, the sub-operations should rather get
  cancelled if no response had been received within a timeout window and an
  error should be sent back to the user. 

## Cancellation in synchronous programs

Before exploring cancellation in asynchronous environments, let's first take a
look on how Cancellation is implemented in "normal" (synchronous) programs:

Every synchronous function is always guaranteed to be run to completion - it
can't be aborted at an arbitrary point of time. Therefore there typically exists
no inbuilt mechanism to cancel an ongoing operation (e.g. on another thread).
Early java versions featured a `Thread.stop` method.
[However it was deprecated since it's flawed](https://docs.oracle.com/javase/1.5.0/docs/guide/misc/threadPrimitiveDeprecation.html)

In order to still support cancellation, different ecosystems defined their own
mechanisms in library form that allow to request and handle a cancellation.

The most well-known form is the `CancellationToken`, as e.g. known from
[C#](https://docs.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken?view=netframework-4.8). A variable of type `CancellationToken` is passed to each
method which is part of the cancellable workflow. Those methods can check
`CancellationToken.IsCancellationRequested` from time to time in order to discover
whether the parent task has requested cancellation. In this case the
subtask can return early (e.g. by throwing an Exception), which leads the
overall method to run to completion.

Other environments utilize variations of this pattern:
- Go uses the [Context](https://blog.golang.org/context) struct in order to
  propagate a cancellation request to subtasks. In Go, the cancellation request
  is signalled via a `Channel` since that easily enables programs to wait for
  either a cancellation request or a state update from another source.
- Java utilizes a thread-local interruption state. If a a thread gets
  `.interrupt()`ed, certain operations will throw and force the subtask to return
  if the exception is not handled otherwise. This can be treated as some kind
  of thread-local `CancellationToken` which is not explicitely passed between
  methods. It comes with the downside of not being usable in asynchronous contexts.

The general workflow of cancellation is similar in those environments:
1. An external source signals a cancellation request.
2. Specific checks inside the sub-operation listen for cancellation-requests and
   lead to early returns - e.g. by throwing exceptions.
3. The operations runs to completion. The initiator of the request will be able
   to retrieve a result for the operation. It either finishes normally, or it
   will signal that the operation aborted early due to the cancellation (e.g. via
   a `OperationCanceledException`).

This behavior has a specific set of properties:
- The cancellation is always only a cancellation request. It can not be enforced.
  Even ifcancellation is requested, the subprocess might continue to run for a certain
  amount of time. Either because no checks for cancellation had been incorporated
  in this specific code path, or because an early abort of the operation would
  cause substantial problems.
- All methods can observe whether a cancellation happend, if a child-method was
  affected by it, and have the ability to react on it. E.g.
  ```csharp
  Result cancellableMethod(CancellationToken ct) {
      var resource = getImportantResource();
      try {
          var result = cancellableSubMethod(ct);
          // The sub-method finished without having being cancelled.
      } catch (OperationCanceledException) {
          // This branch will be taken if the operation was cancelled inside the
          // sub-method.
          // We can run specific code here if required - e.g. for clean-up.
          throw;
      } finally {
          // The source ALWAYS needs to be returned - whether the operation had
          // been cancelled or not.
          returnImportantResource(resource);
      }
  }
  ```

## async/await and cancellation - in other languages

The `async/await` feature allows us to write asynchronous code (not-thread-blocking
code which multiplexes a set of tasks on a lower number of threads)
in a way that resembles traditional synchronous code.
`async/await` and related mechanisms (e.g. stackless coroutines) already exist
in a variety of other programming languages.

The observation of the author is it that cancellation of asynchronous tasks in
those languages follows the pattern that was described for synchronous environments:
- Cancellation is **requested** via a dedicated parameter or an implicit task-local parameter
- Methods are required to observe cancellation status.
- Async methods run to completion even after cancellation.

In particular:
- C# async `Task`s signals cancellation via `CancellationToken` as already described above.
  Lots of .NET [core] framework functions support this parameter in order to
  propagate cancellation requests.
- Kotlin coroutines utilize a task-local cancellation status.
  This adds some extra convenience by parent tasks being able to automatically
  signal child-tasks about the requirement for cancellation.
  The coroutine standard library (e.g. timers and
  channels for coroutines) recognizes cancellation requests and allows the operations
  to abort early when cancelled.
- Javascript doesn't have standardized types for Cancellation. Libraries need to
  define their own. However since `async` Javascript functions will always run
  to completion, the mechanism would also need to follow the described pattern.
- C++ coroutines can utilize a `CancellationToken` defined in the
  [cppcoro library](https://github.com/lewissbaker/cppcoro#Cancellation).

## Cancellation in async Rust

Cancellation in async Rust works differently compared to the mechanisms we look
at so far.

In order to understand how Rusts cancellation mechanism works in detail
and how it impacts users, let's start with a small
example that demonstrates how an async subtask can be cancelled:

```rust
/// This is an operation that will take a long amount of time.
/// We therefore might want to cancel it under certain conditions.
async fn some_long_taking_operation() -> Result<(), Error> {
    // ...
    Ok(())
}

/// This is a timer function that is provided by some async base framework. The
/// method will return (and the generated Future will get completed)
/// when the given duration has elapsed.
async fn delay(duration: Duration) {
    // ...
}

async fn do_operation_with_timeout() -> Result<(), Error> {
    // This might look like it would run the operation. But in reality it is
    // just producing a Future which represents the operation.
    let long_taking_operation_fut = some_long_taking_operation();
    let timer_fut = delay(Duration::from_millis(10_000));

    /// Wait on either the operation to finish or the timer to expire
    select! {
        res = long_taking_operation_fut => {
            // The operation succeeded first. Return the result.
            return res;
        },
        _ = timer_fut => {
            // The timer finished first. The operation is still running as
            // part of long_taking_operation_fut.
            //
            // We could now wait for it to complete by calling
            // long_taking_operation_fut.await. However in this case we want to
            // cancel the operation and return a timeout error to the user.
            //
            // Dropping a Future cancels the associated operation. Therefore we
            // can utilize the following statement to cancel the operation:
            // drop(long_taking_operation_fut);
            // 
            // However we don't need to perform this explicitely, since just
            // returning will lead to long_taking_operation_fut getting dropped
            // and the operation being cancelled.
            return Err(TimeoutError);
        }
    }
}
```

As indicated by this example, cancellation within Rusts `async/await` system
works differently than in the previously described implementations:
- Cancellation is triggered by simply dropping the `Future` which represents the
  async operation.
- Methods are not required to forward cancellation requests or cancellation
  abilities (e.g. by passing on a `CancellationToken`).
- Cancellation is synchronous - since `drop()` is synchronous.
- Cancellation does not consist of separate *cancellation request* and
  *wait for operation to complete* steps. There is only a single *cancel/drop*
  step.
- Cancellation is enforced - submethods can not prefer to ignore or postpone the
  cancellation.

These properties provide some advantages, some disadvantages, and a bit of
surprising behavior:

### The benefits of synchronously-cancellable subtasks

Rusts `async/await` and the underyling `Futures` ecosystem provides the clear
benefit of unconditionally being able cancel a subtask. There does not exist a
requirement for all intermediate methods to also provide
cancellation support by forwarding a `CancellationToken`.

This makes it a lot easier to utilize cancellation later on in a program which
had not been designed with cancellation in mind right from the start.
E.g. if after several month it had been discovered that a certain subtask
requires a timeout it can be added just as described in the example above.
There is no need to refactor lots of code in order to add timeouts.

### The drawbacks of synchronously-cancellable subtasks

Unfortunately this design does not only provide benefits, it also comes with
drawbacks.

#### Support for IO completion based operations

The main drawback is that the underlying operation **must** support
synchronous cancellation. This property is not given for a variety of
asynchronous operations.

Examples for APIs which do not support synchronous cancellation are IO completion
based OS APIs, e.g. `IO completion ports (IOCP)` on windows, the recently
introduced `io_uring` on Linux and a certain amount of other lower
level constructs (e.g. in device-drivers where the end of the operation might
get signalled through an interrupt).

**These APIs can't be exposed through `async fn` in a safe and zero-cost manner.**

In order to understand why, the following example demonstrates how we could try
to implement IOCP based socket support for windows.

Ideally we would like to implement a method like this for an asynchronous Socket:
```rust
async fn receive(self: &mut Socket, buffer: &mut [u8]) -> Result<usize, Error>;
```

This signature is an asynchronous equivalent of the normal synchronous socket API,
We start a receive operation on a socket, which reads data into a target byte array.
There is no requirement for any special higher level buffer type.

We can now start to implement the method on a Socket struct. The following section
provides some pseudo-code on how IOCP based sockets could potentially be wrapped
in a Future.

```rust
struct Socket {
    os_handle: Handle,
}

impl Socket {
    fn receive(&mut self, buffer: &mut [u8]) -> SocketReceiveFuture {
        SocketReceiveFuture::new(self, buffer)
    }
}

struct SocketReceiveFuture<'a> {
    socket: &'a mut Socket,
    buffer: &'a mut [u8],
    op_started: false,
    op_handle: AsyncOperationHandle,
}

impl Future<'a> for SocketReceiveFuture<'a> {
    fn poll(
        &mut self, // We ignore Pin here just to make the example less complicated
        cx: &mut Context<'_>,
    ) -> Poll<Result<usize, Error> {
        match self.op_started {
            false => {
                // Initiate winapi operation
                let (result, op_handle) = WSARecv(
                    self.socket.os_handle,
                    self.buffer.into_c_buffer_pointer(), 1, &self.buffer.len(),
                    more_fancy_winapi_flags...);
                // In winap operations might finish synchronously. In that case
                // the Future is Ready.
                if result != WSA_IO_PENDING {
                    return Poll::Ready(result.into());
                }
                // The operation is pending. Store a reference for it
                self.op_handle = op_handle;
                // We need to wait until the IO completion
                // port signalled that the operation has completed.
                self.op_started = true;
                // Notify an IOCP port/manager that this task is waiting for a
                // completion event on this socket.
                io_completion_port.wait_for_completion(&self.op_handle, cx);
            },
            true => {
                // This should be called once the io_completion_port dequeued
                // the completion event and woke up the task which polls the
                // SocketReceiveFuture. If this happened then this Future is
                // ready.
                if io_completion_port.is_receive_finished(&self.op_handle) {
                    let io_result = WSAGetOverlappedResult(
                        self.socket.os_handle, ...);
                    self.op_started = false;
                    return Poll::ready(io_result.into());
                }
                // Otherwise continue to wait
            }
        }
        Poll::Pending
    }
}
```

We could then use this as
```rust
async fn interact_with_server() {
    let socket = Socket::connect(endpoint).await?;
    let buffer: [u8; 1024] = [0; 1024];
    let bytes_read = socket.receive(buffer).await?;
}
// Do something with received bytes
```

This approach would however only be valid in the absence of cancellation. In
case the async `interact_with_server` operation gets cancelled,
`SocketReceiveFuture` as well as the `buffer` would get dropped immediately.
This introduces a problem: The operation at the OS level might still be running,
and the OS might try to write to the bytes inside `buffer` (which doesn't exist
anymore). This means the cancellation introduced a memory safety issue.

As implementor of the `SocketReceiveFuture` we only have one tool to prevent this
from happening: When the task gets cancelled `SocketReceiveFuture::drop` will
get called, where we could try to cancel the underlying OS operation in order
to prevent the OS from accessing the memory after the `Future` gets destroyed.

Unfortunately it is not possible to synchronously cancel the ongoing receive
operation in this method. There exists a `CancelIoEx` method, but it will only
initiate/request a Cancellation - but it won't make sure that the cancellation
is actually happens and will also not wait until the IO operation completes.

Therefore

```rust
impl<'a> Drop for SocketReceiveFuture<'a> {
    fn drop(&mut self) {
        if self.op_started {
            CancelIoEx(self.op_handle);
        }
    }
}
```

would still not resolve the issue. The background operation could still run
after `drop()` returned and write to `buffer`.

There exists an additional fix for it: We can perform a blocking wait after `CancelIoEx`
in order to wait inside `drop` until the OS has finished processing the operation
and until it does not require the `buffer` anymore.
However since `Cancel` is only a hint to the OS - and since the cancellation might take
a long amount of time or might even never happen - this is not practical in an
async context where the OS thread should never be blocked.

This unfortunately means we have to give up at this point. It is not possible
to implement the method safely and without blocking a thread.

##### Workaround

There exists a workaround for this problem: We can provide the operation an
owned buffer instead of a reference - and make sure that the ownership is kept
intact over the whole duration of the operation. E.g. the caller can pass a
`BytesMut` struct, which we `std::mem::forget()` for the duration of the IOCP
operation and reconstruct it when done. Or we can pass that owned buffer to the
`io_completion_port` which could track oustanding operations together with their
buffers safely - even while they are cancelled. The `io_completion_port` (or
any other central operation manager) would continue to run all the operations in
the background. Even if the user-visible `Socket` and it's associated `Future`s
are gone.

The downside of this workaround is that it requires either an API change or an
extra copy. The API now needs to be
```rust
async fn receive(
    self: &mut Socket,
    buffer: OwnedBufferType) -> Result<OwnedBufferType, Error<OwnedBufferType>>;
```

This requires the user and the API implementor to either agree on specific owned
buffer types that are used or additional generics are required. The API
also gets more complicated to use, since callers always need to make sure to
recover the utilized buffer from the `Result`.

It must also be ok for the calling task to leak the buffer in case the operation
gets cancelled.

Alternatively, the implementation of the Socket can use an owned buffer
internally to read new data from the OS and can copy it to the user-provided
`&mut[u8]` when finished. In this case the original function signature can be kept.
However the downside of this approach is an extra copy of each read byte.

An additional drawback of this mechanism is also that the system now performs
invisible background work that is no longer visible to the foreground application,
but still drains resources.
We can't control or see how much oustanding work the `io_completion_port` still
needs to perform due to cancellations.

##### Summary

Within the current design of Rusts `async` functions and `Future`s it is not
possible to wrap a certain set of asynchronous APIs in a true zero-cost fashion
without degrading interoperability.

While readiness-based asynchronous APIs (as e.g. provided by
`select/poll/epoll/kqueue`) can easily be wrapped, completion based APIs require
workarounds.

This drawback might be not that visible today, since most high performance
server software that could benefit from async/await is running on readiness-based
unix APIs. However it seems like we might see a bigger amount of IO completion
based primitives in the future. E.g. `io_uring` moved into this direction,
and it's documentation describes why it has benefits compared to other models
(e.g. a lower number of required system calls which also get more expensive in the
days of Spectre/Meltdown mitigations). If this happens, the inability to model
model IO completion based operations in simple asynchronous Rust APIs could become
a bigger problem.

### The surprises of synchronously-cancellable subtasks

For new users of `async/await` it can be surprising that cancelled
`async` functions do not run to completion. Instead they can silently return at
any `.await` point.

http://www.randomhacks.net/2019/03/09/in-nightly-rust-await-may-never-return/
already described this behavior a while ago.

As a short recap: A function like

```rust
async fn my_async_method() {
    println!("Step 1");
    timer.delay(1000).await;
    println!("Step 2");
    timer.delay(1000).await;
    println!("Step 3");
    timer.delay(1000).await;
    println!("finished");
}
```

can be cancelled in any of it's `.await` points. Therefore the output of
the function can be anything from

> Step 1

to 

> Step 1  
> Step 2  
> Step 3  
> finished

While this behavior causes no further issues in this example, it can lead to
subtle bugs in other functions.

The following sections describes one example where the author of this article
run into the problem, and discusses some of the fixes for it:

The `async fn` represents a task that is responsible for sending messages that
had been previously been enqueued for sending at another component, which we
refer to as the `message_store`. The `message_store` manages the full lifecycle
of outgoing messages. If the current send task or any other task which interacts
with the connection encounters an error, all tasks that operate on the current
connection get cancelled. They will get restarted if a the connection had been
resumed, and can then resume sending messages.
The send task needs to notify the MessageStore about the outcome of
sending each message. This will allow the store to either request a
retransmission of the message later or to notify other components that the message
had been successfully transmitted.

In this environment, the send task for message had been implemented in roughly
the following fashion:

```rust
impl SendTask {
    async fn execute(&mut self) -> Result<(), Error> {
        loop {
            // Obtain the next message to send from the message store
            let msg_to_send = self.message_store.get_next_outgoing_message();
            if let Some(msg_to_send) = msg_to_send {
                // Send the message using an asynchronous connection type
                let send_res = self.connection.write(msg_to_send.as_bytes()).await;
                // Return the ownership of the message back to the store, so that
                // it can be retransmitted if sending the message failed here.
                self.message_store.signal_completion(
                    msg_to_send, send_res.is_ok());
                if !send_res.is_ok() {
                    return Err(...);
                }
            } else {
                // Wait for an event that signals a new message is available
                self.message_store.has_message_event.wait().await;
            }
        }
    }
}
```

In this example it is crucial for the sending task to return ownership of the
obtained message back to the store after attempting to send it, in order to
make sure that retries for sending the message work. The send task can't just
obtain a reference to the message for the variety of reasons (e.g.
`message_store` keeps all state inside a `Mutex` which needs to get locked again
after each access).

In synchronous code, this snippet would be correct (at least as long no subtle
bugs lead to panics).
However the asynchronous version contains an issue due to the described
Cancellation behavior: If `send_task` gets cancelled while waiting on
`self.connection.write`, the code for returning the message and signaling
completion (`self.message_store.signal_completion`) will never be invoked. This
might lead to either a leak or a crash on the next query to `get_next_outgoing_message`.

Unfortunately the issue is not easy to spot for a beginner, and might also not
be covered by unit-tests (which are less likely to check for error and
cancellation paths).

In order to fix it, we need to make sure that `signal_completion` also gets
called when the operation is cancelled. That means it needs to happen inside
a `drop` function which gets executed when the generated `Future` gets dropped.
The following code demonstrates this, by introducing an additional structure that
acts as a RAII guard for the message:

```rust
struct SendHelper<'a> {
    msg: Option<Message>,
    message_store: MessageStore<'a>,
    send_ok: boolean,
}

impl<'a> SendHelper<'a> {
    async fn do_send<'b>(&'b mut self, connection: &'b Connection) -> bool
    where 'a : 'b {
        if let Some(ref mut msg) = &mut self.msg {
            self.send_ok = self.connection.write(msg.as_bytes()).await.is_ok();
            self.send_ok
        } else {
            false
        }
    }
}

impl<'a> Drop for SendHelper<'a> {
    fn drop(&mut self) {
        // ALWAYS signal the MessageStore about whether the message
        // had been delivered or not.
        let msg = self.msg.take().unwrap();
        self.message_store.signal_completion(msg, self.send_ok);
    }
}

impl SendTask {
    async fn execute(&mut self) -> Result<(), Error> {
        loop {
            // Obtain the next message to send from the message store
            let msg_to_send = self.message_store.get_next_outgoing_message();
            if let Some(msg_to_send) = msg_to_send {
                let mut helper = SendHelper {
                    msg: Some(msg_to_send),
                    message_store: self.message_store,
                    send_ok: false,
                };
                let is_ok = helper.do_send(self.connection).await;
                if !is_ok {
                    return Err(...);
                }
                // At this point `helper` gets dropped and `message_store` will
                // automatically be notified.
            } else {
                // Wait for an event that signals a new message is available
                self.message_store.has_message_event.wait().await;
            }
        }
    }
}
```

As it can be observed, this increases the necessary code size for the example a
lot. In the case of the authors codebase the overhead was even bigger due to a
a variety of trait bounds on the `MessageStore` which all needed to get propagated
to the RAII guard.

Readers might notice that the design for sure would be better if `MessageStore`
would directly return the RAII guard. This is certainly true. However it might
not be aware about the `Connection` type, and the amount of required code
would still stay the same.

In order to avoid the overhead of an additional data structure we can experiment
with a `ScopeGuard`: A closure which gets automatically executed whenever the
owning scope gets destructed. Using a `ScopeGuard` we could try to write the
task as:

```rust
impl SendTask {
    async fn execute(&mut self) -> Result<(), Error> {
        loop {
            // Obtain the next message to send from the message store
            let msg_to_send = self.message_store.get_next_outgoing_message();
            if let Some(msg_to_send) = msg_to_send {
                let mut is_ok = false;

                let _guard = ScopeGuard::new(||{
                    // This code is executed whenever the scope is left, including
                    // the situation when the task gets cancelled.
                    self.message_store.signal_completion(
                        msg_to_send, is_ok);
                });

                // Send the message using an asynchronous connection type
                is_ok = self.connection.write(msg_to_send.as_bytes()).await;
                if !send_res.is_ok() {
                    return Err(...);
                }
                // At the end of this block the message_store would be notified.
            } else {
                // Wait for an event that signals a new message is available
                self.message_store.has_message_event.wait().await;
            }
        }
    }
}
```

However this approach will not pass the borrow checker: `msg_to_send` is used
both inside the closure in an owned fashion as well as by mutable reference
outside of the closure. We can't move `msg_to_send` inside the `ScopeGuard`,
since we would lose the ability to send it that way.

The borrow checker simply won't understand that the code inside `ScopeGuard`
would only be called after `self.connection.write` does not require the message
anymore - e.g. because it is cancelled.

A workaround for this could be to develop a new `ScopeGuard` type which allows
to reference the data that is stored inside it. Another workaround is to use
`RefCell`s to coordinate ownership between the guard and the method.

Rust could potentially also provide a new language construct for asynchronous
cleanup work which would follow different borrow-checker rules. It could understand
that `connection.write` is no longer executing when it starts running. This could
be provided through a hypothetical `defer` or `finally` block, which mimics how
other languages have solved the issue:

```rust
if let Some(msg_to_send) = msg_to_send {
    let mut is_ok = false;

    defer {
        // This code is executed whenever the scope is left. The borrow-checker
        // rules are different and allow to move msg_to_send as long as no
        // other code moved it before.
        self.message_store.signal_completion(
            msg_to_send, is_ok);
    }

    is_ok = self.connection.write(msg_to_send.as_bytes()).await;
    if !send_res.is_ok() {
        return Err(...);
    }
}
```

or

```rust
if let Some(msg_to_send) = msg_to_send {
    let is_ok = try {
        self.connection.write(msg_to_send.as_bytes()).await;
    } finally {
        // This code is also executed in case of a cancellation.
        self.message_store.signal_completion(
            msg_to_send, is_ok);
    }
    if !send_res.is_ok() {
        return Err(...);
    }
}
```

These solutions lead to a significantly lower amount of code.
An interesting observation is that those patterns also reflect
how another language (e.g. C#) might handle an explicit `CancellationException`
that is thrown by `signal_completion` in case of cancellation.

However it's arguable whether they are good ideas or not:
They resemble concepts from other languages which are not idiomatic or required
as much in "synchronous Rust". Therefore introducing them specifically for
asynchronous contexts seems questionable.

The actual mitigation of cancellation related issues might also be only the
secondary concern. The primary concern is how authors of asynchronous code can
gain the ability to detect subtle bugs that are introduced due to
cancellation possibilities, and how they can have confidence about their code
and dependencies being bug-free in scenarios where cancellation is involved.

#### Translation of synchronous to asynchronous Rust code

The described cancellation behavior has an impact on the ability to move
synchronous Rust code into the asynchronous world. On a first glance translating
synchronous Rust code into the async equivalent should be possible by just adding
`async` to all functions, by replacing thread-blocking primitives
(mutexes, sockets, etc) with their async equivalents, and by adding `.await`s.

However due to additional early-returns in case of cancellation this might not
be sufficient. Code that spans `.await` points must be audited for sections
which depend on run-to-completion semantics, and suitable RAII guards
must be inserted that mitigate potential cancellation issues.

## Summary

This document describes the impact of Rusts cancellation mechansims for
asynchronous functions and compares it with how other languages handle
cancellation.

As described, Rusts cancellation mechanisms is very unique and comes with a huge
benefit as well as with a certain set of drawbacks.
We will need to find better ways to deal with the drawbacks in the Future. E.g.
we can explore how to best document and highlight the "early return behavior"
in documentation.

For wrapping IO completion based operations we can at least provide guides which
describe the necessary workarounds in-depth.

While it seems unlikely that we can still discover a mechanisms that enables us
to seemlessly wrap IO completion based operations, we can also perform further
investigations in that direction.

One possible change which would allow to wrap IOCP operations natively and which
would also remove the "early return" behavior is a redefinition of the `Future`
trait: It could be defined in a way where callers are obligated to `poll` the
`Future` until s`Poll::Ready` is returned - without caller being allowed to
perform early `drop`s. 
This change would lead to run-to-completion `async fn`s. But it would also have
some further impliciations. A future article might explore this more in detail.
