# Running Staxel (405710) on Linux with this fork

## The issue

Staxel's FNA/SDL + CefSharp subclassing keeps the window procedure Wine hands it
in a 32-bit field. Wine's `get_winproc_ptr()` only recognises a winproc handle
(`0xffff00xx`) with its high half zero, so the sign-extended form
(`0xffffffffffffff00xx`) is not recognised and is executed as a function pointer.
Every window message then raises `EXCEPTION_ACCESS_VIOLATION`: clicks are ignored
and the process wedges.

```
EXCEPTION_ACCESS_VIOLATION addr=FFFFFFFFFFFF0012   # handle 0xffff0012, sign-extended
EXCEPTION_ACCESS_VIOLATION addr=FFFFFFFFFA641CB0   # procedure 0x6ffffa641cb0, truncated to 32 bits
```

## How it is fixed

The fix lives in `patches/wine-hotfixes/winproc-signext/` in this repository, and
is applied with the rest of the patch stack by
`patches/protonprep-valve-staging.sh`, so a build from this repository includes
it.

- `get_winproc_ptr()` accepts the sign-extended handle as the same handle.
- a procedure truncated to 32 bits is recovered from Wine's winproc table by its
  low 32 bits.
- `get_winproc_params()` falls back to the other A/W variant instead of passing
  the raw handle to the callback, and `get_winproc()` keeps returning the handle,
  the only form that survives a 32-bit round trip.

Full write-up: `patches/wine-hotfixes/winproc-signext/README.md`.

## Requirements

- Linux x86_64, Docker (or podman) and GNU Make. The build runs inside the
  SteamRT 4 SDK container, which is pulled on the first build (~7 GB).
- The patch step runs on the host, so `git`, `patch` and `python3` are needed there too.
- ~50 GB free disk, around 30 minutes per full build.
- Steam with Staxel installed (appid 405710).

The commands use the build name `GE-Proton-Staxel`. Substitute any name, but use
it consistently: the install path and the Steam compatibility-tool list must
match.

## Build

```sh
git clone --recurse-submodules https://github.com/99kennetn/proton-ge-custom
cd proton-ge-custom

./patches/protonprep-valve-staging.sh &> patchlog.txt
grep -iE 'does not apply|FAILED' patchlog.txt   # must print nothing
```

The patch step edits the wine submodule in place, so `git -C wine status` shows
`dlls/win32u/class.c` as modified.

```sh
mkdir build && cd build
../configure.sh --build-name=GE-Proton-Staxel
make redist &> log
```

The build lands in that `build/` directory as `GE-Proton-Staxel.tar.gz`.

## Install

```sh
mkdir -p ~/.steam/steam/compatibilitytools.d
tar -xf build/GE-Proton-Staxel.tar.gz -C ~/.steam/steam/compatibilitytools.d/
```

`~/.local/share/Steam/compatibilitytools.d/` is the same directory on most
installations. Restart Steam so the build appears in the compatibility-tool list.

## Steam settings

1. Staxel → Properties → Compatibility → **Force the use of a specific Steam Play
   compatibility tool** → the name you built (`GE-Proton-Staxel`)
2. Launch options: none needed. This fork turns Xalia off for appid 405710 in
   `proton` itself. If the game still misbehaves, add `PROTON_USE_XALIA=0
   %command%` as a fallback.

Diagnostic run (writes a large `~/steam-405710.log`):

```
PROTON_LOG=+win,+msg %command%
```

## Before the first launch: .NET (one-time, manual)

Staxel's launcher and assemblies need a real .NET Framework CLR. Wine's built-in
Mono is not enough: without a CLR the launcher reports
`Mono runtime detected, dotnet clr recommended` and the client exits on a Mono-only
entry point. This fork does not install it for you. Install it once into the game's
prefix, with the game closed:

```sh
protontricks 405710 dotnet48
```

That step is slow (10-20 minutes) and occasionally does not finish on the first
run — run it again if the check below fails. Installed verbs are recorded in the
prefix's `winetricks.log`, so repeating the command later costs nothing.

Check the result — this file must exist:

```sh
ls ~/.local/share/Steam/steamapps/compatdata/405710/pfx/drive_c/windows/Microsoft.NET/Framework64/v4.0.30319/clr.dll
```

The easiest way to be able to run the install for dotnet is to run the game once to build the compatdata folder (it will fail to launch), and then run the command
```sh
protontricks 405710 dotnet48
```

Once it is in place, do not delete `~/.local/share/Steam/steamapps/compatdata/405710`:
it holds that CLR, and winetricks' `dotnet48` begins with `remove_mono`, which
takes it away again.

## Confirming it works

From the repository root, the patch reached the wine sources:

```sh
grep -c 'fixup_truncated_winproc' wine/dlls/win32u/class.c
# expect 5: the definition plus its four call sites (only after the patch step above)
```

The patch is in the installed build:

```sh
strings ~/.steam/steam/compatibilitytools.d/GE-Proton-Staxel/files/lib/wine/x86_64-unix/win32u.so \
  | grep -c 'recovering 32-bit truncated proc'
# expect 1: the warning string compiled into win32u
```

A session runs without the crash (`~/steam-<appid>.log` is written when the
diagnostic launch options above are used):

```sh
grep -c EXCEPTION_ACCESS_VIOLATION ~/steam-405710.log
# expect 0
```

The prefix has a real CLR:

```sh
grep -c dotnet48 ~/.local/share/Steam/steamapps/compatdata/405710/pfx/winetricks.log
# expect 1 or more
```

With `PROTON_LOG=+win,+msg`, that log also shows the fix working:
`recovering 32-bit truncated proc ...` — roughly 30,000 times per session, all
resolved.

## Expected noise, not a problem

- If SteamVR is installed (and registered as the OpenXR runtime, which its
  installer does), two `rundll32.exe` "This application could not be started"
  dialogs appear at launch: SteamVR's OpenXR probe, which finds no headset
  (`~/.local/share/Steam/logs/xrclient_rundll32.txt`). The game starts and runs
  normally, and with no VR runtime installed this does not happen.
- Staxel asks whether to send logs when you quit. Its report says
  "No immediate issues were found."
