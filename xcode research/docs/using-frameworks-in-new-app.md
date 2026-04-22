# Using `.framework` Bundles in a New App

This guide explains how to add an existing `.framework` to a new Xcode app and make sure it links and loads correctly at runtime.

## Prefer XCFramework When Possible

If you control the binary you are importing:

- Prefer `.xcframework` for modern distribution.
- Use `.framework` only when you know the binary matches the app's platform, architecture, and build context.

A plain `.framework` often fails when the app switches between simulator and device if the bundle was built for only one slice.

## Basic Integration Flow

To use a prebuilt framework in a new app:

1. Add the `.framework` to the Xcode project.
2. Link it to the app target.
3. Embed and sign it if it is a dynamic framework.
4. Import the module in app code.
5. Run the app and verify it launches without a loader error.

## Adding the Framework to the Project

In Xcode:

1. Open the app project.
2. Drag the `.framework` into the Project Navigator.
3. Choose to copy items if you want the framework stored inside the app project directory.
4. Make sure the app target is selected in the add dialog.

A practical folder layout is:

```text
MyApp/
├── MyApp.xcodeproj
├── MyApp/
├── Frameworks/
│   └── VendorSDK.framework
└── Resources/
```

Keep third-party binaries in a dedicated `Frameworks/` folder so the dependency boundary stays visible.

## Link the Framework

After the file is added:

1. Select the app target.
2. Open `General`.
3. Under `Frameworks, Libraries, and Embedded Content`, add the framework if it is not already present.

For a dynamic framework, the usual setting is:

- `Embed & Sign`

For a static framework, the usual setting is:

- `Do Not Embed`

If you are unsure whether the framework is static or dynamic, inspect how it was built before forcing embed settings.

## Import and Use It

Swift:

```swift
import VendorSDK
```

Objective-C:

```objc
@import VendorSDK;
```

If module import fails:

- confirm `Defines Module` was enabled in the framework build
- confirm the module name matches the import name
- confirm the framework actually contains public headers or Swift module metadata

## Runtime Loading Rules

Linking is not enough for dynamic frameworks. The app also needs the framework embedded in its bundle.

At runtime, a dynamic framework is typically copied into:

```text
MyApp.app/Frameworks/
```

If the framework is linked but not embedded, launch usually fails with a loader error such as:

```text
Library not loaded: @rpath/VendorSDK.framework/VendorSDK
```

That usually means one of these is wrong:

- the framework was not embedded
- the framework was not signed
- the binary inside the framework does not match the current platform or architecture
- the app's runpath settings are wrong

## Build Settings to Check

For the app target, review:

- `Framework Search Paths`
- `Runpath Search Paths`
- `Other Linker Flags` if the vendor requires custom flags

The default runpath used by app targets usually includes:

```text
@executable_path/Frameworks
```

Do not change runpaths casually. Most new app targets already have the correct default for embedded frameworks.

## Device vs Simulator

This is the most common failure mode with prebuilt `.framework` files.

Check that the binary supports the environment you are using:

- iPhone simulator needs simulator slices
- physical devices need device slices
- macOS apps need macOS slices

If the vendor gives you only one `.framework`, it may not be portable across all targets. That is why `.xcframework` is preferred.

## If You Built the Framework Yourself

When the framework comes from another local Xcode project:

1. Add the framework project to the workspace or app project.
2. Add the framework product under target dependencies.
3. Link the built product.
4. Embed it in the app target if it is dynamic.

This is usually better than dragging random files from `DerivedData`, because Xcode manages build order and paths more reliably.

## Recommended New-App Structure

For an app that consumes local frameworks, use a layout like this:

```text
MyApp/
├── MyApp.xcodeproj
├── MyApp/
│   ├── AppDelegate.swift
│   ├── SceneDelegate.swift
│   └── Features/
├── Frameworks/
│   ├── VendorSDK.framework
│   └── InternalSDK.xcframework
├── Config/
└── Scripts/
```

Rules that keep this manageable:

- keep binaries out of source folders
- separate internal frameworks from vendor frameworks if both exist
- avoid copying different versions of the same framework into multiple places
- document where each framework came from and how it is updated

## Troubleshooting Checklist

If the app builds but does not launch:

1. Confirm the framework is listed under `Frameworks, Libraries, and Embedded Content`.
2. Confirm the embed mode is correct.
3. Confirm the binary supports the active platform and architecture.
4. Confirm the module imports without manual header hacks.
5. Confirm the framework is present inside the built app bundle.

If the app does not build:

1. Confirm the search paths are correct.
2. Confirm the module name matches the import.
3. Confirm public headers and module metadata were exported correctly.
4. Confirm the framework was built with compatible deployment targets.

## Practical Default

For a new app, the safest approach is:

1. Use `.xcframework` if available.
2. Store binary dependencies in a top-level `Frameworks/` folder.
3. Link the framework through the app target's `General` settings.
4. Use `Embed & Sign` for dynamic frameworks.
5. Validate both simulator and device builds before treating the integration as done.
