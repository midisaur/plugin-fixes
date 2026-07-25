# iLok License Manager / PACE License Support 6.0.0 — Install Notes for Wine 11

What actually makes this work, with all the dead ends stripped out.
Target: `~/.wine-ableton` prefix on PikaOS 4 / Hyprland, wine-d2d1-nspa-11.11
(the ableton-wine Wine tree).

## What's wrong with the unmodified installer

`LicenseSupport.exe` from iLok.com is a 166 MB InstallShield self-extractor
wrapped around a WiX Burn bundle. The Burn bundle fails on Wine 11 with
`0x80070643` because two of its inner MSIs are incompatible with Wine:

1. The Windows-10 launch condition
   `Installed Or (VersionNT64 >= 603 and COMPATIBLE_OS_VERSION="True")`
   evaluates false in Wine (Wine reports OS version 6).
2. The `Wix4SchedServiceConfig_X86` action calls
   `ChangeServiceConfig2` with the `SERVICE_FAILURE_ACTIONS` flag,
   an API Wine does not implement.

The bundle's own logs show `Action ended ... INSTALL. Return value 0` for
every visible action, then Burn returns `0x80070643` and rolls everything
back. The MSI's `InstallFiles`, `RegisterProduct`, and service-install
actions never actually run.

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
export PATH=/home/jer/.local/opt/wine-d2d1-nspa-11.11/bin:$PATH
export WINEPREFIX=/home/jer/.wine-ableton
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
Exec=/home/jer/.local/bin/ilok %f
Type=Application
StartupNotify=true
Path=/home/jer/.wine-ableton
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
