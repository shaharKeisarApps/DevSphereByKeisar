---
id: multibinding-dependency-injection
title: MultiBinding in Dependency Injection
tldr: MultiBinding is a powerful concept in Dependency Injection that allows binding multiple implementations of a dependency into a collection. It enables providing dependencies at runtime for more flexible and dynamic code.
tags: [dependency-injection, dagger, architecture, clean-code]
difficulty: advanced
readTimeMinutes: 6
publishedDate: null
relatedTopics: [ksp-autoservice, understanding-kclass]
---

MultiBinding is a technique used in Dependency Injection to bind multiple implementations of a dependency into a collection, such as a list or map. It enables the injection of multiple objects that share a common interface or superclass, providing a way to manage dependencies dynamically at runtime.

### What is MultiBinding?

MultiBinding allows you to bind several implementations of a common interface or superclass into a collection. This collection can then be injected wherever needed, providing a way to access multiple implementations of a dependency.

For example, if you have several implementations of an `Operation` interface, you can use MultiBinding to inject a list of all available operations into a class that requires them.

### Benefits of MultiBinding

1. **Flexibility**: MultiBinding allows you to provide different implementations of a dependency at runtime, enabling more dynamic and adaptable code.

2. **Modularity**: By grouping related dependencies together, MultiBinding promotes cleaner and more modular code.

3. **Ease of Extension**: You can easily add or remove implementations without changing the dependent classes, making the codebase more maintainable.

### MultiBinding with Dagger

Dagger, a popular DI framework, provides robust support for MultiBinding.

#### Define a Key Annotation

Create a custom annotation to act as a key for the dependencies:

```kotlin
@MapKey
@Target(AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
annotation class OperationKey(val value: KClass<out Operation>)
```

#### Bind Multiple Implementations

Use the `@Binds` and `@IntoMap` annotations to bind multiple implementations:

```kotlin
@Module
@ContributesTo(AppScope::class)
interface OperationModule {
    @Binds
    @IntoMap
    @OperationKey(DialerOperation::class)
    fun bindDialerOperation(operation: DialerOperation): Operation
}
```

#### Inject the Collection

You can now inject the collection of dependencies wherever needed:

```kotlin
class OperationExecutor @Inject constructor(
    private val operations: Map<KClass<out Operation>, Provider<Operation>>
) {
    fun execute(operationClass: KClass<out Operation>) {
        val operation = operations[operationClass]?.get()
        operation?.execute()
    }
}
```

### Real-Life Example

Imagine a scenario where you have an automation system that can perform various operations like dialing, messaging, etc. Each operation is implemented as a separate class, and you want to provide the correct operation at runtime based on user input.

Using MultiBinding, you can bind all available operations into a map and inject them into an executor class. The executor can then select the appropriate operation at runtime, providing a flexible and dynamic solution.
