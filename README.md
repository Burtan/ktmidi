# ktmidi: Kotlin Multiplatform library for MIDI 1.0 and MIDI 2.0

![maven repo](https://img.shields.io/maven-central/v/dev.atsushieno/ktmidi)

ktmidi is a Kotlin Multiplatform library for MIDI Access API and MIDI data processing that covers MIDI 1.0 and MIDI 2.0. 

## Features

It implements the following features (and so on):

- MIDI 1.0 and 2.0 UMPs everywhere.
- `MidiAccess` : MIDI access abstraction API like what Web MIDI API 1.0 is going to provide.
  - There are actual implementations for some platform specific MIDI API within this library, and you can implement your own backend if you need.
  - Unlike `javax.sound.midi` API, this API also covers creating virtual ports wherever possible. [`ktmidi-jvm-desktop`](https://github.com/atsushieno/ktmidi-jvm-desktop) actually contains ALSA MIDI Access implementation that supports it. (Unsupported platforms are left unsupported.)
- `MidiMusic` and `Midi2Music` : reflects Standard MIDI File format structure, with reader and writer. (MIDI 2.0 support is not based on standard, as there is nothing like SMF specification for MIDI 2.0 yet.)
  - No strongly-typed message types (something like NoteOnMessage, NoteOffMessage, and so on). There is no point of defining strongly-typed messages for each mere MIDI status byte - you wouldn't need message type abstraction.
  - No worries, there are MidiCC, MidiRpnType, MidiMetaType, MidiEvent fields (of `Byte` or `Int`) and more, so that you don't have to remember the actual constants.
  - `SmfReader` reads and `SmfWriter` writes to SMF (standard MIDI format) files with MIDI messages, with `SmfTrackMerger` and `SmfTrackSplitter` that help you implement sequential event processing for your own MIDI players, or per-track editors if needed.
- `MidiPlayer` and `Midi2Player`: provides MIDI player functionality: play/pause/stop and fast-forwarding.
  - Midi messages are sent to its "message listeners". If you don't pass a Midi Access instance or a Midi Output instance, it will do nothing but event dispatching.
  - It is based on customizible scheduler `MidiTimer`.
  - Not realtime strict (as on GC-ed language / VM), but would suffice for general usage.

## Using ktmidi

To use ktmidi, add the following lines in the `dependencies` section in `build.gradle`:

```
dependencies {
    implementation 'dev.atsushieno:ktmidi:+' // replace + with the actual version
}
```

The actual artifact might be platform dependent like `dev.atsushieno:ktmidi-android:+` or `dev.atsushieno:ktmidi-js:+`, depending on the project targets.

ktmidi is released at sonatype and hence available at Maven Central.

## Applications

- [atsushieno/mugene-ng](https://github.com/atsushieno/mugene-ng) is a Music Macro Language compiler that aims to support MIDI 2.0 (as its output format (based on this API) as well as MIDI 1.0 SMF.
- [atsushieno/notium](https://github.com/atsushieno/notium-ng) aims to offer the same functionality as mugene-ng, but as an object-oriented Kotlin API.
- [atsushieno/kmmk](https://github.com/atsushieno/kmmk) is a virtual MIDI keyboard application that is based on Jetpack Compose and therefore supposed to work on both Android and Java desktop. (still under construction)

## Platform Access API

For platform MIDI access API, we cover Android MIDI API (in Kotlin), javax.sound.midi API (with limited feature set), and ALSA Sequencer. For dependency resolution reason, ALSA implementation is split from here - see [atsushieno/ktmidi-jvm-desktop](https://github.com/atsushieno/ktmidi-jvm-desktop) project.

ktmidi builds for Kotlin/JVM, Kotlin/JS and Kotlin/Native (though I only develop with Kotlin/JVM so far).

The entire API is still subject to change, and it had been actually radically changing when development was most active.

## MIDI 2.0 support

ktmidi supports MIDI 2.0. Since you can derive from `MidiAccess` abstract API, you can create your own MIDI access implementation and don't have to wait for platform native API to support MIDI 2.0. Note that MIDI 2.0 support is still an experimental work.

It would be useful for general MIDI 2.0 software tools such as MIDI 2.0 UMP player.

Here is a list of MIDI 2.0 extensibility in this API:

- `MidiInput` and `MidiOutput` now has `midiProtocol` property which can be get and/or set. When `MidiCIProtocolValue.MIDI2_V1` is specified, then the I/O object is supposed to process UMPs (Universal MIDI Packets).
- `Midi2Music` is a feature parity with `MidiMusic`, but all the messages are stored as UMPs. However, since SMF concepts of time calculation (namely delta time quantization / specification) is useful, we optionally blend it into UMPs and their JR Timestamp messages are actually fake - they store delta times just like SMF.
- `Midi2Player` is a feature parity with `MidiPlayer`.
- `dev.atsushieno.ktmidi.umpfactory` package contains a bunch of utility functions that are used to construct UMP integer values.

[atsushieno/kmmk](https://github.com) supports "MIDI 2.0 mode" which sends MIDI messages in MIDI 2.0 UMPs. There is also an ongoing experimental project to process MIDI 2.0 UMPs in [audio plugins on Android](https://github.com/atsushieno/android-audio-plugin-framework/tree/main/java/aap-midi-device-service).

### SMF alternative format

Since there is no comparable standard music file format like SMF for MIDI 2.0, we had to come up with our own. Our `Midi2Player` accepts files in the format described below:

```
// Data Format:
//   identifier: 0xAAAAAAAAAAAAAAAA (16 bytes)
//   i32 deltaTimeSpec
//   i32 numTracks
//   tracks
//        identifier: 0xEEEEEEEEEEEEEEEE (16 bytes)
//       i32 numUMPs
//       umps (i32, i64 or i128)
```

If `deltaTimeSpec` is a positive integer, it works like the value in SMF header chunk. There is no "format" specifier in this format - if "numTracks" is 1 then it is obviously compatible with FORMAT 0.

Also, we have a workaround for META events, now that system message has its own message type with fixed (limited) length - in this format they are stored as SYSEX8 messages with all 0s for manufacturer ID, device ID, and sub IDs.

[mugene-ng](https://github.com/atsushieno/mugene-ng) can generate music files based on this format.

## Docs

Right now there is no published API documentation. Dokka plugin is integrated and you can build API docs by `./gradlew dokkaHtml` and it will be generated locally.

For hacking and/or contributing to ktmidi, please have a look at [HACKING.md](HACKING.md).

There are couple of API/implementation design docs:

- [docs/MidiAccess.md](docs/MidiAccess.md)
- [docs/MidiMusic.md](docs/MidiMusic.md)
- [docs/MidiPlayer.md](docs/MidiPlayer.md)

## Historical background

It started as the Kotlin port of C# [managed-midi](https://github.com/atsushieno/managed-midi) library. Also it started with partial copy of [fluidsynth-midi-service-j](https://github.com/atsushieno/fluidsynth-midi-service-j) project.

However everything in this project went far beyond them and now we are making it usable for MIDI 2.0.

## License

ktmidi is distributed under the MIT license.

