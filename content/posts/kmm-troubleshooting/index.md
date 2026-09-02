---
date: "2026-05-03T20:08:21+07:00"
draft: false
title: "KMP Troubleshooting: iOS File IO, Room, and IrLinkageError"
summary: "Reading and writing files via NSString from Kotlin, finding the simulator's app container, and fixing the Koin/lifecycle IrLinkageError."
categories:
  - Code
tags:
  - kmp
  - kotlin
  - ios
  - koin
  - room
  - troubleshooting
---

A running list of the things that cost me time on the iOS side of a Kotlin Multiplatform project.

## File IO from `iosMain`

There's no `java.io.File` in `iosMain`, so file access goes through `NSString`. Both directions are one-liners, but the signatures are easy to get wrong.

**Read a file:**

```kotlin
val content = NSString.create(
    contentsOfFile = path,
    encoding = NSUTF8StringEncoding,
    error = null,
) as? String ?: return
```

The `as? String` matters — Kotlin/Native bridges `NSString` to `String` on the way out, but `create` returns a nullable `Any?`, so you need the cast to get anything usable.

**Write a file:**

```kotlin
NSString.create(content)?.writeToFile(path, true, NSUTF8StringEncoding, null)
```

The second argument is `atomically`. Leave it `true` unless you have a reason not to — it writes to a temp file and renames, so a crash mid-write can't leave you with a truncated file.

**Converting `NSString` back to `String`:**

Sometimes the implicit bridge doesn't kick in and you're stuck holding an `NSString`. This works:

```kotlin
fun NSString.toKotlinString(): String {
    // Warning: awful hack
    return this.stringByAppendingString("")
}
```

Appending an empty string forces the value through the bridge. It is genuinely a hack, but it's the shortest thing that works.

For a real-world reference, [`LogSharer.ios.kt` in music-assistant/mobile-app](https://github.com/music-assistant/mobile-app/blob/6f02b655718100723d243df544fb727bcf862dc2/composeApp/src/iosMain/kotlin/io/music_assistant/client/logging/LogSharer.ios.kt#L36) does file sharing this way.

## Finding files on the simulator

When you've written a file from iOS and want to actually look at it, you need the app's data container. First get the booted simulator's UDID:

```sh
xcrun simctl list devices booted
```

Then ask for the container path:

```sh
xcrun simctl get_app_container <SIMULATOR_UDID> <BUNDLE_ID> data
```

For example:

```sh
xcrun simctl get_app_container 43F06E17-DEAA-4623-832E-EFB5BE8FAE52 com.paulcoding.lnviewer.LNViewer data
```

That prints an absolute path you can `cd` into or open in Finder. The `data` argument is the one you want — `app` gives you the read-only bundle instead.

## Room on iOS

Room supports KMP now, but the database builder is `expect`/`actual` — you have to supply the iOS path yourself.

- [Official Room KMP docs](https://developer.android.com/kotlin/multiplatform/room)
- [`getDatabaseBuilder.ios.kt` in kmp-mvvm-template](https://github.com/hasancbngl/kmp-mvvm-template/blob/90f3f06b910e787cdc39dfc0777e06b9f28ff15b/composeApp/src/iosMain/kotlin/data/local/getDatabaseBuilder.ios.kt#L5) — a short, complete iOS actual implementation

## `IrLinkageError` from Koin

This one is not your code's fault, and the error message does a good job of hiding that:

```text
Uncaught Kotlin exception: kotlin.internal.IrLinkageError: Can not read value from
backing field of property 'androidx_lifecycle_viewmodel_compose_LocalViewModelStoreOwner$stable':
Private backing field of property declared in module
<org.jetbrains.androidx.lifecycle:lifecycle-viewmodel-compose>
can not be accessed in module <io.insert-koin:koin-compose-viewmodel>
```

It crashes at runtime, not at compile time, which is what makes it confusing.

**What's happening:** `koin-compose-viewmodel` 4.0.0 was compiled against an older `lifecycle-viewmodel-compose` (below 2.10.0). In the newer lifecycle artifact, `LocalViewModelStoreOwner`'s `$stable` backing field became private. Koin's compiled-in reference to it no longer resolves, and Kotlin/Native reports that as an IR linkage failure the first time the code path runs.

**The fix:** update Koin. 4.2.1 is built against Kotlin 2.3.20 and the current lifecycle APIs.

```toml
# libs.versions.toml
[versions]
koin = "4.2.1"
```

Downgrading `lifecycle-viewmodel-compose` back below 2.10.0 also stops the crash, but it drags the rest of your Compose stack backwards with it. Moving Koin forward is the better direction.

> Any `IrLinkageError` mentioning a "private backing field" in *someone else's* module means two dependencies were compiled against incompatible versions of a third. Look at the two modules named in the message and find a pair of versions that agree — don't go looking in your own source.
