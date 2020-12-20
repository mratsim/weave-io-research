Async semantics proposal for Swift
==================================

Modern Cocoa development involves a lot of asynchronous programming using blocks and NSOperations. A lot of APIs are exposing blocks and they are more natural to write a lot of logic, so we'll only focus on block-based APIs.

Block-based APIs are hard to use when number of operations grows and dependencies between them become more complicated. In this paper I introduce *asynchronous semantics* and *Promise type* to Swift language (borrowing ideas from design of throw-try-catch and optionals). Functions can opt-in to become *async*, programmer can compose complex logic involving asynchronous operations while compiler produces necessary closures to implement that logic. This proposal does not propose new runtime model, nor "actors" or "coroutines".

Table of contents
-----------------

* [Common problems with block-based APIs](#common-problems-with-block-based-apis)
* [Async semantics](#async-semantics)
* [Error handling](#error-handling)
* [Functions returning Void](#functions-returning-void)
* [Cancellation](#cancellation)
* [Behavior of defer](#behavior-of-defer)
* [Returning multiple values](#returning-multiple-values)
* [Promise types](#promise-types)
* [map, flatMap and reduce](#map-flatmap-and-reduce)
* [Sync and async functions interoperability](#sync-and-async-functions-interoperability)
* [Conversion Examples](#conversion-examples)
* [Blocking calls](#blocking-calls)
* [Thread safety](#thread-safety)
* [Async types](#async-types)
* [Conversion of ObjC APIs](#conversion-of-objc-apis)

Common problems with block-based APIs
-------------------------------------

#### Problem 1: Pyramid of doom

Sequence of simple operations is unnaturally composed in the nested blocks.

```
func makeSandwich1(completionBlock: (result:Sandwich)->Void) {
    cutBread { buns in
        cutCheese { cheeseSlice in
            cutHam { hamSlice in
                cutTomato { tomatoSlice in 
                    completionBlock(Sandwich([buns, cheeseSlice, hamSlice, tomatoSlice]))
                }
            }
        }
    }
}

makeSandwich1 { sandwich in
    eat(sandwich)
}
```

#### Problem 2: Verbosity and old-style error handling

Handling errors becomes difficult and very verbose.

```
func makeSandwich2(completionBlock: (result:Sandwich?, error:NSError?)->Void) {
    cutBread { buns, error in
        guard let buns = buns else {
            completionBlock(nil, error)
            return
        }
        cutCheese { cheeseSlice, error in
            guard let cheeseSlice = cheeseSlice else {
                completionBlock(nil, error)
                return
            }
            cutHam { hamSlice, error in
                guard let hamSlice = hamSlice else {
                    completionBlock(nil, error)
                    return
                }
                cutTomato { tomatoSlice in 
                    guard let tomatoSlice = tomatoSlice else {
                        completionBlock(nil, error)
                        return
                    }
                    completionBlock(Sandwich([buns, cheeseSlice, hamSlice, tomatoSlice]), nil)
                }
            }
        }
    }
}

makeSandwich2 { sandwich, error in
    guard let sandwich = sandwich else {
        error("No sandwich today")
        return
    }
    eat(sandwich)
}
```


#### Problem 3: Forget to call a completion handler

It's easy to bail out by simply returning without calling the appropriate block. When forgotten, the issue is very hard to debug.

```
func makeSandwich3(completionBlock: (result:Sandwich?, error:NSError?)->Void) {
    cutBread { buns, error in
        guard let buns = buns else {
            return // <- forgot to call the block
        }
        cutCheese { cheeseSlice, error in
            guard let cheeseSlice = cheeseSlice else {
                return // <- forgot to call the block
            }
            ...
        }
    }
}
```


#### Problem 4: Forget to return after calling a completion handler

When you do not forget to call the block, you can still forget to return after that.
Thankfully `guard` syntax protects against that to some degree, but it's not always relevant.

```
func makeSandwich4(recipient:Person, completionBlock: (result:Sandwich?, error:NSError?)->Void) {
    if recipient.isVegeterian {
        if let sandwich = cachedVegeterianSandwich {
            completionBlock(cachedVegeterianSandwich) // <- forgot to return after calling the block
        }
    }
    ...
}
```

#### Problem 5: Conditional execution is hard and error-prone

When trying to conditionally execute some asynchronous function, it becomes a huge pain. Half of the code must be wrapped in a helper "continuation" closure that is conditionally executed.

```
func makeSandwich5(recipient:Person, completionBlock: (result:Sandwich)->Void) {
    let continuation:(contents:SandwichContents)->Void = {
      // ... continue and call completionBlock eventually
    }
    if !recipient.isVegeterian {
        continuation(mashroom)
    } else {
        cutHam { hamSlice in
            continuation(hamSlice)
        }
    }
}
```


#### Problem 6: Joining results from independent operations

Some operations can be executed in parallel, but then must be merged. We can once again use helper closures and a semaphore-like counter to make sure the resulting completion handler runs only once.

This is hugely error-prone as one little mistake would lead to completionBlock being called more than once or never at all.

Note: in this example we assume that all blocks run on the same queue or thread that launched them, so there are no race conditions on shared variable. See [Thread safety](#thread-safety) for more details.

```
func makeSandwich6(recipient:Person, completionBlock: (result:Sandwich)->Void) {
    var semaphore = 0
    var buns:Bread! = nil
    var cheese:Cheese! = nil
    var ham:Ham! = nil
    var tomato:Vegetable! = nil
    let continuation:()->Void = {
        if semaphore == 0 {
            completionBlock(Sandwich([buns, cheeseSlice, hamSlice, tomatoSlice]))
        }
    }
    semaphore++
    semaphore++
    semaphore++
    semaphore++
        
    cutBread { breadSlices in
        semaphore--
        buns = breadSlices
        continuation()
    }
    cutCheese { cheeseSlice in
        semaphore--
        cheese = cheeseSlice
        continuation()
    }
    cutHam { hamSlice in
        semaphore--
        ham = hamSlice
        continuation()
    }
    cutTomato { tomatoSlice in 
        semaphore--
        tomato = tomatoSlice
        continuation()
    }
}
```


#### Problem 7: Continuing on the wrong queue/thread

Many APIs call completion block on their own private queues. User code mostly expects to run on its own private queue (or main thread).

Forgetting to dispatch_async back to the correct queue leads to very hard to debug issues that can manifest sporadically or in some unrelated parts of the program.

```
func makeSandwich7(completionBlock: (result:Sandwich)->Void) {
    cutBread { buns in
        dispatch_async(dispatch_get_main_queue()) {
            cutCheese { cheeseSlice in
                dispatch_async(dispatch_get_main_queue()) {
                    cutHam { hamSlice in
                        dispatch_async(dispatch_get_main_queue()) {
                            cutTomato { tomatoSlice in 
                                dispatch_async(dispatch_get_main_queue()) {
                                    completionBlock(Sandwich([buns, cheeseSlice, hamSlice, tomatoSlice]))
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}
```


Async semantics
---------------

I propose an extension to Swift language that allows to solve all of these problems in the most straightforward way.

Functions can opt-in the *async semantics* by using the `async` keyword and replacing closure argument with the usual return type:

```
// Before:
func makeSandwich(completionHandler: (result:Sandwich)->Void)

// After:
async func makeSandwich() -> Sandwich

// Equivalent to:
func makeSandwich() -> Promise<Sandwich>
```

Now our first example can be rewritten in a more natural way:

```
async func cutBread() -> Bread
async func cutCheese() -> Cheese
async func cutHam() -> Ham
async func cutTomato() -> Vegetable

async func makeSandwich() -> Sandwich {
    let bread  = await cutBread()
    let cheese = await cutCheese()
    let ham    = await cutHam()
    let tomato = await cutTomato()
    return Sandwich([bread, cheese, ham, tomato])
}
```

Under the hood, the compiler rewrites this code using nested closures like in example `makeSandwich1`. Note that every operation starts only after the previous one has completed. 

To allow parallel execution, move `await` from the call to the result when it is needed:

```
async func makeSandwich() -> Sandwich {
    let bread  = cutBread()
    let cheese = cutCheese()
    let ham    = cutHam()
    let tomato = cutTomato()
    return Sandwich([await bread, await cheese, await ham, await tomato])
}
```

In the above example all four operations will start one after another, and then the function will wait for completion of every one of them before returning a sandwich.

In other words, `await` instructs compiler to encapsulate all the remaining code into a closure that is executed when the pending operation completes.

Asynchronous code *begins* execution in the same order as written. Some operations may need to begin upon completion of other operations, others may need to run in parallel. By using `await` in appropriate places you may specify which operations wait for which ones.

Note that `await` does not block the thread, it only tells compiler to organize the remaining calls as a continuation of the awaited operation. There is no risk of deadlock when using `await` and it is not possible to use it to make the current thread wait for the result.


Error handling
--------------

Error handling syntax introduced in Swift 2 works naturally with asynchronous model.

```
// Before
func makeSandwich(completionHandler: (result:Sandwich?, error:NSError?)->Void)

// After
async func makeSandwich() throws -> Sandwich

// Equivalent to:
func makeSandwich() -> ThrowingPromise<Sandwich>
```

Our example thus becomes (compare with the example `makeSandwich2`):

```
async func cutBread() throws -> Bread
async func cutCheese() throws -> Cheese
async func cutHam() throws -> Ham
async func cutTomato() throws -> Vegetable

async func makeSandwich() throws -> Sandwich {
    let bread  = try await cutBread()
    let cheese = try await cutCheese()
    let ham    = try await cutHam()
    let tomato = try await cutTomato()
    return Sandwich(bread + cheese + ham + tomato)
}
```

Internally, compiler rewrites the code using closures and exits early if any intermediate call throws an error. Every `try await` will serialize execution, so `cutCheese` will only begin after `cutBread` completes.

To enable throwing async calls execute in parallel, `try await` must be used after all necessary operations has begun.

```
async func makeSandwich() throws -> Sandwich {
    let bread  = cutBread()
    let cheese = cutCheese()
    let ham    = cutHam()
    let tomato = cutTomato()
    guard let bread  = try await bread
    guard let cheese = try await cheese
    guard let ham    = try await ham
    guard let tomato = try await tomato
    return Sandwich(bread + cheese + ham + tomato)
}
```

In this example we begin all operations at once and then first wait for completion of `cutBread` and exit before checking the results of other operations.

Note that this example throws whenever any operation fails, but does not fail with the earliest operation. `await` and `try await` statements also accept a tuple or an array of pending operations.

```
let (bread, cheese) = try await (cutBread(), cutCheese())
```

This example starts both operations simultaneously, but throws as soon as either of two operations complete with failure.

When the number of operations is dynamic, `try await` can be used with an array:

```
var operations = ingredients.map { prepare($0) }
try await operations
```

Or even shorter:

```
try await ingredients.map(prepare)
```

If no operation throws an error, `try await` returns an array of results.



Functions returning Void
------------------------

Functions returning `Void` work with async semantics like all others. The only limitation is it's impossible to bind the result of an operation to a variable.

```
await doSomething() // returns Void
try wait doSomethingThatThrows() // returns Void
try wait ingredients.map(doSomethingThatThrows) // returns an array of Void
```


Cancellation
------------

There is no built-in language support for scheduling tasks and cancellation. Cancellation must be implemented by the programmer where needed. However, coupled with improved error handling, implementing cancellation becomes easy:

```
@IBAction func makeSandwich(sender:AnyObject) {
    async {
        let bread  = try await kitchen.cutBread()
        let cheese = try await kitchen.cutCheese()
        let ham    = try await kitchen.cutHam()
        let tomato = try await kitchen.cutTomato()
        return Sandwich(bread + cheese + ham + tomato)
    } catch CocoaErrorDomain.UserCancelled {
        // Ignore, user quit the kitchen.
    } catch {
        // Some really interesting error happened
        presentError(error)
    }
}

@IBAction func exitKitchen(sender:AnyObject) {
    kitchen.cancel()
}
```

Internally, `kitchen` may use `NSOperations` or custom `cancelled` flag.



Behavior of `defer`
-------------------

In the asynchronous scope `defer` changes behaviour to the asynchronous one. The deferred statement is executed when the asynchronous operation completes.

```
async func makeSandwich() throws -> Sandwich {
    startSpinner()
    defer stopSpinner() // will be called when error is thrown or when all operations complete and a result is returned.
    let bread  = try await cutBread()
    let cheese = try await cutCheese()
    let ham    = try await cutHam()
    let tomato = try await cutTomato()
    return Sandwich(bread + cheese + ham + tomato)
}
```

Returning multiple values
-------------------------

Block-based APIs may have multiple result arguments (not counting an error argument). These are naturally represented by tuples in async functions:

```
// Before
func makeClubSandwich(completionHandler: (part1:Sandwich?, part2:Sandwich?, error:NSError?)->Void)

// After
async func makeSandwich() throws -> (Sandwich,Sandwich)
```


Promise types
-------------

Asynchronous semantics are implemented with *promises* — types that represent a future value. Promises help compiler generate appropriate callback closures and allow programmer to manipulate asynchronous operations.

There are two types to represent a promised value: `Promise<T>` for non-throwing asynchronous functions and `ThrowingPromise<T>` for throwing ones. Both have a write-only property `value` (of type `T`) and the second one also has a write-only property `error` (of type `ErrorType`).
    
`Promise` is a subclass of `ThrowingPromise` (similar to how a non-throwing function is a subtype of a throwing one).

In many cases you do not work with promises directly and compiler often optimizes them away entirely. You might want to use promises explicitly to schedule pending operations manually or to implement async behaviour using legacy block-based APIs.


map, flatMap and reduce
-----------------------

Standard `map`, `flatMap` and `reduce` methods are extended to `Promise` and `ThrowingPromise`. Compiler implements parallel execution of all mapping operations on promises. It is therefore possible to write something like this:

```
urls.map(fetch).map(parse).map(analyze).reduce([]){ $0.append($1) }
```

In the example above, all invocations of `fetch` will happen at once, then `parse`, `analyze` and `append` will happen in order of completion of every operation. Therefore, the data will be processed as soon as it is available, but the order of the resulting array is not guaranteed.

```
extension Promise<T> {
    func map<T,U>(f:(T) -> U) -> Promise<U>
    func flatMap<T,U>(f:(T) -> Promise<U>) -> Promise<U>
    func reduce<U>(initial: U, combine:(U, T) -> U) -> Promise<U>
}

extension ThrowingPromise<T> {
    func map<T,U>(f:(T) -> U) -> ThrowingPromise<U>
    func flatMap<T,U>(f:(T) -> ThrowingPromise<U>) -> ThrowingPromise<U>
    func reduce<U>(initial: U, combine:(U, T) throws -> U) -> ThrowingPromise<U>
}
```


Sync and async functions interoperability
-----------------------------------------

We have already seen how asynchronous functions call synchronous and asynchronous ones, but we haven't explored who synchronous ones call async ones and how async ones can wrap block-based functions.

#### Calling an async function from a non-async function

Similar to throwing functions, every async function "poisons" the calling scope. You must either propagate the semantics to the scope definition (and mark the outer function `async`) or isolate asynchronous behaviour using an `async { }` block.

In case of a simple action method, it must be made clear which part of the method is executed asynchronously.

```
@IBAction func buttonDidClick(sender:AnyObject) {
    async {
        let sandwich = async makeSandwich()
        imageView.image = sandwich.preview
    }
}
```

Async block is helpful to make it clear when the function actually returns. In the example below it is clear that `return true` happens right after async execution begins, but before it finishes.

```
func makeSandwichIfNeeded() -> Bool {
    if !needsMakingSandwich {
        return false
    }
    async {
        let sandwich = async makeSandwich()
        imageView.image = sandwich.preview
    }
    return true
}
```

In this example only the code executed in `async` scope will have async sematics. The rest of the code is running as usual. In fact, `async { }` syntax is equivalent to `do { }`, but a different keyword is required to make it clear that the code will be running asynchronously.

Like `do`, `async` scope supports `catch` blocks that are also executed with asynchronous semantics.

```
func makeSandwichIfNeeded() -> Bool {
    if !needsMakingSandwich {
        return false
    }
    async {
        let sandwich = async try makeSandwich()
        imageView.image = sandwich.preview
    } catch {
        imageView.image = UIImage(named: "no-sandwich.png")
    }
    return true
}
```

In the example above all code within `async-catch` blocks is executed asynchronously while the function returns `true`.

In `catch` part of the `async-catch` statement it is forbidden to throw errors (no one will be able to catch them).

#### Wrapping async function in a block-based function

With `async-catch` pair of blocks it is easy to wrap a result of asynchronous call in a classic block-based API.

```
func makeSandwichOldWay(completionHandler: (Sandwich?, NSError?)->Void) {
    async {
        let sandwich = async try makeSandwich()
        completionHandler(sandwich, nil)
    } catch {
        completionHandler(nil, error)
    }
}
```

#### Wrapping block-based function in async function

Most ObjC APIs are converted to async functions by the compiler automatically (see the details below). However, if there are some block-based APIs left, they can be wrapped in async function easily. To do that, the function must be declared not with `async` prefix, but with explicit `Promise<...>` or `ThrowingPromise<...>` return type. In its body, the function would then create an appropriate promise value, return it and then resolve it with an error or a result later.

```
// Generic function that has parameters, closure and return value
func makeClassicSandwich(kind:SandwichKind, completionHandler:(result:Sandwich?, error:NSError?)->Void) -> Int

// New API wraps the classic one:
func makeNewSandwich(kind:SandwichKind) -> ThrowingPromise<Sandwich> {
    var promise = ThrowingPromise<Sandwich>()
    makeClassicSandwich(kind) { result, error in
        if result != nil {
            promise.value = result
        } else {
            promise.error = error
        }
    }
    return promise
}

// Exported as:
async func makeNewSandwich(kind:SandwichKind) throws -> Sandwich
```

Conversion Examples
-------------------

Async semantics are mostly provided by compiler with very little runtime support (see [Thread safety](#thread-safety) and [Blocking calls](#blocking-calls)). Internally, compiler rewrites the code in a sequence of nested closures to implement the asynchronous logic.

```
async func f1() -> Int {                                      func f1(callback:(i:Int)->()) {
    return 42                                        =>           callback(42) 
}                                                             }
```                                               

```                                               
async func f2() -> String {                                   func f2(callback:(s:String)->()) {
    return String(f1())                              =>           f1 {
}                                                                     callback(String($0))
                                                                  }
                                                              }
```

```
async func f3(a:String?) throws -> String {                   func f3(a:String?, callback:(r:String?, e:ErrorType?)->()) {
    guard let a = a else {                                        guard let a = a else {
        throw MissingString()                                         callback(nil, MissingString())
    }                                                =>               return
    return a + f2()                                               }
}                                                                 f2 {
                                                                      callback(a + $0, nil)
                                                                  }
                                                              }
```


Blocking calls
--------------

To block the current thread waiting for a result use the `sync!` keyword. This is only safe to do when the callee is executing on a different GCD queue or thread. Otherwise, a deadlock occurs and runtime detects it and crashes.

You can use `sync!` instead of `async` in `async {}` blocks and function calls:

```
func buttonClick(sender:AnyObject) {
    let sandwich = sync! makeSandwich()
    imageView.image = sandwich.preview // executes synchronously
}
```

```
func buttonClick(sender:AnyObject) {
    sync! {
        let sandwich1 = async makeSandwich()
        let sandwich2 = async makeSandwich()
        imageView1.image = sandwich1.preview
        imageView2.image = sandwich2.preview
    }
    // Two sandwiches may be prepared in parallel, but the `buttonClick` call is blocked until both sandwiches are ready.
}
```

Note that `await` schedules remaining operations after itself and never blocks the thread, so it cannot produce deadlocks. 


Thread safety
-------------

For the most part, async semantics only specify how sequence of statements is rewritten in a form of nested closures by the compiler. However, compiler provides a minimal support to ensure thread safety for multi-threaded APIs.

Compiler stores a reference to the current dispatch queue in each hidden block that implements async semantics. Non-global dispatch queues are additionally retained for the duration of asynchronous execution. When the block is called, a check is performed if the referenced queue is compatible with the current one (e.g. both are the same or block's target queue down the chain of target queues is the same as the current queue). If the check succeeds, the block is called as-is. If the queue is not compatible, then `dispatch_async` is used to dispatch the block on the queue it references. If the queue was retained, it is released after calling the block.


Async types
-----------

Classic and async functions have independent types. Neither is a subtype of another one. Also, there is no automatic equivalence between a block-based function and an async one (Obj-C block-based functions must opt-in to be converted to async in Swift programs). Therefore, it is possible to have the following functions declared together in one scope, completely independent from each other:

```
func makeSandwich() -> Sandwich
func makeSandwich(completionHandler: (Sandwich)->())
async func makeSandwich() -> Sandwich
```

It is not possible to override async function with a non-async one, or vice versa.



Conversion of ObjC APIs
-----------------------

Existing ObjC APIs can be exported to Swift simultaneously in two variants: block-based and async one. By default, async version is not exported. Functions and methods should be examined for safety (ensure that block is called once and only once) and use `__async` (or `async` in method definitions) instead of `void` in their return type.

Note that both variants are exported to preserve backwards compatibility with existing code.

```objc
// Before
- (void) makeSandwich:(void(^)())completionHandler;
- (void) makeSandwich:(void(^)(Sandwich* __nonnull sandwich))completionHandler;
- (void) makeSandwich:(void(^)(Sandwich* __nullable sandwich, NSError* __nullable error))completionHandler;
- (void) makeSandwich:(void(^)(Sandwich* __nullable half1, Sandwich* __nullable half2, NSError* __nullable error))completionHandler;
- (void) makeSandwich:(void(^)(NSError* __nullable error))completionHandler;

// After
- (async) makeSandwich:(void(^)())completionHandler;
- (async) makeSandwich:(void(^)(Sandwich* __nonnull sandwich))completionHandler;
- (async) makeSandwich:(void(^)(Sandwich* __nullable sandwich, NSError* __nullable error))completionHandler;
- (async) makeSandwich:(void(^)(Sandwich* __nullable half1, Sandwich* __nullable half2, NSError* __nullable error))completionHandler;
- (async) makeSandwich:(void(^)(NSError* __nullable error))completionHandler;
```

The declarations above are exported as:

```
async func makeSandwich()
async func makeSandwich() -> Sandwich
async func makeSandwich() throws -> Sandwich
async func makeSandwich() throws -> (half1:Sandwich, half2:Sandwich)
async func makeSandwich() throws
```


Thanks
------

@groue and @pierlo for invaluable feedback.


