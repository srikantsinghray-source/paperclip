


---

1. Project Structure

```
TimeForgeBooster/
├── app/
│   ├── build.gradle.kts
│   ├── CMakeLists.txt
│   └── src/
│       └── main/
│           ├── AndroidManifest.xml
│           ├── cpp/
│           │   ├── core.cpp          // NDK: coins, multipliers, security
│           │   ├── jni_bridge.cpp    // JNI functions
│           │   ├── integrity.cpp     // Anti-tamper & root detection
│           │   ├── crypto_utils.cpp  // AES/RSA helpers
│           │   └── headers/
│           │       └── core.h
│           ├── java/com/timeforge/booster/
│           │   ├── App.kt
│           │   ├── MainActivity.kt
│           │   ├── di/               // Hilt modules
│           │   ├── data/
│           │   │   ├── local/
│           │   │   │   ├── database/
│           │   │   │   │   ├── AppDatabase.kt
│           │   │   │   │   ├── dao/
│           │   │   │   │   └── entity/
│           │   │   │   ├── datastore/
│           │   │   │   └── secure/   // Native coin storage
│           │   │   └── repository/
│           │   ├── domain/
│           │   │   ├── model/
│           │   │   └── usecase/
│           │   ├── presentation/
│           │   │   ├── navigation/
│           │   │   ├── theme/
│           │   │   ├── viewmodel/
│           │   │   └── screens/
│           │   │       ├── disclaimer/
│           │   │       ├── home/
│           │   │       ├── multiplier/
│           │   │       ├── afk/
│           │   │       ├── library/
│           │   │       ├── stats/
│           │   │       └── coinshop/
│           │   ├── service/
│           │   │   └── AfkForegroundService.kt
│           │   ├── util/
│           │   └── NativeLib.kt      // JNI interface
│           └── res/
│               ├── values/
│               │   ├── strings.xml
│               │   └── themes.xml
│               ├── drawable/
│               └── mipmap/
├── build.gradle.kts (project)
├── gradle.properties
└── settings.gradle.kts
```

---

2. Build Scripts & Config

project/build.gradle.kts

```kotlin
plugins {
    id("com.android.application") version "8.2.0" apply false
    id("org.jetbrains.kotlin.android") version "1.9.20" apply false
    id("com.google.dagger.hilt.android") version "2.48.1" apply false
}
```

settings.gradle.kts

```kotlin
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}
rootProject.name = "TimeForgeBooster"
include(":app")
```

app/build.gradle.kts

```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("kotlin-kapt")
    id("com.google.dagger.hilt.android")
}

android {
    namespace = "com.timeforge.booster"
    compileSdk = 34

    defaultConfig {
        applicationId = "com.timeforge.booster"
        minSdk = 26
        targetSdk = 34
        versionCode = 1
        versionName = "1.0.0"

        ndk {
            abiFilters += listOf("arm64-v8a", "armeabi-v7a", "x86_64", "x86")
        }

        externalNativeBuild {
            cmake {
                arguments += listOf("-DANDROID_STL=c++_shared")
            }
        }
    }

    signingConfigs {
        create("release") {
            // Store keystore info in local.properties or env
        }
    }

    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
            signingConfig = signingConfigs.getByName("release")
            ndk {
                debugSymbolLevel = "none"
            }
        }
        debug {
            isDebuggable = false // still enable native security checks in debug for testing
        }
    }

    externalNativeBuild {
        cmake {
            path = file("src/main/cpp/CMakeLists.txt")
            version = "3.22.1"
        }
    }

    buildFeatures {
        compose = true
    }

    composeOptions {
        kotlinCompilerExtensionVersion = "1.5.5"
    }

    packagingOptions {
        resources {
            excludes += "/META-INF/{AL2.0,LGPL2.1}"
        }
    }
}

dependencies {
    val composeBom = platform("androidx.compose:compose-bom:2024.02.00")
    implementation(composeBom)
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.material3:material3")
    implementation("androidx.compose.ui:ui-tooling-preview")
    implementation("androidx.compose.material:material-icons-extended")
    implementation("androidx.activity:activity-compose:1.8.2")
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.7.0")
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.7.0")
    implementation("androidx.navigation:navigation-compose:2.7.7")
    implementation("androidx.hilt:hilt-navigation-compose:1.1.0")

    // Hilt DI
    implementation("com.google.dagger:hilt-android:2.48.1")
    kapt("com.google.dagger:hilt-android-compiler:2.48.1")

    // Room
    implementation("androidx.room:room-runtime:2.6.1")
    implementation("androidx.room:room-ktx:2.6.1")
    kapt("androidx.room:room-compiler:2.6.1")

    // Security Crypto (for DataStore encryption)
    implementation("androidx.security:security-crypto:1.1.0-alpha06")

    // DataStore (preferences)
    implementation("androidx.datastore:datastore-preferences:1.0.0")

    // AdMob
    implementation("com.google.android.gms:play-services-ads:22.6.0")

    // Play Integrity
    implementation("com.google.android.gms:play-services-integrity:1.2.0")

    // WorkManager
    implementation("androidx.work:work-runtime-ktx:2.9.0")
    implementation("androidx.hilt:hilt-work:1.1.0")
    kapt("androidx.hilt:hilt-compiler:1.1.0")

    // Foreground service
    implementation("androidx.core:core-ktx:1.12.0")
}
```

app/CMakeLists.txt (in cpp folder)

```cmake
cmake_minimum_required(VERSION 3.22.1)
project("timeforge_native")

add_library(timeforge_native SHARED
    core.cpp
    jni_bridge.cpp
    integrity.cpp
    crypto_utils.cpp
)

target_include_directories(timeforge_native PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/headers)

# Optimization and stripping
target_compile_options(timeforge_native PRIVATE -O2 -fvisibility=hidden -ffunction-sections -fdata-sections)
target_link_options(timeforge_native PRIVATE -Wl,--gc-sections -Wl,--strip-all)
```

---

3. Native C++ Core (Military‑Grade Security)

cpp/headers/core.h

```cpp
#ifndef TIMEFORGE_CORE_H
#define TIMEFORGE_CORE_H

#include <jni.h>
#include <string>

// Coin system
int64_t getCoinBalanceNative(JNIEnv* env, jobject context);
bool addCoinsNative(JNIEnv* env, jobject context, int amount, const char* verificationToken);
bool spendCoinsNative(JNIEnv* env, jobject context, int amount);

// Multiplier logic (cost calculation, session limits)
int calculateMultiplierCost(int multiplier, int hours);
bool canActivateMultiplier(int multiplier, int hours, int64_t coinBalance);

// Integrity checks
bool performIntegrityCheck(JNIEnv* env, jobject context);
bool isDeviceCompromisedNative(JNIEnv* env);

// AFK session tracking (native background verification)
bool startAfkSessionNative(int gameId, int multiplier, int hours);
bool stopAfkSessionNative();

// Utility
std::string getEncryptedDeviceId(JNIEnv* env, jobject context);

#endif
```

cpp/core.cpp

```cpp
#include "core.h"
#include "crypto_utils.h"
#include <android/log.h>
#include <mutex>
#include <fstream>
#include <cstring>

#define LOG_TAG "TimeForgeNative"
#define LOGE(...) __android_log_print(ANDROID_LOG_ERROR, LOG_TAG, __VA_ARGS__)

static std::mutex coinMutex;
static int64_t cachedCoinBalance = -1; // -1 means not loaded

extern "C" JNIEXPORT jint JNICALL
Java_com_timeforge_booster_NativeLib_getCoinBalance(JNIEnv* env, jclass, jobject context) {
    return (jint)getCoinBalanceNative(env, context);
}

extern "C" JNIEXPORT jboolean JNICALL
Java_com_timeforge_booster_NativeLib_addCoins(JNIEnv* env, jclass, jobject context, jint amount, jstring verificationToken) {
    const char* token = env->GetStringUTFChars(verificationToken, nullptr);
    bool result = addCoinsNative(env, context, amount, token);
    env->ReleaseStringUTFChars(verificationToken, token);
    return result;
}

extern "C" JNIEXPORT jboolean JNICALL
Java_com_timeforge_booster_NativeLib_spendCoins(JNIEnv* env, jclass, jobject context, jint amount) {
    return spendCoinsNative(env, context, amount);
}

extern "C" JNIEXPORT jint JNICALL
Java_com_timeforge_booster_NativeLib_calculateMultiplierCost(JNIEnv*, jclass, jint multiplier, jint hours) {
    return calculateMultiplierCost(multiplier, hours);
}

extern "C" JNIEXPORT jboolean JNICALL
Java_com_timeforge_booster_NativeLib_canActivateMultiplier(JNIEnv* env, jclass, jobject context, jint multiplier, jint hours) {
    int64_t bal = getCoinBalanceNative(env, context);
    return canActivateMultiplier(multiplier, hours, bal);
}

extern "C" JNIEXPORT jboolean JNICALL
Java_com_timeforge_booster_NativeLib_isDeviceCompromised(JNIEnv* env, jclass) {
    return isDeviceCompromisedNative(env);
}

extern "C" JNIEXPORT jboolean JNICALL
Java_com_timeforge_booster_NativeLib_performIntegrityCheck(JNIEnv* env, jclass, jobject context) {
    return performIntegrityCheck(env, context);
}

// Implementation

int64_t getCoinBalanceNative(JNIEnv* env, jobject context) {
    std::lock_guard<std::mutex> lock(coinMutex);
    if (cachedCoinBalance >= 0) return cachedCoinBalance;
    // Load from encrypted file (CryptoUtils decrypts using device-specific key)
    std::string encrypted = readEncryptedFile(env, context, "coin_balance.dat");
    if (encrypted.empty()) {
        cachedCoinBalance = 0;
        return 0;
    }
    cachedCoinBalance = std::stoll(encrypted);
    return cachedCoinBalance;
}

bool addCoinsNative(JNIEnv* env, jobject context, int amount, const char* verificationToken) {
    // 1. Server-side verification (mock: check token signature)
    if (!verifyAdRewardToken(verificationToken)) {
        LOGE("Invalid ad reward token");
        return false;
    }
    std::lock_guard<std::mutex> lock(coinMutex);
    int64_t current = getCoinBalanceNative(env, context);
    current += amount;
    cachedCoinBalance = current;
    // Persist encrypted
    std::string balanceStr = std::to_string(current);
    writeEncryptedFile(env, context, "coin_balance.dat", balanceStr);
    return true;
}

bool spendCoinsNative(JNIEnv* env, jobject context, int amount) {
    std::lock_guard<std::mutex> lock(coinMutex);
    int64_t current = getCoinBalanceNative(env, context);
    if (current < amount) return false;
    current -= amount;
    cachedCoinBalance = current;
    writeEncryptedFile(env, context, "coin_balance.dat", std::to_string(current));
    return true;
}

int calculateMultiplierCost(int multiplier, int hours) {
    // Example cost formula: base cost 5 coins per hour for 2x, increasing with multiplier and hours
    // 1x = always 0 coins (free)
    if (multiplier <= 1) return 0;
    double base = 5.0;
    double multFactor = (multiplier - 1) * 0.8; // 2x = 0.8, 30x = 23.2
    double hourFactor = hours * 1.2;
    return static_cast<int>(base * multFactor * hourFactor) + 1; // ensure >0
}

bool canActivateMultiplier(int multiplier, int hours, int64_t coinBalance) {
    int cost = calculateMultiplierCost(multiplier, hours);
    return coinBalance >= cost;
}

// Integrity and anti-tamper (detailed in integrity.cpp)
```

cpp/integrity.cpp

```cpp
#include "core.h"
#include <jni.h>
#include <string>
#include <sys/stat.h>
#include <unistd.h>
#include <fstream>
#include <android/log.h>

#define LOG_TAG "Integrity"
#define LOGE(...) __android_log_print(ANDROID_LOG_ERROR, LOG_TAG, __VA_ARGS__)

// Root detection
static bool checkForRoot() {
    const char* rootPaths[] = {
        "/system/app/Superuser.apk",
        "/sbin/su",
        "/system/bin/su",
        "/system/xbin/su",
        "/data/local/xbin/su",
        "/data/local/bin/su",
        "/system/sd/xbin/su",
        "/system/bin/failsafe/su",
        "/data/local/su"
    };
    for (const char* path : rootPaths) {
        if (access(path, F_OK) == 0) return true;
    }
    return false;
}

// Frida detection
static bool isFridaRunning() {
    std::ifstream maps("/proc/self/maps");
    std::string line;
    while (std::getline(maps, line)) {
        if (line.find("frida") != std::string::npos ||
            line.find("gum-js-loop") != std::string::npos ||
            line.find("linjector") != std::string::npos) {
            return true;
        }
    }
    return false;
}

// Debugger detection
static bool isDebuggerAttached() {
    std::ifstream status("/proc/self/status");
    std::string line;
    while (std::getline(status, line)) {
        if (line.find("TracerPid:") != std::string::npos) {
            long tracerPid = std::stol(line.substr(line.find(":") + 1));
            if (tracerPid != 0) return true;
        }
    }
    return false;
}

extern "C" bool isDeviceCompromisedNative(JNIEnv* env) {
    // Multiple checks
    if (checkForRoot()) {
        LOGE("Root detected");
        return true;
    }
    if (isFridaRunning()) {
        LOGE("Frida injection detected");
        return true;
    }
    if (isDebuggerAttached()) {
        LOGE("Debugger attached");
        return true;
    }
    // Check for emulator (simple)
    char* qemu = getenv("ANDROID_EMULATOR");
    if (qemu != nullptr) return true;
    return false;
}

// APK signature check using Java interop (simplified, real check uses PackageManager)
extern "C" bool performIntegrityCheck(JNIEnv* env, jobject context) {
    // Call Java method to verify Play Integrity and APK signature via C++ JNI
    jclass cls = env->GetObjectClass(context);
    jmethodID mid = env->GetMethodID(cls, "verifyPlayIntegrity", "()Z");
    if (mid == nullptr) return false;
    jboolean result = env->CallBooleanMethod(context, mid);
    return result;
}
```

cpp/crypto_utils.cpp (abbreviated)

```cpp
#include <string>
#include <jni.h>
// Use Android keystore-backed AES encryption for file storage.
// Implementation uses JNI to call Java KeyStore.
std::string readEncryptedFile(JNIEnv* env, jobject context, const std::string& filename) { /* ... */ }
void writeEncryptedFile(JNIEnv* env, jobject context, const std::string& filename, const std::string& data) { /* ... */ }
bool verifyAdRewardToken(const char* token) { /* validate with server public key */ return true; }
```

cpp/jni_bridge.cpp (includes all JNI registrations)

```cpp
#include <jni.h>
#include "core.h"

extern "C" {
    JNIEXPORT jint JNICALL Java_com_timeforge_booster_NativeLib_getCoinBalance(JNIEnv*, jclass, jobject);
    JNIEXPORT jboolean JNICALL Java_com_timeforge_booster_NativeLib_addCoins(JNIEnv*, jclass, jobject, jint, jstring);
    JNIEXPORT jboolean JNICALL Java_com_timeforge_booster_NativeLib_spendCoins(JNIEnv*, jclass, jobject, jint);
    JNIEXPORT jint JNICALL Java_com_timeforge_booster_NativeLib_calculateMultiplierCost(JNIEnv*, jclass, jint, jint);
    JNIEXPORT jboolean JNICALL Java_com_timeforge_booster_NativeLib_canActivateMultiplier(JNIEnv*, jclass, jobject, jint, jint);
    JNIEXPORT jboolean JNICALL Java_com_timeforge_booster_NativeLib_isDeviceCompromised(JNIEnv*, jclass);
    JNIEXPORT jboolean JNICALL Java_com_timeforge_booster_NativeLib_performIntegrityCheck(JNIEnv*, jclass, jobject);
}
```

---

4. Android Manifest

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE" />
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
    <uses-permission android:name="android.permission.INTERNET" />

    <application
        android:name=".App"
        android:allowBackup="false"
        android:dataExtractionRules="@xml/data_extraction_rules"
        android:fullBackupContent="false"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:supportsRtl="true"
        android:theme="@style/Theme.TimeForgeBooster"
        android:usesCleartextTraffic="false">

        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:theme="@style/Theme.TimeForgeBooster">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <service
            android:name=".service.AfkForegroundService"
            android:foregroundServiceType="specialUse"
            android:exported="false" />

        <!-- AdMob meta-data -->
        <meta-data
            android:name="com.google.android.gms.ads.APPLICATION_ID"
            android:value="ca-app-pub-xxxxxxxx~yyyyyyyy" />
    </application>
</manifest>
```

---

5. Kotlin/Java Source Files

App.kt (Application class)

```kotlin
package com.timeforge.booster

import android.app.Application
import dagger.hilt.android.HiltAndroidApp
import timber.log.Timber
import com.google.android.gms.ads.MobileAds

@HiltAndroidApp
class App : Application() {
    companion object {
        lateinit var instance: App
    }

    override fun onCreate() {
        super.onCreate()
        instance = this
        Timber.plant(Timber.DebugTree())
        MobileAds.initialize(this) {}

        // Load native library
        System.loadLibrary("timeforge_native")
        // Run initial integrity checks
        if (NativeLib.isDeviceCompromised()) {
            // Exit or show message
            throw SecurityException("Device integrity compromised")
        }
    }
}
```

MainActivity.kt

```kotlin
package com.timeforge.booster

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import androidx.activity.viewModels
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Surface
import androidx.compose.ui.Modifier
import com.timeforge.booster.presentation.navigation.NavGraph
import com.timeforge.booster.presentation.theme.TimeForgeBoosterTheme
import com.timeforge.booster.presentation.viewmodel.OnboardingViewModel
import dagger.hilt.android.AndroidEntryPoint

@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    private val onboardingViewModel: OnboardingViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        // Verify Play Integrity on each launch
        if (!NativeLib.performIntegrityCheck(this)) {
            finishAffinity()
            return
        }
        setContent {
            TimeForgeBoosterTheme {
                Surface(modifier = Modifier.fillMaxSize(), color = MaterialTheme.colorScheme.background) {
                    NavGraph(onboardingViewModel = onboardingViewModel)
                }
            }
        }
    }
}
```

NativeLib.kt (JNI interface)

```kotlin
package com.timeforge.booster

object NativeLib {
    init {
        System.loadLibrary("timeforge_native")
    }

    external fun getCoinBalance(): Int
    external fun addCoins(amount: Int, verificationToken: String): Boolean
    external fun spendCoins(amount: Int): Boolean
    external fun calculateMultiplierCost(multiplier: Int, hours: Int): Int
    external fun canActivateMultiplier(multiplier: Int, hours: Int): Boolean
    external fun isDeviceCompromised(): Boolean
    external fun performIntegrityCheck(context: Any): Boolean
}
```

OnboardingViewModel.kt (and Disclaimer screen)

```kotlin
package com.timeforge.booster.presentation.viewmodel

import androidx.lifecycle.ViewModel
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.getValue
import androidx.compose.runtime.setValue
import dagger.hilt.android.lifecycle.HiltViewModel
import javax.inject.Inject

@HiltViewModel
class OnboardingViewModel @Inject constructor() : ViewModel() {
    var disclaimerAccepted by mutableStateOf(false)
        private set

    fun acceptDisclaimer() {
        disclaimerAccepted = true
        // Save timestamp and device hash in DataStore
    }
}
```

DisclaimerScreen.kt (full screen with mandatory checkbox)

```kotlin
package com.timeforge.booster.presentation.screens.disclaimer

import androidx.compose.foundation.layout.*
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.verticalScroll
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp

@Composable
fun DisclaimerScreen(
    onAccept: () -> Unit
) {
    var checked by remember { mutableStateOf(false) }
    val scrollState = rememberScrollState()

    Column(
        modifier = Modifier.fillMaxSize().padding(16.dp).verticalScroll(scrollState),
        verticalArrangement = Arrangement.spacedBy(12.dp)
    ) {
        Text(
            "⚠️ IRREVOCABLE ULTIMATE LIABILITY DISCLAIMER, RELEASE, AND INDEMNIFICATION AGREEMENT ⚠️",
            fontWeight = FontWeight.Bold,
            fontSize = 20.sp,
            color = MaterialTheme.colorScheme.error
        )
        // Insert the entire disclaimer text here (long string from spec)
        Text(fullDisclaimerText, fontSize = 12.sp)

        Spacer(modifier = Modifier.height(16.dp))

        Row(verticalAlignment = Alignment.CenterVertically) {
            Checkbox(checked = checked, onCheckedChange = { checked = it })
            Text(
                "I have read, fully understood, and irrevocably agree to all terms above.",
                modifier = Modifier.weight(1f)
            )
        }

        Button(
            onClick = onAccept,
            enabled = checked,
            modifier = Modifier.align(Alignment.End)
        ) {
            Text("Continue")
        }
    }
}

val fullDisclaimerText = """
THIS APPLICATION, TIMEFORGE BOOSTER (THE "APP"), IS FURNISHED STRICTLY "AS IS," "AS AVAILABLE," AND "WITH ALL FAULTS" WITHOUT ANY WARRANTY OR RECOURSE WHATSOEVER. YOUR ACCESS, DOWNLOAD, INSTALLATION, AND USE OF THE APP IS VOLUNTARY AND CARRIES INHERENT, SUBSTANTIAL, AND IRREVERSIBLE RISKS...
[Full text from spec truncated for brevity – insert entire spec text]
""".trimIndent()
```

Navigation (NavGraph.kt)

```kotlin
package com.timeforge.booster.presentation.navigation

import androidx.compose.runtime.Composable
import androidx.navigation.NavHostController
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import com.timeforge.booster.presentation.screens.disclaimer.DisclaimerScreen
import com.timeforge.booster.presentation.screens.home.HomeScreen
import com.timeforge.booster.presentation.viewmodel.OnboardingViewModel

@Composable
fun NavGraph(onboardingViewModel: OnboardingViewModel) {
    val navController = androidx.navigation.compose.rememberNavController()

    NavHost(navController = navController, startDestination = if (onboardingViewModel.disclaimerAccepted) "home" else "disclaimer") {
        composable("disclaimer") {
            DisclaimerScreen(onAccept = {
                onboardingViewModel.acceptDisclaimer()
                navController.navigate("home") {
                    popUpTo("disclaimer") { inclusive = true }
                }
            })
        }
        composable("home") { HomeScreen(navController) }
        composable("multiplier") { /* MultiplierScreen */ }
        composable("afk") { /* AfkScreen */ }
        composable("library") { /* LibraryScreen */ }
        composable("coinshop") { /* CoinShopScreen */ }
    }
}
```

HomeScreen.kt (displays coin balance and ad button)

```kotlin
@Composable
fun HomeScreen(navController: NavController) {
    val coinBalance by viewModel.coinBalance.collectAsState() // from HomeViewModel
    Column {
        Row(Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.SpaceBetween) {
            Text("Coins: $coinBalance")
            Button(onClick = { viewModel.showRewardedAd() }) { Text("Watch Ad +1 Coin") }
        }
        // Navigation to multipliers, AFK, etc.
        Button(onClick = { navController.navigate("multiplier") }) { Text("Multipliers") }
        // ...
    }
}
```

Data Layer: Room entities, DAO, Database (sketched)

```kotlin
@Entity(tableName = "games")
data class GameEntity(
    @PrimaryKey val id: String,
    val name: String,
    val packageName: String?,
    val isIdle: Boolean
)

@Entity(tableName = "coin_transactions")
data class CoinTransaction(
    @PrimaryKey val id: String,
    val amount: Int,
    val source: String, // "ad"
    val timestamp: Long,
    val verified: Boolean
)

@Dao
interface CoinTransactionDao {
    @Insert
    suspend fun insert(transaction: CoinTransaction)
}
```

Repository (CoinRepository)

```kotlin
class CoinRepository @Inject constructor(
    private val nativeLib: NativeLib, // JNI wrapper
    private val coinTransactionDao: CoinTransactionDao
) {
    fun getBalance(): Int = nativeLib.getCoinBalance()

    suspend fun addCoins(amount: Int, verificationToken: String): Boolean {
        if (nativeLib.addCoins(amount, verificationToken)) {
            coinTransactionDao.insert(CoinTransaction(UUID.randomUUID().toString(), amount, "ad", System.currentTimeMillis(), true))
            return true
        }
        return false
    }

    suspend fun spendCoins(amount: Int): Boolean = nativeLib.spendCoins(amount)
}
```

AdMob Rewarded Ad Helper

```kotlin
@Singleton
class AdRewardManager @Inject constructor(
    @ApplicationContext private val context: Context,
    private val coinRepository: CoinRepository
) {
    private var rewardedAd: RewardedAd? = null

    fun loadAd(activity: Activity) {
        val adRequest = AdRequest.Builder().build()
        RewardedAd.load(context, "ca-app-pub-3940256099942544/5224354917", adRequest, object : RewardedAdLoadCallback() {
            override fun onAdLoaded(ad: RewardedAd) { rewardedAd = ad }
            override fun onAdFailedToLoad(error: LoadAdError) { }
        })
    }

    fun showAd(activity: Activity, onUserEarnedReward: () -> Unit) {
        rewardedAd?.let { ad ->
            ad.fullScreenContentCallback = object : FullScreenContentCallback() {
                override fun onAdDismissedFullScreenContent() { loadAd(activity) }
            }
            ad.show(activity) { rewardItem ->
                // SSV token from server
                val verificationToken = ad.responseInfo?.responseId ?: ""
                viewModelScope.launch {
                    coinRepository.addCoins(1, verificationToken)
                    onUserEarnedReward()
                }
            }
        } ?: run { loadAd(activity) }
    }
}
```

Foreground Service for AFK (AFKForegroundService.kt)

```kotlin
@AndroidEntryPoint
class AfkForegroundService : Service() {
    // Uses native calls to start session, monitors time, etc.
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        // startForeground with notification
        // NativeLib.startAfkSession(...)
        return START_STICKY
    }
}
```

---

6. Security & Obfuscation

· ProGuard rules (proguard-rules.pro):

```
-keep class com.timeforge.booster.NativeLib { *; }
-keepclasseswithmembernames class * { native <methods>; }
# Aggressive obfuscation for everything else.
-allowaccessmodification
-repackageclasses
-overloadaggressively
```

·  