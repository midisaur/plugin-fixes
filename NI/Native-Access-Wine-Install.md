# Native Access 2 — Install Notes for Wine 11

What actually makes this work, dead ends stripped.
Target: `~/.wine-ableton` on PikaOS 4 / Hyprland, wine-d2d1-nspa-11.11
(the ableton-wine Wine tree).

## What's wrong

Two things break in Wine:

1. **Installer preflight loop.** NA's NSIS installer calls
   `powershell.exe -C "<script>"` to check if NA is already running.
   Wine's `powershell.exe` is a 28 KB builtin stub that ignores `-C`
   and always exits 0. The stub lies ("yes NA is running"), the
   installer re-spawns itself in a loop with the "Native Access is
   running" dialog.

2. **First-run dep screen.** After install, NA tries to talk to a
   running `NTKDaemon.exe` over ZMQ on `127.0.0.1:5146` to get the
   daemon version. If none is running, it falls back to
   `wmic datafile ... get Version /value` (Wine has no `wmic`),
   then tries to install via `sudo-prompt` (Wine has no UAC, so
   "User did not grant permission"). Net effect: stuck on the
   "Please grant permission to install dependencies" screen.

The Wine powershell dispatcher goes through the loader stub at
`$WINE_TREE/lib/wine/x86_64-windows/powershell.exe`, not the on-disk
copy. Patching only `drive_c/...` does not help.

## The fix

### 1. Build MinGW (one-time)

```
sudo apt install -y gcc-mingw-w64-x86-64-win32
```

### 2. Patch the powershell stubs

```sh
cat > /tmp/pwsh-shim.c <<'EOF'
#include <stdlib.h>
#include <string.h>
int main(int argc, char **argv) {
    int i, n=0, c=0, s=0;
    for (i = 0; i < argc; i++) {
        if (!argv[i]) continue;
        if (strstr(argv[i], "Native Access"))   n=1;
        if (strstr(argv[i], "Get-CimInstance")) c=1;
        if (strstr(argv[i], "StartsWith"))      s=1;
    }
    return (n && c && s) ? 1 : 0;
}
EOF
x86_64-w64-mingw32-gcc -O2 -s -o /tmp/pwsh-shim.exe /tmp/pwsh-shim.c

PREFIX=$HOME/.wine-ableton
WT=$HOME/.local/opt/wine-d2d1-nspa-11.11
S=/tmp/pwsh-shim.exe

for f in \
    "$WT/lib/wine/x86_64-windows/powershell.exe" \
    "$PREFIX/drive_c/windows/system32/WindowsPowerShell/v1.0/powershell.exe" \
    "$PREFIX/drive_c/windows/syswow64/WindowsPowerShell/v1.0/powershell.exe" ; do
  [[ -f "$f.wine-stub.bak" ]] || cp -p "$f" "$f.wine-stub.bak"
  cp -f "$S" "$f"
done
```

The shim returns exit 1 only for the installer's preflight
(`Get-CimInstance` + `Native Access` + `StartsWith`). For everything
else it returns 0, mimicking the original Wine stub. A dumb
`return 1;` shim breaks the loop but then leaves NA stuck on the
dep screen.

Also packaged as `~/.local/bin/wine-pwsh-shim-install` and
`wine-pwsh-shim-restore`. Copies of all the helper scripts
(`wine-pwsh-shim-install`, `wine-pwsh-shim-restore`, `native-access`)
are in `~/Downloads/Native-Access-Wine-Scripts/`. Re-run
`wine-pwsh-shim-install` after any `wine-d2d1-nspa-11.11` update
that re-installs the loader stub.

### 3. Run the installer

```sh
WINEPREFIX=~/.wine-ableton wine $HOME/Downloads/Native-Access_2.exe
```

Installer copies `C:\Program Files\Native Instruments\Native
Access\Native Access.exe` (223 MB) plus Electron runtime, then exits.
The dialog loop is gone.

### 4. Start NTKDaemon as a service, then launch NA

The bundled `NTKDaemon 1.31.1 Setup PC.exe` does register
`NTKDaemonService` as a Windows service (only the EULA flow fails in
Wine). Start it, then launch NA:

```sh
WINEPREFIX=~/.wine-ableton wine sc start NTKDaemonService
# expect: STATE : 4 RUNNING
```

NA's ZMQ check finds the daemon, gets the version, skips the
install/permission flow, and proceeds to login.

The launcher at `~/.local/bin/native-access` (used by the .desktop
file) does this automatically before exec'ing `Native Access.exe`.
The service stays running between NA launches (shared wineserver),
so subsequent launches are instant.

If the service ever disappears (e.g. after rebuilding the prefix),
re-run the bundled daemon installer once to re-register it:
```sh
WINEPREFIX=~/.wine-ableton wine \
    "$PREFIX/drive_c/Program Files/Native Instruments/Native Access/resources/daemon/win/NTKDaemon 1.31.1 Setup PC.exe"
```

### 5. Install plugins through NA

Once NA launches successfully and you can log in, install plugins
normally through the NA UI. **Most NI plugins install cleanly** on
this Wine prefix — Battery 4, Komplete Kontrol, Massive X, FM8,
Reaktor, etc. all complete the install in ~1 minute, exit 0 or 100,
and get auto-activated by NTKDaemon.

**One exception:** Kontakt 8 Player's installer bundles an older
`vcredist_x64.exe` (vc_redist 14.32) that conflicts with the
vcredist 14.44 already installed. The bundle exits 0x666, the
wrapper returns 1 to NA, and the install fails. The Kontakt Factory
Selection content library still installs in the same run (it's a
sub-product extracted before the vcredist check).

For Kontakt 8 Player, use the manual fix in
`Kontakt-8-Wine-Install.md` + the `kontakt-install` script (VST3
+ activation; standalone not needed).

## Verification

| Check | Expected |
|---|---|
| `wine powershell.exe -C 'exit 7'` | exit 0 |
| `wine powershell.exe -C 'if ((Get-CimInstance ...startsWith Native Access...).Count -gt 0) { exit 0 } else { exit 1 }'` | exit 1 |
| `wine sc query NTKDaemonService` | `STATE : 4 RUNNING` |
| `ss -tln \| grep -E ':(5146|5563|7865)'` | NTKDaemon listening on 127.0.0.1:5146 (ZMQ), 5563, 7865 |
| NA window | download-path dialog on first run, then full library with 91 items |

## Cleanup (optional)

```sh
wine-pwsh-shim-restore
```

Reverts the three powershell stubs. The shim is also safe to leave
in place during normal NA use.
