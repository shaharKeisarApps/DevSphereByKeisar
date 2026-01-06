---
id: mastering-wildcards
title: Mastering Wildcards in Kotlin - Unbounded, Upper Bounded, and Lower Bounded
tldr: Wildcards in Kotlin provide flexibility with generic types. Use star projection (*) for unbounded, 'out T' for upper bounded (covariance), and 'in T' for lower bounded (contravariance).
tags: [generics, variance, type-system, kotlin]
difficulty: intermediate
readTimeMinutes: 6
publishedDate: null
relatedTopics: [mastering-jvm-wildcard, understanding-kclass]
---

Wildcards are a powerful feature in Kotlin's type system that provide flexibility when working with generic types. They allow you to work with types in a more flexible way while maintaining type safety.

### Types of Wildcards

#### Unbounded Wildcard (Star Projection)

The star projection `*` is Kotlin's equivalent of Java's `?`. It represents an unknown type.

```kotlin
fun printList(list: List<*>) {
    list.forEach { println(it) }
}
```

In this example, `List<*>` can accept a list of any type. You can read from it, but you can't add elements (except null) because the actual type is unknown.

#### Upper Bounded Wildcard (out)

The `out` modifier represents covariance - the wildcard type is an unknown subtype of the bound.

```kotlin
fun copyNumbers(dest: MutableList<in Number>, src: List<out Number>) {
    for (item in src) {
        dest.add(item)
    }
}
```

`List<out Number>` means you can read `Number` or any of its subtypes (like `Int`, `Double`), but you cannot add to the list.

#### Lower Bounded Wildcard (in)

The `in` modifier represents contravariance - the wildcard type is an unknown supertype of the bound.

```kotlin
interface Comparable<in T> {
    fun compareTo(other: T): Int
}
```

With `in T`, you can pass values of type `T` to methods, but you cannot safely read values of type `T` from the interface.

### Practical Example

```kotlin
class Box<T>(var item: T)

// Producer: Use 'out' - you can read but not write
fun getBoxValue(box: Box<out Any>): Any {
    return box.item // OK to read
    // box.item = "new" // Error: cannot write
}

// Consumer: Use 'in' - you can write but not read safely
fun setBoxValue(box: Box<in String>, value: String) {
    box.item = value // OK to write
    // val x: String = box.item // Error: type is Any?
}
```

### PECS Principle

**Producer Extends, Consumer Super** (PECS) is a useful mnemonic:
- Use `out` (extends) when you're only reading (producing) values
- Use `in` (super) when you're only writing (consuming) values
- Use invariant (no modifier) when you're doing both

### Key Takeaways

1. Star projection (`*`) for complete type flexibility
2. `out T` for reading values (covariance)
3. `in T` for writing values (contravariance)
4. Follow PECS for deciding which to use
