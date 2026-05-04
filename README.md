# Beltone Trade — Android AAR SDK

Beltone Trade Flutter module distributed as an Android AAR for native integration.

## Requirements

| Requirement | Version |
|-------------|---------|
| `minSdk` | 24 |
| `compileSdk` | 35 |
| `Java` | 17 |
| `AGP` | 8.x |

## Installation

### 1. Add Maven repositories

In your **`settings.gradle.kts`**:

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()

        // Flutter engine artifacts
        val storageUrl = System.getenv("FLUTTER_STORAGE_BASE_URL") ?: "https://storage.googleapis.com"
        maven { url = uri("$storageUrl/download.flutter.io") }

        // Beltone Trade AAR
        maven { url = uri("https://raw.githubusercontent.com/amratef503092/aar-orange-sdk/main") }
    }
}
```

### 2. Add dependency

In your **`app/build.gradle.kts`**:

```kotlin
configurations.all {
    resolutionStrategy.cacheChangingModulesFor(0, "seconds")
}

dependencies {
    implementation("com.beltonefinancial.beltonetrade:flutter_release:1.0") {
        isChanging = true
    }
}
```

> **Note:** `isChanging = true` ensures Gradle always fetches the latest AAR. Without it, Gradle caches version `1.0` permanently.

### 3. Add NDK ABI filters

In your `defaultConfig`:

```kotlin
android {
    defaultConfig {
        ndk {
            abiFilters += listOf("armeabi-v7a", "arm64-v8a", "x86_64")
        }
    }
}
```

---

## Integration

### Step 1: Start FlutterEngine in Application

Create a custom `Application` class that starts the Flutter engine on app launch:

```kotlin
import android.app.Application
import android.util.Log
import io.flutter.embedding.engine.FlutterEngine
import io.flutter.embedding.engine.FlutterEngineCache
import io.flutter.embedding.engine.dart.DartExecutor
import io.flutter.embedding.engine.loader.FlutterLoader
import io.flutter.plugin.common.MethodChannel

class MyApp : Application() {
    companion object {
        const val ENGINE_ID = "beltone_engine"
        const val HOST_CHANNEL = "com.beltonefinancial.beltonetrade/host"
    }

    lateinit var flutterEngine: FlutterEngine

    override fun onCreate() {
        super.onCreate()

        // Initialize Flutter
        val flutterLoader = FlutterLoader()
        flutterLoader.startInitialization(this)
        flutterLoader.ensureInitializationComplete(this, null)

        // Create and start the engine
        flutterEngine = FlutterEngine(this)

        // Setup host channel (see Step 2)
        setupHostChannel()

        // Run Dart entrypoint
        flutterEngine.dartExecutor.executeDartEntrypoint(
            DartExecutor.DartEntrypoint.createDefault()
        )

        // Cache the engine for later use
        FlutterEngineCache.getInstance().put(ENGINE_ID, flutterEngine)
    }
}
```

> Register your Application class in `AndroidManifest.xml`:
> ```xml
> <application android:name=".MyApp" ... >
> ```

### Step 2: Setup Host Channel & Push Credentials

The Flutter module expects credentials via a **MethodChannel**. You must:

1. **Handle `moduleReady`** — Flutter notifies when it's initialized
2. **Handle `getOrangeCredentials`** — Flutter pulls credentials from native
3. **Push `getCredentials`** ��� Native pushes credentials to Flutter (triggers login)

```kotlin
private fun setupHostChannel() {
    MethodChannel(flutterEngine.dartExecutor.binaryMessenger, HOST_CHANNEL)
        .setMethodCallHandler { call, result ->
            when (call.method) {
                // Flutter is ready
                "moduleReady" -> {
                    Log.d("MyApp", "Flutter module ready")
                    result.success(null)
                }

                // Flutter pulls credentials
                "getOrangeCredentials" -> {
                    if (currentToken.isNotEmpty() && currentMobile.isNotEmpty()) {
                        result.success(mapOf(
                            "token" to currentToken,
                            "mobile" to currentMobile
                        ))
                    } else {
                        result.success(null)
                    }
                }

                else -> result.notImplemented()
            }
        }
}
```

### Step 3: Push Credentials & Open Flutter

Before launching the Flutter screen, push the user's **token** and **mobile** number:

```kotlin
private var currentToken = ""
private var currentMobile = ""

fun openBeltoneTrade(token: String, mobile: String) {
    currentToken = token
    currentMobile = mobile

    // Push credentials to Flutter (triggers auto-login)
    MethodChannel(flutterEngine.dartExecutor.binaryMessenger, HOST_CHANNEL)
        .invokeMethod("getCredentials", mapOf(
            "token" to token,
            "mobile" to mobile
        ))

    // Open Flutter screen
    startActivity(
        FlutterActivity
            .withCachedEngine(ENGINE_ID)
            .build(context)
    )
}
```

### Credential Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `token` | `String` | Orange authentication token (UUID format) |
| `mobile` | `String` | User's mobile number (e.g. `"01008137330"`) |

---

## Sequence Diagram

```
Native App                        Flutter Module
    │                                   │
    │── App Launch ──────────────────►  │ Engine starts, SplashScreen loads
    │                                   │ (waits for credentials)
    │                                   │
    │   User enters token + mobile      │
    │                                   │
    │── invokeMethod("getCredentials")─►│ Stores token + mobile
    │   {token, mobile}                 │ Navigates to SplashScreen
    │                                   │
    │── startActivity(FlutterActivity)─►│ Flutter UI becomes visible
    │                                   │ SplashScreen starts login
    │                                   │ with pushed credentials
    │                                   │
    │◄── "moduleReady" ────────────────│ (optional) Flutter is ready
    │                                   │
    │◄── "onLoginResult" ─────────────│ Login success/failure
    │   {status, token?, error?}        │
    │                                   │
```

---

## Reference Implementation

See the full working example at [android-host](https://github.com/amratef503092/android-host).
