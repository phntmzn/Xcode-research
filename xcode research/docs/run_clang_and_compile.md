```swift
import Foundation

let fm = FileManager.default
let tempDir = fm.temporaryDirectory.appendingPathComponent("objc_build_\(UUID().uuidString)")
try fm.createDirectory(at: tempDir, withIntermediateDirectories: true)

let sourceURL = tempDir.appendingPathComponent("hello.m")
let outputURL = tempDir.appendingPathComponent("hello")

let objcCode = """
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"Hello from Objective-C");
    }
    return 0;
}
"""

try objcCode.write(to: sourceURL, atomically: true, encoding: .utf8)

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
    "-framework", "Foundation",
    "-o", outputURL.path
]

let (compileStatus, compileOutput) = try runProcess("/usr/bin/clang", compileArgs)

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

For a reusable function version:

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

func compileObjectiveC(sourcePath: String, outputPath: String) throws {
    let args = [
        sourcePath,
        "-framework", "Foundation",
        "-o", outputPath
    ]

    let result = try runProcess("/usr/bin/clang", args)
    guard result.status == 0 else {
        throw NSError(domain: "CompileError", code: Int(result.status), userInfo: [
            NSLocalizedDescriptionKey: result.output
        ])
    }
}

do {
    try compileObjectiveC(
        sourcePath: "/tmp/hello.m",
        outputPath: "/tmp/hello"
    )

    let result = try runProcess("/tmp/hello", [])
    print(result.output)
} catch {
    print("Error:", error)
}
```

If you want ARC explicitly, use:

```swift
let args = [
    sourcePath,
    "-fobjc-arc",
    "-framework", "Foundation",
    "-o", outputPath
]
```

If you want, I can make this into a Swift command line tool that compiles every `.m` file in a folder.
