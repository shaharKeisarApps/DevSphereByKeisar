---
id: robolectric-unit-testing
title: Mastering Robolectric for True Unit Testing in Android and KMP
tldr: Robolectric enables genuine unit testing of Android code without emulators by simulating the Android framework. Combined with proper test architecture, it delivers fast, reliable tests for ViewModels, Repositories, and UI logic.
tags: [testing, robolectric, unit-testing, tdd, android, kmp]
difficulty: advanced
readTimeMinutes: 12
publishedDate: null
relatedTopics: [supervised-scope-build-plugins, collectasretainedstate-push-architecture]
---

Traditional Android testing has two frustrating options:
- **Instrumentation Tests**: Accurate but slow (run on emulator/device)
- **JVM Unit Tests**: Fast but crash on Android framework code

```kotlin
// This fails in JVM unit tests
@Test
fun testViewModel() {
    val viewModel = UserViewModel() // Crashes: Method getMainLooper not mocked!
}
```

**Robolectric** solves this by providing a simulated Android environment that runs on the JVM, giving you the speed of unit tests with the accuracy of instrumentation tests.

### Setting Up Robolectric

#### Gradle Configuration

```kotlin
// app/build.gradle.kts
android {
    testOptions {
        unitTests {
            isIncludeAndroidResources = true // Required for Robolectric
            isReturnDefaultValues = false
        }
    }
}

dependencies {
    testImplementation("junit:junit:4.13.2")
    testImplementation("org.robolectric:robolectric:4.11.1")
    testImplementation("androidx.test:core:1.5.0")
    testImplementation("androidx.test:core-ktx:1.5.0")
    testImplementation("androidx.test.ext:junit:1.1.5")
    testImplementation("androidx.test.ext:junit-ktx:1.1.5")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.8.0")
    testImplementation("app.cash.turbine:turbine:1.0.0")
}
```

#### Basic Test Structure

```kotlin
@RunWith(RobolectricTestRunner::class)
@Config(sdk = [33]) // Target SDK
class UserViewModelTest {

    @Test
    fun `user loads successfully`() {
        // Test runs on simulated Android environment
        val viewModel = UserViewModel(fakeRepository)

        // No crashes!
        assertThat(viewModel.state.value).isInstanceOf<UserState.Loading>()
    }
}
```

### Testing ViewModels with Robolectric

#### ViewModel with Android Dependencies

```kotlin
class UserViewModel(
    private val repository: UserRepository,
    savedStateHandle: SavedStateHandle
) : ViewModel() {

    private val userId = savedStateHandle.get<String>("userId") ?: ""

    private val _state = MutableStateFlow<UserState>(UserState.Loading)
    val state: StateFlow<UserState> = _state.asStateFlow()

    init {
        loadUser()
    }

    private fun loadUser() {
        viewModelScope.launch {
            _state.value = UserState.Loading

            repository.getUser(userId)
                .onSuccess { user ->
                    _state.value = UserState.Success(user)
                }
                .onFailure { error ->
                    _state.value = UserState.Error(error.message ?: "Unknown error")
                }
        }
    }
}
```

#### Comprehensive Test

```kotlin
@RunWith(RobolectricTestRunner::class)
@Config(sdk = [33])
class UserViewModelTest {

    private lateinit var repository: FakeUserRepository
    private lateinit var savedStateHandle: SavedStateHandle
    private lateinit var viewModel: UserViewModel

    // Use Turbine for Flow testing
    @Test
    fun `loading state emitted first`() = runTest {
        savedStateHandle = SavedStateHandle(mapOf("userId" to "123"))
        repository = FakeUserRepository()
        viewModel = UserViewModel(repository, savedStateHandle)

        viewModel.state.test {
            // First emission should be Loading
            assertThat(awaitItem()).isEqualTo(UserState.Loading)
            cancelAndIgnoreRemainingEvents()
        }
    }

    @Test
    fun `success state emitted when user loads`() = runTest {
        val testUser = User("123", "John Doe", "john@example.com")
        repository = FakeUserRepository(Result.success(testUser))
        savedStateHandle = SavedStateHandle(mapOf("userId" to "123"))

        viewModel = UserViewModel(repository, savedStateHandle)

        viewModel.state.test {
            assertThat(awaitItem()).isEqualTo(UserState.Loading)
            assertThat(awaitItem()).isEqualTo(UserState.Success(testUser))
        }
    }

    @Test
    fun `error state emitted when repository fails`() = runTest {
        val error = RuntimeException("Network error")
        repository = FakeUserRepository(Result.failure(error))
        savedStateHandle = SavedStateHandle(mapOf("userId" to "123"))

        viewModel = UserViewModel(repository, savedStateHandle)

        viewModel.state.test {
            assertThat(awaitItem()).isEqualTo(UserState.Loading)
            val errorState = awaitItem() as UserState.Error
            assertThat(errorState.message).isEqualTo("Network error")
        }
    }
}
```

### Testing Android Resources

Robolectric provides access to real resources:

```kotlin
@RunWith(RobolectricTestRunner::class)
class ResourceTest {

    private val context = ApplicationProvider.getApplicationContext<Context>()

    @Test
    fun `string resource is correct`() {
        val appName = context.getString(R.string.app_name)
        assertThat(appName).isEqualTo("My App")
    }

    @Test
    fun `color resource is correct`() {
        val primaryColor = ContextCompat.getColor(context, R.color.primary)
        assertThat(primaryColor).isEqualTo(0xFF6200EE.toInt())
    }

    @Test
    fun `drawable exists`() {
        val drawable = ContextCompat.getDrawable(context, R.drawable.ic_launcher)
        assertThat(drawable).isNotNull()
    }
}
```

### Testing Repositories with Room

```kotlin
@RunWith(RobolectricTestRunner::class)
@Config(sdk = [33])
class UserRepositoryTest {

    private lateinit var database: AppDatabase
    private lateinit var userDao: UserDao
    private lateinit var repository: UserRepository

    @Before
    fun setup() {
        // Create in-memory database
        val context = ApplicationProvider.getApplicationContext<Context>()
        database = Room.inMemoryDatabaseBuilder(context, AppDatabase::class.java)
            .allowMainThreadQueries() // OK for tests
            .build()

        userDao = database.userDao()
        repository = UserRepositoryImpl(userDao, fakeApi)
    }

    @After
    fun tearDown() {
        database.close()
    }

    @Test
    fun `repository returns cached user when available`() = runTest {
        // Given: User exists in database
        val cachedUser = UserEntity("123", "Cached User", "cached@example.com")
        userDao.insert(cachedUser)

        // When: Repository fetches user
        val result = repository.getUser("123")

        // Then: Cached user returned, no API call
        assertThat(result.getOrNull()).isEqualTo(cachedUser.toDomain())
        assertThat(fakeApi.callCount).isEqualTo(0)
    }

    @Test
    fun `repository fetches from API when cache miss`() = runTest {
        // Given: User not in database
        val apiUser = User("456", "API User", "api@example.com")
        fakeApi.users["456"] = apiUser

        // When: Repository fetches user
        val result = repository.getUser("456")

        // Then: User fetched from API and cached
        assertThat(result.getOrNull()).isEqualTo(apiUser)
        assertThat(fakeApi.callCount).isEqualTo(1)

        // Verify cached
        val cached = userDao.getUserById("456")
        assertThat(cached).isNotNull()
    }
}
```

### Testing WorkManager

```kotlin
@RunWith(RobolectricTestRunner::class)
@Config(sdk = [33])
class SyncWorkerTest {

    private lateinit var context: Context
    private lateinit var workManager: WorkManager

    @Before
    fun setup() {
        context = ApplicationProvider.getApplicationContext()

        // Initialize WorkManager for tests
        val config = Configuration.Builder()
            .setMinimumLoggingLevel(Log.DEBUG)
            .setExecutor(SynchronousExecutor())
            .build()

        WorkManagerTestInitHelper.initializeTestWorkManager(context, config)
        workManager = WorkManager.getInstance(context)
    }

    @Test
    fun `sync worker runs successfully`() = runTest {
        // Create test worker
        val request = OneTimeWorkRequestBuilder<SyncWorker>()
            .setInputData(workDataOf("userId" to "123"))
            .build()

        // Enqueue and wait
        workManager.enqueue(request).result.get()

        val workInfo = workManager.getWorkInfoById(request.id).get()

        assertThat(workInfo.state).isEqualTo(WorkInfo.State.SUCCEEDED)
    }

    @Test
    fun `sync worker retries on failure`() = runTest {
        // Configure repository to fail
        fakeRepository.shouldFail = true

        val request = OneTimeWorkRequestBuilder<SyncWorker>()
            .setBackoffCriteria(
                BackoffPolicy.LINEAR,
                10,
                TimeUnit.SECONDS
            )
            .build()

        workManager.enqueue(request).result.get()

        val workInfo = workManager.getWorkInfoById(request.id).get()

        assertThat(workInfo.state).isEqualTo(WorkInfo.State.ENQUEUED) // Retry scheduled
        assertThat(workInfo.runAttemptCount).isGreaterThan(0)
    }
}
```

### Testing SharedPreferences

```kotlin
@RunWith(RobolectricTestRunner::class)
class PreferencesManagerTest {

    private lateinit var context: Context
    private lateinit var prefsManager: PreferencesManager

    @Before
    fun setup() {
        context = ApplicationProvider.getApplicationContext()
        prefsManager = PreferencesManager(context)
    }

    @Test
    fun `user preferences are persisted`() {
        prefsManager.setTheme(Theme.DARK)
        prefsManager.setNotificationsEnabled(true)

        assertThat(prefsManager.getTheme()).isEqualTo(Theme.DARK)
        assertThat(prefsManager.areNotificationsEnabled()).isTrue()
    }

    @Test
    fun `preferences survive recreation`() {
        prefsManager.setUsername("testuser")

        // Create new instance (simulates app restart)
        val newManager = PreferencesManager(context)

        assertThat(newManager.getUsername()).isEqualTo("testuser")
    }
}
```

### Testing Broadcast Receivers

```kotlin
@RunWith(RobolectricTestRunner::class)
class NetworkReceiverTest {

    private lateinit var context: Context
    private lateinit var receiver: NetworkChangeReceiver

    @Before
    fun setup() {
        context = ApplicationProvider.getApplicationContext()
        receiver = NetworkChangeReceiver()
    }

    @Test
    fun `receiver handles network connected`() {
        val intent = Intent(ConnectivityManager.CONNECTIVITY_ACTION).apply {
            putExtra(ConnectivityManager.EXTRA_NETWORK_TYPE, ConnectivityManager.TYPE_WIFI)
        }

        receiver.onReceive(context, intent)

        // Verify receiver triggered sync
        verify(syncManager).startSync()
    }

    @Test
    fun `receiver handles network disconnected`() {
        val intent = Intent(ConnectivityManager.CONNECTIVITY_ACTION).apply {
            putExtra(ConnectivityManager.EXTRA_NO_CONNECTIVITY, true)
        }

        receiver.onReceive(context, intent)

        // Verify receiver cancelled sync
        verify(syncManager).cancelSync()
    }
}
```

### Advanced: Testing Activities and Fragments

```kotlin
@RunWith(RobolectricTestRunner::class)
@Config(sdk = [33])
class MainActivityTest {

    @Test
    fun `activity launches successfully`() {
        val scenario = ActivityScenario.launch(MainActivity::class.java)

        scenario.onActivity { activity ->
            assertThat(activity.isFinishing).isFalse()
        }
    }

    @Test
    fun `clicking button navigates to detail`() {
        val scenario = ActivityScenario.launch(MainActivity::class.java)

        scenario.onActivity { activity ->
            val button = activity.findViewById<Button>(R.id.detailButton)
            button.performClick()

            // Verify navigation occurred
            val navController = activity.findNavController(R.id.nav_host)
            assertThat(navController.currentDestination?.id)
                .isEqualTo(R.id.detailFragment)
        }
    }
}
```

### Testing Compose UI (with Robolectric)

```kotlin
@RunWith(RobolectricTestRunner::class)
@Config(sdk = [33])
class UserProfileScreenTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun `user profile displays correctly`() {
        val user = User("123", "John Doe", "john@example.com")

        composeTestRule.setContent {
            UserProfileScreen(user = user)
        }

        composeTestRule.onNodeWithText("John Doe").assertIsDisplayed()
        composeTestRule.onNodeWithText("john@example.com").assertIsDisplayed()
    }

    @Test
    fun `clicking edit button triggers callback`() {
        var editClicked = false
        val user = User("123", "John Doe", "john@example.com")

        composeTestRule.setContent {
            UserProfileScreen(
                user = user,
                onEditClick = { editClicked = true }
            )
        }

        composeTestRule.onNodeWithContentDescription("Edit").performClick()

        assertThat(editClicked).isTrue()
    }
}
```

### Performance and Best Practices

#### Parallel Test Execution

```kotlin
// gradle.properties
org.gradle.parallel=true
org.gradle.workers.max=4

// build.gradle.kts
tasks.withType<Test> {
    maxParallelForks = Runtime.getRuntime().availableProcessors()

    testLogging {
        events("passed", "skipped", "failed")
        showStandardStreams = false
    }
}
```

#### Test Configuration

```kotlin
@RunWith(RobolectricTestRunner::class)
@Config(
    sdk = [33],
    application = TestApplication::class, // Custom test application
    qualifiers = "en-rUS-w360dp-h640dp-xhdpi" // Specific device config
)
class CustomConfigTest {
    // Tests with specific configuration
}
```

#### Shared Test Setup

```kotlin
open class BaseRobolectricTest {

    @get:Rule
    val coroutineRule = MainCoroutineRule()

    protected lateinit var context: Context

    @Before
    fun baseSetup() {
        context = ApplicationProvider.getApplicationContext()
    }

    protected fun runTest(
        testBody: suspend TestScope.() -> Unit
    ) = kotlinx.coroutines.test.runTest {
        testBody()
    }
}

// Usage
class UserViewModelTest : BaseRobolectricTest() {
    @Test
    fun `test with shared setup`() = runTest {
        // Test logic
    }
}
```

### Integration with KMP

For KMP projects, use Robolectric in Android-specific tests:

```kotlin
// shared/src/androidUnitTest/kotlin/
@RunWith(RobolectricTestRunner::class)
class AndroidSpecificRepositoryTest {

    @Test
    fun `repository uses Android context correctly`() {
        val context = ApplicationProvider.getApplicationContext<Context>()
        val repository = AndroidUserRepository(context)

        // Test Android-specific behavior
    }
}
```

### CI/CD Integration

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Run Robolectric tests
        run: ./gradlew testDebugUnitTest --continue

      - name: Publish test results
        uses: EnricoMi/publish-unit-test-result-action@v2
        if: always()
        with:
          files: '**/build/test-results/**/*.xml'
```

### Conclusion

Robolectric transforms Android testing by providing:
- **JVM speed** without emulator overhead
- **Real Android framework** behavior
- **Hermetic tests** that don't require devices
- **Fast CI/CD** pipelines
- **True unit testing** for Android code

Combined with proper test architecture (fakes, test doubles, coroutine testing), it enables TDD and comprehensive test coverage for Android and KMP projects.