# Native Access 2 — Install Notes for Wine 11

How to get Native Access 2 installed and the dep screen unstuck
in a single Wine prefix, using only the bundled installers plus
`wine sc start`.

Target: any `~/.wine-ableton`-style prefix on Linux + Hyprland,
`wine-d2d1-nspa-11.11` (the `ableton-linux` Wine tree).

## Two obstacles to be aware of

The installer has two distinct preflight hooks that show up in
Wine debug output. On `wine-d2d1-nspa-11.11`, only the second
one is actually blocking; the first one is a non-issue (despite
what some Wine-on-Linux tutorials claim).

### 1. NSIS installer's `powershell.exe` preflight (diagnostic, not a fix)

For reference only. The NA NSIS installer runs a `powershell.exe
-C` preflight **before** copying any files. The check is:

```
powershell.exe -NoProfile -Command
  if ((Get-CimInstance -ClassName Win32_Process |
       ? {$_.Path -and $_.Path.StartsWith(
              'C:\Program Files\Native Instruments\Native Access',
              'CurrentCultureIgnoreCase')}).Count -gt 0)
  { exit 0 } else { exit 1 }
```

In Wine debug output (`WINEDEBUG=+loaddll,+seh,+winediag`),
this shows up as:

```
fixme:powershell:wmain stub.
fixme:powershell:wmain argv[0] L"C:\\windows\\system32\\...\\powershell.exe"
fixme:powershell:wmain argv[1] L"-C"
fixme:powershell:wmain argv[2] L"if ((Get-CimInstance ... StartsWith ..."
```

The same preflight fires **5+ times in rapid succession** when
NA's installer is launched as `wine Native-Access_2.exe`. Wine's
loader-stub dispatcher routes bare-name `powershell.exe`
invocations through `wineps.dll!wmain`, which returns 0 for
every argument set.

**This is not an installer loop**, despite the rapid-fire
`wmain` warnings. The NSIS installer continues past the
preflight, copies all 223 MB of NA's Electron runtime, and
exits normally on `wine-d2d1-nspa-11.11`. The preflight is just
a "kill any running instance before upgrading" check, and the
installer treats "no instance running" as fine.

**Nothing was done about the powershell stub.** No shim, no
`DllOverrides` entry, no on-disk Wine-tree modification. The
28 KB loader stub that ships with Wine 11 is left in place. If
you see `wmain` warnings during install, that's normal Wine
output and not an error.

### 2. First-run dep screen (blocking — this is the real fix)

After NA installs, on first launch it tries to talk to a
running `NTKDaemon.exe` over ZMQ (local IPC) to read the daemon
version. If none is running, NA falls back to `wmic datafile
... get Version /value` (Wine has no `wmic`), then tries to
install via `sudo-prompt` (Wine has no UAC), and you get "User
did not grant permission." Net effect: NA sits on the "Please
grant permission to install dependencies" screen.

This is the only installer obstacle that actually blocks NA
from starting.

## Install

### 1. Install NTKDaemon directly, not through NA

The Inno Setup installer bundled inside the NA bundle
(`resources/daemon/win/NTKDaemon 1.31.1 Setup PC.exe`)
registers `NTKDaemonService` as a real Windows service when run
directly. NA's bundled `elevate.exe` + UAC wrapper does not
work in Wine; bypass it by running the Inno Setup installer
manually.

```sh
WINEPREFIX=~/.wine-ableton wine \
    "$WINEPREFIX/drive_c/Program Files/Native Instruments/Native Access/resources/daemon/win/NTKDaemon 1.31.1 Setup PC.exe"
```

Accept the EULA prompt in the Inno UI. The installer copies
`NTKDaemon.exe` + `crashpad_handler.exe` to `C:\Program Files\Common
Files\Native Instruments\NTK\` and registers
`NTKDaemonService` in the SCM database. No reboot needed.

Skip this step if `wine sc query NTKDaemonService` already shows
the service registered.

### 2. Run the NA installer

```sh
WINEPREFIX=~/.wine-ableton wine $HOME/Downloads/Native-Access_2.exe
```

Copies `C:\Program Files\Native Instruments\Native Access\Native
Access.exe` (223 MB) plus the Electron runtime, then exits. NA does
not need to be running during this step.

### 3. Launch NA via `~/.local/bin/native-access`

```sh
WINEPREFIX=~/.wine-ableton ~/.local/bin/native-access
```

The launcher script chains two things in one process:

1. `wine sc start NTKDaemonService` — registers NTKDaemon with the
   SCM, starts it, and verifies `STATE : 4 RUNNING`. NA's ZMQ check
   then finds the daemon (NAT's IPC port registered by
   NTKDaemonService), gets the version, and skips the
   install/permission flow.
2. `wine 'C:\Program Files\Native Instruments\Native Access\Native Access.exe'`
   — runs NA in the foreground so the launcher script stays alive
   for the duration of the NA session. Exits cleanly when NA closes.

The launcher sets the env vars NA needs under Wayland
(`WINE_D3D_CONFIG=csmt=0x1`, `WINE_DISABLE_UNIX_MOUNT_REPARSE=1`,
etc.) and pins the Wine tree to
`~/.local/opt/wine-d2d1-nspa-11.11/bin/wine` (the system `wine`
on PATH may resolve to a different build and would corrupt the
prefix's running state).

**Override the prefix** with `NATIVE_ACCESS_WINEPREFIX=/path/to/prefix`
if NA isn't installed in the default `~/.wine-ableton`.

The service stays registered in the SCM database across NA
launches (shared wineserver), so subsequent launches are
instant. If the service disappears (e.g. after `wineserver -k` or
rebuilding the prefix), re-run the Inno Setup installer in
step 1 once to re-register it.

Wire the launcher into a `.desktop` file for a menu entry:

```ini
[Desktop Entry]
Type=Application
Name=Native Access
Exec=/home/jer/.local/bin/native-access
StartupWMClass=native access.exe
Categories=AudioVideo;Audio;
```

The `StartupWMClass` MUST match the lowercase exe filename with
spaces preserved (verified via `hyprctl clients -j`); otherwise
Hyprland can't bind the window and NA lands in the unclassified
default layer, picking up the wrong window rule.

## Install plugins through NA

After the library loads, install plugins normally through NA's
UI. Battery 4, Komplete Kontrol, Massive X, FM8, Reaktor, etc.
all complete the install through NA cleanly. The single exception
is Kontakt 8 Player; see `Kontakt-8-Wine-Install.md`.

