# Structuring a New Dynamic Library Project

This guide describes a practical Xcode layout for building a reusable Apple-platform dynamic library. In most modern Apple projects, the library is shipped as a `.framework` or `.xcframework` instead of exposing a raw `.dylib` directly.

## Recommended Product Choice

- Use a dynamic framework target when the library is consumed by apps you control in Xcode.
- Use an `.xcframework` when you plan to distribute a binary to multiple apps, multiple platforms, or multiple CPU architectures.
- Use a raw `.dylib` only when you have a very specific loader or packaging requirement. For app development, frameworks are the normal integration point.

## Suggested Project Layout

```text
MyLibrary/
├── MyLibrary.xcodeproj
├── Sources/
│   └── MyLibrary/
│       ├── Public/
│       │   ├── MyLibrary.h
│       │   └── MyLibraryAPI.h
│       ├── Internal/
│       │   ├── CacheManager.swift
│       │   └── RequestBuilder.swift
│       └── Features/
│           ├── Authentication/
│           └── Networking/
├── Tests/
│   └── MyLibraryTests/
│       ├── AuthenticationTests.swift
│       └── NetworkingTests.swift
├── Examples/
│   └── DemoApp/
├── Scripts/
│   ├── build_xcframework.sh
│   └── package_release.sh
└── Docs/
    ├── architecture.md
    └── integration.md
```

## Target Setup in Xcode

Create at least these targets:

1. `MyLibrary`
2. `MyLibraryTests`
3. `DemoApp` (optional but strongly recommended)

Use these conventions:

- Put all exported API in `Sources/MyLibrary/Public/`.
- Keep non-exported implementation in `Internal/` or feature folders.
- Keep tests outside the library target so the public API is exercised like a consumer would use it.
- Add a small demo app early. It catches integration mistakes faster than unit tests alone.

## Public vs Internal Surface

Design the library so the public boundary is obvious:

- Swift: mark public API as `public` or `open` only when extension is intentional.
- Objective-C: place exported headers in the target's `Public` headers section.
- Hide implementation details behind `internal`, `fileprivate`, or private Objective-C headers.
- Avoid leaking third-party types through your public API unless you want to make them part of your compatibility contract.

## Build Settings That Matter

For a framework-style dynamic library, review these settings:

- `Mach-O Type`: `Dynamic Library`
- `Defines Module`: `Yes`
- `Build Libraries for Distribution`: `Yes` if you need stable Swift module interfaces across compiler versions
- `Skip Install`: `No` for archive/distribution targets
- `Install Objective-C Compatibility Header`: `Yes` if Swift code needs Objective-C exposure
- `Product Module Name`: keep it stable once consumers depend on it
- `Current Library Version` and `Compatibility Version`: update deliberately for binary compatibility

For mixed Swift and Objective-C code:

- Provide a small umbrella header.
- Keep imports inside the framework module where possible.
- Verify the generated `ModuleName-Swift.h` header is only used where appropriate.

## Dependency Structure

Keep dependencies explicit and shallow:

- Prefer depending on other frameworks through Xcode target dependencies or Swift Package Manager.
- Avoid circular dependencies between frameworks.
- If the library wraps another binary library, isolate that adapter layer instead of spreading direct calls across the codebase.
- If you expect external distribution, prefer vendoring as an `.xcframework` instead of passing around local build products.

## Versioning Guidance

Treat the library like a product, not just a folder of source files:

- Keep a changelog.
- Version releases semantically.
- Separate breaking API changes from additive changes.
- Rebuild and test the demo app before every release.

## Testing Strategy

A useful minimum set:

- Unit tests for public behavior
- Integration tests for file system, networking, persistence, or lifecycle behavior
- Demo app smoke test to confirm embed/load behavior
- Archive build validation for the platforms you support

If the library will be distributed in binary form, test:

- device build
- simulator build
- archive export
- app launch with the embedded framework

## Shipping as an XCFramework

If distribution is a real goal, make the project produce an `.xcframework` from day one:

1. Archive for each target platform and SDK.
2. Combine archives with `xcodebuild -create-xcframework`.
3. Publish the resulting `.xcframework` instead of a single `.framework`.

This is preferred because a plain `.framework` usually represents one platform and one build variant, while an `.xcframework` packages the full set cleanly.

## Example Ownership Boundaries

A maintainable split looks like this:

- `Public/`: types consumers import and call directly
- `Features/`: domain logic grouped by capability
- `Internal/`: shared implementation details
- `Tests/`: consumer-style validation
- `Examples/`: integration reference
- `Docs/`: API, release, and setup notes

## Common Mistakes

- Exporting too much API too early
- Mixing demo app code into the library target
- Depending on app-specific resources from the framework
- Shipping a simulator-only `.framework`
- Forgetting to embed and sign the framework in the consuming app
- Treating a raw local build artifact as a reusable distribution format

## Practical Default

If you are starting fresh, use this baseline:

1. Create a framework target, not a raw `.dylib`.
2. Keep `Sources`, `Tests`, `Examples`, and `Docs` separate.
3. Build toward `.xcframework` distribution even if the first consumer is only one app.
4. Use a demo app to validate that the framework actually links, embeds, and loads at runtime.
