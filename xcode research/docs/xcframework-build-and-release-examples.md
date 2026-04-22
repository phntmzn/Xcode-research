# XCFramework Build and Release Examples

This guide focuses on packaging a framework for reuse as an `.xcframework`.

## Example Output Layout

```text
build/
├── MyLibrary-iOS.xcarchive
├── MyLibrary-Sim.xcarchive
├── MyLibrary-macOS.xcarchive
├── MyLibrary.xcframework
└── MyLibrary.xcframework.zip
```

## Archive Commands

### iOS device archive

```bash
xcodebuild archive \
  -project MyLibrary.xcodeproj \
  -scheme MyLibrary \
  -configuration Release \
  -sdk iphoneos \
  -archivePath build/MyLibrary-iOS \
  SKIP_INSTALL=NO \
  BUILD_LIBRARY_FOR_DISTRIBUTION=YES
```

### iOS simulator archive

```bash
xcodebuild archive \
  -project MyLibrary.xcodeproj \
  -scheme MyLibrary \
  -configuration Release \
  -sdk iphonesimulator \
  -archivePath build/MyLibrary-Sim \
  SKIP_INSTALL=NO \
  BUILD_LIBRARY_FOR_DISTRIBUTION=YES
```

### macOS archive

```bash
xcodebuild archive \
  -project MyLibrary.xcodeproj \
  -scheme MyLibrary \
  -configuration Release \
  -sdk macosx \
  -archivePath build/MyLibrary-macOS \
  SKIP_INSTALL=NO \
  BUILD_LIBRARY_FOR_DISTRIBUTION=YES
```

## Create the XCFramework

```bash
xcodebuild -create-xcframework \
  -framework build/MyLibrary-iOS.xcarchive/Products/Library/Frameworks/MyLibrary.framework \
  -debug-symbols build/MyLibrary-iOS.xcarchive/dSYMs/MyLibrary.framework.dSYM \
  -framework build/MyLibrary-Sim.xcarchive/Products/Library/Frameworks/MyLibrary.framework \
  -debug-symbols build/MyLibrary-Sim.xcarchive/dSYMs/MyLibrary.framework.dSYM \
  -framework build/MyLibrary-macOS.xcarchive/Products/Library/Frameworks/MyLibrary.framework \
  -debug-symbols build/MyLibrary-macOS.xcarchive/dSYMs/MyLibrary.framework.dSYM \
  -output build/MyLibrary.xcframework
```

## Full Build Script Example

### `Scripts/release_xcframework.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

SCHEME="MyLibrary"
PROJECT="MyLibrary.xcodeproj"
CONFIGURATION="Release"
BUILD_DIR="$(pwd)/build"
OUTPUT_NAME="MyLibrary"

rm -rf "${BUILD_DIR}"
mkdir -p "${BUILD_DIR}"

archive() {
  local sdk="$1"
  local archive_path="$2"

  xcodebuild archive \
    -project "${PROJECT}" \
    -scheme "${SCHEME}" \
    -configuration "${CONFIGURATION}" \
    -sdk "${sdk}" \
    -archivePath "${archive_path}" \
    SKIP_INSTALL=NO \
    BUILD_LIBRARY_FOR_DISTRIBUTION=YES
}

archive iphoneos "${BUILD_DIR}/${OUTPUT_NAME}-iOS"
archive iphonesimulator "${BUILD_DIR}/${OUTPUT_NAME}-Sim"
archive macosx "${BUILD_DIR}/${OUTPUT_NAME}-macOS"

xcodebuild -create-xcframework \
  -framework "${BUILD_DIR}/${OUTPUT_NAME}-iOS.xcarchive/Products/Library/Frameworks/${OUTPUT_NAME}.framework" \
  -debug-symbols "${BUILD_DIR}/${OUTPUT_NAME}-iOS.xcarchive/dSYMs/${OUTPUT_NAME}.framework.dSYM" \
  -framework "${BUILD_DIR}/${OUTPUT_NAME}-Sim.xcarchive/Products/Library/Frameworks/${OUTPUT_NAME}.framework" \
  -debug-symbols "${BUILD_DIR}/${OUTPUT_NAME}-Sim.xcarchive/dSYMs/${OUTPUT_NAME}.framework.dSYM" \
  -framework "${BUILD_DIR}/${OUTPUT_NAME}-macOS.xcarchive/Products/Library/Frameworks/${OUTPUT_NAME}.framework" \
  -debug-symbols "${BUILD_DIR}/${OUTPUT_NAME}-macOS.xcarchive/dSYMs/${OUTPUT_NAME}.framework.dSYM" \
  -output "${BUILD_DIR}/${OUTPUT_NAME}.xcframework"

(
  cd "${BUILD_DIR}"
  ditto -c -k --sequesterRsrc --keepParent "${OUTPUT_NAME}.xcframework" "${OUTPUT_NAME}.xcframework.zip"
)

echo "Created ${BUILD_DIR}/${OUTPUT_NAME}.xcframework"
echo "Created ${BUILD_DIR}/${OUTPUT_NAME}.xcframework.zip"
```

## Verify the XCFramework Contents

### Show all files inside the packaged binary

```bash
find build/MyLibrary.xcframework -maxdepth 4 -print
```

### Print the package manifest

```bash
plutil -p build/MyLibrary.xcframework/Info.plist
```

### Inspect slices in a contained framework binary

```bash
lipo -info build/MyLibrary.xcframework/ios-arm64/MyLibrary.framework/MyLibrary
```

### Inspect linked libraries

```bash
otool -L build/MyLibrary.xcframework/ios-arm64/MyLibrary.framework/MyLibrary
```

### Verify code signing details

```bash
codesign -dv --verbose=4 build/MyLibrary.xcframework/ios-arm64/MyLibrary.framework 2>&1
```

## Consume the XCFramework from Swift Package Manager

### Create a checksum for the zipped artifact

```bash
swift package compute-checksum build/MyLibrary.xcframework.zip
```

### `Package.swift` with a remote binary target

```swift
// swift-tools-version: 5.10
import PackageDescription

let package = Package(
    name: "MyLibraryWrapper",
    platforms: [
        .iOS(.v16),
        .macOS(.v13)
    ],
    products: [
        .library(name: "MyLibraryWrapper", targets: ["MyLibraryWrapper"])
    ],
    targets: [
        .binaryTarget(
            name: "MyLibrary",
            url: "https://example.com/releases/MyLibrary.xcframework.zip",
            checksum: "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
        ),
        .target(
            name: "MyLibraryWrapper",
            dependencies: ["MyLibrary"]
        )
    ]
)
```

### `Package.swift` with a local binary target

```swift
// swift-tools-version: 5.10
import PackageDescription

let package = Package(
    name: "LocalBinaryDemo",
    targets: [
        .binaryTarget(
            name: "MyLibrary",
            path: "./Frameworks/MyLibrary.xcframework"
        ),
        .executableTarget(
            name: "LocalBinaryDemo",
            dependencies: ["MyLibrary"]
        )
    ]
)
```

## CI Example

### `.github/workflows/release-xcframework.yml`

```yaml
name: Release XCFramework

on:
  workflow_dispatch:
  push:
    tags:
      - 'v*'

jobs:
  build:
    runs-on: macos-14

    steps:
      - uses: actions/checkout@v4

      - name: Select Xcode
        run: sudo xcode-select -s /Applications/Xcode.app

      - name: Build XCFramework
        run: bash Scripts/release_xcframework.sh

      - name: Show output
        run: find build -maxdepth 3 -print

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: MyLibrary-xcframework
          path: build/MyLibrary.xcframework.zip
```

## Distribution Notes as Code

### Stable build settings

```xcconfig
SKIP_INSTALL = NO
BUILD_LIBRARY_FOR_DISTRIBUTION = YES
DEFINES_MODULE = YES
APPLICATION_EXTENSION_API_ONLY = YES
```

### Clean version metadata

```xcconfig
MARKETING_VERSION = 1.2.0
CURRENT_PROJECT_VERSION = 120
DYLIB_CURRENT_VERSION = 1.2.0
DYLIB_COMPATIBILITY_VERSION = 1.0.0
```

## Post-Release Smoke Test

### Create a temporary app build against the packaged artifact

```bash
xcodebuild \
  -project ExampleConsumer.xcodeproj \
  -scheme ExampleConsumer \
  -destination 'platform=iOS Simulator,name=iPhone 16' \
  build
```

### Inspect the embedded packaged binary in the consumer app

```bash
find ~/Library/Developer/Xcode/DerivedData -path '*ExampleConsumer.app/Frameworks/MyLibrary.framework/*' -maxdepth 2 -print
```

## Common Failures

### Simulator slice missing

```text
Could not find module 'MyLibrary' for target 'x86_64-apple-ios-simulator'
```

### Binary interface not built for distribution

```text
Module compiled with Swift 5.x cannot be imported by the Swift 5.y compiler
```

### Runtime embed failure

```text
Library not loaded: @rpath/MyLibrary.framework/MyLibrary
```

## Practical Pattern

If you want a durable release process:

1. Archive per SDK.
2. Build an `.xcframework`.
3. Zip it.
4. Compute a checksum if SwiftPM is involved.
5. Smoke-test a consumer app against the packaged artifact before calling the release done.
