# iConfig — Claude Handoff

## What this project is

iConfig is a Qt desktop app for configuring iConnectivity MIDI/audio devices (iConnectMIDI, iConnectAudio, mio families). It talks to devices over USB via SysEx/MIDI using a companion static library, GeneSysLib.

This is a fork of the abandoned iConnectivity source release, ported to build on modern Apple Silicon Macs.

## What was done in the porting session

Starting from source that had never been committed for arm64, the following was done to get a clean build on macOS 26 / Apple Silicon with Homebrew Qt 6.11 and Clang 21:

- Installed `rtmidi`, `qt`, and `boost` via Homebrew (rtmidi was the critical missing piece — it was referenced but never committed)
- Updated both `.pro` files (`GeneSysLib/Qt/GeneSysLib.pro`, `iConfig/iConnectivityiConfig.pro`) for C++17, libc++, macOS 12.0 target, Homebrew paths
- Switched from compiling `RtMidi.cpp` locally to linking against the Homebrew `librtmidi` shared library
- Fixed the GeneSysLib static library output path so the app can find it
- Made ~40 source files compile cleanly under Qt 6 and C++17 (see README "What changed from upstream" for the full list)
- App installed to `/Applications/iConfig Neo.app` for testing alongside any legacy version

## Build process (quick reference)

```sh
cd GeneSysLib/Qt && qmake GeneSysLib.pro && make -j$(sysctl -n hw.logicalcpu) && cd ../..
cd iConfig && qmake iConnectivityiConfig.pro && make -j$(sysctl -n hw.logicalcpu)
```

Output: `iConfig/iConnectivity iConfig.app`

To reinstall to /Applications after a rebuild:
```sh
cp -r "/Users/joemcmahon/Code/iConfig/iConfig/iConnectivity iConfig.app" "/Applications/iConfig Neo.app"
```

## Testing status

- **UI navigation**: verified — all major screens open without crashing
- **On-device communication**: NOT yet tested with physical hardware
- The porting changes are all mechanical API fixes with no intentional logic changes, but untested paths may have issues

## Architecture

```
GeneSysLib/          C++ library — all device protocol logic
  Base/              SysEx parsing, MIDI I/O (via rtmidi), command dispatch
  Audio/             AudioV1 (older devices), AudioV2, Mixer
  Device/            Device identity, gizmo enumeration
  MIDI/              Port routing, filtering, remapping
  Qt/                GeneSysLib.pro — builds libGeneSysLib.a

iConfig/             Qt desktop app
  Main.cpp           Entry point
  MainWindow.*       Top-level window, device switching
  DeviceInfo.*       Central device state object (talks to GeneSysLib)
  AudioRelated/      Audio config and patchbay UI
  MIDIRelated/       MIDI routing/filter/remap UI
  MixerRelated/      Mixer UI (AudioV2 devices only)
  FirmwareRelated/   Firmware upgrade/check dialogs
  Presets/           Save/restore preset files
  qtsinglapplication/ Third-party single-instance wrapper
```

## Key files to know

| File | Why it matters |
|------|----------------|
| `GeneSysLib/Base/Communicator.cpp` | All rtmidi I/O — first place to look for MIDI connectivity issues |
| `GeneSysLib/Base/SysexParser.cpp` | Parses incoming SysEx — first place to look for garbled data |
| `iConfig/DeviceInfo.cpp` | Manages device state, query sequencing — central to everything |
| `iConfig/MainWindow.cpp` | Top-level UI logic, device switching |
| `GeneSysLib/Qt/GeneSysLib.pro` | Library build config |
| `iConfig/iConnectivityiConfig.pro` | App build config |

## Things to watch for during hardware testing

- **Device not detected**: check rtmidi can see the port (`Communicator::getInCount()` / `getOutCount()`)
- **UI shows device but data doesn't load**: likely a SysEx parsing issue in `SysexParser.cpp` or a command sequencing issue in `DeviceInfo.cpp`
- **Mixer/Audio controls don't update**: AudioV2 devices use a meter polling timer — check `TimerThread` and the `RereadMeters` screen path in `DeviceInfo.cpp`
- **Crash on device disconnect**: the communicator closes ports asynchronously — there's a mutex in `Communicator.cpp` around that path that was already tricky in the original code
- **Preset save/load broken**: files go to `QStandardPaths::AppDataLocation` (changed from `QDesktopServices::DataLocation` in the port) — on macOS this is `~/Library/Application Support/iConnectivity iConfig/presets/`

## Dependencies (all via Homebrew)

| Package | Version tested | Purpose |
|---------|---------------|---------|
| qt | 6.11.1 | UI framework |
| rtmidi | 6.0.0 | MIDI I/O |
| boost | 1.90.0 | Utilities used throughout GeneSysLib |

## Remote

```
git@github.com:joemcmahon/iConfig.git
```
Fork of `iConnectivity/iConfig`.
