## Background work

From the [Kotlin Hands-on Couroutines example](https://play.kotlinlang.org/hands-on/Introduction%20to%20Coroutines%20and%20Channels/01_Introduction)

### Blocking

If everything happens on the same thread: api call, post processing, etc, the UI freezes unil the resuls are fetched, finished and displayed. BAD 🙅🏻‍♀️

### Launching a new thread

It's possible to easily launch a new thread for any background work:

```kotlin
thread {
	fetchContributors()
}
```

This will **block** a background thread, not the main UI one.

It's important to remember to come back to the UI thread when displaying the results, or doing any other UI changes, otherwise nothing will happen, or the app will just crash.

Whilst the rsults are being fetched, the whole thread blocks - waits for the repo result, and then waits for each of the contributors result. Only then the thread returns.

### Retrofit callback API

Using `Call.enqueue` is useful as it *asychronously* starts the HTTP request and takes a callback as an argument - specifying that should happen after the request had finished.

```kotlin
    enqueue(object : Callback<T> {
        override fun onResponse(call: Call<T>, response: Response<T>) {
            //do something with the reponse
        }

        override fun onFailure(call: Call<T>, t: Throwable) {
            log.error("Call failed", t)
        }
    })
```

There's no guarantee that each `enqueue` call will be called from the same thread. So collecting and processing the final results might be tricky. Usually requirees the use of thread-safe objects to keep track of when all items were processed.

### Suspending functions

_"..instead of blocking the thread, we suspend the coroutine"_

`launch` starts a new computation. This computation is responsible for loading the data and showing the results. This computation is suspendable: while performing the network requests, it gets "suspended" and releases the underlying thread. When the network request returns the result, the computation is resumed.

Coroutines are computations that run on top of threads and can be suspended. By saying "suspended", we mean that the corresponding computation can be paused, removed from the thread, and stored in memory. Meanwhile, the thread is free to be occupied with other activities:

## Coroutines

**Coroutine scope** - its lifetime. Has a reference to a coroutine **context**.

*Coroutine scope* is responsible for the structure and parent-child relationships between different coroutines. We always start new coroutines inside a scope. *Coroutine context* stores additional technical information used to run a given coroutine, like the dispatcher specifying the thread or threads the coroutine should be scheduled on.

Both the below builders define a coroutine scope.

This one blocks the thread that invoked it though

```kotlin
runBlocking {
  launch {
    // suspend functions go here
  }
}
```

This one doesn't

```kotlin
coroutineScope { // scope - creates a scope without starting a new coroutine
  launch { //scope
    // suspend functions go here
  }
}
```

#### Structured concurrency

You want to make sure that you nest all your coroutines, and more specifically, their scopes. This way you'll have a parent-child coroutine structure and you'll be able to cancel the whole flow if the user decides to cancel the operation.

For example:

```kotlin
runBlocking { // creates new scope
  launch {
    // another scope
    async {
      // another scope
    }
  }
}
```

The parent coroutine doesn't finish until all its children do.

However, if using `GlobalScope.async` or `GlobalScope.launch` - you get no structure. The coroutines started from a global scope are all independent and their lifetime is only limited to the whole application. If you cancel the main `global` coroutine, none of its children are cancelled. 

### Threads vs coroutines

Threads run in parallel to each other, coroutines run concurrently. Parallel tasks literally run at the same time, concurrent tasks can start, run, and complete in overlapping time periods. It doesn't necessarily mean they'll ever both be running at the same instant.

### Block vs suspend

From [here](https://medium.com/better-programming/kotlin-coroutines-for-beginners-a54d7fedb206):
A process is blocked when there’s some external reason it can’t be restarted; for example, an I/O device is unavailable or a semaphore file is locked.
A process being suspended means that the OS has stopped executing it, but that could just be for time slicing (multitasking). There’s no implication the process can’t be resumed immediately.

The difference between **blocking** and **suspending** is that if a thread is blocked, no other work happens. If the thread is suspended, other work happens until the result is available.

This means that *suspend functions* in Kotlin can be stopped and resumed and then stopped again and again (by the implementation, not the programmer), letting other work to be done on the thread, interleaving it with the long running, *suspending* task. The name might be misleading, as it **doesn't mean** that the suspend function stops its own, or any other task execution.

Once the suspend function finishes its task and is able to return, the thread will just resume executing as normal, sequentially.

In this example, `updateParticipantName` is a suspend function, because it calls another suspend function - `sendNetworkRequest`. So the thread, when hitting `sendNetworkRequest` will suspend, but once the result is returned and that function finishes executing, the thread is back to normal execution - evaluates the if statement, logs the result and saves it, without interleaving any other tasks the thread might otherwise be doing.

```
suspend fun updateParticipantName(name: String) {

=>  val result = sendNetworkRequest(name)
    if (result == successful) {
      Log.d(result)
      saveResult(result)
  }
}

suspend fun sendNetworkRequest(data: String) {
  ...
}
```


### Hot vs cold

A **hot data source** is when you've got a function which can be active (executing) and returning you data even before or after you've called it. For example, when you initiate a server request in the background, where it can asynchronously return multiple objects.

It's useful to think about **streams** in this instance, when values are arriving one by one.

A **cold data source** is when a function is active only at the time of calling it - not before and not after. Suspending functions are therefore _cold_, because they _suspend_ other operations until a value is returned.

### Examples

```kotlin
fun main() = GlobalScope.launch { //this: CoroutineScope
     print("Main start")
     coroutineScope { //Creates a coroutine scope
         launch {
             delay(500L)
             println("Task from nested launch")
         }
     }
     println("Main end") //This line is not printed until the nested launch completes
 }

fun printWhatever() {
     runBlocking { 
         delay(200)
         print("whatever")
     } 
 }

fun globalFunction() {
     print("Started!")
     main()
     printWhatever()
     print("Finished!")
 }

globalFunction()
```

Produces:

```
Started! //global
Main start //main
whatever //whatever
Finished! // global
Task from nested launch //main
Main end //main
```

Proves that `globalFunction()` happily finishes, whilst `main` still continues. Also, `main` doesn't wait for `printWhatever`, but just run by itself, when it's called.

If the whole thing were running blocking, each function would wait for each execution to finish first.

### Channels vs Flows

#### Channels

1.  a `Channel` is one-time use only - once you stop it you can't re-vive it, you need to use a new one. That's important for stopping and restarting channels on `onPause` and `onResume`
2.  the buffered channels *do not* buffer items when there are no subscribers

#### Types of channels

There are 4 types of channels:

- **Rendezvous** -`Channel<String>()`
  - Either `send` or `receive` suspend. If a `send` is called, but there's no receiver yet, it suspends. If `receive` was called, but nothing was sent yet, that suspends.
- **Buffered** - `Channel<String>(10)`
  - `send` suspends only if the limited number of elements has been sent (the channel is full)
- **Conflated** - `Channel<String>(CONFLATED)`
  - `send` never suspends - a new element will overwrite the previously sent element
- **Unlimited** - `Channel<String>(UNLIMITED)`
  - `send` will never suspend. `receive` suspends only if it's listening to an empty channel. The channel is not _truly_ unlimited - it's limited to `Int.MAX` (2147483647)

**Example:**

```kotlin
fun main() = runBlocking<Unit> {
    val channel = Channel<String>()
    launch { // producer A
        channel.send("A1")
        channel.send("A2")
        log("A done")
    }
    launch { // producer B
        channel.send("B1")
        log("B done")
    }
    launch { // consumer/receiver
        repeat(3) {
            val x = channel.receive()
            log(x)
        }
    }
}
fun log(message: Any?) {
    println("[${Thread.currentThread().name}] $message")
}
```

Prints:

```
[main @coroutine#4] A1
[main @coroutine#4] B1
[main @coroutine#2] A done
[main @coroutine#3] B done
[main @coroutine#4] A2
```

**WHY??**

How many coroutines here? 4 - runBlocking, producers A and B and the receiving one

https://www.youtube.com/watch?v=HpWQUoVURWQ&ab_channel=JetBrainsTV

1. Producer A sends **`A1`** and suspends the whole coroutine, because there is no `receive` call yet
2. Because the first coroutine **suspended**, the thread is free now, and another coroutine can start.
3. Producer B sends **`B1`** and also suspends the coroutine, becuse there is no `receive` call yet
4. The thread is free again and the third coroutine starts - we get a `receive` call. That's where the _`rendezvous`_ happens
5. The Producer A coroutine is ready to be awake again, but the thread is busy receiving and printing elements. It prints the received **`A1`**element.
6. Then **`B1`** is received and Producer B coroutine is ready to be awake again.
7. Now it's time for the 3rd iteration of the loop in the receiver coroutine. There's nothing in the channel though, so it suspends.
8. Then we can awaken the Producer A coroutine - the next call is sending **`A2`** - there is an awaiting `receive` call, so a _`rendezvous`_ happens.
9. However, the receiver coroutine isn't executed straight away, because the main thread is currently still busy - it's ready to we awake, but only after the Producer  A coroutine, and then Producer B coroutines are done again.
10. The Producer A coroutine can continue - the _`rendezvous`_ happened, no need to wait for anything. So we print out the log `A done`. The coroutine finished all its tasks, it finished and can be garbage collected.
11. Next we awaken Producer B coroutine, it prints out `B done`
12. Then the last scheduled cortoutine is run, which receives the **`A2`** value.

If you add `B2` to Producer B before it's done, and iterate 4 times, the printout is different:

```
[main @coroutine#4] A1
[main @coroutine#4] B1
[main @coroutine#2] A done
[main @coroutine#4] A2
[main @coroutine#4] B2
[main @coroutine#3] B done
```

This is because when `A2` is sent, the `rendezvous` happens. The Producer B is awake, it sends `B2` but there isn't a corresponding `receive` call, so that suspends. Then the cosumer receives `A2`, prints it out. Then it calls `receive` again, and there's another element in the channel already, so prints `B2`. The receiver routine finishes, the last one awakens, and prints out `B done`.