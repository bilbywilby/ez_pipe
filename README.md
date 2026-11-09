## ADB Wi-Fi 2.0 (New in Android 17 / Platform Tools 37)

The old wireless debugging was notoriously flaky. Android 17 + SDK Platform Tools **37.0.0** (Feb 2026) replaced the entire stack with a custom ~4,000-line **Rust mDNS library** (`libadbmdns`), replacing both Bonjour and legacy mDNS:

| Old behavior | ADB Wi-Fi 2.0 |
|---|---|
| Connection drops on network change or device sleep | Stays connected across network changes and reboots |
| Manual IP + port pairing every time | **Auto-discovery** via mDNS; device appears in Android Studio's Device Manager |
| No network trust awareness | Daemon **auto-disables** on untrusted networks, re-enables on trusted ones (SSID + BSSID matching) |
| QR code pairing could stall indefinitely | Live list of discoverable local devices in the pairing dialog |

**Measured improvements:** 32% higher auto-connection success rate, 66% faster connections (90th percentile).

### Setup (replaces the old `adb pair` / `adb connect` flow)

1. Update to **Platform Tools 37.0.0+** and **Android Studio Quail 3+**
2. On the Pixel: **Settings → System → Developer options → Wireless debugging** (toggle on)
3. The device **automatically appears** in Android Studio's **Device Manager**
4. Click it → scan the QR code or enter the pairing code
5. Done — no `adb pair`, no `adb connect`, no IP hunting

### CLI equivalent (if you prefer the terminal)

```bash
# The old flow still works, but auto-discovery is now the default:
adb devices          # wireless device should appear automatically

# If you still want to connect manually:
adb connect <ip>:<port>
```

### Security note: CVE-2026-0073

A critical vulnerability was discovered where the **Wireless ADB authentication process could be fully bypassed**. A follow-up feature lets developers choose which network interface `adbd` listens on, reducing exposure. If you're on an untrusted network, ADB Wi-Fi 2.0 now **automatically disables itself**.

---

## 2026 Play Store Requirements (Now in Effect)

As of today (Nov 9, 2026), these rules are **active**:

| Requirement | Deadline | Status |
|---|---|---|
| **Target API 36** (Android 16) for new apps & updates | Aug 31, 2026 | **Enforced** (extension to Nov 1 available for eligible apps) |
| **16 KB page-size** support for NDK apps on 64-bit devices | Nov 1, 2025 | **Enforced** |
| **Play App Signing** (Google holds the app signing key) | Default since Aug 2021 | **Standard** |
| **AAB** (not APK) for new Play submissions | Ongoing | **Required** |
| **Developer Verification** (Brazil, Indonesia, Singapore, Thailand) | Sep 30, 2026 | **Enforced in those 4 countries** |
| **Developer Verification** (global, including US) | 2027 | **Upcoming** |

### What Target API 36 Actually Changes

```kotlin
// build.gradle.kts
android {
    defaultConfig {
        targetSdk = 36  // was 35 before Aug 2026
    }
}
```

- **Large screens** (smallest width ≥ 600dp): all orientations, all aspect ratios, and full resizability are now **baseline**. Legacy `screenOrientation`, `resizeableActivity=false`, and aspect ratio constraints are ignored.
- **Compose-first**: Jetpack Compose is now the standard. View toolkit (`android.widget.*`, Fragments, RecyclerView, ViewPager) is in **maintenance mode** — critical bug fixes only, no new features.
- **16 KB page size**: If your app uses NDK native libraries, they must be rebuilt with 16 KB alignment. Check with:
  ```bash
  # Check if a .so file is 16KB-aligned
  readelf -l libfoo.so | grep -A1 "LOAD"
  # Page sizes should show 0x4000 (16384), not 0x1000 (4096)
  ```

---

## CI/CD Pipeline (GitHub Actions + Fastlane)

The standard 2026 pipeline for a Pixel-targeted app:

```yaml
# .github/workflows/android-ci.yml
name: Android CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'

      - name: Decode keystore
        run: |
          echo "${{ secrets.ANDROID_KEYSTORE_BASE64 }}" | base64 -d > release.keystore

      - name: Grant execute permission for gradlew
        run: chmod +x gradlew

      - name: Build AAB
        run: ./gradlew bundleRelease

      - name: Build APK (for internal QA / sideload)
        run: ./gradlew assembleRelease

      - name: Upload AAB artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-release.aab
          path: app/build/outputs/bundle/release/app-release.aab

      - name: Upload APK artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-release.apk
          path: app/build/outputs/apk/release/app-release.apk

      - name: Run unit tests
        run: ./gradlew testRelease

      - name: Run lint
        run: ./gradlew lintRelease
        continue-on-error: true

  deploy-internal:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-ruby@v1
        with:
          ruby-version: '3.3'
      - name: Setup Fastlane
        run: bundle install
      - name: Upload to Play Internal Track
        run: bundle exec fastlane android internal
        env:
          ANDROID_KEYSTORE_BASE64: ${{ secrets.ANDROID_KEYSTORE_BASE64 }}
          ANDROID_KEYSTORE_PASSWORD: ${{ secrets.ANDROID_KEYSTORE_PASSWORD }}
          ANDROID_KEY_ALIAS: ${{ secrets.ANDROID_KEY_ALIAS }}
          ANDROID_KEY_PASSWORD: ${{ secrets.ANDROID_KEY_PASSWORD }}
          PLAY_STORE_SERVICE_ACCOUNT_JSON: ${{ secrets.PLAY_STORE_SERVICE_ACCOUNT_JSON }}
```

### Fastlane `Fastfile`

```ruby
platform :android do
  desc "Build and upload to Play internal track"
  lane :internal do
    gradle(task: "bundleRelease")

    upload_to_play_store(
      track: "internal",
      aab: "app/build/outputs/bundle/release/app-release.aab",
      json_key_data: ENV['PLAY_STORE_SERVICE_ACCOUNT_JSON'],
      skip_upload_metadata: false,
      skip_upload_images: false
    )
  end

  desc "Promote internal → closed → production"
  lane :promote do |options|
    promote_play_track(
      from_track: options[:from] || "internal",
      to_track:   options[:to]   || "production"
    )
  end

  desc "Generate release notes from commit log"
  lane :changelog do
    write_changelog_to_gitHub(
      version_number: "1.0.0",
      git_hub_owner: "yourorg",
      git_hub_repo: "yourapp"
    )
  end
end
```

### Key CI/CD Best Practices (2026)

- **Cache Gradle dependencies** keyed on `build.gradle` hash — cuts cold builds from ~15 min to ~4 min
- **Never commit keystores** — store as base64 in CI secrets or a KMS (AWS Secrets Manager, HashiCorp Vault, GCP KMS)
- **Use Play App Signing** — Google holds the production key; you only manage the upload key (recoverable via Play Console support)
- **Produce both AAB and APK** in the same pipeline — AAB for Play, APK for internal QA / sideload / MDM
- **Run E2E tests only on merge to main**, not every PR
- **Upload `mapping.txt`** (R8/ProGuard) so release crashes are deobfuscable

---

## Advanced Debugging on a Pixel

### Logcat Mastery

```bash
# Filter by package + log level (W = warning and above)
adb logcat -p W -s "MyApp:*" "AndroidRuntime:*"

# Filter by PID
adb shell pidof com.myapp
adb logcat --pid=$(adb shell pidof com.myapp)

# Capture to a file (useful for bug reports)
adb logcat -d > logcat_$(date +%Y%m%d_%H%M%S).txt

# Watch only crashes
adb logcat -b crash

# Clear the buffer first, then capture
adb logcat -c
# ... reproduce the bug ...
adb logcat -d > crash_log.txt

# Filter by tag AND message pattern
adb logcat | grep -E "MyApp|FATAL"
```

### Performance Profiling

```bash
# Systrace (CPU, GPU, binder, memory)
adb shell perfetto -o /data/misc/perfetto-traces/trace.perfetto \
  -t 10s -b 64mb \
  sched freq idle am wm gfx view binder_driver hal dalvik io

adb pull /data/misc/perfetto-traces/trace.perfetto
# Open at https://ui.perfetto.dev

# Memory info for a specific app
adb shell dumpsys meminfo com.myapp

# CPU usage
adb shell top -n 1 | grep myapp

# GPU rendering mode (for frame drops)
adb shell setprop debug.hwui.show_dirty_rect true
adb shell setprop debug.hwui.profile true
adb logcat -s "HwUI"
```

### Screenshot & Screen Recording

```bash
# Screenshot
adb shell screencap -p /sdcard/screen.png
adb pull /sdcard/screen.png

# 10-second screen recording
adb shell screenrecord --time-limit 10 /sdcard/recording.mp4
adb pull /sdcard/recording.mp4
```

### Network Inspection

```bash
# Enable verbose HTTP logging (app must be debuggable)
adb shell setprop http.debuglevel 5

# Capture traffic (requires a proxy like mitmproxy on your PC)
adb shell settings put global http_proxy 192.168.1.100:8080

# Disable proxy when done
adb shell settings put global http_proxy :0
```

### Force-State Manipulation

```bash
# Force an activity to a specific state
adb shell am start -n com.myapp/.MainActivity
adb shell am force-stop com.myapp

# Simulate low battery
adb shell dumpsys battery set level 5
adb shell dumpsys battery reset

# Simulate airplane mode
adb shell am broadcast -a android.intent.action.AIRPLANE_MODE --ez state true

# Change locale
adb shell am broadcast -a com.myapp.LOCALE_CHANGED --es locale "fr-FR"
```

---

## APK Size Optimization

```bash
# Check APK size breakdown
aapt dump resources app-release.apk | head -50

# Check DEX size
aapt dump dexdump app-release.apk | grep "Class #0"

# Check native lib sizes
unzip -l app-release.apk | grep "\.so"
```

### Gradle Config (maximize size reduction)

```kotlin
// app/build.gradle.kts
android {
    buildTypes {
        release {
            isMinifyEnabled = true          // R8 / ProGuard
            isShrinkResources = true        // Remove unused resources
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")

            // R8 full mode (default since AGP 8.0)
            // For even more aggressive shrinking:
            // isOptimizeEnabled = true      // AGP 8.4+
        }
    }

    // Only ship the ABIs you need
    defaultConfig {
        ndk {
            abiFilters += listOf("arm64-v8a")  // Pixels are arm64
        }
    }

    // Compress native libs
    packaging {
        jniLibs {
            useLegacyPackaging = false  // Compress .so files
        }
    }
}
```

### R8 / ProGuard Rules (common gotchas)

```proguard
# Keep serialization classes
-keep class com.myapp.model.** { *; }

# Keep reflection targets
-keepclassmembers class * {
    @com.google.gson.annotations.SerializedName <fields>;
}

# Keep native method signatures
-keepclasseswithmembernames class * {
    native <methods>;
}
```

---

## Reverse Engineering / Decompiling (For Analysis)

```bash
# Decompile an APK to Java (smali)
jadx -d output_dir/ app.apk

# View the manifest
aapt dump xmltree app.apk AndroidManifest.xml

# Extract native libs
unzip app.apk "lib/*" -d extracted/

# Check for hardcoded secrets (quick and dirty)
strings app.apk | grep -iE "api[_-]?key|secret|password|token"

# Check exported activities (security audit)
aapt dump xmltree app.apk AndroidManifest.xml | grep -A5 "activity"
```

> **Legal note:** Decompiling for security research or interoperability is generally fair use, but redistributing decompiled code is not.

---

## Quick-Reference: 2026 Dev Loop

```bash
# 1. Build (target API 36, Compose, 16KB-aligned)
./gradlew bundleRelease assembleRelease

# 2. Install to Pixel (wireless, no cable needed with ADB Wi-Fi 2.0)
adb install -r -g app/build/outputs/apk/release/app-release.apk

# 3. Profile
adb shell perfetto -o /data/misc/perfetto-traces/t.perfetto -t 10s -b 64mb sched gfx binder_driver
adb pull /data/misc/perfetto-traces/t.perfetto

# 4. Ship to Play (via CI)
bundle exec fastlane android internal
bundle exec fastlane android promote from:internal to:production
'''


