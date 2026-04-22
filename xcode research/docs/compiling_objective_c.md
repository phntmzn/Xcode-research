Yes. The usual ways are:

## 1. Call Objective-C from Swift in the same Xcode project

If you have `.m` and `.h` files in an app project, add a bridging header.

### Objective-C file

```objc
// Greeter.h
#import <Foundation/Foundation.h>

@interface Greeter : NSObject
- (void)sayHello;
@end
```

```objc
// Greeter.m
#import "Greeter.h"

@implementation Greeter
- (void)sayHello {
    NSLog(@"Hello from Objective-C");
}
@end
```

### Bridging header

```objc
// YourProject-Bridging-Header.h
#import "Greeter.h"
```

### Swift

```swift
let g = Greeter()
g.sayHello()
```

In Xcode, set:

`Build Settings > Objective-C Bridging Header`

to something like:

```text
YourProject/YourProject-Bridging-Header.h
```

---

## 2. Compile Objective-C as a library/framework, then use it in Swift

This is better for reusable code.

### Objective-C header

```objc
// MathBox.h
#import <Foundation/Foundation.h>

@interface MathBox : NSObject
- (int)addA:(int)a b:(int)b;
@end
```

### Objective-C implementation

```objc
// MathBox.m
#import "MathBox.h"

@implementation MathBox
- (int)addA:(int)a b:(int)b {
    return a + b;
}
@end
```

After adding it to the target and exposing the header through the bridging header, Swift can use:

```swift
let box = MathBox()
print(box.addA(3, b: 4))
```

---

## 3. Compile Objective-C manually from Swift with `Process`

If you mean “have Swift run `clang` and compile an Objective-C file,” you can do that too.

```swift
import Foundation

let source = "/tmp/hello.m"
let output = "/tmp/hello"

let objcCode = """
#import <Foundation/Foundation.h>

int main() {
    @autoreleasepool {
        NSLog(@"Hello from Objective-C");
    }
    return 0;
}
"""

try objcCode.write(toFile: source, atomically: true, encoding: .utf8)

let process = Process()
process.executableURL = URL(fileURLWithPath: "/usr/bin/clang")
process.arguments = [
    source,
    "-framework", "Foundation",
    "-o", output
]

let pipe = Pipe()
process.standardOutput = pipe
process.standardError = pipe

try process.run()
process.waitUntilExit()

let data = pipe.fileHandleForReading.readDataToEndOfFile()
print(String(data: data, encoding: .utf8) ?? "")
print("Exit:", process.terminationStatus)
```

Then run the compiled program from Swift:

```swift
let run = Process()
run.executableURL = URL(fileURLWithPath: output)
try run.run()
run.waitUntilExit()
```

---

## 4. Swift Package Manager note

SwiftPM does not support bridging headers the same way Xcode app targets do. For mixed Swift + Objective-C in packages, you usually expose Objective-C via a Clang target/module map.

That is more setup-heavy than an Xcode app target.

---

## 5. Important rule

- `.h` / `.m` = Objective-C
- `.swift` = Swift
- Swift does not directly “compile Objective-C code inside a Swift file”
- It either:
  - imports compiled Objective-C code, or
  - invokes a compiler like `clang`

---

## Smallest Xcode mixed example

Project files:

```text
MyApp/
├── AppDelegate.swift
├── ViewController.swift
├── Greeter.h
├── Greeter.m
└── MyApp-Bridging-Header.h
```

Bridging header:

```objc
#import "Greeter.h"
```

Swift usage:

```swift
let greeter = Greeter()
greeter.sayHello()
```

If you want, I can write a full minimal mixed Swift + Objective-C Xcode project tree.
