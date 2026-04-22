```swift
let bashScript = """
#!/bin/bash

echo "Hello from bash"

name="world"
echo "Hi, $name"

for i in 1 2 3
do
    echo "Number: $i"
done
"""
```

If you want to run it from Swift too:

```swift
import Foundation

let bashScript = """
#!/bin/bash

echo "Hello from bash"

name="world"
echo "Hi, $name"

for i in 1 2 3
do
    echo "Number: $i"
done
"""

let tempURL = FileManager.default.temporaryDirectory.appendingPathComponent("script.sh")
try bashScript.write(to: tempURL, atomically: true, encoding: .utf8)

try FileManager.default.setAttributes([.posixPermissions: 0o755], ofItemAtPath: tempURL.path)

let process = Process()
process.executableURL = URL(fileURLWithPath: "/bin/bash")
process.arguments = [tempURL.path]

try process.run()
process.waitUntilExit()
```

If the bash script contains `\( ... )`, escape it in Swift like this:

```swift
let bashScript = """
echo "\\(notSwiftInterpolation)"
"""
```
