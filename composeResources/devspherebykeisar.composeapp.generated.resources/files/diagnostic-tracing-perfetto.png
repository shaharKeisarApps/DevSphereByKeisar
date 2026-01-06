---
id: diagnostic-tracing-perfetto
title: Advanced Diagnostics with Kotlin Tracing and Perfetto Performance Analysis
tldr: Perfetto tracing provides microsecond-precision performance diagnostics for Kotlin apps. Combined with custom trace sections, it reveals bottlenecks in coroutines, compose recompositions, and background work that traditional profilers miss.
tags: [performance, diagnostics, tracing, perfetto, profiling, debugging]
difficulty: advanced
readTimeMinutes: 13
publishedDate: null
relatedTopics: [collectasretainedstate-push-architecture, supervised-scope-build-plugins]
---

Traditional CPU profilers sample stack traces periodically (typically every 10ms). This misses:
- **Microsecond-level operations** - Compose recompositions, coroutine suspensions
- **Async work** - Coroutine context switches, dispatcher changes
- **System interactions** - Frame rendering, GC, I/O waits

**Perfetto tracing** captures every event with nanosecond precision, revealing the complete execution story.

### Understanding Perfetto

Perfetto is Android's system-wide tracing framework that captures:
- CPU scheduling (which thread ran when)
- Frame rendering (composition, layout, draw phases)
- Custom trace sections (your code)
- System events (GC, binder calls, I/O)

#### Trace Anatomy

```kotlin
// Traditional profiler sees: someFunction took 100ms
fun someFunction() {
    // What actually happened:
    // - Network call: 80ms
    // - JSON parsing: 15ms
    // - Database write: 5ms
    // Profiler can't distinguish these!
}

// With Perfetto tracing
fun someFunction() = trace("someFunction") {
    trace("network") {
        api.fetchData() // See exactly: 80ms
    }
    trace("parsing") {
        json.parse(data) // See exactly: 15ms
    }
    trace("database") {
        db.insert(parsed) // See exactly: 5ms
    }
}
```

### Setting Up Perfetto Tracing

#### 1. Add Tracing Dependencies

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.tracing:tracing-ktx:1.2.0")
    implementation("androidx.tracing:tracing-perfetto:1.0.0")

    // For binary tracing (production)
    implementation("androidx.tracing:tracing-perfetto-binary:1.0.0")
}
```

#### 2. Initialize Tracing

```kotlin
// In Application.onCreate()
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        // Enable Perfetto tracing
        PerfettoTrace.enable(
            TracingConfig.Builder()
                .setBufferSizeKb(20_000) // 20MB buffer
                .setBufferFillPolicy(BufferFillPolicy.DISCARD)
                .build()
        )

        // Register tracing for entire app lifecycle
        lifecycle.addObserver(
            AppTracingLifecycleObserver()
        )
    }
}
```

### Core Tracing Patterns

#### Function-Level Tracing

```kotlin
// Extension function for clean tracing
inline fun <T> trace(sectionName: String, block: () -> T): T {
    Trace.beginSection(sectionName)
    try {
        return block()
    } finally {
        Trace.endSection()
    }
}

// Usage
fun loadUserData(userId: String): User = trace("loadUserData") {
    val userData = trace("fetchUserApi") {
        api.getUser(userId)
    }

    val preferences = trace("fetchPreferences") {
        prefsRepository.getPreferences(userId)
    }

    trace("combineData") {
        User(userData, preferences)
    }
}
```

#### Coroutine Tracing

Trace coroutine execution across context switches:

```kotlin
// Tracing dispatcher wrapper
class TracingCoroutineDispatcher(
    private val delegate: CoroutineDispatcher,
    private val name: String
) : CoroutineDispatcher() {

    override fun dispatch(context: CoroutineContext, block: Runnable) {
        delegate.dispatch(context) {
            trace("$name.dispatch") {
                block.run()
            }
        }
    }
}

// Use in DI setup
val tracingIODispatcher = TracingCoroutineDispatcher(
    Dispatchers.IO,
    "IO"
)

// Now see IO work in traces
suspend fun fetchData() = withContext(tracingIODispatcher) {
    // This shows as "IO.dispatch" in Perfetto!
    api.getData()
}
```

#### Repository Pattern Tracing

```kotlin
class UserRepository(
    private val api: UserApi,
    private val db: UserDao,
    private val cache: UserCache
) {
    suspend fun getUser(userId: String): User = trace("UserRepo.getUser") {
        // Check cache first
        trace("cache.check") {
            cache.get(userId)
        }?.let { return@trace it }

        // Try database
        val dbUser = trace("db.query") {
            db.getUserById(userId)
        }

        if (dbUser != null && !isStale(dbUser)) {
            return@trace dbUser
        }

        // Fetch from network
        val networkUser = trace("network.fetch") {
            api.getUser(userId)
        }

        // Update cache & db in parallel
        coroutineScope {
            launch { trace("cache.update") { cache.put(networkUser) } }
            launch { trace("db.update") { db.insert(networkUser) } }
        }

        networkUser
    }
}
```

### Compose Recomposition Tracing

Identify expensive recompositions:

```kotlin
@Composable
fun UserProfile(
    user: User,
    modifier: Modifier = Modifier
) = trace("Compose.UserProfile") {
    Column(modifier = modifier) {
        // Trace expensive composables
        TraceComposable("UserProfile.Avatar") {
            UserAvatar(user.avatarUrl)
        }

        TraceComposable("UserProfile.Details") {
            UserDetails(user)
        }

        TraceComposable("UserProfile.Actions") {
            UserActions(user.id)
        }
    }
}

@Composable
inline fun TraceComposable(
    name: String,
    content: @Composable () -> Unit
) {
    Trace.beginAsyncSection(name, name.hashCode())
    content()
    Trace.endAsyncSection(name, name.hashCode())
}
```

### Flow Tracing

Track Flow emissions and transformations:

```kotlin
fun <T> Flow<T>.traced(name: String): Flow<T> = flow {
    trace("Flow.$name") {
        collect { value ->
            trace("Flow.$name.emit") {
                emit(value)
            }
        }
    }
}

// Usage
fun observeUsers(): Flow<List<User>> =
    db.observeUsers()
        .traced("database")
        .map { users ->
            trace("map.filterActive") {
                users.filter { it.isActive }
            }
        }
        .traced("filtered")
        .map { users ->
            trace("map.sortByName") {
                users.sortedBy { it.name }
            }
        }
        .traced("sorted")
```

### Advanced: Async Trace Sections

For work that spans multiple frames or threads:

```kotlin
class ImageLoader {
    private val asyncSections = ConcurrentHashMap<String, Int>()

    suspend fun loadImage(url: String): Bitmap {
        val cookie = url.hashCode()

        Trace.beginAsyncSection("ImageLoad", cookie)
        try {
            val cached = trace("checkCache") {
                cache.get(url)
            }
            if (cached != null) {
                Trace.endAsyncSection("ImageLoad", cookie)
                return cached
            }

            val downloaded = trace("download") {
                downloadImage(url)
            }

            val decoded = trace("decode") {
                BitmapFactory.decodeByteArray(downloaded, 0, downloaded.size)
            }

            trace("cacheStore") {
                cache.put(url, decoded)
            }

            return decoded
        } finally {
            Trace.endAsyncSection("ImageLoad", cookie)
        }
    }
}
```

### Capturing and Analyzing Traces

#### Capture Trace

```kotlin
// Programmatic capture
suspend fun captureTrace(durationSeconds: Int): File {
    val traceFile = File(context.cacheDir, "trace_${System.currentTimeMillis()}.perfetto-trace")

    PerfettoTrace.record(
        traceFile,
        TracingConfig.Builder()
            .setBufferSizeKb(50_000)
            .build()
    ) {
        // Perform operations to trace
        delay(durationSeconds.seconds)
    }

    return traceFile
}
```

#### ADB Capture

```bash
# Capture 10 second trace
adb shell perfetto \
  -c - --txt \
  -o /data/misc/perfetto-traces/trace.perfetto-trace <<EOF
buffers: {
    size_kb: 20000
    fill_policy: DISCARD
}
data_sources: {
    config {
        name: "linux.ftrace"
        ftrace_config {
            ftrace_events: "sched/sched_switch"
            ftrace_events: "power/suspend_resume"
            ftrace_events: "sched/sched_wakeup"
        }
    }
}
duration_ms: 10000
EOF

# Pull trace
adb pull /data/misc/perfetto-traces/trace.perfetto-trace .
```

#### Open in Perfetto UI

Visit https://ui.perfetto.dev and load the trace file.

### Reading Perfetto Traces

#### Key Metrics to Look For

**Frame Rendering:**
- Composition time (target: <8ms for 120fps)
- Layout time (target: <5ms)
- Draw time (target: <3ms)

**Coroutine Dispatchers:**
- Thread pool saturation
- Context switch frequency
- Blocking operations on Main dispatcher

**Custom Traces:**
- Longest sections (optimization candidates)
- Unexpected sequential work (parallelize?)
- Redundant work (caching needed?)

#### Example Analysis

```
Timeline View:
|----Composition (15ms)---|
  |--UserProfile (12ms)--|
    |--Avatar (8ms)--|    ← Bottleneck!
    |--Details (3ms)-|
    |--Actions (1ms)-|

Diagnosis: Avatar loading is slow
Solution: Add image caching or load thumbnails
```

### Production Tracing

For production apps, use conditional tracing:

```kotlin
inline fun <T> traceInDebug(sectionName: String, block: () -> T): T {
    return if (BuildConfig.DEBUG) {
        trace(sectionName, block)
    } else {
        block()
    }
}

// Or trace only slow operations
inline fun <T> traceIfSlow(
    sectionName: String,
    thresholdMs: Long = 16,
    block: () -> T
): T {
    val start = System.nanoTime()
    val result = block()
    val durationMs = (System.nanoTime() - start) / 1_000_000

    if (durationMs > thresholdMs) {
        // Log slow operation
        Log.w("Performance", "$sectionName took ${durationMs}ms")

        // Could trigger trace capture
        if (durationMs > 100) {
            triggerSlowOperationTrace()
        }
    }

    return result
}
```

### Automated Performance Testing

Capture traces in tests:

```kotlin
@Test
fun testUserLoadPerformance() = runTest {
    val traceFile = File.createTempFile("test_trace", ".perfetto-trace")

    PerfettoTrace.record(traceFile) {
        val user = repository.getUser("test_user_id")
        assertThat(user).isNotNull()
    }

    // Analyze trace programmatically
    val trace = PerfettoTrace.parse(traceFile)
    val getUserDuration = trace.findSection("UserRepo.getUser")?.duration

    assertThat(getUserDuration).isLessThan(100.milliseconds)
}
```

### Integration with CI/CD

```kotlin
// Gradle task for performance regression testing
tasks.register("performanceTest") {
    doLast {
        val traces = runPerformanceTests()

        traces.forEach { (testName, traceFile) ->
            val analysis = analyzePerfettoTrace(traceFile)

            println("📊 $testName:")
            analysis.slowSections.forEach { section ->
                println("  ⚠️ ${section.name}: ${section.duration}ms")
            }

            if (analysis.hasRegressions()) {
                throw GradleException("Performance regression detected!")
            }
        }
    }
}
```

### Best Practices

1. **Trace Strategically**: Don't trace every function - focus on:
   - Repository operations
   - Network calls
   - Database queries
   - Compose recompositions
   - Image/data processing

2. **Use Hierarchical Naming**: `"UserRepo.getUser"`, `"UserRepo.getUser.cache"`

3. **Avoid Allocations in Trace Names**: Reuse strings

4. **Profile First, Then Trace**: Use profiler to find hot areas, then add traces

5. **Automate Trace Analysis**: CI/CD performance regression tests

### Conclusion

Perfetto tracing provides unprecedented visibility into Kotlin app performance:
- **Microsecond precision** vs profiler's 10ms sampling
- **Complete execution picture** including async work
- **Frame-accurate** compose recomposition tracking
- **Production-safe** with minimal overhead
- **Integrated** with Android's performance tools

Combined with strategic trace placement, it becomes an indispensable tool for building high-performance Kotlin applications.