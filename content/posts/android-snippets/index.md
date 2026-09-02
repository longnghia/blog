---
date: "2026-02-13T17:03:35+07:00"
draft: false
title: "Android Snippets: Koin, Crash Handler, Serialization"
summary: "Setup code I copy into every new Android project — Koin DI through the version catalog, an uncaught-exception crash logger, and kotlinx.serialization wiring."
categories:
  - Code
tags:
  - android
  - kotlin
  - koin
  - gradle
  - serialization
---

Three pieces of boilerplate I end up writing at the start of every Android project. Keeping them here so I stop digging through old repos for them.

## Dependency injection with Koin

Koin over Hilt, mostly because there's no annotation processor in the critical path and build times stay reasonable. Everything goes through the version catalog.

```toml
# libs.versions.toml

[versions]
koin = "4.1.0"
koin-annotations = "2.3.0"
koin-plugin = "0.3.0"

[libraries]
koin-androidx-compose = { module = "io.insert-koin:koin-androidx-compose", version.ref = "koin" }
koin-androidx-workmanager = { module = "io.insert-koin:koin-androidx-workmanager", version.ref = "koin" }
koin-core = { module = "io.insert-koin:koin-core", version.ref = "koin" }
koin-ktor = { module = "io.insert-koin:koin-ktor", version.ref = "koin" }
koin-logger-slf4j = { module = "io.insert-koin:koin-logger-slf4j", version.ref = "koin" }
koin-annotations = { module = "io.insert-koin:koin-annotations", version.ref = "koin-annotations" }

[plugins]
koin-compiler = { id = "io.insert-koin.compiler.plugin", version.ref = "koin-plugin" }
```

```kotlin
// build.gradle.kts

// Only needed if you use the annotation-based module declarations:
// plugins {
//     alias(libs.plugins.koin.compiler)
// }

dependencies {
    implementation(libs.koin.core)
    implementation(libs.koin.androidx.compose)
    implementation(libs.koin.androidx.workmanager)
    implementation(libs.koin.ktor)
    implementation(libs.koin.logger.slf4j)
    implementation(libs.koin.annotations)
}
```

Start Koin in `Application.onCreate`. `androidContext` is what makes `get<Context>()` work anywhere downstream, and it's the line people forget:

```kotlin
// App.kt

class App : Application() {
    override fun onCreate() {
        super.onCreate()
        startKoin {
            androidLogger()
            androidContext(this@App)
            modules(appModule)
            modules(networkModule)
        }
    }
}
```

Then the modules themselves. `viewModelOf` and `singleOf` take constructor references, so you don't have to spell out the dependency list — Koin resolves the parameters:

```kotlin
// AppModule.kt

val appModule = module {
    viewModelOf(::MainViewModel)
    singleOf(::Downloader)
    single { CustomJson }
}

val networkModule = module {
    single<HttpClient> {
        HttpClient(CIO) {
            engine {
                requestTimeout = 8000
            }
            install(ContentNegotiation) {
                json(CustomJson)
            }
            install(HttpTimeout)
        }
    }
}
```

## kotlinx.serialization

The plugin has to be applied in two places, which is the usual source of "serializer not found" errors.

```kotlin
// Project-level build.gradle.kts
plugins {
    kotlin("plugin.serialization") version "1.9.0" apply false
}

// Module-level build.gradle.kts
plugins {
    kotlin("android")
    id("org.jetbrains.kotlin.plugin.serialization")
}

dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.9.0")
}
```

### One shared `Json` instance

`Json { ... }` builds a new configuration every time you call it, and each one carries its own serializer cache. Define it once and inject it — that's the `CustomJson` referenced in `appModule` and in the Ktor `ContentNegotiation` block above, so both the HTTP client and anything doing manual parsing behave identically:

```kotlin
val CustomJson = Json {
    ignoreUnknownKeys = true    // don't crash when the API adds a field
    coerceInputValues = true    // null for a non-null field falls back to the default
    encodeDefaults = false      // keep request bodies small
    isLenient = true
}
```

`ignoreUnknownKeys` is the important one. Without it, a backend adding a single field to a response breaks your app in production.

## Crash handler

Compose's error screens don't help once the app is on someone else's device, and a full crash-reporting SDK is more than a side project needs. This writes the stack trace to a file you can pull later.

Adapted from [EhViewer's `CrashHandler`](https://github.com/FooIbar/EhViewer/blob/main/app/src/main/kotlin/com/hippo/ehviewer/util/CrashHandler.kt):

```kotlin
object CrashHandler {
    fun install() {
        val handler = Thread.getDefaultUncaughtExceptionHandler()
        Thread.setDefaultUncaughtExceptionHandler { t, e ->
            runCatching { saveCrashLog(e) }
            handler?.uncaughtException(t, e)
        }
    }

    private fun getThrowableInfo(t: Throwable, writer: PrintWriter) {
        t.printStackTrace(writer)
        var cause = t.cause
        while (cause != null) {
            cause.printStackTrace(writer)
            cause = cause.cause
        }
    }

    private fun saveCrashLog(e: Throwable) {
        val nowString = System.currentTimeMillis()
        val fileName = "crash-$nowString.log"
        val file = File(appContext.crashLogDir, fileName)

        runCatching {
            file.printWriter().use { writer ->
                writer.write("======== CrashInfo ========\n")
                getThrowableInfo(e, writer)
                writer.write("\n")
            }
        }.onFailure {
            log(it)
            file.delete()
        }
    }
}
```

Three details that make it work rather than make things worse:

- It **captures the previous handler and calls it** afterward. Replacing it outright means the process never dies properly and you get an ANR instead of a crash.
- Both the install and the write are wrapped in `runCatching`. An exception thrown from inside an uncaught-exception handler is not a good time.
- It **walks the cause chain** manually. `printStackTrace` on a wrapped exception often gives you the wrapper and elides the thing you actually need.

Call it first thing in `onCreate`, before Koin:

```kotlin
override fun onCreate() {
    super.onCreate()
    CrashHandler.install()
    startKoin { /* ... */ }
}
```

Crashes during DI setup are exactly the ones you can't reproduce, so the handler needs to already be in place.
