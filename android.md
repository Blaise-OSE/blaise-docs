# Android SDK Integration Guide

How to integrate `BlaiseSDK` into an Android application. The SDK exposes a single
Kotlin class with one public method — discover the device, request USB permission,
open the connection, and acquire the RAW capture, all behind one call.

## Requirements

- Android `minSdk` 24+
- Kotlin coroutines on the consumer side (the SDK exposes a `Flow` and uses `suspend`
  internally)
- A USB host-capable Android device for `DeviceMode.HARDWARE`
- A **GitHub Personal Access Token** with the `read:packages` scope, so Gradle can pull
  the SDK from GitHub Packages

## Installation

Artifacts are published to GitHub Packages.

`settings.gradle.kts`:

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven {
            name = "GitHubPackages-Blaise"
            url = uri("https://maven.pkg.github.com/Forward-Edge-AI-Inc/Blaise.Android.Library")
            credentials {
                username = providers.gradleProperty("gpr.user")
                    .orElse(providers.environmentVariable("GITHUB_ACTOR")).get()
                password = providers.gradleProperty("gpr.token")
                    .orElse(providers.environmentVariable("GITHUB_TOKEN")).get()
            }
        }
    }
}
```

App-level `build.gradle.kts`:

```kotlin
dependencies {
    implementation("com.forwardedgeai.blaise:library:<version>")
}
```

### Configuring credentials

Do **not** commit literal credentials to a project-level `gradle.properties` — configure
them outside the project instead:

**Option A — user-level `~/.gradle/gradle.properties`:**

```properties
gpr.user=<your-github-login>
gpr.token=<your-PAT-with-read:packages>
```

**Option B — environment variables:**

```bash
export GITHUB_ACTOR=<your-github-login>
export GITHUB_TOKEN=<your-PAT-with-read:packages>
```

The GitHub Packages repository only handles the `com.forwardedgeai.blaise` group, so
missing credentials will not affect resolution of `androidx.*` / `google.*` artifacts.

## Quick start

Construct a `BlaiseSDK` with the desired mode and collect the flow returned by `scan()`:

```kotlin
import com.forwardedgeai.library.BlaiseSDK
import com.forwardedgeai.library.BlaiseSDK.ScanEvent
import com.forwardedgeai.library.DeviceMode

val sdk = BlaiseSDK(DeviceMode.HARDWARE)

sdk.scan(context).collect { event ->
    when (event) {
        ScanEvent.Connecting -> showPreparingUi()
        is ScanEvent.Capturing -> updateProgress(event.progress)
        is ScanEvent.Completed -> handleRawFile(event.rawPath)
    }
}
```

`BlaiseSDK` holds no state between scans — construct a new instance per scan, per
screen, or for the lifetime of the process; all work equivalently. `deviceMode` is fixed
at construction; to switch modes, construct a new instance.

A `DeviceMode.DEMO` mode is included so integrators can wire up the full flow without
owning physical hardware — it follows the exact same
`Connecting → Capturing → Completed` event shape as `HARDWARE`, using a bundled sample
capture instead of real device I/O.

## Event model

The flow always emits, in order, exactly one `Connecting`, one or more `Capturing`, and
exactly one terminal `Completed`. After `Completed` the flow completes.

| Event | Meaning |
|-------|---------|
| `Connecting` | Discovering the device, requesting USB permission, and opening / configuring the connection. Show a "preparing" indicator rather than a progress ring — there is no progress data yet, and the phase's duration is driven by the user answering the permission dialog. |
| `Capturing(progress: Float?)` | Active acquisition phase. `progress` is `0f..1f`, or `null` if not known. Emitted repeatedly while capture is in flight. |
| `Completed(rawPath: String)` | Terminal event. Carries the absolute path to the captured `.raw` file in the caller's cache directory. The flow completes immediately after. |

The path delivered in `Completed.rawPath` points to unprocessed pixel data — a 2D MIPI
RAW10 image of the diffraction pattern the sample's Raman-scattered light produces on
the device's grating. It is **not** a 1D spectrum or a classification result — decoding
and inference are the integrator's responsibility. `scan()` writes its output to
`context.cacheDir`; the OS may evict the file at any time, so copy or move it out before
relying on it long-term.

## Error handling

Exceptions are surfaced when the collector hits the failing emission. All exceptions are
Kotlin `object`s extending `RuntimeException` — there is no shared base exception type,
so catch the concrete ones you care about:

| Exception | Phase | Meaning |
|-----------|-------|---------|
| `BlaiseNoDevicesDiscoveredException` | `Connecting` | `UsbManager.deviceList` was empty (HARDWARE only). |
| `BlaiseDevicePermissionException` | `Connecting` | The user denied the USB permission prompt, or permission was revoked before connection setup completed (HARDWARE only). |
| `BlaiseUnableToConnectException` | `Connecting` / `Capturing` | Could not open the device, claim the required interface, or configure the line (HARDWARE only). |
| `BlaiseIOException` | `Capturing` | The device returned an unexpected response or a bulk transfer failed (HARDWARE only). |

```kotlin
import kotlinx.coroutines.CancellationException

try {
    BlaiseSDK(DeviceMode.HARDWARE).scan(context).collect { event ->
        when (event) {
            ScanEvent.Connecting -> showPreparing()
            is ScanEvent.Capturing -> updateProgress(event.progress)
            is ScanEvent.Completed -> handleRawFile(event.rawPath)
        }
    }
} catch (e: CancellationException) {
    throw e // cooperative cancellation must propagate
} catch (e: BlaiseNoDevicesDiscoveredException) {
    showError("Connect your Blaise device and try again.")
} catch (e: BlaiseDevicePermissionException) {
    showError("USB permission denied.")
} catch (e: BlaiseUnableToConnectException) {
    showError("Could not connect to the spectrometer.")
} catch (e: BlaiseIOException) {
    showError("Communication with the spectrometer failed.")
}
```

The flow is cold: cancelling the collecting coroutine before `Completed` cancels the
in-flight scan and releases the USB connection (device teardown runs in `try/finally`
regardless of success, failure, or cancellation). The one exception is the system USB
permission dialog — cancelling while it is on screen does not unregister the broadcast
receiver or resume the suspended coroutine.

## Reference implementation

A worked example — decoding the RAW capture and running on-device inference — ships in
the [blaise-applications](https://github.com/Blaise-OSE/blaise-applications) reference
Android sample.
