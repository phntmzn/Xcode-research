```swift
import Foundation

let fm = FileManager.default
let tempDir = fm.temporaryDirectory.appendingPathComponent("swift_build_\(UUID().uuidString)")
try fm.createDirectory(at: tempDir, withIntermediateDirectories: true)

let sourceURL = tempDir.appendingPathComponent("hello.swift")
let outputURL = tempDir.appendingPathComponent("hello")

let swiftCode = """
import Foundation

print("Hello from compiled Swift")
"""

try swiftCode.write(to: sourceURL, atomically: true, encoding: .utf8)

func runProcess(_ launchPath: String, _ args: [String]) throws -> (Int32, String) {
    let process = Process()
    let pipe = Pipe()

    process.executableURL = URL(fileURLWithPath: launchPath)
    process.arguments = args
    process.standardOutput = pipe
    process.standardError = pipe

    try process.run()
    process.waitUntilExit()

    let data = pipe.fileHandleForReading.readDataToEndOfFile()
    let text = String(data: data, encoding: .utf8) ?? ""
    return (process.terminationStatus, text)
}

let compileArgs = [
    sourceURL.path,
    "-o", outputURL.path
]

let (compileStatus, compileOutput) = try runProcess("/usr/bin/swiftc", compileArgs)

print("Compile exit status:", compileStatus)
if !compileOutput.isEmpty {
    print(compileOutput)
}

guard compileStatus == 0 else {
    fatalError("Compilation failed")
}

let (runStatus, runOutput) = try runProcess(outputURL.path, [])

print("Run exit status:", runStatus)
if !runOutput.isEmpty {
    print(runOutput)
}
```

Reusable version:

```swift
import Foundation

func runProcess(_ launchPath: String, _ args: [String]) throws -> (status: Int32, output: String) {
    let process = Process()
    let pipe = Pipe()

    process.executableURL = URL(fileURLWithPath: launchPath)
    process.arguments = args
    process.standardOutput = pipe
    process.standardError = pipe

    try process.run()
    process.waitUntilExit()

    let data = pipe.fileHandleForReading.readDataToEndOfFile()
    return (process.terminationStatus, String(data: data, encoding: .utf8) ?? "")
}

func compileSwift(sourcePath: String, outputPath: String) throws {
    let args = [
        sourcePath,
        "-o", outputPath
    ]

    let result = try runProcess("/usr/bin/swiftc", args)
    guard result.status == 0 else {
        throw NSError(
            domain: "SwiftCompileError",
            code: Int(result.status),
            userInfo: [NSLocalizedDescriptionKey: result.output]
        )
    }
}

do {
    try compileSwift(
        sourcePath: "/tmp/hello.swift",
        outputPath: "/tmp/hello"
    )

    let result = try runProcess("/tmp/hello", [])
    print(result.output)
} catch {
    print("Error:", error)
}
```

Multiline string example for the Swift source:

```swift
let swiftCode = """
import Foundation

struct Greeter {
    let name: String

    func greet() {
        print("Hello, \\(name)")
    }
}

Greeter(name: "Swift").greet()
"""
```

For optimization, add flags like:

```swift
let args = [
    sourcePath,
    "-O",
    "-o", outputPath
]
```

Or parse as a script with `swift` instead of compiling:

```swift
let result = try runProcess("/usr/bin/swift", [sourcePath])
print(result.output)
```

If you want, I can write the folder version that compiles every `.swift` file with `swiftc`.
