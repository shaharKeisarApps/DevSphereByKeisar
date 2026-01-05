---
id: kmp-resources-all-targets
title: Mastering Resources in Kotlin Multiplatform - A Comprehensive Guide for All Targets
tldr: Compose Resources provides type-safe, multiplatform access to strings, images, fonts, and files across JVM, Android, iOS, Web, and Desktop. Proper architecture with expect/actual patterns and platform-specific configurations ensures seamless resource handling.
tags: [kmp, multiplatform, resources, compose-resources, ios, web, desktop]
difficulty: advanced
readTimeMinutes: 15
publishedDate: null
relatedTopics: [supervised-scope-build-plugins, collectasretainedstate-push-architecture]
---

## The Resource Challenge in KMP

Each platform has different resource systems:
- **Android**: `R.string.app_name`, `R.drawable.ic_launcher`
- **iOS**: `NSLocalizedString`, `UIImage(named:)`
- **Web**: `<img src="assets/logo.png">`, `fetch('/data.json')`
- **Desktop**: `ImageIO.read()`, `ResourceBundle`

Kotlin Multiplatform needs a unified approach that:
- Works across all targets
- Provides type safety
- Supports localization
- Handles images, fonts, strings, and files

## Compose Resources: The Modern Solution

Compose Resources (from Jetpack Compose Multiplatform) provides a unified, type-safe API for all resources.

### Setup

```kotlin
// gradle/libs.versions.toml
[versions]
compose = "1.6.0"

[libraries]
compose-resources = { module = "org.jetbrains.compose.components:components-resources", version.ref = "compose" }

[plugins]
jetbrainsCompose = { id = "org.jetbrains.compose", version.ref = "compose" }
```

```kotlin
// build.gradle.kts
plugins {
    kotlin("multiplatform")
    id("org.jetbrains.compose")
}

kotlin {
    sourceSets {
        val commonMain by getting {
            dependencies {
                implementation(compose.runtime)
                implementation(compose.foundation)
                implementation(compose.components.resources)
            }
        }
    }
}
```

### Resource Directory Structure

```
commonMain/
├── composeResources/
│   ├── drawable/
│   │   ├── ic_launcher.png
│   │   ├── ic_launcher.xml (Android vector)
│   │   └── logo.svg
│   ├── drawable-hdpi/
│   │   └── ic_launcher.png
│   ├── drawable-xhdpi/
│   │   └── ic_launcher.png
│   ├── font/
│   │   ├── roboto_regular.ttf
│   │   └── roboto_bold.ttf
│   ├── files/
│   │   ├── config.json
│   │   └── data.csv
│   └── values/
│       ├── strings.xml
│       └── strings_es.xml (Spanish localization)
```

## Working with Strings

### Define Strings

```xml
<!-- commonMain/composeResources/values/strings.xml -->
<resources>
    <string name="app_name">My App</string>
    <string name="welcome_message">Welcome, %s!</string>
    <string name="item_count">You have %d items</string>
    <string name="greeting">Hello</string>
</resources>

<!-- commonMain/composeResources/values/strings_es.xml -->
<resources>
    <string name="app_name">Mi Aplicación</string>
    <string name="welcome_message">¡Bienvenido, %s!</string>
    <string name="item_count">Tienes %d elementos</string>
    <string name="greeting">Hola</string>
</resources>
```

### Access Strings in Code

```kotlin
import devspherebykeisar.composeapp.generated.resources.Res
import devspherebykeisar.composeapp.generated.resources.*
import org.jetbrains.compose.resources.stringResource

@Composable
fun WelcomeScreen(username: String) {
    Column {
        // Simple string
        Text(stringResource(Res.string.greeting))

        // Formatted string
        Text(stringResource(Res.string.welcome_message, username))

        // Plurals (requires additional setup)
        Text(stringResource(Res.string.item_count, 5))
    }
}

// Non-Composable usage
suspend fun getAppName(): String {
    return Res.string.app_name.getString()
}
```

## Working with Images

### Drawable Resources

```kotlin
import devspherebykeisar.composeapp.generated.resources.Res
import devspherebykeisar.composeapp.generated.resources.ic_launcher
import devspherebykeisar.composeapp.generated.resources.logo
import org.jetbrains.compose.resources.painterResource

@Composable
fun AppLogo() {
    Image(
        painter = painterResource(Res.drawable.logo),
        contentDescription = "App Logo",
        modifier = Modifier.size(120.dp)
    )
}

// Vector drawable (Android XML)
@Composable
fun VectorIcon() {
    Icon(
        painter = painterResource(Res.drawable.ic_launcher),
        contentDescription = "App Icon"
    )
}
```

### Platform-Specific Image Handling

```kotlin
// Automatically selects correct density
@Composable
fun AdaptiveImage() {
    // On Android: uses hdpi/xhdpi/xxhdpi versions automatically
    // On iOS: uses @2x/@3x versions
    // On Desktop/Web: uses base version
    Image(
        painter = painterResource(Res.drawable.ic_launcher),
        contentDescription = "Launcher Icon"
    )
}
```

## Working with Fonts

### Font Resources

```kotlin
import devspherebykeisar.composeapp.generated.resources.roboto_bold
import devspherebykeisar.composeapp.generated.resources.roboto_regular
import org.jetbrains.compose.resources.Font

@Composable
fun CustomFontText() {
    val robotoBold = FontFamily(Font(Res.font.roboto_bold))
    val robotoRegular = FontFamily(Font(Res.font.roboto_regular))

    Column {
        Text(
            text = "Bold Text",
            fontFamily = robotoBold
        )

        Text(
            text = "Regular Text",
            fontFamily = robotoRegular
        )
    }
}

// Create font family for typography
val AppTypography = Typography(
    displayLarge = TextStyle(
        fontFamily = FontFamily(Font(Res.font.roboto_bold)),
        fontSize = 57.sp
    ),
    bodyLarge = TextStyle(
        fontFamily = FontFamily(Font(Res.font.roboto_regular)),
        fontSize = 16.sp
    )
)
```

## Working with Files

### Raw Resource Files

```kotlin
import devspherebykeisar.composeapp.generated.resources.Res
import org.jetbrains.compose.resources.ExperimentalResourceApi

@OptIn(ExperimentalResourceApi::class)
suspend fun loadConfig(): String {
    return Res.readBytes("files/config.json").decodeToString()
}

@OptIn(ExperimentalResourceApi::class)
suspend fun loadCsvData(): List<String> {
    val csv = Res.readBytes("files/data.csv").decodeToString()
    return csv.lines()
}

// In a repository
class ConfigRepository {
    @OptIn(ExperimentalResourceApi::class)
    suspend fun getConfig(): Config {
        val jsonString = Res.readBytes("files/config.json").decodeToString()
        return Json.decodeFromString<Config>(jsonString)
    }
}
```

## Platform-Specific Resources with expect/actual

For truly platform-specific resources:

```kotlin
// commonMain
expect class PlatformResources {
    fun getSystemLanguage(): String
    fun getAssetPath(filename: String): String
}

// androidMain
actual class PlatformResources {
    actual fun getSystemLanguage(): String {
        return Locale.getDefault().language
    }

    actual fun getAssetPath(filename: String): String {
        return "file:///android_asset/$filename"
    }
}

// iosMain
actual class PlatformResources {
    actual fun getSystemLanguage(): String {
        return NSLocale.currentLocale.languageCode
    }

    actual fun getAssetPath(filename: String): String {
        return NSBundle.mainBundle.pathForResource(
            filename.substringBeforeLast('.'),
            filename.substringAfterLast('.')
        ) ?: ""
    }
}

// wasmJsMain
actual class PlatformResources {
    actual fun getSystemLanguage(): String {
        return js("navigator.language").toString()
    }

    actual fun getAssetPath(filename: String): String {
        return "/assets/$filename"
    }
}

// desktopMain
actual class PlatformResources {
    actual fun getSystemLanguage(): String {
        return System.getProperty("user.language") ?: "en"
    }

    actual fun getAssetPath(filename: String): String {
        return javaClass.classLoader.getResource(filename)?.path ?: ""
    }
}
```

## Advanced: Dynamic Resource Loading

```kotlin
class ResourceManager {
    private val cache = mutableMapOf<String, Any>()

    @OptIn(ExperimentalResourceApi::class)
    suspend fun <T> loadResource(
        path: String,
        parser: (ByteArray) -> T
    ): T {
        return cache.getOrPut(path) {
            val bytes = Res.readBytes(path)
            parser(bytes)
        } as T
    }

    suspend fun loadJsonConfig(path: String): JsonObject {
        return loadResource(path) { bytes ->
            Json.parseToJsonElement(bytes.decodeToString()).jsonObject
        }
    }

    suspend fun loadImage(path: String): ImageBitmap {
        return loadResource(path) { bytes ->
            bytes.toImageBitmap()
        }
    }
}
```

## Localization Best Practices

### Plurals Support

```xml
<!-- values/strings.xml -->
<resources>
    <plurals name="items_count">
        <item quantity="zero">No items</item>
        <item quantity="one">One item</item>
        <item quantity="other">%d items</item>
    </plurals>
</resources>
```

```kotlin
@Composable
fun ItemCounter(count: Int) {
    Text(
        text = pluralStringResource(Res.plurals.items_count, count, count)
    )
}
```

### Locale Selection

```kotlin
@Composable
fun LocalizedApp() {
    // Get system locale automatically
    val locale = LocalLocale.current

    MaterialTheme {
        // Resources automatically use correct locale
        AppContent()
    }
}

// Manual locale override
@Composable
fun LocalizedAppWithOverride(localeCode: String) {
    val locale = Locale(localeCode)

    CompositionLocalProvider(LocalLocale provides locale) {
        MaterialTheme {
            AppContent()
        }
    }
}
```

## iOS-Specific Considerations

### Asset Catalog Integration

```kotlin
// iosMain - Bridge to iOS asset catalog
actual class PlatformResources {
    fun loadUIImage(named: String): UIImage? {
        return UIImage.imageNamed(named)
    }
}

// Usage in commonMain through expect/actual
expect fun loadPlatformImage(name: String): ImageBitmap

// iosMain
actual fun loadPlatformImage(name: String): ImageBitmap {
    val uiImage = UIImage.imageNamed(name)
    return uiImage?.toImageBitmap() ?: error("Image not found")
}
```

### iOS Bundle Resources

```kotlin
// iosMain
@OptIn(ExperimentalResourceApi::class)
fun loadFromBundle(filename: String): ByteArray {
    val bundle = NSBundle.mainBundle
    val path = bundle.pathForResource(
        filename.substringBeforeLast('.'),
        filename.substringAfterLast('.')
    ) ?: error("Resource not found: $filename")

    val data = NSData.dataWithContentsOfFile(path)
        ?: error("Failed to load: $filename")

    return data.toByteArray()
}
```

## Web/Wasm-Specific Handling

### Asset Loading for Web

```kotlin
// wasmJsMain
@OptIn(ExperimentalResourceApi::class)
suspend fun loadWebAsset(path: String): ByteArray {
    return window.fetch(path)
        .await()
        .arrayBuffer()
        .await()
        .toByteArray()
}

// Public assets in web deployment
// Place in: composeApp/src/wasmJsMain/resources/
// Accessed at: /path/to/file
```

### Webpack Configuration for Resources

```kotlin
// build.gradle.kts
kotlin {
    wasmJs {
        browser {
            commonWebpackConfig {
                // Configure webpack to handle resources
                outputFileName = "app.js"

                // Copy resources to output directory
                devServer = (devServer ?: KotlinWebpackConfig.DevServer()).apply {
                    static = (static ?: mutableListOf()).apply {
                        add(project.projectDir.path)
                        add(project.projectDir.path + "/composeResources")
                    }
                }
            }
        }
    }
}
```

## Desktop-Specific Resources

### JAR Resource Packaging

```kotlin
// desktopMain
@OptIn(ExperimentalResourceApi::class)
fun loadDesktopResource(path: String): ByteArray {
    return object {}.javaClass.classLoader
        .getResourceAsStream(path)
        ?.readBytes()
        ?: error("Resource not found: $path")
}

// Resources are packaged in JAR and accessed via classloader
```

### Native File System Access

```kotlin
// desktopMain - User documents directory
fun getUserDocumentsPath(): String {
    return System.getProperty("user.home") + "/Documents/MyApp"
}

// Load from file system
suspend fun loadFromFileSystem(filename: String): ByteArray {
    val file = File(getUserDocumentsPath(), filename)
    return file.readBytes()
}
```

## Testing Resources

```kotlin
@Test
fun testStringResources() = runTest {
    // Load string resource in test
    val appName = Res.string.app_name.getString()
    assertThat(appName).isEqualTo("My App")
}

@Test
fun testFileResources() = runTest {
    @OptIn(ExperimentalResourceApi::class)
    val config = Res.readBytes("files/config.json").decodeToString()
    val jsonObject = Json.parseToJsonElement(config).jsonObject

    assertThat(jsonObject["version"]?.jsonPrimitive?.content).isEqualTo("1.0.0")
}
```

## Performance Optimization

### Resource Caching

```kotlin
object ResourceCache {
    private val stringCache = ConcurrentHashMap<String, String>()
    private val imageCache = ConcurrentHashMap<String, ImageBitmap>()

    @OptIn(ExperimentalResourceApi::class)
    suspend fun getCachedString(key: String, loader: suspend () -> String): String {
        return stringCache.getOrPut(key) { loader() }
    }

    suspend fun getCachedImage(key: String, loader: suspend () -> ImageBitmap): ImageBitmap {
        return imageCache.getOrPut(key) { loader() }
    }
}
```

### Lazy Loading

```kotlin
@Composable
fun LazyLoadedImage(resourceId: DrawableResource) {
    var bitmap by remember { mutableStateOf<ImageBitmap?>(null) }

    LaunchedEffect(resourceId) {
        bitmap = loadImageBitmap(resourceId)
    }

    bitmap?.let { bmp ->
        Image(bitmap = bmp, contentDescription = null)
    } ?: CircularProgressIndicator()
}
```

## Conclusion

Proper resource handling in KMP requires:
- **Compose Resources** for common resources across targets
- **expect/actual** for platform-specific resources
- **Proper configuration** for each target (iOS bundle, Web webpack, Desktop JAR)
- **Caching strategies** for performance
- **Localization support** for multi-language apps

This comprehensive approach enables seamless resource management across all Kotlin Multiplatform targets while maintaining type safety and developer productivity.