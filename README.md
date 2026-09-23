# Kaffeine Fixes (2.0.19)

A patched fork of [KDE Kaffeine](https://invent.kde.org/multimedia/kaffeine) 2.0.19 resolving critical bugs related to modern desktop environments (Wayland), digital TV tuners (ATSC/DVB), and program schedule timing.

---

## Issues Resolved

### 1. ATSC / DVB Channel Addition Crash (`SIGSEGV`)
- **Bug**: In Kaffeine 2.0.19, `sqlFindFreeKey()` in `src/sqlinterface.h` used `std::prev(container.constEnd(), -1)` which advanced the iterator forward past `constEnd()`, returning a null node and causing an instant segmentation fault when saving new TV channels. Furthermore, reverse stepping on Qt 5's `QMap::constEnd()` sentinel header node returned null.
- **Fix**: Replaced with safe forward traversal across the container to determine the maximum existing primary key, and safely decremented iterators using `operator--` in `dvbchannel.cpp`.

### 2. Video Playing in Separate Floating VLC Window on Wayland
- **Bug**: Kaffeine embeds its video player using `libvlc_media_player_set_xwindow()`. Under native Wayland sessions (`QT_QPA_PLATFORM=wayland`), Qt does not expose an X11 window ID, and LibVLC 3.0 does not support native Wayland subsurfaces. LibVLC falls back to spawning an external detached window.
- **Fix**: Added automatic detection in `src/main.cpp` before `QApplication` starts. If running inside a Wayland desktop session, Kaffeine automatically selects the `xcb` (XWayland) platform backend, allowing LibVLC to dock video cleanly inside Kaffeine's interface.

### 3. TV Tuner Locking Up After Updating Scan Data
- **Bug**: Updating scan data over the internet launches KDE KIO download workers (`kio_http_cache_cleaner`). Because `dvb_fe_open2()` opened `/dev/dvb/adapterX/frontendY` without `O_CLOEXEC`, the background helper processes inherited the open tuner file descriptor, locking the device (`EBUSY`) and producing:
  > *"Didn't find a device with valid settings or access permissions are wrong. Please check the Configure Television window."*
- **Fix**: Switched frontend opening to `dvb_fe_open_flags(..., O_RDWR | O_CLOEXEC)` so background workers never inherit or lock tuner hardware.

### 4. Program Guide (EPG) 1 Hour Early in Non-DST Zones (e.g. Arizona)
- **Bug**: Broadcasters often transmit ATSC EIT schedule timestamps referenced to GPS/UTC assuming Daylight Saving Time (e.g., MDT UTC-6). In regions that do not observe DST (such as Arizona, `America/Phoenix` UTC-7), converting the timestamp to local time causes all programs to display 1 hour too early (e.g., 7:00 PM shows display at 6:00 PM). Kaffeine previously lacked any EPG time offset option.
- **Fix**: Added a configurable **"EPG time offset (hours)"** setting (`-12` to `+12` hours) accessible in **Television → Configure Television**, persisting to `~/.config/kaffeinerc` (`EpgTimeOffset`).

---

## Quick Install (Pre-compiled Binary)

Download the latest release tarball from the [Releases](https://github.com/PlasmaDrifter/kaffeine-fixes/releases) page:

```bash
tar -xzf kaffeine-2.0.19-patched-linux-x86_64.tar.gz
cd kaffeine-2.0.19-patched-linux-x86_64
./install.sh
```

The installer copies `kaffeine` to `~/.local/bin/kaffeine` and creates the desktop launcher override at `~/.local/share/applications/org.kde.kaffeine.desktop`.

---

## Building from Source

### Dependencies
On Fedora / Nobara:
```bash
sudo dnf install -y cmake g++ make extra-cmake-modules \
    libdvbv5-devel vlc-devel qt5-qtbase-devel qt5-qtx11extras-devel \
    kf5-kio-devel kf5-ki18n-devel kf5-kcoreaddons-devel kf5-kwidgetsaddons-devel \
    kf5-kxmlgui-devel kf5-kwindowsystem-devel kf5-solid-devel kf5-kdbusaddons-devel
```

On Ubuntu / Debian:
```bash
sudo apt install -y cmake g++ extra-cmake-modules \
    libdvbv5-dev libvlc-dev qtbase5-dev libqt5x11extras5-dev \
    libkf5kio-dev libkf5i18n-dev libkf5coreaddons-dev libkf5widgetsaddons-dev \
    libkf5xmlgui-dev libkf5windowsystem-dev libkf5solid-dev libkf5dbusaddons-dev
```

### Build & Install
```bash
git clone https://github.com/PlasmaDrifter/kaffeine-fixes.git
cd kaffeine-fixes
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j$(nproc)
cp src/kaffeine ~/.local/bin/kaffeine
```

---

## License
Kaffeine is licensed under the [GNU General Public License v2 (GPL-2.0)](COPYING).
