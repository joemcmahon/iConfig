# iConfig

iConfig is a desktop configuration application for iConnectivity MIDI and audio interfaces, including the iConnectMIDI, iConnectAudio, and mio product families.

This is a community-maintained fork of the [original iConnectivity source release](https://github.com/iConnectivity/iConfig). iConnectivity has not published a build for Apple Silicon, and the original codebase no longer compiles against modern toolchains. This fork ports it to **macOS arm64** (Apple Silicon) with Qt 6, C++17, and current Homebrew dependencies.

## Status

Builds and runs on Apple Silicon Macs running macOS 12 Monterey or later. The UI is functional — device selection, MIDI routing, audio configuration, mixer, patchbay, and firmware upgrade dialogs all work.

## Prerequisites

### 1. Xcode Command Line Tools

If you haven't already:

```sh
xcode-select --install
```

### 2. Homebrew

If you haven't already, install [Homebrew](https://brew.sh):

```sh
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

After installing, follow any instructions Homebrew prints about adding it to your shell's PATH. On Apple Silicon this typically means adding this to your `~/.zprofile`:

```sh
eval "$(/opt/homebrew/bin/brew shellenv)"
```

Then reload your shell:

```sh
source ~/.zprofile
```

### 3. Dependencies

```sh
brew install qt rtmidi boost
```

This installs Qt 6, rtmidi 6, and Boost. The build takes a few minutes if packages aren't cached.

### 4. Verify qmake is available

```sh
qmake --version
```

You should see something like:

```
QMake version 3.1
Using Qt version 6.x.x in /opt/homebrew/lib
```

If `qmake` is not found, your Homebrew bin directory may not be on your PATH. Check that `eval "$(/opt/homebrew/bin/brew shellenv)"` is in your shell profile and re-open your terminal.

If you have multiple Qt installations (e.g. from MacPorts or a manual Qt installer), make sure `/opt/homebrew/bin` comes first in your PATH, or invoke qmake with its full path `/opt/homebrew/bin/qmake`.

## Building

Clone the repository:

```sh
git clone https://github.com/joemcmahon/iConfig.git
cd iConfig
```

### Step 1 — Build the GeneSysLib static library

```sh
cd GeneSysLib/Qt
qmake GeneSysLib.pro
make -j$(sysctl -n hw.logicalcpu)
cd ../..
```

This produces `GeneSysLib/Qt/libGeneSysLib.a`. Warnings from Boost about deprecated headers are expected and harmless.

### Step 2 — Build the iConfig app

```sh
cd iConfig
qmake iConnectivityiConfig.pro
make -j$(sysctl -n hw.logicalcpu)
```

This produces `iConfig/iConnectivity iConfig.app`.

### Step 3 — Run

```sh
open "iConnectivity iConfig.app"
```

Or double-click the app bundle in Finder.

### Clean build

If you need to start fresh (e.g. after switching branches or pulling updates):

```sh
# Clean GeneSysLib
cd GeneSysLib/Qt
make distclean
qmake GeneSysLib.pro
make -j$(sysctl -n hw.logicalcpu)
cd ../..

# Clean iConfig
cd iConfig
make distclean
qmake iConnectivityiConfig.pro
make -j$(sysctl -n hw.logicalcpu)
```

## Troubleshooting

**`qmake: command not found`**
Homebrew's bin directory is not on your PATH. See the "Verify qmake is available" step above.

**`fatal error: 'RtMidi.h' file not found`**
rtmidi is not installed or Homebrew's include path isn't being picked up. Run `brew install rtmidi` and confirm `/opt/homebrew/include/rtmidi/RtMidi.h` exists.

**`fatal error: 'boost/...' file not found`**
Boost is not installed. Run `brew install boost`.

**Linker error: `library not found for -lGeneSysLib`**
GeneSysLib wasn't built first, or the build output is missing. Complete Step 1 before Step 2.

**App opens but shows no devices**
This is expected if no iConnectivity device is connected. Connect your device via USB and relaunch.

**macOS Gatekeeper blocks the app**
Because the app is built locally and not signed with an Apple Developer certificate, macOS may warn you. Right-click the app in Finder and choose Open, then confirm in the dialog.

## What changed from upstream

The original source targeted Qt 4.8/5, `libstdc++`, macOS 10.6, and a locally-vendored `rtmidi 2.1.1` that was never committed to the repository. The following changes were made to build on a modern Apple Silicon Mac:

**Build system**
- Switch to C++17, `libc++`, macOS 12.0 deployment target
- Replace MacPorts and hardcoded Xcode SDK paths with Homebrew paths
- Link against Homebrew-installed `librtmidi` (6.0.0) instead of compiling from a missing local copy
- Fix GeneSysLib static library path referenced by the app project
- Add Qt `network` and `core5compat` modules

**C++17 / Boost 1.90 fixes**
- Replace removed `boost/tr1/unordered_map.hpp` with `<unordered_map>`
- Replace removed `std::auto_ptr` with `std::unique_ptr`
- Qualify ambiguous `advance()`/`distance()` calls (both `boost` and `std` namespaces were in scope)
- Add missing standard library includes to `MyAlgorithms.h`

**Qt 6 API fixes**
- `QRegExp`/`QRegExpValidator` → `QRegularExpression`/`QRegularExpressionValidator`
- `QHeaderView::setResizeMode` → `setSectionResizeMode`
- `QLayout::setMargin(n)` → `setContentsMargins(n,n,n,n)`
- `QStyleOption::init()` → `initFrom()`
- `QDateTime::toTime_t()` → `toSecsSinceEpoch()`
- `QDesktopServices::storageLocation(DataLocation)` → `QStandardPaths::writableLocation(AppDataLocation)`
- `QString::sprintf()` → `QString::asprintf()`
- `QMouseEvent::posF()` → `position()`
- `QList::toStdList()` / `toSet()` / `toList()` → standard iterator-range construction
- `QChar::toAscii()` / `QString::toAscii()` → `toLatin1()`
- Removed `QSysInfo::MacintoshVersion` / `MV_10_8` (removed from Qt 6)
- Fixed `class QStringList` forward declaration (now a typedef in Qt 6)
- Added missing `#include <QActionGroup>` and `#include <QPainterPath>`
- Fixed unimplemented constructor in `QtSingleApplicationWrapper` that Qt 6's moc tried to call
- Fixed a pre-existing bug where a `bool` was concatenated into a `QString` error message

## License

GPLv3 — see [LICENSE.md](LICENSE.md).
