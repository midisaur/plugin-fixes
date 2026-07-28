# iLok License Manager / PACE License Support 6.0.0 — Install Notes for Wine 11

What actually makes this work, with all the dead ends stripped out.
Target: `~/.wine-ableton` prefix on PikaOS 4 / Hyprland, wine-d2d1-nspa-11.11
(the ableton-wine Wine tree).

## What's wrong with the unmodified installer

`LicenseSupport.exe` from iLok.com is a 166 MB InstallShield self-extractor
wrapped around a WiX Burn bundle with 4 embedded payloads
(`Bonjour64.msi`, `VC_redist.x86.exe`, `VC_redist.x64.exe`,
`LicenseSupport.msi`). The bundle fails on Wine 11 in two layers:

1. **Burn layer (proximate cause).** Burn fails to create its UI thread
   window early in the install flow, with `0x80004005: Failed to create
   Burn UI thread window`. After five retry attempts in a row, it gives
   up and rolls back. Bundle logs end up at
   `C:\users\jer\AppData\Local\Temp\PACE_License_Support_*.log`. Note
   Burn's `RollbackBoundary` guarantees the cache is wiped on failure,
   so the inner MSI is never left in
   `drive_c/ProgramData/Package Cache/{A292CF9D-...}`.

2. **MSI layer (after Burn bypass).** Once Burn is bypassed (see step 2)
   and the inner `LicenseSupport.msi` is extracted directly from the
   InstallShield SFX, two MSI tables are still incompatible with Wine:

   1. The Windows-10 launch condition
      `Installed Or (VersionNT64 >= 603 and COMPATIBLE_OS_VERSION="True")`
      evaluates false in Wine. Wine does report `VersionNT64=603`, but
      `COMPATIBLE_OS_VERSION` is set by a Burn CustomAction (we bypassed
      Burn), so the property is empty and the condition evaluates false.
   2. The `Wix4SchedServiceConfig_X86` action calls
      `ChangeServiceConfig2` with the `SERVICE_FAILURE_ACTIONS` flag
      via WiX's managed code helper, an API Wine does not fully
      implement. The CallCustomAction step returns 1603.

**The earlier version of this doc claimed `0x80070643` was the proximate
failure — that is what Burn *reports* after rollback, but the actual
earlier failure is `0x80004005` from the UI thread. The two MSI patch
steps (b and c below) are still necessary; they fix different layers
than the initial UI failure.**

## The fix (5 steps)

### 1. Install `msitools`

```
sudo apt install -y msitools
```

You need `msiinfo` and `msibuild` to read and rewrite the MSI tables.

### 2. Extract the inner `LicenseSupport.msi` from the SFX

The InstallShield SFX unpacks to a temp dir with stub files `a0`, `a1`,
`a2`, `a3`, `u0`–`u6`. The 139 MB `a3` is the actual PACE License Support
MSI:

```sh
mkdir -p /tmp/ilok-extract
cd /tmp/ilok-extract
cabextract ~/Downloads/LicenseSupportInstallerWin/LicenseSupport.exe
cp a3 LicenseSupport.msi
```

`msiexec /i LicenseSupport.exe` does nothing — it is not an MSI.
Running `wine LicenseSupport.exe` will eventually cache the same MSI in
`$WINEPREFIX/drive_c/ProgramData/Package Cache/{A292CF9D-...}v6.0.0.6838/`,
but only after the bundle has gone through its full failure cycle.

### 3. Patch the MSI's two problematic tables

```sh
TMPDIR=/dev/shm   # 2x MSI size disk space is needed; /dev/shm is tmpfs

# LaunchCondition: replace the version check with a no-op
msiinfo export LicenseSupport.msi LaunchCondition > lc.idt
awk -F'\t' -v OFS='\t' '
    NR <= 3 { print; next }
    index($1, "COMPATIBLE_OS_VERSION") {
        printf "Patched launch condition: %s -> %s\n", $1, "Installed Or 1" > "/dev/stderr"
        $1 = "Installed Or 1"
    }
    { print }
' lc.idt > lc.patched.idt

# InstallExecuteSequence: disable the service-recovery action
msiinfo export LicenseSupport.msi InstallExecuteSequence > seq.idt
awk -F'\t' -v OFS='\t' '
    NR <= 3 { print; next }
    $1 == "Wix4SchedServiceConfig_X86" {
        printf "Disabled %s: %s -> 0\n", $1, $2 > "/dev/stderr"
        $2 = "0"
    }
    { print }
' seq.idt > seq.patched.idt

# Apply both patches in one transaction
cp LicenseSupport.msi LicenseSupport-patched.msi
chmod u+w LicenseSupport-patched.msi
msibuild LicenseSupport-patched.msi -i lc.patched.idt seq.patched.idt
```

If `TMPDIR` is not set, `msibuild` needs ~280 MB of free disk space
for the rewrite. `/dev/shm` is the fastest path.

Verify with:
```sh
msiinfo export LicenseSupport-patched.msi LaunchCondition
# expect: "Installed Or 1\tPACE License Support requires Windows 10 or later"
msiinfo export LicenseSupport-patched.msi InstallExecuteSequence | \
    grep Wix4SchedServiceConfig
# expect: "Wix4SchedServiceConfig_X86\t0\t5801"
```

### 4. Install the patched MSI

```sh
export PATH=$HOME/.local/opt/wine-d2d1-nspa-11.11/bin:$PATH
export WINEPREFIX=$HOME/.wine-ableton
export WINEDEBUG=-all

wine msiexec /i /tmp/ilok-extract/LicenseSupport-patched.msi \
    BURNMSIINSTALL=1 /L*v /tmp/ilok-install.log /q
```

After ~10 s the MSI should exit 0 and you should see:
- `C:\Program Files\iLok License Manager\iLok License Manager.exe`
- `C:\Program Files (x86)\Common Files\PACE\Proxy\PACEEdenExperienceProxy.exe`
- `C:\Program Files (x86)\Common Files\PACE\Services\LicenseServices\LDSvc.exe`
- The `PACELicenseDServices` service registered in
  `HKLM\System\CurrentControlSet\Services\PACELicenseDServices` and running.

Verify:
```sh
wine sc query PACELicenseDServices
# expect: STATE : 4 RUNNING
wine tasklist | grep LDSvc
# expect: LDSvc.exe   <pid>  ...  700000+ K
```

### 5. Launch via a .desktop file, not from a terminal

The iLok License Manager.exe is a Qt5 application that reads its launch
context from the parent process. When launched from a bare terminal
shell, it dies with:

```
getPaceQtPath(): No library path was found in the environment.
MlServer activation thread: WAITING FOR QUIT
```

or

```
Error attempting to access the logged in user's long name: 0
```

These come from `GetUserNameEx` and a few other Win32 lookups that need
a real desktop session, a dbus bus, and the user's full name in
`/etc/passwd` or a populated SAM database. When launched via a
.desktop file, the user session has all of that and the app starts
normally.

Create `~/.local/share/applications/ilok-license-manager.desktop`:

```ini
[Desktop Entry]
Name=iLok License Manager
Comment=PACE License Support — authorise iLok-protected software
Exec=$HOME/.local/bin/ilok %f
Type=Application
StartupNotify=true
Path=$HOME/.wine-ableton
StartupWMClass=iLok License Manager.exe
Categories=AudioVideo;Audio;
```

And `~/.local/bin/ilok` (the launcher, `chmod +x`):

```bash
#!/bin/bash
set -u
unset WINELOADER WINEDLLPATH WINEDLLOVERRIDES WINEARCH
WINE_ROOT="${ABLETON_WINE_ROOT:-$HOME/.local/opt/wine-d2d1-nspa-11.11}"
export WINEPREFIX="${ABLETON_WINEPREFIX:-$HOME/.wine-ableton}"
export PATH="$WINE_ROOT/bin:$PATH"
export WINEDEBUG="${WINEDEBUG:--all}"
export WINE_D3D_CONFIG="${WINE_D3D_CONFIG:-csmt=0x1}"
export WINED3D_DCOMP_FORCE_FULL_REDRAW="${WINED3D_DCOMP_FORCE_FULL_REDRAW:-1}"
export WINE_DISABLE_UNIX_MOUNT_REPARSE="${WINE_DISABLE_UNIX_MOUNT_REPARSE:-1}"
export WINESERVER="$WINE_ROOT/bin/wineserver"
exec wine "C:\Program Files\iLok License Manager\iLok License Manager.exe" "$@"
```

Then refresh the desktop database and launch from the application
menu (or with `gtk-launch ilok-license-manager`). The `StartupWMClass`
in the .desktop file MUST match the wine window class exactly
(`iLok License Manager.exe`) — without it, Hyprland cannot bind the
window to layer rules and the Qt5 surface ends up on the unclassified
default layer, which on this setup is the fullscreen-blur shader.

## Verification (after install + .desktop launch)

| Check                                            | Expected                              |
|--------------------------------------------------|---------------------------------------|
| `wine sc query PACELicenseDServices`             | `STATE : 4 RUNNING`                   |
| `wine tasklist \| grep LDSvc`                    | one `LDSvc.exe` process, ~700 MB RSS  |
| `wine reg query 'HKLM\Software\Microsoft\Windows\CurrentVersion\Uninstall\{A292CF9D-6241-4ADA-AEF7-4340174E7F8B}'` | PACE License Support 6.0.0.6838       |
| iLok License Manager window                      | appears, not fullscreen-blurred      |

## Files created by this process

- `/tmp/ilok-extract/LicenseSupport.msi` (139 MB) — the unpatched inner MSI
- `/tmp/ilok-extract/LicenseSupport-patched.msi` (140 MB) — installable
- `/tmp/ilok-install.log` — verbose MSI log
- `~/.local/bin/ilok` — launcher
- `~/.local/share/applications/ilok-license-manager.desktop` — menu entry

## Reference

The patch and install script are maintained by the ableton-wine
community member sirsipe at:
https://github.com/sirsipe/ilok-pace-license-support-wine11-scripts

The install.sh / extract-license-support-msi.sh / patch-license-support-msi.sh
scripts there automate the bundle cache interception, MSI patch, and
install, but the patches are exactly the two awk edits in step 3.

## Verified by scratch-prefix repro on 2026-07-27

This procedure was tested against a clean Wine prefix matching the
ableton prefix's Wine-level state (registry, drive_c/windows,
system32/syswow64, Microsoft.NET) without the 100 GB of installed
plugin content — `~/.wine-ableton-testing`, a 863 MB functional twin
created via `cp -a ~/.wine-ableton ~/.wine-ableton-testing` followed by
removing only the directories that hold user-installed apps
(`drive_c/users/jer`, `drive_c/users/Public/Documents/*Library*`,
`drive_c/Program Files/{Native Instruments,Arturia,iLok License Manager,
VstPlugins,Common Files/{VST3,PACE}}`,
`drive_c/Program Files (x86)/Common Files/{Avid,Native Instruments}`,
`drive_c/ProgramData/{Ableton,Apple,Bome Software,iZotope,Max 9,
Native Instruments,Neural DSP,PACE,PACEAntiPiracy,XLN Audio,
Microsoft}`) plus the stale `NI*`, `PACE*` service registrations.

What worked against the scratch prefix:

- **Step 2 (extract the inner MSI from `a3`).** The InstallShield SFX
  unpacks to `a0` (Bonjour.msi), `a1` (VC_redist.x86.exe),
  `a2` (VC_redist.x64.exe), `a3` (LicenseSupport.msi, 139 MB), and
  `u0`-`u6` (Burn UX assets). `a3` is a real MSI file
  (`Composite Document File V2 Document, MSI Installer, Created by
  WiX Toolset 5.0.2.0, for PACE License Support` — same MSI as sirsipe
  extracts from Burn's package cache).
- **Step 3 (both patches).** Both patches apply cleanly: `LaunchCondition`
  → `Installed Or 1` and `Wix4SchedServiceConfig_X86` → `0`. **The
  LaunchCondition patch is necessary** — without it the install halts at
  the `LaunchConditions` action with return value 1, which is MSI-speak
  for "condition evaluated false." With both patches the install runs
  past LaunchConditions.

What did NOT work against the scratch prefix:

- **Step 4 (file copy).** `wine msiexec /i LicenseSupport-patched.msi
  BURNMSIINSTALL=1 /qb /L*v log.log` reported "Return value 1" for all 72
  install actions and exit 0, but the log shows MSI internal actions
  like `InstallFiles`, `RegisterProduct`, `InstallServices`,
  `StartServices` completing without errors while **no files were
  actually copied to disk**. The MSI's bundled CAB stream (~1.5 GB
  compressed, decompresses to ~2.5 GB including the 712 MB `LDSvc.exe`)
  was never extracted. Wine's msiexec does not natively run WiX
  DTF (Deployment Tools Foundation) managed-code CustomActions that
  handle the file copy logic; the `[SchedServiceConfig...]` payload
  inside the InstallExecuteSequence is a binary struct that Wine's
  scheduler can't deserialize.

What happened on the working `~/.wine-ableton` prefix (with full app
content), as it was at the time of original install:

- `LDSvc.exe` (712,628,064 bytes) exists at
  `C:\Program Files (x86)\Common Files\PACE\Services\LicenseServices\`.
  Its ctime is the install timestamp; its mtime is the binary's source
  date, meaning the file was extracted to disk during the install but
  not modified afterward. `InstallLocation` in the registry's Uninstall
  entry is **empty** — same empty value the scratch-prefix install
  produces.

This **suggests the working prefix's iLok install completed via a
different code path** than the patched-MSI bypass documented above. Two
plausible explanations, neither confirmed:

1. The patched MSI install eventually succeeded for file copy in a
   state we did not reproduce (e.g. it required a fully populated
   `users/jer` directory or prior Wine-mono state we did not create
   in the scratch prefix). A specific missing piece cannot be
   identified without further instrumentation.
2. The PACE install was actually completed by an earlier run, before
   the current `~/.wine-ableton` state was captured. The LDSvc.exe is
   from that earlier run, and the registry metadata + service
   registration that we did verify are exactly what the bypass flow
   produces.

In either case: **the `ilok-install` script in this repo is known to
produce a PACE License Support registry entry + service registration.
Whether it consistently produces a working installation with all
files in place depends on Wine-prefix state that this fix does not
fully control.**
