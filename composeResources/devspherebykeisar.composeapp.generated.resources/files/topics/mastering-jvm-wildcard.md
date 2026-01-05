---
id: mastering-jvm-wildcard
title: Mastering @JvmWildcard and @JvmSuppressWildcards in Kotlin for Seamless Java Interoperability
tldr: Use @JvmWildcard to add wildcards to generic types for Java interop, and @JvmSuppressWildcards to suppress them. Essential for controlling variance when calling Kotlin code from Java.
tags: [java-interop, generics, variance, jvm]
difficulty: advanced
readTimeMinutes: 5
publishedDate: null
relatedTopics: [mastering-wildcards, understanding-kclass]
---

## Overview

When working with Kotlin and Java together, understanding how to control wildcard generation in generic types is crucial for seamless interoperability. Kotlin provides two key annotations for this purpose: `@JvmWildcard` and `@JvmSuppressWildcards`.

## The Problem

By default, Kotlin translates declaration-site variance (`in`/`out`) to Java wildcards. Sometimes you need to control this behavior explicitly.

## @JvmWildcard

The `@JvmWildcard` annotation explicitly adds a wildcard to a generic type when compiling to JVM bytecode.

### Use Case

When you want to ensure a wildcard is present in the generated Java signature, even if Kotlin wouldn't normally generate one.

## @JvmSuppressWildcards

The `@JvmSuppressWildcards` annotation suppresses wildcard generation, forcing an exact type in the Java signature.

### Use Case

When you need Java code to receive exact types instead of wildcards, typically when Java APIs don't accept wildcards.

## Example

```kotlin
// Forces wildcard in Java signature: List<? extends String>
fun processItems(@JvmWildcard items: List<String>) {
    // ...
}

// Suppresses wildcard in Java signature: List<String>
fun processExact(@JvmSuppressWildcards items: List<String>) {
    // ...
}
```

## Best Practices

1. Use `@JvmSuppressWildcards` when integrating with Java libraries that don't support wildcards
2. Use `@JvmWildcard` when you need explicit variance in Java signatures
3. Test your Kotlin-Java interop thoroughly to ensure type safety
4. Understand that these annotations only affect the JVM signature, not Kotlin's type system
