Use an **Xcode Cocoa Framework** layout, not a Swift package.

## Tiny macOS `.framework` idea

```text
GoyslopFileWriters/
├── GoyslopFileWriters.xcodeproj
├── GoyslopFileWriters/
│   ├── Info.plist
│   ├── GoyslopFileWriters.h
│   ├── FileWriterError.swift
│   ├── MIDINote.swift
│   ├── MIDITrack.swift
│   ├── MIDIFileWriter.swift
│   ├── ChordWriter.swift
│   └── TempoMap.swift
├── GoyslopFileWritersTests/
│   └── GoyslopFileWritersTests.swift
└── README.md
```

## Framework target type
In Xcode:

**File → New → Project → Framework**
Choose:
- macOS
- Framework
- Language: Swift

## What each file does

### `GoyslopFileWriters.h`
Public umbrella header for the framework.

```objc
#import <Foundation/Foundation.h>

//! Project version number for GoyslopFileWriters.
FOUNDATION_EXPORT double GoyslopFileWritersVersionNumber;

//! Project version string for GoyslopFileWriters.
FOUNDATION_EXPORT const unsigned char GoyslopFileWritersVersionString[];
```

### `FileWriterError.swift`

```swift
import Foundation

public enum FileWriterError: Error {
    case couldNotCreateSequence
    case couldNotCreateTrack
    case couldNotSetTempo
    case couldNotWriteFile
    case invalidPath
}
```

### `MIDINote.swift`

```swift
import Foundation

public struct MIDINote {
    public let note: UInt8
    public let velocity: UInt8
    public let beat: Double
    public let duration: Double
    public let channel: UInt8

    public init(
        note: UInt8,
        velocity: UInt8 = 100,
        beat: Double,
        duration: Double,
        channel: UInt8 = 0
    ) {
        self.note = note
        self.velocity = velocity
        self.beat = beat
        self.duration = duration
        self.channel = channel
    }
}
```

### `MIDITrack.swift`

```swift
import Foundation

public struct MIDITrack {
    public var notes: [MIDINote]

    public init(notes: [MIDINote] = []) {
        self.notes = notes
    }

    public mutating func add(_ note: MIDINote) {
        notes.append(note)
    }
}
```

### `TempoMap.swift`

```swift
import Foundation

public struct TempoMap {
    public let bpm: Double

    public init(bpm: Double = 120.0) {
        self.bpm = bpm
    }
}
```

### `MIDIFileWriter.swift`
Uses `AudioToolbox`.

```swift
import Foundation
import AudioToolbox

public final class MIDIFileWriter {
    public init() {}

    public func write(
        tracks: [MIDITrack],
        tempo: TempoMap = TempoMap(),
        to url: URL
    ) throws {
        var sequence: MusicSequence?
        NewMusicSequence(&sequence)

        guard let sequence else {
            throw FileWriterError.couldNotCreateSequence
        }

        var tempoTrack: MusicTrack?
        MusicSequenceGetTempoTrack(sequence, &tempoTrack)

        if let tempoTrack {
            let status = MusicTrackNewExtendedTempoEvent(tempoTrack, 0, tempo.bpm)
            guard status == noErr else {
                throw FileWriterError.couldNotSetTempo
            }
        }

        for trackModel in tracks {
            var track: MusicTrack?
            let createStatus = MusicSequenceNewTrack(sequence, &track)
            guard createStatus == noErr, let track else {
                throw FileWriterError.couldNotCreateTrack
            }

            for n in trackModel.notes {
                var msg = MIDINoteMessage(
                    channel: n.channel,
                    note: n.note,
                    velocity: n.velocity,
                    releaseVelocity: 0,
                    duration: Float32(n.duration)
                )

                let status = MusicTrackNewMIDINoteEvent(track, n.beat, &msg)
                guard status == noErr else {
                    throw FileWriterError.couldNotWriteFile
                }
            }
        }

        let nsURL = url as NSURL
        let writeStatus = MusicSequenceFileCreate(
            sequence,
            nsURL,
            .midiType,
            .eraseFile,
            480
        )

        guard writeStatus == noErr else {
            throw FileWriterError.couldNotWriteFile
        }
    }
}
```

### `ChordWriter.swift`

```swift
import Foundation

public final class ChordWriter {
    public init() {}

    public func triad(root: UInt8, beat: Double, duration: Double) -> MIDITrack {
        MIDITrack(notes: [
            MIDINote(note: root,     beat: beat, duration: duration),
            MIDINote(note: root + 4, beat: beat, duration: duration),
            MIDINote(note: root + 7, beat: beat, duration: duration)
        ])
    }

    public func minorTriad(root: UInt8, beat: Double, duration: Double) -> MIDITrack {
        MIDITrack(notes: [
            MIDINote(note: root,     beat: beat, duration: duration),
            MIDINote(note: root + 3, beat: beat, duration: duration),
            MIDINote(note: root + 7, beat: beat, duration: duration)
        ])
    }
}
```

## Example use in another macOS app

```swift
import Foundation
import GoyslopFileWriters

let writer = MIDIFileWriter()
let chords = ChordWriter()

let track1 = chords.minorTriad(root: 60, beat: 0, duration: 2)
let track2 = chords.triad(root: 65, beat: 2, duration: 2)

let out = URL(fileURLWithPath: "/Users/you/Desktop/test.mid")
try writer.write(tracks: [track1, track2], tempo: TempoMap(bpm: 120), to: out)
```

## Good tiny framework variants
You could split this into several small frameworks:

```text
GoyslopMIDIWriter.framework
GoyslopChordWriter.framework
GoyslopArpWriter.framework
GoyslopDrumPatternWriter.framework
GoyslopTempoTools.framework
```

## Even smaller Xcode framework tree

```text
GoyslopMIDIWriter/
├── GoyslopMIDIWriter.xcodeproj
├── GoyslopMIDIWriter/
│   ├── Info.plist
│   ├── GoyslopMIDIWriter.h
│   ├── MIDIFileWriter.swift
│   ├── MIDINote.swift
│   └── FileWriterError.swift
└── GoyslopMIDIWriterTests/
    └── GoyslopMIDIWriterTests.swift
```

## Build output
After building in Xcode, the framework ends up around:

```text
~/Library/Developer/Xcode/DerivedData/.../Build/Products/Debug/GoyslopFileWriters.framework
```

If you want, I can write the **full Xcode-ready file contents** for this framework next, including `Info.plist` and a test file.

Here are **3 tiny macOS `.framework` variants** using **Swift + AudioToolbox**.

## 1. `GoyslopMIDIWriter.framework`
Smallest plain MIDI file writer.

```text
GoyslopMIDIWriter/
├── GoyslopMIDIWriter.xcodeproj
├── GoyslopMIDIWriter/
│   ├── Info.plist
│   ├── GoyslopMIDIWriter.h
│   ├── MIDIFileWriter.swift
│   ├── MIDINote.swift
│   └── FileWriterError.swift
└── GoyslopMIDIWriterTests/
    └── GoyslopMIDIWriterTests.swift
```

### Purpose
- write `.mid`
- add single notes
- set tempo

### Core files
`MIDINote.swift`
```swift
import Foundation

public struct MIDINote {
    public let note: UInt8
    public let velocity: UInt8
    public let beat: Double
    public let duration: Double

    public init(note: UInt8, velocity: UInt8 = 100, beat: Double, duration: Double) {
        self.note = note
        self.velocity = velocity
        self.beat = beat
        self.duration = duration
    }
}
```

`FileWriterError.swift`
```swift
import Foundation

public enum FileWriterError: Error {
    case sequenceCreateFailed
    case trackCreateFailed
    case tempoFailed
    case writeFailed
}
```

---

## 2. `GoyslopChordWriter.framework`
Tiny chord-based file writer.

```text
GoyslopChordWriter/
├── GoyslopChordWriter.xcodeproj
├── GoyslopChordWriter/
│   ├── Info.plist
│   ├── GoyslopChordWriter.h
│   ├── Chord.swift
│   ├── ChordWriter.swift
│   ├── MIDIFileWriter.swift
│   └── FileWriterError.swift
└── GoyslopChordWriterTests/
    └── GoyslopChordWriterTests.swift
```

### Purpose
- write triads
- write minor chords
- export progression as `.mid`

### Core files
`Chord.swift`
```swift
import Foundation

public struct Chord {
    public let notes: [UInt8]
    public let beat: Double
    public let duration: Double

    public init(notes: [UInt8], beat: Double, duration: Double) {
        self.notes = notes
        self.beat = beat
        self.duration = duration
    }
}
```

`ChordWriter.swift`
```swift
import Foundation

public final class ChordWriter {
    public init() {}

    public func major(root: UInt8, beat: Double, duration: Double) -> Chord {
        Chord(notes: [root, root + 4, root + 7], beat: beat, duration: duration)
    }

    public func minor(root: UInt8, beat: Double, duration: Double) -> Chord {
        Chord(notes: [root, root + 3, root + 7], beat: beat, duration: duration)
    }
}
```

---

## 3. `GoyslopPatternWriter.framework`
Tiny loop and pattern writer.

```text
GoyslopPatternWriter/
├── GoyslopPatternWriter.xcodeproj
├── GoyslopPatternWriter/
│   ├── Info.plist
│   ├── GoyslopPatternWriter.h
│   ├── PatternStep.swift
│   ├── PatternWriter.swift
│   ├── MIDIFileWriter.swift
│   └── FileWriterError.swift
└── GoyslopPatternWriterTests/
    └── GoyslopPatternWriterTests.swift
```

### Purpose
- write repeating note patterns
- make 1-bar or 4-bar loops
- export simple arps or drum-like grids

### Core files
`PatternStep.swift`
```swift
import Foundation

public struct PatternStep {
    public let note: UInt8
    public let beat: Double
    public let duration: Double
    public let velocity: UInt8

    public init(note: UInt8, beat: Double, duration: Double, velocity: UInt8 = 100) {
        self.note = note
        self.beat = beat
        self.duration = duration
        self.velocity = velocity
    }
}
```

`PatternWriter.swift`
```swift
import Foundation

public final class PatternWriter {
    public init() {}

    public func fourOnTheFloor(note: UInt8 = 36) -> [PatternStep] {
        [
            PatternStep(note: note, beat: 0, duration: 0.5),
            PatternStep(note: note, beat: 1, duration: 0.5),
            PatternStep(note: note, beat: 2, duration: 0.5),
            PatternStep(note: note, beat: 3, duration: 0.5)
        ]
    }
}
```

## Best 3-way split

```text
1. GoyslopMIDIWriter.framework   // raw note events
2. GoyslopChordWriter.framework  // chord helpers
3. GoyslopPatternWriter.framework // loops and grids
```

## Smallest shared `MIDIFileWriter.swift`
This can be reused in all 3:

```swift
import Foundation
import AudioToolbox

public final class MIDIFileWriter {
    public init() {}

    public func writeNotes(_ notes: [MIDINote], bpm: Double, to url: URL) throws {
        var sequence: MusicSequence?
        NewMusicSequence(&sequence)
        guard let sequence else { throw FileWriterError.sequenceCreateFailed }

        var tempoTrack: MusicTrack?
        MusicSequenceGetTempoTrack(sequence, &tempoTrack)
        if let tempoTrack {
            guard MusicTrackNewExtendedTempoEvent(tempoTrack, 0, bpm) == noErr else {
                throw FileWriterError.tempoFailed
            }
        }

        var track: MusicTrack?
        guard MusicSequenceNewTrack(sequence, &track) == noErr, let track else {
            throw FileWriterError.trackCreateFailed
        }

        for n in notes {
            var msg = MIDINoteMessage(
                channel: 0,
                note: n.note,
                velocity: n.velocity,
                releaseVelocity: 0,
                duration: Float32(n.duration)
            )
            MusicTrackNewMIDINoteEvent(track, n.beat, &msg)
        }

        guard MusicSequenceFileCreate(sequence, url as CFURL, .midiType, .eraseFile, 480) == noErr else {
            throw FileWriterError.writeFailed
        }
    }
}
```

## Tiny umbrella header
Same pattern for each framework:

```objc
#import <Foundation/Foundation.h>

FOUNDATION_EXPORT double GoyslopMIDIWriterVersionNumber;
FOUNDATION_EXPORT const unsigned char GoyslopMIDIWriterVersionString[];
```

Next I can write the **full Xcode-ready tree for all 3 frameworks**.
