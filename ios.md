# iOS SDK Integration Guide

How to integrate `BlaiseSDK` into an iOS application. The SDK exposes a `Blaise`
namespace: construct a `Blaise.Client` for the desired device mode and consume its
`scan()` as an `AsyncThrowingStream`.

## Requirements

- iOS 16.0 or later
- Xcode 15 or later
- CocoaPods

## Installation

The SDK is distributed via CocoaPods, pulled directly from its repository by tag.

Add it to your `Podfile`:

```ruby
pod 'BlaiseSDK', :git => 'https://github.com/Forward-Edge-AI-Inc/Blaise.iOS.SDK.git', :tag => 'v0.0.4'
```

Then install:

```bash
pod install
```

Open the generated `.xcworkspace` (**not** the `.xcodeproj`) — opening the project alone
leaves the pods, and `import BlaiseSDK`, unresolved. CocoaPods authenticates against this
private repository using your local git credentials.

## Quick start

Construct a `Blaise.Client` for the desired mode and iterate the scan stream:

```swift
import BlaiseSDK

// Verify the SDK is wired up correctly.
print("Blaise SDK \(Blaise.version)")

// Construct a client for the desired mode (fixed at construction).
let client = Blaise.Client(mode: .demo)   // or .hardware

do {
    for try await event in client.scan() {
        switch event {
        case .connecting:              print("connecting…")
        case .capturing(let progress): print("capturing \(progress ?? 0)")
        case .completed(let rawPath):  print("captured RAW at \(rawPath)")
        }
    }
} catch {
    print("Scan failed: \(error)")
}
```

`scan()` is the transport: it streams the spectrometer's RAW bytes into a file and hands
back the path via `.completed(rawPath:)`. Parsing that capture and running inference on
it is the host app's job — the SDK never decides PASS/FAIL.

A client's mode is fixed at construction; each call to `scan()` creates a fresh stream.
`.demo` mode simulates acquisition by writing a synthesized capture to a temporary file,
so integrators can build the full flow without owning physical hardware — it follows the
same `connecting → capturing → completed` event shape as `.hardware`.

Cancelling the consuming `Task` cancels the scan: `for try await` ties the stream's
lifetime to the surrounding task, so leaving the screen (e.g. in SwiftUI, the view's
`.task` disappearing) cancels the in-flight scan through the stream's `onTermination`.

## API surface

```swift
public enum Blaise {
    public static let version: String

    public enum DeviceMode {
        case demo                       // synthesized capture, runs anywhere
        case hardware                   // real device
    }

    public final class Client {
        public init(mode: DeviceMode)   // mode fixed at construction
        public func scan() -> AsyncThrowingStream<ScanEvent, Swift.Error>
    }

    public enum ScanEvent: Equatable {
        case connecting                     // pre-capture (discover/permission/open)
        case capturing(progress: Double?)   // 0…1, repeated
        case completed(rawPath: String)     // path to the captured .raw; stream finishes after
    }

    public enum Error: Swift.Error, Equatable {
        case spectrometerUnreachable
        case cancelled
        case internalError(String)
    }
}
```

## Event model

| Event | Meaning |
|-------|---------|
| `.connecting` | Pre-capture phase — discovering the device, requesting permission, opening the connection. No progress data yet. |
| `.capturing(progress:)` | Active acquisition phase. `progress` is `0…1`, or `nil` if not known. Emitted repeatedly while capture is in flight. |
| `.completed(rawPath:)` | Terminal event. Carries the path to the captured `.raw` file. The stream finishes immediately after. |

The path delivered in `.completed(rawPath:)` points to unprocessed RAW pixel data, not a
decoded spectrum or a classification result — decoding and inference are the
integrator's responsibility.

## Error handling

`scan()` throws its errors **into the stream**, so a single `do/catch` around the
`for try await` loop covers them:

```swift
do {
    for try await event in client.scan() {
        switch event {
        case .connecting:             isPreparing = true
        case .capturing(let p):       if let p { progress = p }
        case .completed(let rawPath): finish(with: process(rawPath: rawPath))
        }
    }
} catch is CancellationError {
    // view went away / user cancelled — nothing to surface
} catch let error as Blaise.Error {
    switch error {
    case .spectrometerUnreachable: errorMessage = "Couldn't reach the spectrometer…"
    case .internalError(let detail): errorMessage = "Internal error: \(detail)"
    case .cancelled: break          // silent
    }
}
```

## Versioning

Semantic versioning. Pre-kit-ship internal builds use `0.x.y`; the kit-ship release is
`1.0.0`.

## Reference implementation

A worked example — decoding the RAW capture and running on-device inference with
TensorFlow Lite — ships in the
[blaise-applications](https://github.com/Blaise-OSE/blaise-applications) reference iOS
sample.
