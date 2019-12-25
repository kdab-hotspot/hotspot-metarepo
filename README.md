# KDAB Hotspot Profiler Cross-platform build system

We unofficially bring KDAB's awesome Hotspot to MacOS and Windows.

Current status: only builds `perfparse` on Linux or MacOS; hotspot GUI needs more work on KDE bundling.

## Checking out the sources

```
git clone --recurse-submodules https://github.com/kdab-hotspot/hotspot-metarepo.git
```

## Building on Linux

Prerequisites for Qt XCB plugin support:

```
sudo apt install libfontconfig1-dev libfreetype6-dev libx11-dev libxext-dev libxfixes-dev libxi-dev libxrender-dev libxcb1-dev libx11-xcb-dev libxcb-glx0-dev libxkbcommon-x11-dev libxcb-xinput-dev libxcb-xinerama0-dev libxcb-keysyms1-dev libxcb-icccm4-dev libxcb-render-util0-dev libxcb-image0-dev
```

```
mkdir build
cd build
cmake -G "Unix Makefiles" -DCMAKE_BUILD_TYPE=Release ..
cmake --build . -- -j4
```

## Building on MacOS

TODO

## Building on Windows

```
mkdir build
cd build
cmake -G "Visual Studio 16 2019" -A Win64 ..
cmake --build . --config Release
```

