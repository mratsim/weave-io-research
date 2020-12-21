# [F#] Implementing Coroutines (async/await) using continuations

[Coroutines](https://en.wikipedia.org/wiki/Coroutine) or async/await is not a new concept as it was named in 1958 by Melvin Conway. Coroutines had support in languages such as Simula(1962), Smalltalk(1972) and Modula-2(1978) but I think one can argue that coroutines came into mainstream with C# 5(2012) when Microsoft added async/await to C#.

## The problem with subroutines

A thread can only execute a single subroutine at a time as a subroutine starts and then runs to completion. In a server environment subroutines can be problematic as in order to service connections concurrently we would usually would start a thread per connection, this was the idiom a few years ago. A thread depending on the operating system and settings needs about 1 MiB of user stack space meaning 1,024 threads needs 1 GiB of for just user stack space. In addition there's the cost of kernel stack space, book-keeping and scheduling of threads. This drives cost of VM/hardware. This is sometimes known as the [C10K problem](https://en.wikipedia.org/wiki/C10k_problem).

## Why coroutines can help

A coroutine is like a subroutine except it can also pause/yield the execution when the coroutine begins an asynchronous IO operation to let the executing thread do something else while we are waiting for the IO operation to complete. Once the IO operation is completed the coroutine gets picked up by an idle thread and continues to execute. This enables us to service say 1 million connections by having 1 million coroutines executed by 100 threads.

Coroutines are cheaper than threads because the memory footprint requirement is smaller and doesn't require lots of _continuous_ RAM (a rather steep requirement). In addition, the scheduling is simpler because it is cooperative rather than preemptive.

## Why coroutines sometimes hurts

What the C# community has learnt over the years is that the niceness of coroutines has a price.

For one thing the call stack is gone which has huge implications for debugging. Microsoft has improved the tooling over the years to give us back the illusion of call stack.

Another common issue is that we sometimes try to fit the round shape of the coroutine peg into the square hole of a subroutine like so:

```csharp
var task = MyFunctionIsAsync(); // Returns a coroutine as a Task
var result = task.Result; // Sometimes result in a dead-lock?!?!
```

A coroutine might yield execution and to later resume it but by calling `Task.Result` this thread is now blocked waiting for the coroutine to complete. Depending on the `SynchronizationContext` it can mean that the coroutine can only execute on the thread that started it but this thread is blocked waiting for the coroutine to finish => dead-lock.

IMHO; it was a mistake to add `Task.Result` because it made it easy to use coroutines wrongly but what is done is done. It is quite simple; don't call `Task.Result` or `Task.Wait`.

## Implementing coroutines using continuations

C# has specific compiler support to support coroutines and behind the scenes generates state machines when it can and uses continuation when it can't.

I want to demonstrate how to implement coroutines in F# using continuations. This is how F# `Async` works internally but I will skip exception support and accept that there is potential for stack overflow for long coroutine chains. There are solutions for this which are used by F#  `Async` but I want to keep the code clean to not hide the simple underlying idea.

A simple coroutine definition in F# could look like this:

```fsharp
type [<Struct>] Coroutine<'T> = Co of (('T -> unit) -> unit)
```

We start the `Coroutine<_>` by giving it a continuation/callback function: `r: 'T -> unit` that the `Coroutine<_>` invokes when the result `'T` is available.

### The basics

When the result is available the `Coroutine<_>` invokes the continuation `r` immediately like in the `result` coroutine.

```fsharp
let result v =
  Co <| fun r ->
    r v
```

Sometimes we have to wait on a `Task<_>` to complete before we can execute the continuation `r`:

```fsharp
// Creates a Coroutine<_> from a Task<_>
let ofTask (t : Task<_>) =
  Co <| fun r ->
    let cw (t : Task<_>) = r t.Result
    t.ContinueWith (Action<Task<'T>> cw) |> ignore

// Creates a Coroutine<unit> from a Task
let ofUnitTask (t : Task) =
  Co <| fun r ->
    let cw (_ : Task) = r ()
    t.ContinueWith (Action<Task> cw) |> ignore
```

There's an important difference between `Task<_>` and `Coroutine<_>`. The task begins executing immediately on creation but `Coroutine<_>` begins executing once it receives the continuation `r`.

### The Monad

In order to complete the `Coroutine<_>` Monad we define `bind` and `combine`:

```fsharp
let bind (Co c) f =
  Co <|
    fun r ->
      let cr v =
        let (Co d) = f v
        d r
      c cr

let combine (Co c) (Co d) =
  Co <|
    fun r ->
      let cr _ =
        d r
      c cr
```

### Supporting let! syntax

In order to support an experience that looks like async/await in C# we can now define a [computation expression builder](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/computation-expressions):

```fsharp
type CoroutineBuilder () =
  class
    // To support let!
    member x.Bind       (c, f)  : Coroutine<_> = bind     c               f
    member x.Bind       (t, f)  : Coroutine<_> = bind     (ofTask t)      f
    member x.Bind       (t, f)  : Coroutine<_> = bind     (ofUnitTask t)  f
    // To support do!
    member x.Combine    (c, d)  : Coroutine<_> = combine  c               d
    member x.Combine    (t, d)  : Coroutine<_> = combine  (ofUnitTask t)  d
    // To support return
    member x.Return     v       : Coroutine<_> = result   v
    // To support return!
    member x.ReturnFrom c       : Coroutine<_> = c
    member x.ReturnFrom t       : Coroutine<_> = ofTask   t
    member x.Zero       ()      : Coroutine<_> = result   ()
  end
let coroutine = CoroutineBuilder ()
```

### Running a `Coroutine<_>`

We also need a way to invoke a `Coroutine<_>`

```fsharp
let invoke (Co c) r =
  let wc _ = c r
  ThreadPool.QueueUserWorkItem (WaitCallback wc) |> ignore
```

The `Coroutine<_>` will execute in the thread pool to begin with and when done it will call the continuation `r` with the result.

By design we don't expose a `.Result` property, the only way to receive the result of `Coroutine<_>` is through the continuation `r`. This design is to ensure there's no surprise dead-locks from trying to force a coroutine into a subroutine. Note; the continuation `r` can be executed by any thread.

### Downloading google homepage

All of this enables us to write a simple `Coroutine<_>` like this:

```fsharp
let downloadFromUri uri =
  coroutine {
    let wc    = new WebClient ()
    let! txt  = wc.DownloadStringTaskAsync (Uri uri)
    // Coroutine<_> doesn't support use!
    wc.Dispose ()
    return txt
  }

let exampleDownloadFromGoogle =
  coroutine {
    let! txt = downloadFromUri "https://www.google.com/"
    return txt.Length
  }

[<EntryPoint>]
let main argv =
  invoke exampleDownloadFromGoogle <| printfn "Result: %A"

  printfn "Press any key to exit"
  Console.ReadKey () |> ignore

  0
```

Prints the size of the google home that at the time of writing was:

```
Press any key to exit
Result: 45782
```

Note; `Press any key to exit` is printed before the result, this is because `invoke` enqueues a `Coroutine<_>` to be executed by the thread pool and immediately returns. After google responds the `Coroutine<_>` continues executing on the thread pool and ultimately prints the size of the google homepage.

### Downloading google & bing homepages

We can download from both google & bing:

```fsharp
let exampleDownloadFromGoogleAndBing =
  coroutine {
    let! google = downloadFromUri "https://www.google.com/"
    let! bing   = downloadFromUri "https://www.bing.com/"
    return google.Length, bing.Length
  }
```

Prints:

```
Press any key to exit
Result: (45748, 90632)
```

This is good because it doesn't block the executing thread while downloading the homepage, this is bad because we download the homepages sequentially. It would be better to download both at the same time.

### Downloading google & bing homepage in parallel

Using `runInParallel` this is easy to do.

```fsharp
let exampleDownloadFromGoogleAndBingInParallel =
  coroutine {
    // Start the download coroutines in parallel
    let! cgoogle = runInParallel <| downloadFromUri "https://www.google.com/"
    let! cbing   = runInParallel <| downloadFromUri "https://www.bing.com/"

    // Wait for the coroutines to complete
    let! google  = cgoogle
    let! bing    = cbing

    return google.Length, bing.Length
  }
```

This downloads the homepages in parallel. How does it work?

`runInParallel` has the signature: `Coroutine<'T> -> Coroutine<Coroutine<'T>>`. The reason for this is that a `Coroutine<_>` is started when it is passed the continuation `r` but in this case we like the continuation to start additional `Coroutine<_>` before the first `Coroutine<_>` is done. So `runInParallel` starts the `Coroutine<_>` it is passed and then immediately returns a new `Coroutine<_>` that we can wait on for the result. This enables the continuation to start multiple `Coroutine<_>` and then wait for the results.

This is how `Async.StartChild` works as well.

The implementation of `runInParallel` is a bit more intricate than the previous functions but the basic idea is this:

1. Start the `Coroutine<_>` passed to with a special continuation. This continuation checks a shared state if there's a receiver for the result? If yes, invoke the receiver with the result. Otherwise, store the result.
2. Create a new `Coroutine<_>` that once it receives a receiver checks the shared state to see if there's a result available. If yes, invoke the receiver with the value. Otherwise, store the  receiver.

We have to account for multiple executing threads which complicates the code even further.

```fsharp
let runInParallel (Co c) =
  Co <| fun r ->
    let mutable state = Initial

    // Some of the cases in the matches are error cases but as mentioned
    //  for this blog I avoid handling error cases.

    let cr v =
      let update s =
        match s with
        | Initial       -> ValueNone    , HasValue v
        | HasReceiver r -> ValueSome r  , Done
        | HasValue    _ -> ValueNone    , HasValue v
        | Done          -> ValueNone    , Done
      match cas &state update with
      | ValueSome r -> r v
      | ValueNone   -> ()

    // Starts the coroutine
    c <| cr

    let cco cr =
      let update s =
        match s with
          | Initial       -> ValueNone   , HasReceiver cr
          | HasReceiver _ -> ValueNone   , HasReceiver cr
          | HasValue    v -> ValueSome v , Done
          | Done          -> ValueNone   , Done
      match cas &state update with
      | ValueSome v -> cr v
      | ValueNone   -> ()

    // Passes a new coroutine to the receiver immediately
    r <| Co cco
```

### Downloading many pages in parallel

We can make the pattern above a bit more general like so:

```fsharp
let exampleDownloadManyPagesInParallel =
  let download uri =
    coroutine {
      let! text = downloadFromUri uri
      return text.Length
    } |> debugf "Download: %s" (string uri)
  coroutine {
    let uris        = Array.init 10 (fun i -> sprintf "https://gist.github.com/mrange?page=%d" (i + 1))
    let! allLengths = uris |> Array.map download |> runAllInParallel 3
    return allLengths
  } |> time
```

This prints:

```
Press any key to exit
DEBUG - Download: https://gist.github.com/mrange?page=1 - INVOKE
DEBUG - Download: https://gist.github.com/mrange?page=2 - INVOKE
DEBUG - Download: https://gist.github.com/mrange?page=3 - INVOKE
DEBUG - Download: https://gist.github.com/mrange?page=1 - RESULT: 93883
DEBUG - Download: https://gist.github.com/mrange?page=2 - RESULT: 94160
DEBUG - Download: https://gist.github.com/mrange?page=3 - RESULT: 94602
DEBUG - Download: https://gist.github.com/mrange?page=4 - INVOKE
DEBUG - Download: https://gist.github.com/mrange?page=5 - INVOKE
DEBUG - Download: https://gist.github.com/mrange?page=6 - INVOKE
DEBUG - Download: https://gist.github.com/mrange?page=4 - RESULT: 97397
DEBUG - Download: https://gist.github.com/mrange?page=5 - RESULT: 93125
DEBUG - Download: https://gist.github.com/mrange?page=6 - RESULT: 89598
DEBUG - Download: https://gist.github.com/mrange?page=7 - INVOKE
DEBUG - Download: https://gist.github.com/mrange?page=8 - INVOKE
DEBUG - Download: https://gist.github.com/mrange?page=9 - INVOKE
DEBUG - Download: https://gist.github.com/mrange?page=7 - RESULT: 95235
DEBUG - Download: https://gist.github.com/mrange?page=8 - RESULT: 92106
DEBUG - Download: https://gist.github.com/mrange?page=9 - RESULT: 94957
DEBUG - Download: https://gist.github.com/mrange?page=10 - INVOKE
DEBUG - Download: https://gist.github.com/mrange?page=10 - RESULT: 91992
Result: (2342L, [|93883; 94160; 94602; 97397; 93125; 89598; 95235; 92106; 94957; 91992|])
```

Thanks to `debugf` we see that it seems tasks are invoked in batches of 3 as intended.

`debugf` is used to debug `Coroutine<_>`. As mentioned in the prelude a `Coroutine<_>` implies no call stack (the elimination of the call stack is kind of the point), this makes debugging harder. `debugf` helps by printing to the console when a named `Coroutine<_>` is started and completed:

```fsharp
let debug nm (Co c) =
  Co <| fun r ->
    printfn "DEBUG - %s - INVOKE" nm
    let cr v =
      printfn "DEBUG - %s - RESULT: %A" nm v
      r v
    c cr

let debugf fmt = kprintf debug fmt
```

`runAllInParallel` has the signature: `int -> Coroutine<'T> array -> Coroutine<'T array>`. That is it runs the coroutines in the input array in parallel. The integer decides how many it runs in parallel at the same time.

`runAllInParallel` works by creating batches of `Coroutine<_>`. All `Coroutine<_>` in a batch is executed in parallel, when all `Coroutine<_>` in the batch has run to completion the next batch is started.

A better approach would be to keep always keep the maximum number of `Coroutine<_>` running at all times but batching was easier to implement for the purpose of this blog:

```fsharp
let runAllInParallel parallelism (cos : _ array) =
  let chunks = cos |> Array.indexed |> Array.chunkBySize parallelism
  Co <| fun r ->
    let result = Array.zeroCreate cos.Length
    let rec loop i =
      if i < chunks.Length then
        let chunk = chunks.[i]
        let children = chunk |> Array.map (fun (j, co) -> runInParallel co |> join |> map (fun v -> j, v))
        let mutable remaining = children.Length
        let cr (j, v) =
          result.[j] <- v
          if Interlocked.Decrement &remaining = 0 then loop (i + 1)
        for (Co c) in children do
          c cr
      else
        r result
    loop 0
```

### What about CPU bound problems?

`Coroutine<_>` is intended for IO bound tasks where the CPU is sitting most of the time waiting on IO. It is possible using `Coroutine<_>` for CPU bound tasks but you need to explicitly switch to a worker thread. For example; let's say you want to compute the mandelbrot set:

```fsharp
let exampleCPUBoundProblem =

  let d = 2048
  let lines (pixels : byte array) f t =
    // Mandelbrot specific code deleted, it's in the full source code below
    ()

  let colines (pixels : byte array) f t =
    coroutine {
      // Switches to a thread pool thread that will do the work
      do! switchToThreadPool
      lines pixels f t
      return ()
    }
  coroutine {
    let pixels = Array.zeroCreate (r*d)

    let  s  = d / 4
    // Run 4 Coroutine<_> in parallel
    let! s0 = runInParallel <| colines pixels (0*s) (1*s)
    let! s1 = runInParallel <| colines pixels (1*s) (2*s)
    let! s2 = runInParallel <| colines pixels (2*s) (3*s)
    let! s3 = runInParallel <| colines pixels (3*s) (4*s)

    do! s0
    do! s1
    do! s2
    do! s3

    let img     = File.Create "mandelbrot.pbm"
    let sw      = new StreamWriter (img)

    do! sw.WriteAsync  (sprintf "P4\n%d %d\n" d d)
    do! sw.FlushAsync  ()
    do! img.WriteAsync (pixels, 0, pixels.Length)

    // Coroutine<_> doesn't support use!
    sw.Dispose ()
    img.Dispose ()

    return ()
  } |> time
```

The key here is `do! switchToThreadPool`, without it the `colines` would execute on the calling thread until completion and then execute the next `colines` essentially a sequential operation. With `do! switchToThreadPool` the `colines` transfer execution to the thread pool and the continuation can start the other `colines` `Coroutine<_>` each running in the thread pool.

```fsharp
let switchToThreadPool =
  Co <| fun r ->
    // This makes the continuation execute on the thread pool
    //  the calling thread can continue executing other code
    let wc _ = r ()
    ThreadPool.QueueUserWorkItem (WaitCallback wc) |> ignore
```

With `do! switchToThreadPool` it prints:

```
Result: (748L, ())
```

This means the execution took 748 ms.

Without `do! switchToThreadPool` it prints:

```
Result: (1648L, ())
```

This means the execution took 1648 ms.

The speed up we gain from utilizing multiple cores. The reason for the rather poor speed up is that we just split the problem into 4 chunks and starts executing but the way the mandelbrot works the chunks are not even in CPU demand so `colines` will complete unevenly giving poor usage of the CPU resources. Like mentioned; coroutines are intended for IO bound tasks. For CPU bound tasks we should look for other abstractions such as `Parallel.ForEach` which utilizes work stealing algorithms to mitigate the issue of uneven CPU demand.

# Conclusion

There you have it, a functioning `Coroutine<_>` library in F# with the massive caveat that if a continuation throws an exception it all breaks down hard. This is fixable and `Async` in F# supports exceptions by having special continuation functions for exceptions and cancellation of coroutines.

In addition; `Async` in F# also avoids stack overflows by implementing a common functional pattern called trampolines.

As I wanted to keep down the complexity in order to not hide the idea of coroutines using continuations in a forest of error-handling code I just ignored all of it.

To summarize, a coroutine based on continuations can look like this:

```fsharp
type [<Struct>] Coroutine<'T> = Co of (('T -> unit) -> unit)
```

The coroutine is started when it is passed a continuation function, when the result is available the continuation function is invoked.


# [F#] Full source code

```fsharp
module MinimalisticCoroutine =

  type [<Struct>] Coroutine<'T> = Co of (('T -> unit) -> unit)

  module Coroutine =
    open FSharp.Core.Printf

    open System
    open System.Diagnostics
    open System.Threading
    open System.Threading.Tasks

    module Details =
      let inline refEq<'T when 'T : not struct> (a : 'T) (b : 'T) = Object.ReferenceEquals (a, b)

      module Loops =
        let rec cas (rs : byref<'T>) (cs : 'T) u : 'U =
          let v, ns   = u cs
          let acs     = Interlocked.CompareExchange(&rs, ns, cs)
          if refEq acs cs then v
          else cas &rs acs u

      let sw =
        let sw = Stopwatch ()
        sw.Start ()
        sw

      let cas (rs : byref<_>) u =
        let cs = rs
        Interlocked.MemoryBarrier ()
        Loops.cas &rs cs u

      type ChildState<'T> =
        | Initial
        | HasReceiver of ('T -> unit)
        | HasValue    of 'T
        | Done

    open Details

    let result v =
      Co <| fun r ->
        r v

    let bind (Co c) f =
      Co <|
        fun r ->
          let cr v =
            let (Co d) = f v
            d r
          c cr

    let combine (Co c) (Co d) =
      Co <|
        fun r ->
          let cr _ =
            d r
          c cr

    let apply (Co c) (Co d) =
      Co <|
        fun r ->
          let cr f =
            let dr v = r (f v)
            d dr
          c cr

    let map m (Co c) =
      Co <| fun r ->
        let cr v = r (m v)
        c cr

    let unfold uf z =
      Co <| fun r ->
        let ra = ResizeArray 16
        let rec cr p =
          match p with
          | None        -> r (ra.ToArray ())
          | Some (s, v) ->
            ra.Add v
            let (Co c) = uf s
            c cr
        let (Co c) = uf z
        c cr

    let debug nm (Co c) =
      Co <| fun r ->
        printfn "DEBUG - %s - INVOKE" nm
        let cr v =
          printfn "DEBUG - %s - RESULT: %A" nm v
          r v
        c cr

    let debugf fmt = kprintf debug fmt

    let time (Co c) =
      Co <| fun r ->
        let before = sw.ElapsedMilliseconds
        let cr v =
          let after = sw.ElapsedMilliseconds
          r (after - before, v)
        c cr

    let switchToNewThread =
      Co <| fun r ->
        let ts () =  r ()
        let t = Thread (ThreadStart ts)
        t.IsBackground  <- true
        t.Name          <- "Coroutine thread"
        t.Start ()

    let switchToThreadPool =
      Co <| fun r ->
        let wc _ = r ()
        ThreadPool.QueueUserWorkItem (WaitCallback wc) |> ignore

    let join (Co c) =
      Co <| fun r ->
        let cr (Co d) = d r
        c cr

    let runInParallel (Co c) =
      Co <| fun r ->
        let mutable state = Initial

        let cr v =
          let update s =
            match s with
            | Initial       -> ValueNone    , HasValue v
            | HasReceiver r -> ValueSome r  , Done
            | HasValue    _ -> ValueNone    , HasValue v
            | Done          -> ValueNone    , Done
          match cas &state update with
          | ValueSome r -> r v
          | ValueNone   -> ()

        c <| cr

        let cco cr =
          let update s =
            match s with
              | Initial       -> ValueNone   , HasReceiver cr
              | HasReceiver _ -> ValueNone   , HasReceiver cr
              | HasValue    v -> ValueSome v , Done
              | Done          -> ValueNone   , Done
          match cas &state update with
          | ValueSome v -> cr v
          | ValueNone   -> ()

        r <| Co cco

    let runAllInParallel parallelism (cos : _ array) =
      let chunks = cos |> Array.indexed |> Array.chunkBySize parallelism
      Co <| fun r ->
        let result = Array.zeroCreate cos.Length
        let rec loop i =
          if i < chunks.Length then
            let chunk = chunks.[i]
            let children = chunk |> Array.map (fun (j, co) -> runInParallel co |> join |> map (fun v -> j, v))
            let mutable remaining = children.Length
            let cr (j, v) =
              result.[j] <- v
              if Interlocked.Decrement &remaining = 0 then loop (i + 1)
            for (Co c) in children do
              c cr
          else
            r result
        loop 0

    let ofTask (t : Task<_>) =
      Co <| fun r ->
        let cw (t : Task<_>) = r t.Result
        t.ContinueWith (Action<Task<'T>> cw) |> ignore

    let ofUnitTask (t : Task) =
      Co <| fun r ->
        let cw (_ : Task) = r ()
        t.ContinueWith (Action<Task> cw) |> ignore

    let invoke (Co c) r =
      let wc _ = c r
      ThreadPool.QueueUserWorkItem (WaitCallback wc) |> ignore

    type Builder () =
      class
        member x.Bind       (c, f)  : Coroutine<_> = bind     c               f
        member x.Bind       (t, f)  : Coroutine<_> = bind     (ofTask t)      f
        member x.Bind       (t, f)  : Coroutine<_> = bind     (ofUnitTask t)  f
        member x.Combine    (c, d)  : Coroutine<_> = combine  c               d
        member x.Combine    (t, d)  : Coroutine<_> = combine  (ofUnitTask t)  d
        member x.Return     v       : Coroutine<_> = result   v
        member x.ReturnFrom c       : Coroutine<_> = c
        member x.ReturnFrom t       : Coroutine<_> = ofTask   t
        member x.Zero       ()      : Coroutine<_> = result   ()
      end
  let coroutine = Coroutine.Builder ()

  type Coroutine<'T> with
    static member (>>=) (c, f) = Coroutine.bind     f c
    static member (>>.) (c, d) = Coroutine.combine  c d
    static member (<*>) (c, d) = Coroutine.apply    c d
    static member (|>>) (c, m) = Coroutine.map      m c

open MinimalisticCoroutine

open System
open System.IO
open System.Net
open System.Threading.Tasks

open Coroutine

let downloadFromUri uri =
  coroutine {
    let wc    = new WebClient ()
    let! txt  = wc.DownloadStringTaskAsync (Uri uri)
    wc.Dispose ()
    return txt
  }

let exampleDownloadFromGoogle =
  coroutine {
    let! txt = downloadFromUri "https://www.google.com/"
    return txt.Length
  }

let exampleDownloadFromGoogleAndBing =
  coroutine {
    let! google = downloadFromUri "https://www.google.com/"
    let! bing   = downloadFromUri "https://www.bing.com/"
    return google.Length, bing.Length
  }

let exampleDownloadFromGoogleAndBingInParallel =
  coroutine {
    // Start the download coroutines in parallel
    let! cgoogle = runInParallel <| downloadFromUri "https://www.google.com/"
    let! cbing   = runInParallel <| downloadFromUri "https://www.bing.com/"

    // Wait for the coroutines to complete
    let! google  = cgoogle
    let! bing    = cbing

    return google.Length, bing.Length
  }

let exampleDownloadManyPagesInParallel =
  let download uri =
    coroutine {
      let! text = downloadFromUri uri
      return text.Length
    } |> debugf "Download: %s" (string uri)
  coroutine {
    let uris        = Array.init 10 (fun i -> sprintf "https://gist.github.com/mrange?page=%d" (i + 1))
    let! allLengths = uris |> Array.map download |> runAllInParallel 3
    return allLengths
  } |> time

let exampleCPUBoundProblem =
  let d = 2048
  let r = d >>> 3
  let m = 250
  let s = 2.0 / float d

  let rec mandelbrot re im cre cim i =
    if i > 0 then
      let re2   = re*re
      let im2   = im*im
      let reim  = re*im
      if re2 + im2 > 4.0 then
        i
      else
        mandelbrot (re2 - im2 + cre) (reim + reim + cim) cre cim (i - 1)
    else
      i

  let lines (pixels : byte array) f t =
    for y = f to (t - 1) do
      let yo  = y*r
      let cim = float y*s - 1.0
      for xo = 0 to (r - 1)  do
        let x0            = xo <<< 3
        let mutable pixel = 0uy
        for p = 0 to 7 do
          let x = x0 + p
          let cre = float x*s - 1.5
          let i   = mandelbrot cre cim cre cim m
          if i = 0 then
            pixel   <-pixel ||| (1uy <<< (7 - p))
        pixels.[yo + xo] <- pixel

  let colines (pixels : byte array) f t =
    coroutine {
      do! switchToThreadPool
      lines pixels f t
      return ()
    }
  coroutine {
    let pixels = Array.zeroCreate (r*d)

    let  s  = d / 4
    let! s0 = runInParallel <| colines pixels (0*s) (1*s)
    let! s1 = runInParallel <| colines pixels (1*s) (2*s)
    let! s2 = runInParallel <| colines pixels (2*s) (3*s)
    let! s3 = runInParallel <| colines pixels (3*s) (4*s)

    do! s0
    do! s1
    do! s2
    do! s3

    let img     = File.Create "mandelbrot.pbm"
    let sw      = new StreamWriter (img)

    do! sw.WriteAsync  (sprintf "P4\n%d %d\n" d d)
    do! sw.FlushAsync  ()
    do! img.WriteAsync (pixels, 0, pixels.Length)

    sw.Dispose ()
    img.Dispose ()

    return ()
  } |> time

[<EntryPoint>]
let main argv =
  Environment.CurrentDirectory <- AppDomain.CurrentDomain.BaseDirectory

  invoke exampleDownloadFromGoogleAndBingInParallel <| printfn "Result: %A"

  printfn "Press any key to exit"
  Console.ReadKey () |> ignore

  0
```