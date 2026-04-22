# Framework Integration Code Examples

This guide shows concrete examples for using an existing `.framework` or `.xcframework` in a new app target.

## Example App Layout

```text
MyApp/
├── MyApp.xcodeproj
├── MyApp/
│   ├── MyAppApp.swift
│   ├── ContentView.swift
│   ├── AppDelegate.swift
│   └── Features/
├── Frameworks/
│   ├── VendorSDK.framework
│   └── InternalSDK.xcframework
├── Config/
│   ├── Base.xcconfig
│   └── Release.xcconfig
└── Scripts/
    └── verify_embedded_frameworks.sh
```

## SwiftUI App Import Example

### `MyApp/MyAppApp.swift`

```swift
import SwiftUI
import VendorSDK

@main
struct MyAppApp: App {
    @StateObject private var model = AppModel()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environmentObject(model)
        }
    }
}
```

### `MyApp/AppModel.swift`

```swift
import Combine
import Foundation
import VendorSDK

@MainActor
final class AppModel: ObservableObject {
    @Published var status = "Not started"

    private let client: VendorClient

    init() {
        client = VendorClient(configuration: .init(apiKey: "demo-key"))
    }

    func loadStatus() async {
        do {
            let result = try await client.fetchStatus()
            status = "Loaded: \(result.description)"
        } catch {
            status = "Error: \(error)"
        }
    }
}
```

### `MyApp/ContentView.swift`

```swift
import SwiftUI

struct ContentView: View {
    @EnvironmentObject private var model: AppModel

    var body: some View {
        VStack(spacing: 12) {
            Text("Vendor SDK Demo")
                .font(.headline)

            Text(model.status)
                .font(.body.monospaced())

            Button("Refresh") {
                Task {
                    await model.loadStatus()
                }
            }
        }
        .padding()
        .task {
            await model.loadStatus()
        }
    }
}
```

## UIKit Import Example

### `MyApp/ViewController.swift`

```swift
import UIKit
import VendorSDK

final class ViewController: UIViewController {
    private let client = VendorClient(configuration: .init(apiKey: "demo-key"))
    private let label = UILabel()

    override func viewDidLoad() {
        super.viewDidLoad()

        view.backgroundColor = .systemBackground
        label.numberOfLines = 0
        label.translatesAutoresizingMaskIntoConstraints = false

        view.addSubview(label)

        NSLayoutConstraint.activate([
            label.leadingAnchor.constraint(equalTo: view.layoutMarginsGuide.leadingAnchor),
            label.trailingAnchor.constraint(equalTo: view.layoutMarginsGuide.trailingAnchor),
            label.centerYAnchor.constraint(equalTo: view.centerYAnchor)
        ])

        Task {
            await load()
        }
    }

    @MainActor
    private func load() async {
        do {
            let result = try await client.fetchStatus()
            label.text = "Framework response: \(result)"
        } catch {
            label.text = "Framework error: \(error)"
        }
    }
}
```

## Objective-C App Import Example

### `MyApp/AppDelegate.m`

```objc
#import "AppDelegate.h"
@import VendorSDK;

@implementation AppDelegate

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    VendorConfiguration *configuration = [[VendorConfiguration alloc] initWithAPIKey:@"demo-key"];
    VendorClient *client = [[VendorClient alloc] initWithConfiguration:configuration];

    [client fetchStatusWithCompletion:^(VendorStatus * _Nullable status, NSError * _Nullable error) {
        if (error) {
            NSLog(@"Framework error: %@", error);
            return;
        }

        NSLog(@"Framework status: %@", status);
    }];

    return YES;
}

@end
```

## App Target XCConfig Examples

### `Config/Base.xcconfig`

```xcconfig
FRAMEWORK_SEARCH_PATHS = $(inherited) $(PROJECT_DIR)/Frameworks
LIBRARY_SEARCH_PATHS = $(inherited)
LD_RUNPATH_SEARCH_PATHS = $(inherited) @executable_path/Frameworks @loader_path/Frameworks
OTHER_LDFLAGS = $(inherited)
ENABLE_USER_SCRIPT_SANDBOXING = YES
```

### Explicit linker flags if a vendor requires them

```xcconfig
OTHER_LDFLAGS = $(inherited) -framework VendorSDK -ObjC
```

Most of the time, Xcode handles the link entry automatically when the framework is added through the app target settings.

## Manual Build Phase Script Example

Use this only when you need an explicit verification step.

### `Scripts/verify_embedded_frameworks.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

APP_BUNDLE="${TARGET_BUILD_DIR}/${WRAPPER_NAME}"
FRAMEWORK_BINARY="${APP_BUNDLE}/Frameworks/VendorSDK.framework/VendorSDK"

if [[ ! -f "${FRAMEWORK_BINARY}" ]]; then
  echo "error: VendorSDK.framework is not embedded in ${APP_BUNDLE}"
  exit 1
fi

echo "Embedded framework verified at ${FRAMEWORK_BINARY}"
```

## Verify the Framework Was Embedded

### Show frameworks inside the built app bundle

```bash
find ~/Library/Developer/Xcode/DerivedData -path '*Build/Products/Debug-iphonesimulator/MyApp.app/Frameworks/*' -maxdepth 3
```

### Check the app binary's linked frameworks

```bash
otool -L ~/Library/Developer/Xcode/DerivedData/.../Build/Products/Debug-iphonesimulator/MyApp.app/MyApp
```

Example output:

```text
@rpath/VendorSDK.framework/VendorSDK
@rpath/InternalSDK.framework/InternalSDK
/System/Library/Frameworks/UIKit.framework/UIKit
```

### Check the framework architecture slices

```bash
lipo -info MyApp/Frameworks/VendorSDK.framework/VendorSDK
```

Example output:

```text
Architectures in the fat file: VendorSDK are: arm64 x86_64
```

### Check the framework code signature

```bash
codesign --verify --deep --strict --verbose=2 \
  ~/Library/Developer/Xcode/DerivedData/.../Build/Products/Debug-iphoneos/MyApp.app
```

### Check runpaths on the app binary

```bash
otool -l ~/Library/Developer/Xcode/DerivedData/.../Build/Products/Debug-iphonesimulator/MyApp.app/MyApp | rg -n 'LC_RPATH|path'
```

## Using a Local Framework Target Instead of Dragging Files

If the framework comes from another local project, a workspace-style setup is cleaner.

### Build the app from a workspace

```bash
xcodebuild \
  -workspace MyWorkspace.xcworkspace \
  -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 16' \
  build
```

### Reference the framework target as a dependency

```text
MyApp target
├── Target Dependencies
│   └── MyLibrary.framework
└── Frameworks, Libraries, and Embedded Content
    └── MyLibrary.framework (Embed & Sign)
```

## Example Runtime Error and Fix

### Error

```text
dyld[12345]: Library not loaded: @rpath/VendorSDK.framework/VendorSDK
  Referenced from: /.../MyApp.app/MyApp
  Reason: tried: '/.../MyApp.app/Frameworks/VendorSDK.framework/VendorSDK' (no such file)
```

### What usually fixes it

```text
1. Add VendorSDK.framework under Frameworks, Libraries, and Embedded Content.
2. Set the entry to Embed & Sign.
3. Rebuild the app.
4. Verify the file exists under MyApp.app/Frameworks/.
```

## Example Static vs Dynamic Handling

### Dynamic framework

```text
Link: Yes
Embed: Yes
Signing: Yes
```

### Static framework

```text
Link: Yes
Embed: No
Signing: Not applicable as a copied runtime bundle
```

If you embed a static framework as if it were dynamic, you usually create confusion rather than a working app bundle.

## Example Post-Build Inspection Commands

### Show product settings

```bash
xcodebuild \
  -project MyApp.xcodeproj \
  -scheme MyApp \
  -showBuildSettings | rg 'FRAMEWORK_SEARCH_PATHS|LD_RUNPATH_SEARCH_PATHS|TARGET_BUILD_DIR|WRAPPER_NAME'
```

### Show framework contents

```bash
find MyApp/Frameworks/VendorSDK.framework -maxdepth 3 -print
```

### Inspect a binary inside the framework

```bash
file MyApp/Frameworks/VendorSDK.framework/VendorSDK
```

### Inspect linked libraries of the framework itself

```bash
otool -L MyApp/Frameworks/VendorSDK.framework/VendorSDK
```

## Local `xcframework` Consumption Example

If the vendor gives you an `.xcframework`, keep it in the same top-level folder:

```text
MyApp/
└── Frameworks/
    └── VendorSDK.xcframework
```

Xcode resolves the correct slice when the product is added to the app target.

## Practical Pattern

If you want the least fragile setup:

- keep third-party binaries under `Frameworks/`
- prefer `.xcframework` over `.framework`
- let Xcode manage linking through target settings
- use `Embed & Sign` only for dynamic frameworks
- verify the final `.app` bundle with `find`, `otool`, `lipo`, and `codesign`
