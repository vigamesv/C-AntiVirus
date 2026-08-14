# C++AntiVirus

A minimal, cross-platform (Windows + Linux) desktop scanner that walks a
folder you choose, hashes every file (SHA-256), and checks each hash against
[VirusTotal](https://docs.virustotal.com/reference/overview)'s existing
database. Detections show full per-engine detail: which of VirusTotal's 70+
AV engines flagged the file, what they called it, and their detection
method/definitions date.

**Important — how it actually works:** this tool does **hash-only lookups**.
It never uploads your files' contents to VirusTotal — only a SHA-256
fingerprint. That's fast and keeps your data private, but it means:

- Files VirusTotal has never seen before come back as "unknown to VT," not
  "clean." This is not a substitute for a real, always-on antivirus (Windows
  Defender, etc.) — it's a good second-opinion / spot-check tool.
- The free "Public" API key is limited to **4 requests/minute and 500/day**.
  Scanning a folder with thousands of files will take a while and/or hit the
  daily cap — the app self-throttles to respect this, configurable in the UI.
- Get a free API key at virustotal.com (sign up → your profile icon → API
  key). Paste it into the app; it's stored locally in your user config
  directory and only ever sent to `virustotal.com`.

## What's in this repo

```
CMakeLists.txt      CMake build (fetches ImGui/GLFW/json/tinyfd automatically)
build.sh             One-command CMake build wrapper (Linux/macOS)
build_gxx.sh         g++-only build (no CMake) - auto-locates the source,
                      auto-fetches all dependencies, and produces a single
                      C++AntiVirus.exe
src/
  c++antivirus.cpp      The entire application: SHA-256, config persistence,
                      HTTP client, VirusTotal API client, background
                      directory scanner, and the Dear ImGui UI/app loop
```

The app is intentionally kept in one source file for easy reading and
editing. Third-party libraries (Dear ImGui, GLFW, nlohmann/json,
tinyfiledialogs) are pulled in automatically — via CMake's `FetchContent` if
you build with CMake, or via `git clone`/direct download if you use
`build_gxx.sh`. The one dependency you **do** need locally either way is
**libcurl** (or GLFW, if not already present for your toolchain).

## Build — Windows, using only g++ (no CMake, no vcpkg, no MSVC)

If you'd rather not install CMake or Visual Studio at all, `build_gxx.sh`
drives the entire build with a single `g++` invocation. Run it from a Bash
shell (MSYS2 MinGW64, or Git Bash with a MinGW-w64 toolchain on your PATH).

**One-time, if you're on MSYS2:**
```bash
pacman -S --needed git mingw-w64-x86_64-toolchain mingw-w64-x86_64-glfw
```
(libcurl is fetched automatically by the script — no package needed for it.)

**If you're using a standalone MinGW-w64 toolchain instead of MSYS2**, make
sure `g++` and `git` are on your PATH, and that a GLFW dev package is
available for it; libcurl is handled automatically either way.

**Then:**
```bash
chmod +x build_gxx.sh
./build_gxx.sh
```

The script auto-locates `src/vtscanner.cpp` (even on an external/USB drive),
vendors in ImGui, nlohmann/json, and tinyfiledialogs via `git clone`,
downloads the official prebuilt libcurl-for-MinGW package, and compiles
everything with one `g++` command. Output: `C++AntiVirus.exe`, plus one
`libcurl-*.dll` that must stay next to it (curl is linked dynamically;
everything else is statically linked into the exe). If you'd rather point it
at a specific source file, pass the path directly:
```bash
./build_gxx.sh /path/to/src/vtscanner.cpp
```

## Build — Windows, using CMake

The easiest path is MSVC + [vcpkg](https://github.com/microsoft/vcpkg) for
libcurl, which gets you a single, dependency-free `.exe`:

```powershell
git clone https://github.com/microsoft/vcpkg
.\vcpkg\bootstrap-vcpkg.bat
.\vcpkg\vcpkg install curl:x64-windows-static

cmake -B build -DCMAKE_BUILD_TYPE=Release ^
      -DCMAKE_TOOLCHAIN_FILE=vcpkg\scripts\buildsystems\vcpkg.cmake ^
      -DVCPKG_TARGET_TRIPLET=x64-windows-static
cmake --build build --config Release
```

The result is one file, no DLLs to ship alongside it (the CMakeLists.txt
already statically links the MSVC runtime via `CMAKE_MSVC_RUNTIME_LIBRARY`
and links `curl:x64-windows-static` statically). Double-click it to run; no
installer/setup step required.

**MinGW alternative (still via CMake):** if you'd rather not use MSVC,
install [MSYS2](https://www.msys2.org/), then in a MinGW64 shell:

```bash
pacman -S mingw-w64-x86_64-toolchain mingw-w64-x86_64-cmake \
          mingw-w64-x86_64-curl mingw-w64-x86_64-glfw
cmake -B build -G "MinGW Makefiles" -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
```

The CMakeLists.txt passes `-static -static-libgcc -static-libstdc++` for
MinGW builds so the resulting `.exe` doesn't need MinGW runtime DLLs either.

## Build — Linux

```bash
sudo apt update
sudo apt install build-essential cmake libcurl4-openssl-dev \
                  libgl1-mesa-dev libx11-dev libxrandr-dev \
                  libxinerama-dev libxcursor-dev libxi-dev

./build.sh
./build/vtscanner
```

`build.sh` is just a thin wrapper around `cmake -B build && cmake --build
build`, so you never have to remember the exact CMake invocation. It
produces a single executable that dynamically links against standard system
libraries (libc, libcurl, libGL, X11) present on essentially every desktop
Linux install, so you can copy just that one binary to another machine with
the same libraries and run it directly — no installer needed. If you want a
fully self-contained artifact to hand to someone else, wrap it with
`linuxdeploy` or build an AppImage; that's an optional extra step, not
required to run it yourself.

## Usage

1. Launch `C++AntiVirus.exe` (Windows) or `vtscanner` (Linux).
2. Paste your VirusTotal API key.
3. Click **Browse...** and pick a folder (e.g. `Downloads`, `C:\Users\you`,
   or your whole home directory — bigger folders just take longer and use
   more of your daily API quota).
4. Click **Start Scan**. Progress and live detection count show at the top;
   results stream into the table below as each file is checked.
5. Click **Details** on any flagged file to see the full per-engine
   breakdown (engine name, category, exact detection name, definitions
   date) and a link to the file's full VirusTotal report.

Settings (API key, last folder, requests/minute) are remembered between
runs.

## Notes / honest limitations

- This was written and reviewed carefully but **not compiled/run in a full
  desktop GUI environment** as part of producing it — build it locally with
  the commands above before relying on it. The code follows the standard
  Dear ImGui + GLFW + OpenGL3 example pattern exactly, so it should build
  cleanly with the dependency versions pinned in `CMakeLists.txt`.
- Hash-only lookups mean zero-day/never-seen-before malware won't be
  flagged. Consider this a complement to, not a replacement for, real-time
  antivirus.
- Respect VirusTotal's [terms of service](https://www.virustotal.com/gui/terms-of-service)
  and rate limits — the built-in limiter defaults to the Public API's 4
  requests/minute.
