---
id: deferred-and-await
title: Understanding Deferred and Await in Kotlin Coroutines
tldr: Deferred objects and the await() function are essential tools for managing asynchronous operations in Kotlin Coroutines. By returning a Deferred object from a coroutine, you can access the result of a long-running operation later, when it becomes available.
tags: [coroutines, async, concurrency, kotlin]
difficulty: intermediate
readTimeMinutes: 5
publishedDate: null
relatedTopics: [hot-cold-streams]
---

## Overview

In Kotlin Coroutines, Deferred objects represent the result of a long-running or asynchronous operation. A coroutine can return a Deferred object, and then continue executing other code while the operation is ongoing. Later, when the result becomes available, you can use the **`await()`** function to suspend the coroutine until the result is ready.

## Example: Starting a Car Engine

Here's an example using the scenario of starting a car engine:

```kotlin
suspend fun startEngine(): Deferred<String> = coroutineScope {
    async { // Simulate long-running operation
        delay(5000)
        "Engine started!"
    }
}

suspend fun driveCar() { // Start the engine and get a Deferred object
    val engineDeferred = startEngine()
    // Do some other stuff while the engine starts
    println("Buckle up and adjust your mirrors...")
    // Wait for the engine to start and get the result
    val engineResult = engineDeferred.await()
    // Drive the car with the engine started
    println(engineResult + " Let's hit the road!")
}
```

In this example, `startEngine()` returns a `Deferred<String>` that represents the asynchronous engine starting operation. Meanwhile, `driveCar()` can continue with other tasks like buckling up. When it's time to drive, it calls `await()` to get the final result.

## Key Takeaways

- **Deferred objects** represent future results of asynchronous operations
- **`await()`** suspends execution until the result is ready
- You can do other work while waiting for deferred results
- This pattern enables efficient concurrent programming
