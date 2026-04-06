# Beltone Trade — Android Integration Guide

---

## What is this?

Beltone Trade is a Flutter screen that opens inside your app.

The user taps a button → Flutter opens → user does stuff → Flutter closes → user is back in your app.

**Your job:** 2 things:
1. Add the AAR dependency (Step 0 below)
2. Write the code so your app and Flutter can talk (Steps 1-3)

---

## Step 0: Add the AAR to your project

In your `settings.gradle.kts`, add this Maven repo:

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()

        // Beltone Trade
        maven {
            url = uri("https://raw.githubusercontent.com/amratef503092/aar-orange-sdk/main")
        }
    }
}
```

In your `app/build.gradle.kts`:

```kotlin
dependencies {
    releaseImplementation("com.beltonefinancial.beltonetrade:flutter_release:1.0")
}
```

Sync Gradle. If it says BUILD SUCCESSFUL → go to Step 1.

---

## How does it work?

Your app and Flutter send messages to each other using **MethodChannel**.

Think of it like a phone call:
- Flutter calls you → you pick up and answer
- You call Flutter → Flutter picks up and answers

Every call has a **name**. We use 3 names:

| Name | What it does |
|---|---|
| `com.beltone.trade/auth` | Flutter asks for login info |
| `com.beltone.trade/wallet` | Flutter tells you the payment result |
| `com.myorange/navigation` | Flutter says "close me" |

---

## You need to do 3 things

| # | What | One sentence |
|---|---|---|
| 1 | **Login** | Give Flutter the user's credentials (or token) |
| 2 | **Payment** | Save the payment result when Flutter sends it |
| 3 | **Close** | Close the Flutter screen when Flutter says "I'm done" |

---

## Step 1: Create the Application class

This is the main file. Copy it into your project.

> If you already have an Application class, just add the code inside `onCreate()`.

```kotlin
import android.app.Activity
import android.app.Application
import android.os.Bundle
import android.util.Log
import io.flutter.embedding.engine.FlutterEngine
import io.flutter.embedding.engine.FlutterEngineCache
import io.flutter.embedding.engine.dart.DartExecutor
import io.flutter.embedding.engine.loader.FlutterLoader
import io.flutter.plugin.common.MethodChannel
import java.lang.ref.WeakReference

// This holds the payment result
data class ChargeResult(
    val amount: Double,
    val currency: String,
    val method: String,
    val transactionId: String
)

class YourApp : Application() {

    lateinit var flutterEngine: FlutterEngine
    private var currentActivity: WeakReference<Activity>? = null

    // Flutter saves the payment result here
    var lastChargeResult: ChargeResult? = null
        private set

    // Your Activity calls this to get the result
    fun consumeChargeResult(): ChargeResult? {
        val r = lastChargeResult
        lastChargeResult = null
        return r
    }

    override fun onCreate() {
        super.onCreate()

        // --- Track which screen is open (so we can close it later) ---
        registerActivityLifecycleCallbacks(object : ActivityLifecycleCallbacks {
            override fun onActivityResumed(a: Activity) { currentActivity = WeakReference(a) }
            override fun onActivityPaused(a: Activity)  { currentActivity = null }
            override fun onActivityCreated(a: Activity, b: Bundle?) {}
            override fun onActivityStarted(a: Activity) {}
            override fun onActivityStopped(a: Activity) {}
            override fun onActivitySaveInstanceState(a: Activity, b: Bundle) {}
            override fun onActivityDestroyed(a: Activity) {}
        })

        // --- Start Flutter engine ---
        val loader = FlutterLoader()
        loader.startInitialization(this)
        loader.ensureInitializationComplete(this, null)
        flutterEngine = FlutterEngine(this)

        // 1️⃣ LOGIN
        setupAuthChannel()

        // 2️⃣ PAYMENT RESULT
        setupWalletChannel()

        // 3️⃣ CLOSE FLUTTER
        setupNavigationChannel()

        // --- Run Flutter ---
        flutterEngine.dartExecutor.executeDartEntrypoint(
            DartExecutor.DartEntrypoint.createDefault()
        )
        FlutterEngineCache.getInstance().put("beltone_engine", flutterEngine)
    }


    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    //  1️⃣  LOGIN
    //  Channel: "com.beltone.trade/auth"
    //
    //  You have 2 options. Pick one, delete the other.
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

    private fun setupAuthChannel() {

        // ╔═══════════════════════════════════════════╗
        // ║  OPTION A — DEMO (username + password)    ║
        // ║  Use this for testing                     ║
        // ╚═══════════════════════════════════════════╝

        MethodChannel(
            flutterEngine.dartExecutor.binaryMessenger,
            "com.beltone.trade/auth"
        ).setMethodCallHandler { call, result ->
            when (call.method) {

                "getCredentials" -> {
                    val username = "agaber"       // ← change this
                    val password = "Ag@13579"     // ← change this
                    result.success(mapOf("username" to username, "password" to password))
                    Log.d("Beltone", "Sent credentials to Flutter")
                }

                "onLoginResult" -> {
                    val status = call.argument<String>("status")
                    val token  = call.argument<String>("token")
                    val error  = call.argument<String>("error")
                    if (status == "success") Log.d("Beltone", "Login OK")
                    else                     Log.e("Beltone", "Login FAILED: $error")
                    result.success(null)
                }

                else -> result.notImplemented()
            }
        }

        // ╔═══════════════════════════════════════════╗
        // ║  OPTION B — PRODUCTION (JWT token)        ║
        // ║  Use this when you have real auth         ║
        // ║                                           ║
        // ║  To use: delete OPTION A, uncomment this  ║
        // ╚═══════════════════════════════════════════╝
        //
        // MethodChannel(
        //     flutterEngine.dartExecutor.binaryMessenger,
        //     "com.beltone.trade/auth"
        // ).setMethodCallHandler { call, result ->
        //     when (call.method) {
        //
        //         "getToken" -> {
        //             val token: String? = YourAuthManager.getToken()  // ← your real token
        //             if (token != null) {
        //                 result.success(mapOf("token" to token))
        //                 Log.d("Beltone", "Token sent to Flutter")
        //             } else {
        //                 result.success(null)
        //             }
        //         }
        //
        //         "onAuthResult" -> {
        //             val status = call.argument<String>("status")
        //             Log.d("Beltone", "Auth result: $status")
        //             result.success(null)
        //         }
        //
        //         else -> result.notImplemented()
        //     }
        // }
    }


    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    //  2️⃣  PAYMENT RESULT
    //  Channel: "com.beltone.trade/wallet"
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

    private fun setupWalletChannel() {
        MethodChannel(
            flutterEngine.dartExecutor.binaryMessenger,
            "com.beltone.trade/wallet"
        ).setMethodCallHandler { call, result ->
            when (call.method) {

                // Payment worked
                "onChargeSuccess" -> {
                    val amount        = call.argument<Double>("amount") ?: 0.0
                    val currency      = call.argument<String>("currency") ?: "EGP"
                    val method        = call.argument<String>("method") ?: ""
                    val transactionId = call.argument<String>("transactionId") ?: ""

                    Log.d("Beltone", "Payment OK: $amount $currency")
                    lastChargeResult = ChargeResult(amount, currency, method, transactionId)
                    result.success(null)
                }

                // Payment failed
                "onChargeFailed" -> {
                    val error = call.argument<String>("error")
                    Log.e("Beltone", "Payment FAILED: $error")
                    result.success(null)
                }

                // Flutter is closing
                "onFlutterClose" -> {
                    Log.d("Beltone", "Flutter closing")
                    currentActivity?.get()?.finish()
                    result.success(null)
                }

                else -> result.notImplemented()
            }
        }
    }


    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    //  3️⃣  CLOSE FLUTTER
    //  Channel: "com.myorange/navigation"
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

    private fun setupNavigationChannel() {
        MethodChannel(
            flutterEngine.dartExecutor.binaryMessenger,
            "com.myorange/navigation"
        ).setMethodCallHandler { call, result ->
            when (call.method) {
                "goBackToNative" -> {
                    Log.d("Beltone", "Flutter done, closing")
                    currentActivity?.get()?.finish()
                    result.success(null)
                }
                else -> result.notImplemented()
            }
        }
    }
}
```

---

## Step 2: Open the Flutter screen

When the user taps a button in your Activity:

```kotlin
import io.flutter.embedding.android.FlutterActivity

button.setOnClickListener {
    startActivity(
        FlutterActivity
            .withCachedEngine("beltone_engine")
            .build(this)
    )
}
```

That's it. One line opens Flutter.

---

## Step 3: Show the payment result

When Flutter closes, the user comes back to your Activity.

Add this to read the result:

```kotlin
override fun onResume() {
    super.onResume()

    val app = application as YourApp
    val result = app.consumeChargeResult()

    if (result != null) {
        // The user paid! Show it in your UI
        // result.amount        → 500.0
        // result.currency      → "EGP"
        // result.method        → "orange_cash"
        // result.transactionId → "TXN-123456"
    }
}
```

---

## The full flow (what happens)

```
User taps button
    ↓
Flutter opens
    ↓
Flutter asks: "getCredentials" (or "getToken")
    ↓
You send login info
    ↓
Flutter logs in → goes to home screen
    ↓
User goes to Wallet → Cash In → enters amount
    ↓
User taps "Confirm"
    ↓
Payment succeeds
    ↓
Flutter sends you: "onChargeSuccess" (amount, currency, etc.)
    ↓
Flutter shows success message (3 seconds)
    ↓
Flutter calls: "onFlutterClose" → screen closes
    ↓
Your Activity resumes → you read the result → show it
```

---

## Quick reference — all methods

### `com.beltone.trade/auth` (Login)

**Demo:**

| Method | Who calls | What |
|---|---|---|
| `getCredentials` | Flutter asks you | Send: `{ "username": "...", "password": "..." }` |
| `onLoginResult` | Flutter tells you | Receive: `{ "status": "success", "token": "..." }` |

**Production:**

| Method | Who calls | What |
|---|---|---|
| `getToken` | Flutter asks you | Send: `{ "token": "eyJ..." }` or `null` |
| `onAuthResult` | Flutter tells you | Receive: `{ "status": "success" }` or `{ "status": "failed" }` |

### `com.beltone.trade/wallet` (Payment)

| Method | Who calls | What |
|---|---|---|
| `onChargeSuccess` | Flutter tells you | `{ "amount": 500.0, "currency": "EGP", "method": "orange_cash", "transactionId": "TXN-123" }` |
| `onChargeFailed` | Flutter tells you | `{ "amount": 500.0, "error": "..." }` |
| `onFlutterClose` | Flutter tells you | `{ "chargedAmount": 500.0 }` — close the screen |

### `com.myorange/navigation` (Close)

| Method | Who calls | What |
|---|---|---|
| `goBackToNative` | Flutter tells you | Close the Flutter screen |

---

## FAQ

**Q: Demo or Production?**
A: Testing? Use Demo. Real app with real users? Use Production.

**Q: What if login fails?**
A: Flutter tells you via `onLoginResult` (demo) or `onAuthResult` (prod) with `status = "failed"`.

**Q: What if user goes back without paying?**
A: `consumeChargeResult()` returns `null`. That's normal.

**Q: What if payment fails?**
A: Flutter handles it. User sees error inside Flutter. You get `onChargeFailed` in log.

**Q: What if Flutter crashes?**
A: Flutter screen closes. Your app keeps working.

---

## How to test

Run this in terminal:

```bash
adb logcat | grep Beltone
```

You should see:

```
Sent credentials to Flutter     ← login info sent
Login OK                        ← login worked
Payment OK: 500.0 EGP           ← payment done
Flutter done, closing           ← Flutter closed
```

---

## Something not working?

| Problem | Fix |
|---|---|
| Flutter shows login screen | Wrong username/password or bad token |
| No payment result | Add `consumeChargeResult()` in `onResume()` |
| Flutter doesn't close | Check navigation channel is set up |
| App crashes | Make sure `FlutterEngineCache` is in `onCreate()` |
| Nothing works | Check channel names — must be **exact** match |

---

## Checklist

- [ ] Application class has `setupAuthChannel()` — Demo or Production
- [ ] Application class has `setupWalletChannel()`
- [ ] Application class has `setupNavigationChannel()`
- [ ] Login info is correct (credentials or token)
- [ ] Activity reads `consumeChargeResult()` in `onResume()`
- [ ] Button opens Flutter with `FlutterActivity.withCachedEngine("beltone_engine")`
- [ ] Tested with `adb logcat | grep Beltone`

---

**Done!**
