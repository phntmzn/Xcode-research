# Dynamic Library Code Examples

This guide is intentionally example-first. It shows one practical way to author a reusable dynamic framework for Apple platforms.

## Example Repository Layout

```text
MyLibrary/
├── MyLibrary.xcodeproj
├── Config/
│   ├── Base.xcconfig
│   ├── Debug.xcconfig
│   └── Release.xcconfig
├── Sources/
│   └── MyLibrary/
│       ├── Public/
│       │   ├── MyLibrary.h
│       │   ├── MyLibraryAPI.h
│       │   ├── GreetingService.swift
│       │   ├── APIClient.swift
│       │   └── UserSession.swift
│       ├── Internal/
│       │   ├── DefaultDateProvider.swift
│       │   ├── HTTPTransport.swift
│       │   └── TokenStore.swift
│       └── Resources/
│           └── MyLibrary.bundle
├── Tests/
│   └── MyLibraryTests/
│       ├── GreetingServiceTests.swift
│       ├── APIClientTests.swift
│       └── UserSessionTests.swift
└── Examples/
    └── DemoApp/
```

## Swift Public API Example

### `Sources/MyLibrary/Public/GreetingService.swift`

```swift
import Foundation

public protocol Greeter {
    func greeting(for name: String) -> String
}

public struct GreetingConfiguration: Sendable {
    public var prefix: String
    public var dateFormat: String

    public init(prefix: String = "Hello", dateFormat: String = "yyyy-MM-dd") {
        self.prefix = prefix
        self.dateFormat = dateFormat
    }
}

public final class GreetingService: Greeter, Sendable {
    private let configuration: GreetingConfiguration
    private let dateProvider: DateProviding

    public init(configuration: GreetingConfiguration = .init()) {
        self.init(configuration: configuration, dateProvider: DefaultDateProvider())
    }

    init(
        configuration: GreetingConfiguration,
        dateProvider: DateProviding
    ) {
        self.configuration = configuration
        self.dateProvider = dateProvider
    }

    public func greeting(for name: String) -> String {
        let formatter = DateFormatter()
        formatter.dateFormat = configuration.dateFormat
        let dateString = formatter.string(from: dateProvider.now())
        return "\(configuration.prefix), \(name). Today is \(dateString)."
    }
}
```

### `Sources/MyLibrary/Internal/DefaultDateProvider.swift`

```swift
import Foundation

protocol DateProviding: Sendable {
    func now() -> Date
}

struct DefaultDateProvider: DateProviding {
    func now() -> Date {
        Date()
    }
}
```

## Async API Client Example

### `Sources/MyLibrary/Public/APIClient.swift`

```swift
import Foundation

public struct APIRequest: Sendable {
    public var path: String
    public var method: String
    public var headers: [String: String]
    public var body: Data?

    public init(
        path: String,
        method: String = "GET",
        headers: [String: String] = [:],
        body: Data? = nil
    ) {
        self.path = path
        self.method = method
        self.headers = headers
        self.body = body
    }
}

public struct APIError: Error, Sendable {
    public let message: String

    public init(_ message: String) {
        self.message = message
    }
}

public protocol TokenProviding: Sendable {
    func currentToken() async -> String?
}

public final class APIClient: Sendable {
    private let baseURL: URL
    private let transport: HTTPTransporting
    private let tokenProvider: TokenProviding?

    public init(baseURL: URL, tokenProvider: TokenProviding? = nil) {
        self.init(
            baseURL: baseURL,
            transport: HTTPTransport(),
            tokenProvider: tokenProvider
        )
    }

    init(
        baseURL: URL,
        transport: HTTPTransporting,
        tokenProvider: TokenProviding? = nil
    ) {
        self.baseURL = baseURL
        self.transport = transport
        self.tokenProvider = tokenProvider
    }

    public func send(_ request: APIRequest) async throws -> Data {
        guard let url = URL(string: request.path, relativeTo: baseURL) else {
            throw APIError("Invalid request path: \(request.path)")
        }

        var urlRequest = URLRequest(url: url)
        urlRequest.httpMethod = request.method
        urlRequest.httpBody = request.body

        for (name, value) in request.headers {
            urlRequest.setValue(value, forHTTPHeaderField: name)
        }

        if let token = await tokenProvider?.currentToken() {
            urlRequest.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        }

        let (data, response) = try await transport.send(urlRequest)

        guard let httpResponse = response as? HTTPURLResponse else {
            throw APIError("Response was not an HTTPURLResponse")
        }

        guard (200..<300).contains(httpResponse.statusCode) else {
            throw APIError("Unexpected status code: \(httpResponse.statusCode)")
        }

        return data
    }
}
```

### `Sources/MyLibrary/Internal/HTTPTransport.swift`

```swift
import Foundation

protocol HTTPTransporting: Sendable {
    func send(_ request: URLRequest) async throws -> (Data, URLResponse)
}

struct HTTPTransport: HTTPTransporting {
    func send(_ request: URLRequest) async throws -> (Data, URLResponse) {
        try await URLSession.shared.data(for: request)
    }
}
```

### `Sources/MyLibrary/Internal/TokenStore.swift`

```swift
import Foundation

public actor InMemoryTokenStore: TokenProviding {
    private var token: String?

    public init(token: String? = nil) {
        self.token = token
    }

    public func update(token: String?) {
        self.token = token
    }

    public func currentToken() async -> String? {
        token
    }
}
```

## Objective-C Umbrella Header Example

### `Sources/MyLibrary/Public/MyLibrary.h`

```objc
#import <Foundation/Foundation.h>

FOUNDATION_EXPORT double MyLibraryVersionNumber;
FOUNDATION_EXPORT const unsigned char MyLibraryVersionString[];

#import <MyLibrary/MyLibraryAPI.h>
#import <MyLibrary/MLYGreeter.h>
```

### `Sources/MyLibrary/Public/MyLibraryAPI.h`

```objc
#import <Foundation/Foundation.h>

#if defined(__cplusplus)
    #define MYLIB_EXTERN extern "C"
#else
    #define MYLIB_EXTERN extern
#endif

#if defined(__GNUC__)
    #define MYLIB_EXPORT __attribute__((visibility("default")))
#else
    #define MYLIB_EXPORT
#endif

#define MYLIB_EXTERN_EXPORT MYLIB_EXTERN MYLIB_EXPORT

#if __has_attribute(swift_name)
    #define MYLIB_SWIFT_NAME(_name) __attribute__((swift_name(#_name)))
#else
    #define MYLIB_SWIFT_NAME(_name)
#endif
```

### `Sources/MyLibrary/Public/MLYGreeter.h`

```objc
#import <Foundation/Foundation.h>
#import <MyLibrary/MyLibraryAPI.h>

NS_ASSUME_NONNULL_BEGIN

MYLIB_EXTERN_EXPORT NSString * MLYLibraryDisplayName(void) MYLIB_SWIFT_NAME(libraryDisplayName());

@interface MLYGreeter : NSObject

- (instancetype)initWithPrefix:(NSString *)prefix;
- (NSString *)greetingForName:(NSString *)name;

@end

NS_ASSUME_NONNULL_END
```

### `Sources/MyLibrary/Internal/MLYGreeter.m`

```objc
#import "MLYGreeter.h"

NSString * MLYLibraryDisplayName(void) {
    return @"MyLibrary";
}

@interface MLYGreeter ()
@property (nonatomic, copy) NSString *prefix;
@end

@implementation MLYGreeter

- (instancetype)initWithPrefix:(NSString *)prefix {
    self = [super init];
    if (self) {
        _prefix = [prefix copy];
    }
    return self;
}

- (NSString *)greetingForName:(NSString *)name {
    return [NSString stringWithFormat:@"%@, %@.", self.prefix, name];
}

@end
```

## Swift API Exposed to Objective-C

### `Sources/MyLibrary/Public/UserSession.swift`

```swift
import Foundation

@objcMembers
public final class UserSession: NSObject {
    public let userID: String
    public private(set) var isAuthenticated: Bool

    public init(userID: String, isAuthenticated: Bool = false) {
        self.userID = userID
        self.isAuthenticated = isAuthenticated
    }

    public func markAuthenticated() {
        isAuthenticated = true
    }
}
```

### Objective-C consumer using generated Swift header

```objc
#import <MyLibrary/MyLibrary-Swift.h>

UserSession *session = [[UserSession alloc] initWithUserID:@"42" isAuthenticated:NO];
[session markAuthenticated];
```

## Resource Lookup Example

### `Sources/MyLibrary/Public/BundleLocator.swift`

```swift
import Foundation

public enum BundleLocator {
    public static let module: Bundle = {
        let candidates = [
            Bundle(for: BundleSentinel.self),
            Bundle.main,
            Bundle.allFrameworks.first(where: { $0.bundleIdentifier?.contains("MyLibrary") == true })
        ]

        for candidate in candidates.compactMap({ $0 }) {
            if let url = candidate.url(forResource: "MyLibrary", withExtension: "bundle"),
               let bundle = Bundle(url: url) {
                return bundle
            }
        }

        return Bundle(for: BundleSentinel.self)
    }()
}

private final class BundleSentinel {}
```

### Loading a bundled JSON file

```swift
import Foundation

public enum ExampleJSONLoader {
    public static func load(named name: String) throws -> Data {
        guard let url = BundleLocator.module.url(forResource: name, withExtension: "json") else {
            throw APIError("Missing resource: \(name).json")
        }

        return try Data(contentsOf: url)
    }
}
```

## Unit Test Examples

### `Tests/MyLibraryTests/GreetingServiceTests.swift`

```swift
import XCTest
@testable import MyLibrary

final class GreetingServiceTests: XCTestCase {
    func testGreetingUsesConfiguredPrefix() {
        let service = GreetingService(
            configuration: .init(prefix: "Welcome", dateFormat: "yyyy"),
            dateProvider: StubDateProvider()
        )

        let value = service.greeting(for: "Taylor")

        XCTAssertEqual(value, "Welcome, Taylor. Today is 2026.")
    }
}

private struct StubDateProvider: DateProviding {
    func now() -> Date {
        Date(timeIntervalSince1970: 1_767_225_600)
    }
}
```

### `Tests/MyLibraryTests/APIClientTests.swift`

```swift
import XCTest
@testable import MyLibrary

final class APIClientTests: XCTestCase {
    func testSendAddsBearerToken() async throws {
        let transport = MockTransport()
        let tokenStore = InMemoryTokenStore(token: "secret-token")
        let client = APIClient(
            baseURL: URL(string: "https://example.com")!,
            transport: transport,
            tokenProvider: tokenStore
        )

        _ = try await client.send(.init(path: "/users"))

        let authHeader = await transport.lastRequest?.value(forHTTPHeaderField: "Authorization")
        XCTAssertEqual(authHeader, "Bearer secret-token")
    }
}

private actor MockTransport: HTTPTransporting {
    private(set) var lastRequest: URLRequest?

    func send(_ request: URLRequest) async throws -> (Data, URLResponse) {
        lastRequest = request
        let response = HTTPURLResponse(
            url: request.url!,
            statusCode: 200,
            httpVersion: nil,
            headerFields: nil
        )!
        return (Data("{}".utf8), response)
    }
}
```

## Demo App Consumer Example

### `Examples/DemoApp/DemoAppApp.swift`

```swift
import SwiftUI
import MyLibrary

@main
struct DemoAppApp: App {
    private let greeter = GreetingService()

    var body: some Scene {
        WindowGroup {
            ContentView(message: greeter.greeting(for: "World"))
        }
    }
}
```

### `Examples/DemoApp/ContentView.swift`

```swift
import SwiftUI
import MyLibrary

struct ContentView: View {
    let message: String
    @State private var responseText = "Loading..."

    var body: some View {
        VStack(spacing: 16) {
            Text(message)
                .font(.title2)

            Text(responseText)
                .font(.body.monospaced())
        }
        .padding()
        .task {
            let client = APIClient(baseURL: URL(string: "https://example.com")!)
            do {
                let data = try await client.send(.init(path: "/status"))
                responseText = String(decoding: data, as: UTF8.self)
            } catch {
                responseText = String(describing: error)
            }
        }
    }
}
```

## XCConfig Examples

### `Config/Base.xcconfig`

```xcconfig
PRODUCT_NAME = $(TARGET_NAME)
PRODUCT_MODULE_NAME = MyLibrary
DEFINES_MODULE = YES
SKIP_INSTALL = NO
BUILD_LIBRARY_FOR_DISTRIBUTION = YES
APPLICATION_EXTENSION_API_ONLY = YES
ENABLE_BITCODE = NO
SWIFT_VERSION = 5.0
CLANG_ENABLE_MODULES = YES
DYLIB_COMPATIBILITY_VERSION = 1
DYLIB_CURRENT_VERSION = 1
```

### `Config/Debug.xcconfig`

```xcconfig
#include "Base.xcconfig"

SWIFT_ACTIVE_COMPILATION_CONDITIONS = DEBUG
GCC_PREPROCESSOR_DEFINITIONS = $(inherited) DEBUG=1
ONLY_ACTIVE_ARCH = YES
```

### `Config/Release.xcconfig`

```xcconfig
#include "Base.xcconfig"

SWIFT_OPTIMIZATION_LEVEL = -O
DEAD_CODE_STRIPPING = YES
COPY_PHASE_STRIP = NO
VALIDATE_PRODUCT = YES
```

## Module Map Example

Use this only if you need a custom Clang module map.

### `Sources/MyLibrary/Public/module.modulemap`

```modulemap
framework module MyLibrary {
    umbrella header "MyLibrary.h"

    export *
    module * { export * }
}
```

## Common `xcodebuild` Commands

### Build the framework for the simulator

```bash
xcodebuild \
  -project MyLibrary.xcodeproj \
  -scheme MyLibrary \
  -sdk iphonesimulator \
  -configuration Debug \
  build
```

### Run the tests

```bash
xcodebuild \
  -project MyLibrary.xcodeproj \
  -scheme MyLibrary \
  -destination 'platform=iOS Simulator,name=iPhone 16' \
  test
```

### Archive the framework for distribution

```bash
xcodebuild \
  -project MyLibrary.xcodeproj \
  -scheme MyLibrary \
  -configuration Release \
  -sdk iphoneos \
  SKIP_INSTALL=NO \
  BUILD_LIBRARY_FOR_DISTRIBUTION=YES \
  archive \
  -archivePath build/MyLibrary-iOS
```

## Minimal Release Script Example

### `Scripts/build_xcframework.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

SCHEME="MyLibrary"
PROJECT="MyLibrary.xcodeproj"
BUILD_DIR="$(pwd)/build"

rm -rf "$BUILD_DIR"
mkdir -p "$BUILD_DIR"

xcodebuild archive \
  -project "$PROJECT" \
  -scheme "$SCHEME" \
  -sdk iphoneos \
  -archivePath "$BUILD_DIR/MyLibrary-iOS" \
  SKIP_INSTALL=NO \
  BUILD_LIBRARY_FOR_DISTRIBUTION=YES

xcodebuild archive \
  -project "$PROJECT" \
  -scheme "$SCHEME" \
  -sdk iphonesimulator \
  -archivePath "$BUILD_DIR/MyLibrary-Sim" \
  SKIP_INSTALL=NO \
  BUILD_LIBRARY_FOR_DISTRIBUTION=YES

xcodebuild -create-xcframework \
  -framework "$BUILD_DIR/MyLibrary-iOS.xcarchive/Products/Library/Frameworks/MyLibrary.framework" \
  -framework "$BUILD_DIR/MyLibrary-Sim.xcarchive/Products/Library/Frameworks/MyLibrary.framework" \
  -output "$BUILD_DIR/MyLibrary.xcframework"
```

## Practical Pattern

If you want a clean baseline, copy these ideas:

- keep public API in `Public/`
- hide internals behind protocols and actors
- add a demo app instead of trusting unit tests alone
- build with `BUILD_LIBRARY_FOR_DISTRIBUTION=YES` if binary reuse matters
- ship an `.xcframework` instead of handing consumers random build products
