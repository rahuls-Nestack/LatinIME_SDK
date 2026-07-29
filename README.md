# LatinIME SDK — Third-Party Integration Guide (v2)

**Version:** 2.0  
**Supersedes:** [PARTNER_INTEGRATION_GUIDE.md](../test-app_git/docs/PARTNER_INTEGRATION_GUIDE.md) (v1 — auth-only)  
**Last updated:** 2026-07-28  
**Reference harness:** `test-app-sdk`  
**Artifact repository:** [LatinIME_SDK on GitHub](https://github.com/rahuls-Nestack/LatinIME_SDK) (local clone often named `LatinIME_SDK` / `KSLMA_EPLMA_SDK`)

This guide explains how to embed LatinIME SDK **AAR files** into your Android application. Two approaches are supported:


| Method                | Best for                                                              | Network at build time           |
| --------------------- | --------------------------------------------------------------------- | ------------------------------- |
| **1. Manual AAR**     | Partners who receive AAR files directly (email, portal, share folder) | No                              |
| **2. Git repository** | Partners with read access to the private LatinIME_SDK GitHub repo     | Yes (first Gradle sync / clone) |


---

## What’s new in v2 (vs v1)


| Area        | v1                  | v2                                                           |
| ----------- | ------------------- | ------------------------------------------------------------ |
| Modules     | Auth only           | **Auth + Shell + Keyboard**                                  |
| Flavors     | kslma, eplma        | **kslma, eplma, partner**                                    |
| After login | Stay on auth UI     | **MainActivity / Settings** (shell)                          |
| Keyboard    | Must **not** appear | **IME appears** in system keyboard list                      |
| Public API  | Minimal             | `**SdkInitializer`**, `**KeyboardSdk`**, `**AuthErrorCode**` |
| Auth gate   | N/A                 | Unauthenticated users **cannot type** (redirect to Login)    |
| Branding    | Product AARs        | Host can override strings/drawables; partner is white-label  |


---

## Current delivery scope

Integrate **three release AARs per flavor**:


| Module   | Alias filename                  | Purpose                             |
| -------- | ------------------------------- | ----------------------------------- |
| Auth     | `{flavor}-auth-release.aar`     | Login / license (`LoginActivity`)   |
| Shell    | `{flavor}-shell-release.aar`    | MainActivity, Settings (post-login) |
| Keyboard | `{flavor}-keyboard-release.aar` | IME service, typing, encryption     |


`{flavor}` = `kslma` | `eplma` | `partner`

**Also published (same bits, alternate name):**

- `latinime-auth-{flavor}-release.aar`
- `latinime-shell-{flavor}-release.aar`
- `latinime-keyboard-{flavor}-release.aar`

In the **LatinIME_SDK** git repo, AARs are typically stored under:

```
LatinIME_SDK/
  auth/      latinime-auth-*-release.aar   (and/or {flavor}-auth-release.aar)
  shell/     latinime-shell-*-release.aar
  keyboard/  latinime-keyboard-*-release.aar
```

**Use one flavor’s three AARs together.** Do not mix kslma auth with eplma keyboard in the same app variant.

### Expected behavior after integration

1. App launches → **LoginActivity** (unless already licensed).
2. After successful auth → **MainActivity** / settings from shell.
3. **Settings → System → Languages & input → On-screen keyboard** lists the product IME:
  - kslma → **Keystroke Lock**
  - eplma → **EndpointLock**
  - partner → **Secure Keyboard** (host may override display name via resources)
4. If the user opens a text field **without** a valid license → IME hides and opens **LoginActivity** (auth gate).
5. With a valid license → typing works; release builds must not log plaintext keystrokes.

### Minimum requirements


| Setting      | Value                                                       |
| ------------ | ----------------------------------------------------------- |
| `minSdk`     | 30                                                          |
| `compileSdk` | 35 recommended                                              |
| `targetSdk`  | 36 recommended                                              |
| AndroidX     | Required                                                    |
| NDK / ABI    | AARs ship **arm64-v8a** (+ **x86_64** for emulator testing) |


---

## Shared integration steps (both methods)

### 1. Transitive dependencies

AARs do **not** bundle every runtime library. Add to your app module:

```gradle
dependencies {
    implementation 'com.google.code.findbugs:jsr305:3.0.2'
    implementation 'androidx.appcompat:appcompat:1.6.1'
    implementation 'com.google.android.material:material:1.9.0'
    implementation 'androidx.constraintlayout:constraintlayout:2.2.1'
    implementation 'com.android.volley:volley:1.2.1'
    // Required when consuming keyboard AAR via files() / flatDir
    implementation 'androidx.lifecycle:lifecycle-runtime-ktx:2.6.2'
    implementation 'androidx.lifecycle:lifecycle-viewmodel-ktx:2.6.2'
}
```

Optional (if you use Firebase / Crashlytics in the host, as the reference harness does):

```gradle
implementation platform('com.google.firebase:firebase-bom:34.12.0')
implementation 'com.google.firebase:firebase-crashlytics'
implementation 'com.google.firebase:firebase-analytics'
```

### 2. `gradle.properties` (project root)

```properties
android.useAndroidX=true
android.nonTransitiveRClass=false
android.suppressUnsupportedCompileSdk=35
```

> **Note:** The keyboard AAR historically expects `nonTransitiveRClass=false` when merging LatinIME resources. Prefer `false` unless ACS confirms otherwise for your drop.

### 3. Initialize the SDK in `Application`

```java
public class MyApplication extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
        SdkInitializer.init(this, new SdkInitializer.Config()
                .environment(SdkEnvironment.PRODUCTION));
        // Optional diagnostics:
        // Log.i("SDK", KeyboardSdk.getIntegrationStatus(this).toString());
    }
}
```

Register in the manifest: `android:name=".MyApplication"`.

### 4. Host application manifest

Keep the host manifest thin. Login / Main / IME services merge from the AARs.

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <application
        android:name=".MyApplication"
        android:label="@string/app_name"
        android:allowBackup="true"
        android:supportsRtl="true"
        tools:replace="android:label">

        <!-- Optional: strip base AOSP LatinIME if it ever merges -->
        <service
            android:name="com.android.inputmethod.latin.LatinIME"
            tools:node="remove" />
    </application>
</manifest>
```

**Flavor-specific:** if you ship multiple flavors, each flavor should **not** strip its own IME, and **should** strip other product IMEs. Example for partner:

```xml
<!-- app/src/partner/AndroidManifest.xml -->
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">
    <application>
        <service
            android:name="com.advancedcybersecurity.android.kslma.LatinIME"
            tools:node="remove" />
        <service
            android:name="com.advancedcybersecurity.android.eplma.LatinIME"
            tools:node="remove" />
    </application>
</manifest>
```

IME service classes (merged from keyboard AAR):


| Flavor  | IME service                                          |
| ------- | ---------------------------------------------------- |
| kslma   | `com.advancedcybersecurity.android.kslma.LatinIME`   |
| eplma   | `com.advancedcybersecurity.android.eplma.LatinIME`   |
| partner | `com.advancedcybersecurity.android.partner.LatinIME` |


### 5. Product flavors

```gradle
android {
    flavorDimensions 'default'
    productFlavors {
        kslma {
            dimension 'default'
            applicationIdSuffix '.kslma'
        }
        eplma {
            dimension 'default'
            applicationIdSuffix '.eplma'
        }
        partner {
            dimension 'default'
            applicationIdSuffix '.partner'
        }
    }
}
```

Ship **only the flavors you license**. One set of three AARs per variant.

### 6. Branding overrides (optional)

Same resource `name` in the **host** `res/` wins at merge time (rebuild host app only):

```xml
<!-- app/src/partner/res/values/strings.xml -->
<resources>
    <string name="app_name">Your Brand Keyboard</string>
    <string name="action_sign_in">Sign In</string>
</resources>
```

SDK default for `action_sign_in` is **Activate**. Partner white-label IME label defaults to **Secure Keyboard**.

### 7. Public API (optional but recommended)

```java
// Auth state for host UI
AuthErrorCode code = KeyboardSdk.getAuthErrorCode(context);
boolean ok = KeyboardSdk.isAuthorized(context);

// Open login if needed
KeyboardSdk.requireAuthentication(context);

// System IME settings
KeyboardSdk.openInputMethodSettings(context);

// Setup wizard (enable / select keyboard) when keyboard module is present
KeyboardSdk.launchSetupWizard(context);

// Snapshot for support / logging
SdkIntegrationStatus status = KeyboardSdk.getIntegrationStatus(context);
```

**AuthErrorCode** values include: `OK`, `NOT_INITIALIZED`, `NOT_AUTHENTICATED`, `LICENSE_EXPIRED`, `LICENSE_SUSPENDED`, `NETWORK_ERROR`, `INVALID_CREDENTIALS`, `UNKNOWN`.

### 8. Release minify (R8)

If `minifyEnabled true` on release:

1. Keep ACS consumer rules from the AARs (packaged via `consumerProguardFiles`).
2. Add host keeps as a safety net (especially with `files()` / flatDir):

```proguard
-keep class com.android.inputmethod.latin.enc.** { *; }
-keep class com.android.inputmethod.latin.api.** { *; }
-keep class com.advancedcybersecurity.sdk.** { *; }
-keep class * extends android.inputmethodservice.InputMethodService { *; }
-keep class com.advancedcybersecurity.android.**.LatinIME { *; }
-keep class com.android.inputmethod.latin.ui.** { *; }
-keepclassmembers class * { native <methods>; }
```

---

# Method 1 — Manual AAR dependency

## Step 1 — Obtain AARs

Receive from ACS, or download from the LatinIME_SDK repository folders (`auth/`, `shell/`, `keyboard/`).

For production, use **release** AARs for your licensed flavor(s).

## Step 2 — Copy into the project

```powershell
mkdir app\libs -Force
copy path\to\kslma-auth-release.aar     app\libs\
copy path\to\kslma-shell-release.aar    app\libs\
copy path\to\kslma-keyboard-release.aar app\libs\
```

Recommended layout:

```
your-app/
  app/
    libs/
      kslma-auth-release.aar
      kslma-shell-release.aar
      kslma-keyboard-release.aar
    build.gradle
    src/main/AndroidManifest.xml
```

## Step 3 — Register `flatDir` (or use `files()`)

**Option A — `settings.gradle`:**

```gradle
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
        flatDir { dirs file("app/libs") }
    }
}
```

**Option B — `files()` (no flatDir):**

```gradle
dependencies {
    kslmaImplementation files(
            'libs/kslma-auth-release.aar',
            'libs/kslma-shell-release.aar',
            'libs/kslma-keyboard-release.aar')
}
```

## Step 4 — Dependencies in `app/build.gradle`

**Single flavor (KSLMA):**

```gradle
dependencies {
    implementation(name: 'kslma-auth-release', ext: 'aar')
    implementation(name: 'kslma-shell-release', ext: 'aar')
    implementation(name: 'kslma-keyboard-release', ext: 'aar')
    // + transitive deps from Shared steps
}
```

**Multiple flavors:**

```gradle
dependencies {
    kslmaImplementation(name: 'kslma-auth-release', ext: 'aar')
    kslmaImplementation(name: 'kslma-shell-release', ext: 'aar')
    kslmaImplementation(name: 'kslma-keyboard-release', ext: 'aar')

    eplmaImplementation(name: 'eplma-auth-release', ext: 'aar')
    eplmaImplementation(name: 'eplma-shell-release', ext: 'aar')
    eplmaImplementation(name: 'eplma-keyboard-release', ext: 'aar')

    partnerImplementation(name: 'partner-auth-release', ext: 'aar')
    partnerImplementation(name: 'partner-shell-release', ext: 'aar')
    partnerImplementation(name: 'partner-keyboard-release', ext: 'aar')
}
```

If your drop uses `latinime-*-{flavor}-release.aar` names, either rename to the alias form or reference the long names in `files()`.

## Step 5 — Build and install

```powershell
.\gradlew.bat :app:assembleKslmaDebug
adb install -r app\build\outputs\apk\kslma\debug\app-kslma-debug.apk
```

## Step 6 — Verify on device


| Step                                       | Expected                                              |
| ------------------------------------------ | ----------------------------------------------------- |
| Install / open app                         | LoginActivity (or Main if already licensed)           |
| Complete login                             | MainActivity / settings available                     |
| Settings → Keyboards                       | Product IME **appears**                               |
| Enable + select IME → type                 | Characters appear                                     |
| Clear license / fresh install → open field | LoginActivity; typing blocked until auth              |
| Release build + type                       | No plaintext under logcat `Nestack_EncryptionLogging` |


## Updating AARs

1. Replace the three files in `app/libs/` (or LatinIME_SDK clone).
2. `.\gradlew.bat clean :app:assembleKslmaRelease`
3. Reinstall.

---

# Method 2 — Git repository integration

Use when your team has **read access** to the private LatinIME_SDK GitHub repository.

## Prerequisites

1. GitHub access to [https://dev.azure.com/jim0634/acs-eplma-sdk/_git/acs-eplma-sdk](https://dev.azure.com/jim0634/acs-eplma-sdk/_git/acs-eplma-sdk) (confirm URL with ACS).
2. PAT with `**repo`** scope (or fine-grained read on that repo).
3. Git on the build machine.

## Step 1 — `gradle.properties` (do not commit secrets)

```properties
android.useAndroidX=true
android.nonTransitiveRClass=false
android.suppressUnsupportedCompileSdk=35

latinimeSdkGitRef=main
gpr.token=YOUR_GITHUB_PAT
```

```
# .gitignore
gradle.properties
LatinIME_SDK/
KSLMA_EPLMA_SDK/
```

## Step 2 — Clone layout expectation

After clone, ensure these exist (names may use `latinime-` prefix):

```
LatinIME_SDK/auth/...
LatinIME_SDK/shell/...
LatinIME_SDK/keyboard/...
```

Point `flatDir` at each folder, or copy alias AARs into `app/libs` as in Method 1.

Example `flatDir` with subfolders:

```gradle
flatDir {
    dirs file("LatinIME_SDK/auth"),
         file("LatinIME_SDK/shell"),
         file("LatinIME_SDK/keyboard")
}
```

Then depend on `latinime-auth-kslma-release` / `latinime-shell-kslma-release` / `latinime-keyboard-kslma-release` (or whatever filenames ACS publishes).

## Step 3 — Refresh after a new drop

```powershell
Remove-Item -Recurse -Force LatinIME_SDK
.\gradlew.bat :app:assembleKslmaDebug
```

Pin a tag when ACS provides one:

```properties
latinimeSdkGitRef=v2.0.0
```

---

# Command reference


| Action                                                    | Command                                                                        |
| --------------------------------------------------------- | ------------------------------------------------------------------------------ |
| Build KSLMA debug                                         | `.\gradlew.bat :app:assembleKslmaDebug`                                        |
| Build EPLMA release (minify)                              | `.\gradlew.bat :app:assembleEplmaRelease`                                      |
| Build partner debug                                       | `.\gradlew.bat :app:assemblePartnerDebug`                                      |
| Install partner release                                   | `adb install -r app\build\outputs\apk\partner\release\app-partner-release.apk` |
| Logcat partner IME                                        | `adb logcat -s PartnerLatinIME:I`                                              |
| Logcat encryption (must be empty of plaintext in release) | `adb logcat -s Nestack_EncryptionLogging:D`                                    |
| List packages                                             | `adb shell pm list packages                                                    |


---

# Troubleshooting


| Symptom                          | Likely cause                          | Fix                                                                                |
| -------------------------------- | ------------------------------------- | ---------------------------------------------------------------------------------- |
| Missing `*-auth-release.aar`     | Wrong folder / filename               | Use `{flavor}-auth-release.aar` or `latinime-auth-{flavor}-release.aar`            |
| Login works but no MainActivity  | Shell AAR missing                     | Add `{flavor}-shell-release.aar`                                                   |
| IME not in keyboard list         | Keyboard AAR missing or stripped      | Add keyboard AAR; remove `tools:node="remove"` for your flavor IME                 |
| `Incomplete R` / missing layouts | Namespace clash or incomplete AAR set | Use matching auth+shell+keyboard from **same** flavor/build; update to latest AARs |
| `NoClassDefFoundError` Lifecycle | Missing KTX deps                      | Add `lifecycle-runtime-ktx` / `lifecycle-viewmodel-ktx`                            |
| Crash on first key / JNI         | Wrong ABI                             | Use arm64 device or x86_64 emulator AAR build                                      |
| Typing works when logged out     | Old keyboard without auth gate        | Update to post–Phase 5 keyboard AAR                                                |
| R8 crash on release              | Missing keep rules                    | Apply consumer / host ProGuard keeps above                                         |
| Duplicate classes                | Two flavors’ AARs in one variant      | One flavor’s three AARs only                                                       |


---

# Support

- **Repo / PAT access:** Your ACS integration contact  
- **Reference projects:** `test-app-sdk` ( harness), `test-app_git` (v1 git-clone pattern)  
- **Internal build/install cheat sheet:** `sdk_Docs/BUILD_AND_INSTALL.md`  
- **This document (v2):** `sdk_Docs/PARTNER_INTEGRATION_GUIDE_v2.md` (+ HTML twin)

---

# Document history


| Version | Scope                                                                                                    |
| ------- | -------------------------------------------------------------------------------------------------------- |
| **v1**  | auth-only (`kslma` / `eplma` auth AARs)                                                                  |
| **v2**  | auth + shell + keyboard; partner flavor; SdkInitializer / KeyboardSdk; auth gate; branding; minify notes |


