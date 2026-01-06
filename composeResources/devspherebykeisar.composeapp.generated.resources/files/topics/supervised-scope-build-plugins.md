---
id: supervised-scope-build-plugins
title: Mastering Supervised Scope Setup with Build Logic Plugin Conventions
tldr: Build logic plugins with convention patterns create reusable, type-safe Gradle configurations. Combined with SupervisorScope for coroutine safety, they enable scalable, maintainable multi-module projects with structured error handling.
tags: [gradle, build-configuration, coroutines, convention-plugins, kotlin-dsl]
difficulty: advanced
readTimeMinutes: 11
publishedDate: null
relatedTopics: [ksp-autoservice, metro-vs-hilt-internals]
---

As Kotlin Multiplatform projects grow, build configuration becomes unwieldy. Duplicated build logic across modules leads to inconsistency, difficult maintenance, and configuration drift. The solution? **Convention plugins** combined with **SupervisorScope** for resilient build-time tasks.

### Convention Plugins: Reusable Build Logic

Convention plugins encapsulate build configuration patterns into reusable, composable units.

#### Traditional Approach (Anti-Pattern)

```kotlin
// build.gradle.kts in EVERY module - lots of duplication!
plugins {
    kotlin("multiplatform")
    kotlin("plugin.serialization")
    id("org.jetbrains.compose")
}

kotlin {
    jvm()
    js(IR) {
        browser()
    }

    sourceSets {
        val commonMain by getting {
            dependencies {
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
                implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.0")
            }
        }
    }
}
```

#### Convention Plugin Approach

Create a shared convention plugin in `build-logic/convention/`:

```kotlin
// build-logic/convention/src/main/kotlin/com.myproject.kmp.gradle.kts
import org.jetbrains.kotlin.gradle.dsl.KotlinMultiplatformExtension

plugins {
    kotlin("multiplatform")
    kotlin("plugin.serialization")
}

extensions.configure<KotlinMultiplatformExtension> {
    // Centralized target configuration
    jvm {
        compilations.all {
            kotlinOptions.jvmTarget = "17"
        }
    }

    js(IR) {
        browser {
            commonWebpackConfig {
                cssSupport.enabled = true
            }
        }
    }

    // iOS targets with consistent configuration
    listOf(
        iosX64(),
        iosArm64(),
        iosSimulatorArm64()
    ).forEach { target ->
        target.binaries.framework {
            baseName = project.name
            isStatic = true
        }
    }

    sourceSets {
        val commonMain by getting {
            dependencies {
                implementation(libs.kotlinx.coroutines.core)
                implementation(libs.kotlinx.serialization.json)
            }
        }

        val commonTest by getting {
            dependencies {
                implementation(kotlin("test"))
                implementation(libs.kotlinx.coroutines.test)
            }
        }
    }
}
```

Now modules simply apply the convention:

```kotlin
// feature-module/build.gradle.kts
plugins {
    id("com.myproject.kmp") // One line!
}
```

### SupervisorScope in Build Scripts

When build scripts execute coroutines (for code generation, validation, etc.), **SupervisorScope** ensures one failure doesn't crash the entire build.

#### Why SupervisorScope in Gradle?

```kotlin
// Without SupervisorScope - one task failure kills everything
tasks.register("generateModels") {
    doLast {
        runBlocking {
            launch { generateUserModel() } // If this fails...
            launch { generateProductModel() } // ...this never runs
            launch { generateOrderModel() }
        }
    }
}
```

#### With SupervisorScope

```kotlin
import kotlinx.coroutines.*

tasks.register("generateModels") {
    doLast {
        runBlocking {
            supervisorScope {
                // Each launch is independent
                val userJob = launch {
                    try {
                        generateUserModel()
                        logger.lifecycle("✅ User model generated")
                    } catch (e: Exception) {
                        logger.error("❌ User model failed: ${e.message}")
                    }
                }

                val productJob = launch {
                    try {
                        generateProductModel()
                        logger.lifecycle("✅ Product model generated")
                    } catch (e: Exception) {
                        logger.error("❌ Product model failed: ${e.message}")
                    }
                }

                val orderJob = launch {
                    try {
                        generateOrderModel()
                        logger.lifecycle("✅ Order model generated")
                    } catch (e: Exception) {
                        logger.error("❌ Order model failed: ${e.message}")
                    }
                }

                // Wait for all, collect results
                awaitAll(userJob, productJob, orderJob)
            }
        }
    }
}
```

### Advanced Convention Plugin Patterns

#### Composable Conventions

Build conventions from smaller, focused plugins:

```kotlin
// build-logic/convention/src/main/kotlin/KmpTargetsConvention.kt
class KmpTargetsConvention : Plugin<Project> {
    override fun apply(target: Project) {
        target.extensions.configure<KotlinMultiplatformExtension> {
            jvm()
            js(IR) { browser() }

            // iOS targets
            iosX64()
            iosArm64()
            iosSimulatorArm64()
        }
    }
}

// build-logic/convention/src/main/kotlin/KmpDependenciesConvention.kt
class KmpDependenciesConvention : Plugin<Project> {
    override fun apply(target: Project) {
        target.extensions.configure<KotlinMultiplatformExtension> {
            sourceSets {
                val commonMain by getting {
                    dependencies {
                        implementation(target.libs.kotlinx.coroutines.core)
                        implementation(target.libs.kotlinx.serialization.json)
                    }
                }
            }
        }
    }
}

// Composite convention plugin
class KmpLibraryConvention : Plugin<Project> {
    override fun apply(target: Project) {
        with(target.pluginManager) {
            apply("org.jetbrains.kotlin.multiplatform")
            apply("org.jetbrains.kotlin.plugin.serialization")
        }

        // Apply sub-conventions
        target.plugins.apply(KmpTargetsConvention::class.java)
        target.plugins.apply(KmpDependenciesConvention::class.java)
    }
}
```

#### Type-Safe Configuration Extensions

Create DSLs for project-specific configuration:

```kotlin
// Extension for API configuration
interface ApiConfiguration {
    val baseUrl: Property<String>
    val timeout: Property<Duration>
    val enableLogging: Property<Boolean>
}

abstract class ApiConfigurationExtension @Inject constructor(
    objects: ObjectFactory
) : ApiConfiguration {
    override val baseUrl: Property<String> =
        objects.property(String::class.java).convention("https://api.example.com")

    override val timeout: Property<Duration> =
        objects.property(Duration::class.java).convention(Duration.ofSeconds(30))

    override val enableLogging: Property<Boolean> =
        objects.property(Boolean::class.java).convention(false)
}

// Convention plugin using the extension
class ApiConvention : Plugin<Project> {
    override fun apply(target: Project) {
        val extension = target.extensions.create(
            "apiConfig",
            ApiConfigurationExtension::class.java
        )

        // Generate configuration code based on extension
        target.tasks.register("generateApiConfig") {
            val baseUrl = extension.baseUrl
            val timeout = extension.timeout
            val logging = extension.enableLogging

            doLast {
                val configFile = target.file("src/commonMain/kotlin/ApiConfig.kt")
                configFile.writeText("""
                    object ApiConfig {
                        const val BASE_URL = "${baseUrl.get()}"
                        const val TIMEOUT_SECONDS = ${timeout.get().seconds}L
                        const val ENABLE_LOGGING = ${logging.get()}
                    }
                """.trimIndent())
            }
        }
    }
}
```

Usage in project:

```kotlin
plugins {
    id("com.myproject.api")
}

apiConfig {
    baseUrl.set("https://production.api.com")
    timeout.set(Duration.ofMinutes(2))
    enableLogging.set(true)
}
```

### Build-Time Code Generation with SupervisorScope

Generate code in parallel with fault isolation:

```kotlin
abstract class CodeGenerationTask : DefaultTask() {
    @get:Input
    abstract val modules: ListProperty<String>

    @get:OutputDirectory
    abstract val outputDir: DirectoryProperty

    @TaskAction
    fun generate() = runBlocking {
        supervisorScope {
            val results = modules.get().map { moduleName ->
                async(Dispatchers.IO) {
                    try {
                        val code = generateCodeForModule(moduleName)
                        val file = outputDir.file("${moduleName}Module.kt").get().asFile
                        file.parentFile.mkdirs()
                        file.writeText(code)

                        Result.success(moduleName)
                    } catch (e: Exception) {
                        logger.error("Failed to generate $moduleName: ${e.message}", e)
                        Result.failure<String>(e)
                    }
                }
            }

            // Await all and collect results
            val outcomes = results.awaitAll()

            val successful = outcomes.count { it.isSuccess }
            val failed = outcomes.count { it.isFailure }

            logger.lifecycle("✅ Generated $successful modules successfully")
            if (failed > 0) {
                logger.warn("⚠️ Failed to generate $failed modules")
            }

            // Fail task only if ALL generations failed
            if (successful == 0) {
                throw GradleException("All code generation failed")
            }
        }
    }

    private fun generateCodeForModule(moduleName: String): String {
        // Your generation logic
        return """
            package generated

            object ${moduleName}Module {
                // Generated code
            }
        """.trimIndent()
    }
}
```

### Version Catalogs with Conventions

Centralize dependency management:

```toml
# gradle/libs.versions.toml
[versions]
kotlin = "2.0.0"
coroutines = "1.8.0"
serialization = "1.6.3"

[libraries]
kotlinx-coroutines-core = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-core", version.ref = "coroutines" }
kotlinx-coroutines-test = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-test", version.ref = "coroutines" }
kotlinx-serialization-json = { module = "org.jetbrains.kotlinx:kotlinx-serialization-json", version.ref = "serialization" }

[bundles]
common-kmp = ["kotlinx-coroutines-core", "kotlinx-serialization-json"]
common-kmp-test = ["kotlinx-coroutines-test"]

[plugins]
kotlin-multiplatform = { id = "org.jetbrains.kotlin.multiplatform", version.ref = "kotlin" }
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
```

Reference in convention plugins:

```kotlin
dependencies {
    add("commonMainImplementation", libs.bundles.common.kmp)
    add("commonTestImplementation", libs.bundles.common.kmp.test)
}
```

### Testing Conventions

Create test conventions for consistent test setup:

```kotlin
class KmpTestConvention : Plugin<Project> {
    override fun apply(target: Project) {
        target.extensions.configure<KotlinMultiplatformExtension> {
            sourceSets {
                val commonTest by getting {
                    dependencies {
                        implementation(kotlin("test"))
                        implementation(target.libs.kotlinx.coroutines.test)
                        implementation(target.libs.turbine)
                        implementation(target.libs.assertk)
                    }
                }
            }
        }

        target.tasks.withType<Test> {
            useJUnitPlatform()

            testLogging {
                events("passed", "skipped", "failed")
                showStandardStreams = true
            }

            // Parallel test execution with SupervisorScope-like behavior
            maxParallelForks = Runtime.getRuntime().availableProcessors()

            reports {
                html.required.set(true)
                junitXml.required.set(true)
            }
        }
    }
}
```

### Error Handling in Build Scripts

Structured error handling with SupervisorScope:

```kotlin
tasks.register("validateProject") {
    doLast {
        runBlocking {
            supervisorScope {
                val validations = listOf(
                    async { validateDependencies() },
                    async { validateSourceSets() },
                    async { validateResources() },
                    async { validateBuildConfig() }
                )

                val results = validations.map { deferred ->
                    try {
                        deferred.await()
                    } catch (e: Exception) {
                        ValidationResult.Failure(e.message ?: "Unknown error")
                    }
                }

                val failures = results.filterIsInstance<ValidationResult.Failure>()

                if (failures.isNotEmpty()) {
                    val report = failures.joinToString("\n") { "  - ${it.reason}" }
                    logger.warn("⚠️ Validation issues:\n$report")

                    // Don't fail build, just warn
                    // Or throw if critical
                    if (failures.size > 2) {
                        throw GradleException("Too many validation failures")
                    }
                }
            }
        }
    }
}

sealed class ValidationResult {
    object Success : ValidationResult()
    data class Failure(val reason: String) : ValidationResult()
}
```

### Conclusion

Convention plugins combined with SupervisorScope create maintainable, fault-tolerant build configurations:

**Benefits:**
- **DRY Build Logic**: Write once, apply everywhere
- **Type Safety**: Leverage Kotlin's type system in build scripts
- **Fault Tolerance**: SupervisorScope isolates failures
- **Scalability**: Easy to add new modules with consistent configuration
- **Testability**: Convention plugins can be unit tested
- **Composability**: Build complex configurations from simple conventions

This pattern scales from small projects to massive multi-module architectures, providing consistency, maintainability, and resilience throughout your build process.