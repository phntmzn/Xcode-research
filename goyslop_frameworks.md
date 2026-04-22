Use an Xcode Cocoa Framework layout, not a Swift package.

Tiny macOS .framework idea

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

Framework target type

In Xcode:

File → New → Project → Framework
Choose:
	•	macOS
	•	Framework
	•	Language: Swift

What each file does

GoyslopFileWriters.h

Public umbrella header for the framework.

#import <Foundation/Foundation.h>

//! Project version number for GoyslopFileWriters.
FOUNDATION_EXPORT double GoyslopFileWritersVersionNumber;

//! Project version string for GoyslopFileWriters.
FOUNDATION_EXPORT const unsigned char GoyslopFileWritersVersionString[];

FileWriterError.swift

import Foundation

public enum FileWriterError: Error {
    case couldNotCreateSequence
    case couldNotCreateTrack
    case couldNotSetTempo
    case couldNotWriteFile
    case invalidPath
}

MIDINote.swift

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

MIDITrack.swift

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

TempoMap.swift

import Foundation

public struct TempoMap {
    public let bpm: Double

    public init(bpm: Double = 120.0) {
        self.bpm = bpm
    }
}

MIDIFileWriter.swift

Uses AudioToolbox.

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

ChordWriter.swift

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

Example use in another macOS app

import Foundation
import GoyslopFileWriters

let writer = MIDIFileWriter()
let chords = ChordWriter()

let track1 = chords.minorTriad(root: 60, beat: 0, duration: 2)
let track2 = chords.triad(root: 65, beat: 2, duration: 2)

let out = URL(fileURLWithPath: "/Users/you/Desktop/test.mid")
try writer.write(tracks: [track1, track2], tempo: TempoMap(bpm: 120), to: out)

Good tiny framework variants

You could split this into several small frameworks:

GoyslopMIDIWriter.framework
GoyslopChordWriter.framework
GoyslopArpWriter.framework
GoyslopDrumPatternWriter.framework
GoyslopTempoTools.framework

Even smaller Xcode framework tree

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

Build output

After building in Xcode, the framework ends up around:

~/Library/Developer/Xcode/DerivedData/.../Build/Products/Debug/GoyslopFileWriters.framework

If you want, I can write the full Xcode-ready file contents for this framework next, including Info.plist and a test file.
