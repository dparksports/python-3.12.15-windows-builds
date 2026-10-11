# Python 3.12.15 — Windows 10/11 Builds

This repository contains the complete, unmodified [CPython](https://github.com/python/cpython)
3.12.15 source tree together with everything needed to reproduce native Windows
builds of it, and publishes ready-to-use portable ZIP packages on the
[Releases](../../releases) page.

> **These are community builds, not the official python.org installers.** The
> source is unmodified; the binaries were produced with the toolchain described
> below. Licensed under the [Python Software Foundation License](LICENSE).

---

## Downloads

| Package | For Windows 10/11… | Notes |
|---|---|---|
| `python-3.12.15-windows-x64.zip` | x64 (Intel/AMD 64-bit) | Native. Also runs on Windows on ARM via emulation. |
| `python-3.12.15-windows-x86.zip` | x86, x64, ARM64 | 32-bit build; runs everywhere via native/emulated 32-bit support. |
| `python-3.12.15-windows-arm64.zip` | ARM64 (Snapdragon etc.) | Native ARM64. *Requires the ARM64 cross-compiler to build — see [ARM64 note](#arm64-note).* |

### Using a package

1. Extract the ZIP to any folder (e.g. `C:\Python312`).
2. Run `python.exe` from that folder. No installer, no registry, no admin
   rights needed — fully portable.
3. pip is included. To upgrade it and install packages:

   ```
   python -m pip install --upgrade pip
   python -m pip install <package>
   ```

Each package contains `python.exe`, the standard library (`Lib\`), extension
modules (`DLLs\`), Tcl/Tk 8.6 + IDLE, sqlite3, OpenSSL, and the headers/libs
(`include\`, `libs\`) for building C extensions.

---

## Reproducing the build

Built and verified on Windows 11 x64 (October 2026). The whole process takes
about 15 minutes on a modern machine.

### Prerequisites

- **Windows 10 or 11 x64** host (any edition).
- **Visual Studio 2026 Community** (or **Visual Studio 2022 17.x**) with:
  - Workload: **Desktop development with C++**
  - Component: **MSVC v143/v145 — C++ x64/x86 build tools**
  - Component: **Windows 10/11 SDK** (10.0.26100 was used)
  - *For ARM64 only:* component **MSVC v145 — C++ ARM64 build tools**
    (Individual components → search "ARM64")
- **Git for Windows** (used to fetch the bundled third-party libraries).

No other tools are required — the build bootstraps itself.

### Step 1 — Get the source

```
git clone <this repo>
cd python-3.12.15-windows-builds
```

### Step 2 — Fetch the external dependencies

From the repo root:

```
PCbuild\get_externals.bat
```

This downloads pre-built/pre-pinned copies of bzip2, xz, zlib, sqlite,
libffi, OpenSSL and Tcl/Tk into `externals\` (a few hundred MB). It only needs
Git and an internet connection.

### Step 3 — Build

From the repo root, in **cmd** (not PowerShell, to keep the env syntax exact):

**With Visual Studio 2022** nothing special is needed:

```
PCbuild\build.bat -p x64   -c Release
PCbuild\build.bat -p Win32 -c Release
PCbuild\build.bat -p ARM64 -c Release
```

**With Visual Studio 2026** two environment variables are currently required,
because the CPython 3.12 project files predate VS 2026:

```bat
rem 1. The projects' ToolsVersion=15.0 makes MSBuild 18 default to the
rem    obsolete v140 (VS2015) toolset. Point it at the installed toolset:
set PlatformToolset=v145

rem 2. If your VS install is fully registered you can skip this one; it
rem    bypasses the vswhere lookup when it is not:
set MSBUILD=C:\Program Files\Microsoft Visual Studio\18\Community\MSBuild\Current\Bin\MSBuild.exe

PCbuild\build.bat -p x64   -c Release
PCbuild\build.bat -p Win32 -c Release
PCbuild\build.bat -p ARM64 -c Release
```

Build outputs land in `PCbuild\amd64\`, `PCbuild\win32\` and `PCbuild\arm64\`
respectively. Build time is 1–2 minutes per architecture. The build is
incremental — re-running after a source change only recompiles what changed.

Two warnings are expected and harmless:
- `warning : Toolset v145 is not used for official builds` — v145 (MSVC 14.5x)
  is binary-compatible with the v143 used for official builds; the test suite
  passes identically.
- `git describe … exited with code 128` — only appears when the source is not
  a git checkout with tags; it just omits the git tag from the version banner.

### Step 4 — Verify

```
PCbuild\amd64\python.exe -c "import sys, sqlite3, ssl, ctypes, tkinter; print(sys.version)"
PCbuild\amd64\python.exe -m test -j8 test_sqlite3 test_ssl test_ctypes test_hashlib test_zlib test_bz2 test_lzma
```

### Step 5 — Package the portable ZIPs

`PC\layout` (run with the freshly built interpreter) assembles a
full, self-contained layout:

```
PCbuild\amd64\python.exe PC\layout -s . -b PCbuild\amd64 --arch amd64 ^
    --include-stable --include-pip --include-tcltk --include-idle --include-venv --include-dev ^
    --zip dist\python-3.12.15-windows-x64.zip

PCbuild\win32\python.exe PC\layout -s . -b PCbuild\win32 --arch win32 ^
    --include-stable --include-pip --include-tcltk --include-idle --include-venv --include-dev ^
    --zip dist\python-3.12.15-windows-x86.zip
```

(Use the ARM64 equivalents once the ARM64 build exists — see below.)

### <a name="arm64-note"></a>ARM64 note

Cross-compiling ARM64 on an x64 host requires the **MSVC v145 — C++ ARM64
build tools** VS component (it is *not* installed by the default
"Desktop development with C++" workload). Add it via
*Tools → Get Tools and Features → Individual components* (or select it in the
installer), then run `PCbuild\build.bat -p ARM64 -c Release` as above.

---

## Toolchain used for the published binaries

| Component | Version |
|---|---|
| Source | CPython 3.12.15 (unmodified official source tarball) |
| Compiler | Microsoft (R) C/C++ Optimizing Compiler 14.51.36231 (toolset v145) |
| MSBuild | 18.10.1 (Visual Studio 2026 Community) |
| Windows SDK | 10.0.26100.0 |
| Tcl/Tk | 8.6.15.2 |
| SQLite | 3.49.1 |
| OpenSSL | 3.0.16 (11 Feb 2025) |
| libffi | 3.4.4 |
| bzip2 / zlib / xz | 1.0.8 / 1.3.1 / 5.2.5 |

Version banner of the built interpreter:

```
Python 3.12.15 (main, Oct 10 2026, 21:16:55) [MSC v.1951 64 bit (AMD64)] on win32
```

## License

Python is Copyright © 2001-2026 Python Software Foundation and is distributed
under the [PSF License](LICENSE) (see also `Misc/` for third-party notices,
including the licenses of the bundled OpenSSL, Tcl/Tk, SQLite, libffi, bzip2,
zlib and xz components).
