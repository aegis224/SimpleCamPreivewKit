# SimpleCamPreviewKit

Swift XCFramework simplified usage of camera preview

## Features
- Querying Camera Device by Camera Resolution, Position, and Type
- Preview of Camera Streaming based on SwiftUI View

## Notes
- Streamed Image type is CIImage, which Pixel Format is 32 BGRA

## Supported Platform
- iOS 13.0
- iPadOS 13.0
- macOS 12.4

## Installation
To install SimpleCamPreviewKit using [Swift Package Manager](https://github.com/apple/swift-package-manager) you can follow the [tutorial published by Apple](https://developer.apple.com/documentation/xcode/adding_package_dependencies_to_your_app) using the URL for this repo with the current version:

1. In Xcode, select “File” → “Add Packages...”
1. Enter https://github.com/aegis224/SimpleCamPreviewKit.git

or you can add the following dependency to your `Package.swift`:

```swift
.package(url: "https://github.com/aegis224/SimpleCamPreviewKit.git", branch: "main")
```

## How to use
### 0. Config info.plist
As this framework using camera, you have to add `NSCameraUsageDescription` key (which will be shown as `Privacy - Camera Usage Description` in Xcode) and write reason for using camera.
### 1. Querying Camera
Before initializing or running camera, please check following two conditions
1. Authorization for access camera
```swift
import AVFoundation

// Check Authorization
let authorization = AVCaptureDevice.authorizationStatus(for: .video)
switch authorization {
	case .authorized:
		break
	case .notDetermined:
		// Ask user to allow access for camera
	default:
		// Ask user to allow access for camera
}
```
2. Camera Specification
	- Resolution
		- Frequently Used Resolution (height x width)
			- VGA (640 x 480)
			- HD (1280 x 720)
			- 1440 x 1080
			- FHD (1920 x 1080)
	- Max FPS
		- Frequently Used FPS
			- 24
			- 30
			- 60
	- Position
		- Front
		- Back
		- External (will be used for macOS Continuity Camera)
	- Camera Device Type
		- Normal
		- UltraWideAngle
			- Front/Back Camera in some iPad Series
			- Back Camera in some iPhone Series
```swift
// Check Existence of Camera that support following specifications
import AVFoundation

let position: AVCaptureDevice.Position = .front
let width: Int32 = 1080
let height: Int32 = 1920
let maxFPS: Int = 60
// For ultra wide angle camera, use .builtInUltraWideCamera
let deviceType: AVCaptureDevice.DeviceType = .builtInWideAngleCamera 

let cameraSpec: CameraSpec = .init(
	position: position,
	width: width,
	height: height,
	maxFPS: maxFPS,
	deviceType: deviceType
)

guard cameraSpec.isAvailable
else {
	// Camera is not found for supporting above cameraSpec
}

```

### 2. Examples
```swift
import SimpleCamPreviewKit
import SwiftUI

// SwiftUI View where using CameraView
struct ExampleView {
	@StateObject private var cameraViewModel: CameraViewModel

	init(cameraSpec: CameraSpec, orientation: Orientation) {
		let cameraViewModel = CameraViewModel(cameraSpec: cameraSpec, orientation: orientation)
		self._cameraViewModel = .init(wrappedValue: cameraViewModel)
	}

	var body: some View {
		CameraView(image: cameraViewModel.currentFrame)
			.task(priority: .background) {
				do {
					try await cameraViewModel.configCamera()
				} catch {
					// Handle error which Error Type is CameraError
				}
			}
			.onDisappear {
				cameraViewModel.terminate()
			}
	}
}
```

### 3. Subscribe to per frame event
CameraViewModel has following getter property, which will be usefule when you want to execute extra job per frame.
```swift
var imageStream: AnyPubliser<CIImage, Never> { get }
```

Assume there's other class has to execute something per frame. (For example, real-time AI image processing)
```swift
import Combine
import CoreImage

final class OtherManager: ObservableObject {
	private var cancellableBags: Set<AnyCancellable> = .init()

	func inject(imageStream: AnyPublisher<CIImage, Never>) {
		imageStream
			.sink { perFrameEvent($0) }
			.store(in: &cancellableBags)
	}

	// Per Frame Event Function
	private func perFrameEvent(newFrame: CIImage) {
		// TO-DO Your own event handler
	}

	func terminate() {
		cancellableBags.forEach { $0.cancel() }
	}
}
```

So extend to 2's Example
```swift
import SimpleCamPreviewKit
import SwiftUI

// SwiftUI View where using CameraView
struct ExampleView {
	@StateObject private var cameraViewModel: CameraViewModel
	@StateObject private var otherManager: OtherManager = .init()

	init(cameraSpec: CameraSpec, orientation: Orientation) {
		let cameraViewModel = CameraViewModel(cameraSpec: cameraSpec, orientation: orientation)
		self._cameraViewModel = .init(wrappedValue: cameraViewModel)
	}

	var body: some View {
		CameraView(image: cameraViewModel.currentFrame)
			.task(priority: .background) {
				do {
					try await cameraViewModel.configCamera()
					otherManager.inject(imageStream: cameraViewModel.imageStream)
				} catch {
					// Handle error which Error Type is CameraError
				}
			}
			.onDisappear {
				cameraViewModel.terminate()
				otherManager.terminate()
			}
	}
}
```

### 4. Error
CamerViewModelError is error which you may meet using  CameraViewModel's `config()` method.
```swift
CameraViewModelError: Error {
	// Will be thrown out when the app is not authorized for camera access.
	// So please check camera authorization
	case unAuthorized
	// Will be thrown out when you config with unsupported CameraSpec.
	// So please check camera compatibility using CameraSpec.isAvailable
	case cameraIsNotConfigured
	// Will be thrown out when you call CameraViewModel typed Instance's config() more than one time
	case cameraIsAlreadyConfigured
	// Can be thrown out CameraViewModel typed Instance's config() for other reasons with localized error
	case other(LocalizedError)
	// Can be thrown out CameraViewModel typed Instance's config() for unknown reasons
	case unknown
}
```

## License
Please Refer to LICENSE.md.
