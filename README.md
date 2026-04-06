# Beltone Trade — Android AAR

## How to use

### 1. Add this repo as a Maven repository

In your `settings.gradle.kts`:

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()

        // Beltone Trade AAR
        maven {
            url = uri("https://raw.githubusercontent.com/amratef503092/aar-orange-sdk/main")
        }
    }
}
```

### 2. Add the dependency

In your `app/build.gradle.kts`:

```kotlin
dependencies {
    releaseImplementation("com.beltonefinancial.beltonetrade:flutter_release:1.0")
}
```

### 3. Sync Gradle

Android Studio → File → Sync Project with Gradle Files

Done!
