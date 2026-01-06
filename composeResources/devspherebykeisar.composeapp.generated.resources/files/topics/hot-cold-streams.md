---
id: hot-cold-streams
title: Hot and Cold Streams in Reactive Programming
tldr: In reactive programming, hot streams emit events regardless of subscribers, while cold streams emit only when subscribed. In Kotlin, use SharedFlow for hot streams and Flow for cold streams. In Swift, use Timer.publish for hot and Future for cold.
tags: [reactive-programming, flow, coroutines, swift, cross-platform]
difficulty: intermediate
readTimeMinutes: 8
publishedDate: null
relatedTopics: [deferred-and-await]
---

The current week includes both hot and cold days, just like the streams in Reactive Programming.

In reactive programming, there are two types of streams: hot and cold. Hot streams emit events regardless of whether there are subscribers, while cold streams emit events only when there is at least one subscriber.

### Hot Streams

#### Swift (Combine Framework)

In Swift's Combine framework, a hot stream can be created using the `Timer.publish` operator. This operator creates a publisher that sends a value at a regular interval.

```swift
import Combine

// Create a hot publisher that sends the current speed of a car every 1 second
let hotPublisher = Timer.publish(every: 1, on: .main, in: .default)
    .autoconnect()
    .map { _ in
        // Simulate the current speed of the car
        return Int.random(in: 0...120)
    }
    .share()
```

The `.autoconnect()` operator ensures that the publisher starts sending values as soon as the first subscriber attaches. The `.share()` operator shares the publisher among subscribers, ensuring they all receive the same events.

#### Kotlin (Flow Framework)

In Kotlin's Flow framework, a hot stream can be created using the `MutableSharedFlow` type:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

// Create a hot stream that sends the current speed of a car every second
val hotFlow = MutableSharedFlow<Int>(replay = 1)

CoroutineScope.launch {
    while (true) {
        // Simulate the current speed of the car
        val speed = (0..120).random()
        hotFlow.emit(speed)
        delay(1000)
    }
}
```

### Cold Streams

#### Swift (Combine Framework)

A cold stream can be created using the `Future` type, which sends a single value when requested by a subscriber:

```swift
import Combine

// Create a cold publisher that sends the current speed of a car when requested
let coldPublisher = Future<Int, Never> { promise in
    // Simulate the current speed of the car
    let speed = Int.random(in: 0...120)
    promise(.success(speed))
}

// Subscriber 1
let subscriber1 = coldPublisher
    .sink { speed in
        print("Current speed: \(speed)")
    }
```

#### Kotlin (Flow Framework)

In Kotlin, a cold stream can be created using the `flowOf` function:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

// Create a cold stream that sends the current speed of a car
val coldFlow = flowOf((0..120).random())

// Subscriber 1
val job1 = CoroutineScope.launch {
    coldFlow.collect { speed ->
        println("Current speed: $speed")
    }
}
```

### Conclusion

Understanding the difference between hot and cold streams is important for designing efficient and responsive applications. Hot streams can be used to create streams that emit events at a fixed rate, while cold streams can be used to create streams that emit events from a collection or that are expensive to compute.

Have a sweet cold ~ hot week! 🌤️ 🌦️
