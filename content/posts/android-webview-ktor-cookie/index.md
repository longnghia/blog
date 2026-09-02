---
date: "2026-05-06T00:08:15+07:00"
draft: false
title: "Sharing Cookies Between WebView and Ktor"
summary: "A CookiesStorage backed by Android's CookieManager, so a WebView login session carries over to Ktor requests — plus the iOS/KMP equivalent."
categories:
  - Code
tags:
  - android
  - ktor
  - webview
  - cookies
  - kmp
  - compose
---

{{< gpt >}}

A common shape for apps that wrap an existing web product: the user logs in through a `WebView`, because the login flow is already built and may involve OAuth redirects or a captcha you don't want to reimplement. After that, the app wants to call the JSON API directly with Ktor.

The problem is that these are two separate cookie jars. The `WebView` gets the session cookie from `Set-Cookie` and stores it in Android's `CookieManager`. Ktor keeps its own in-memory `AcceptAllCookiesStorage` and knows nothing about it. So the login succeeds and the very next API call comes back `401`.

The fix is to stop keeping two jars. `CookiesStorage` is an interface — implement it against `CookieManager` and Ktor reads and writes the same store the `WebView` uses.

## Setup

```kotlin
// build.gradle.kts
dependencies {
    implementation("io.ktor:ktor-client-android:2.3.x")
    implementation("io.ktor:ktor-client-core:2.3.x")
    implementation("io.ktor:ktor-client-content-negotiation:2.3.x")
}
```

## 1. Create a Shared Cookie Manager

```kotlin
import android.webkit.CookieManager
import io.ktor.client.plugins.cookies.*
import io.ktor.http.*

class SharedCookieStorage : CookiesStorage {

    private val webViewCookieManager = CookieManager.getInstance()

    override suspend fun get(requestUrl: Url): List<Cookie> {
        val urlString = requestUrl.toString()
        val cookieString = webViewCookieManager.getCookie(urlString) ?: return emptyList()

        return cookieString.split(";").mapNotNull { rawCookie ->
            val parts = rawCookie.trim().split("=", limit = 2)
            if (parts.size == 2) {
                Cookie(name = parts[0].trim(), value = parts[1].trim())
            } else null
        }
    }

    override suspend fun addCookie(requestUrl: Url, cookie: Cookie) {
        val urlString = requestUrl.toString()
        val cookieString = buildString {
            append("${cookie.name}=${cookie.value}")
            cookie.domain?.let { append("; Domain=$it") }
            cookie.path?.let { append("; Path=$it") }
            cookie.maxAge?.let { append("; Max-Age=$it") }
            cookie.expires?.let { append("; Expires=${it.toHttpDate()}") }
            if (cookie.secure) append("; Secure")
            if (cookie.httpOnly) append("; HttpOnly")
        }
        webViewCookieManager.setCookie(urlString, cookieString)
        webViewCookieManager.flush()
    }

    override fun close() { /* no-op */ }
}
```

## 2. Configure Ktor Client

```kotlin
import io.ktor.client.*
import io.ktor.client.engine.android.*
import io.ktor.client.plugins.cookies.*

object HttpClientProvider {

    val client = HttpClient(Android) {
        install(HttpCookies) {
            storage = SharedCookieStorage()
        }
        // Add other plugins as needed
    }
}
```

## 3. Enable Cookies in WebView

```kotlin
import android.webkit.CookieManager
import android.webkit.WebView

fun configureWebView(webView: WebView) {
    CookieManager.getInstance().apply {
        setAcceptCookie(true)
        setAcceptThirdPartyCookies(webView, true)
    }
}
```

## 4. Jetpack Compose WebView + Ktor Integration

```kotlin
import android.annotation.SuppressLint
import android.webkit.WebView
import android.webkit.WebViewClient
import androidx.compose.runtime.*
import androidx.compose.ui.viewinterop.AndroidView
import io.ktor.client.request.*
import io.ktor.client.statement.*

// Compose WebView that shares cookies with Ktor
@SuppressLint("SetJavaScriptEnabled")
@Composable
fun SharedCookieWebView(
    url: String,
    modifier: Modifier = Modifier,
    onPageFinished: (String) -> Unit = {}
) {
    AndroidView(
        modifier = modifier,
        factory = { context ->
            WebView(context).apply {
                configureWebView(this)
                settings.javaScriptEnabled = true
                webViewClient = object : WebViewClient() {
                    override fun onPageFinished(view: WebView, url: String) {
                        super.onPageFinished(view, url)
                        // Flush cookies so Ktor can pick them up immediately
                        CookieManager.getInstance().flush()
                        onPageFinished(url)
                    }
                }
                loadUrl(url)
            }
        },
        update = { webView ->
            webView.loadUrl(url)
        }
    )
}

// ViewModel using Ktor with shared cookies
@HiltViewModel
class MainViewModel @Inject constructor() : ViewModel() {

    private val client = HttpClientProvider.client

    fun fetchWithSharedCookies(url: String) {
        viewModelScope.launch {
            // Ktor will automatically use cookies set by WebView
            val response = client.get(url)
            val body = response.bodyAsText()
            // handle response
        }
    }
}
```

## 5. Usage in a Screen

```kotlin
@Composable
fun LoginScreen(viewModel: MainViewModel = hiltViewModel()) {
    val loginUrl = "https://example.com/login"
    val apiUrl = "https://example.com/api/data"

    SharedCookieWebView(
        url = loginUrl,
        onPageFinished = { currentUrl ->
            // Once login completes, Ktor can use the session cookie
            if (currentUrl.contains("/dashboard")) {
                viewModel.fetchWithSharedCookies(apiUrl)
            }
        }
    )
}
```

## How It Works

| Step | What Happens                                                      |
| ---- | ----------------------------------------------------------------- |
| 1    | User logs in via WebView → browser sets cookies via `Set-Cookie`  |
| 2    | `CookieManager` stores them system-wide                           |
| 3    | Ktor's `SharedCookieStorage.get()` reads from `CookieManager`     |
| 4    | Ktor sends the same session cookies on API requests               |
| 5    | Ktor responses with `Set-Cookie` → written back via `addCookie()` |

## Key Points

- **`CookieManager.flush()`** — call this after WebView page loads to persist cookies to disk before Ktor reads them.
- **Same domain required** — cookies are domain-scoped; WebView and Ktor must hit the same domain.
- **Thread safety** — `CookieManager` is thread-safe, but wrap `addCookie` in a `withContext(Dispatchers.Main)` if you hit issues on older APIs.
- **HTTPS** — `Secure` cookies only work over HTTPS in both WebView and Ktor.

## iOS and KMP

In Kotlin Multiplatform, on iOS, you bridge between Ktor's CookiesStorage and WKWebsiteDataStore / HTTPCookieStorage.

```kt
import io.ktor.client.plugins.cookies.*
import io.ktor.http.*
import platform.Foundation.*
import platform.WebKit.*

class IosCookieStorage : CookiesStorage {

    // WKWebView uses this store — must use .default to share with WKWebView
    private val wkCookieStore = WKWebsiteDataStore.defaultDataStore().httpCookieStore

    // NSHTTPCookieStorage is the system-wide store (URLSession, etc.)
    private val nsCookieStorage = NSHTTPCookieStorage.sharedHTTPCookieStorage

    override suspend fun get(requestUrl: Url): List<Cookie> {
        return suspendCancellableCoroutine { continuation ->
            wkCookieStore.getAllCookies { nsCookies ->
                val cookies = (nsCookies as List<NSHTTPCookie>)
                    .filter { it.matchesUrl(requestUrl) }
                    .map { it.toKtorCookie() }
                continuation.resume(cookies)
            }
        }
    }

    override suspend fun addCookie(requestUrl: Url, cookie: Cookie) {
        val nsCookie = cookie.toNSHTTPCookie(requestUrl) ?: return

        suspendCancellableCoroutine { continuation ->
            wkCookieStore.setCookie(nsCookie) {
                // Also sync to NSHTTPCookieStorage for URLSession sharing
                nsCookieStorage.setCookie(nsCookie)
                continuation.resume(Unit)
            }
        }
    }

    override fun close() { /* no-op */ }
}
```

The four extension helpers referenced above — `matchesUrl`, `toKtorCookie`, and `toNSHTTPCookie` — are yours to write. They're mechanical but tedious:

- `NSHTTPCookie.matchesUrl(Url)` — compare `domain` (allowing a leading `.` for subdomain cookies) and check that the cookie's `path` is a prefix of the request path
- `NSHTTPCookie.toKtorCookie()` — copy `name`, `value`, `domain`, `path`, `expiresDate`, `secure`, `HTTPOnly` across
- `Cookie.toNSHTTPCookie(Url)` — build an `NSHTTPCookie` from a properties map; it returns null if a required key is missing, which is why the caller uses `?: return`

Note the two stores on the iOS side. `WKWebsiteDataStore.defaultDataStore().httpCookieStore` is what `WKWebView` reads. `NSHTTPCookieStorage.sharedHTTPCookieStorage` is what `URLSession` reads. Writing to both keeps the session usable whichever client makes the next request — and it's why `addCookie` above sets the cookie twice.

Also worth knowing: `WKWebsiteDataStore` is asynchronous, hence the `suspendCancellableCoroutine` wrappers. There is no synchronous read, so `get()` genuinely has to suspend.
