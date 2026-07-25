# Kontakt 8 Player — Wine 11 install (manual, VST3 only)

What actually makes this work, dead ends stripped.
Target: `~/.wine-ableton` on PikaOS 4 / Hyprland, wine-d2d1-nspa-11.11.

## What's wrong with the unmodified installer

Kontakt 8.11.1 Setup PC.exe is an InstallAware + InstallScript bundle.
Before it copies any application files, it launches
`vcredist_x64.exe` (vc_redist 14.32.31332) from its OFFLINE payload to
satisfy a hardcoded prereq. Wine already has vcredist 14.44.35211
installed (from the Native Access install). The vcredist bundle
correctly refuses to downgrade and exits `0x666` (BundleInstallResult:
newer version present). The InstallScript wrapper then returns
`exit 1` to NA, which marks the install as failed even though the
file copy hadn't started yet — `Kontakt 8.vst3` and `Kontakt 8.exe`
are never copied out of the wrapper's temp dir.

The Kontakt Factory Selection content library (a separate sub-product)
DOES install cleanly in the same run, because the wrapper extracts
that sub-product's payload before the vcredist check.

**Most NI products do NOT have this problem.** Battery 4, Komplete
Kontrol, Massive X, FM8, and others install cleanly through NA on
the same Wine prefix. The vcredist-bundling is specific to Kontakt
8.11.1's installer. This is the one-off workaround for that one
product.

## What the manual fix produces

- VST3 plugin at the standard Windows location
  (`C:\Program Files\Common Files\VST3\Kontakt 8.vst3`, ~177 MB,
  PE32+ DLL with `GetPluginFactory` export — standard VST3 entry
  point). Discovered by any VST3 scanner.
- Registry entry `HKLM\SOFTWARE\Native Instruments\Kontakt 8` so
  NTKDaemon's product scan finds it.
- `installed_products/Kontakt 8.json` marker.
- A working activation JWT in
  `drive_c/users/Public/Documents/Native Instruments/Native
  Access/ras3/<UPID>.jwt` written by NTKDaemon after restart.

**Not produced:** the standalone `Kontakt 8.exe` (you don't need
it — only the VST3 and the sample content are useful in a DAW), and
the AAX plugin.

## The fix (4 steps)

### 1. Trigger one failed NA install of Kontakt 8

Open Native Access, find Kontakt 8 Player, click Install. The
download will complete and the wrapper will run, then NA will
report "Installation failed" after the vcredist check. **Leave the
failure artifacts alone** — the wrapper extracts the OFFLINE payload
to a temp dir like
`drive_c/users/jer/AppData/Local/Temp/miaXXXX.tmp/data/OFFLINE/...`
which is what the fix needs.

Verify the payload is there:

```sh
ls -la "$HOME/.wine-ableton/drive_c/users/jer/AppData/Local/Temp/mia"*.tmp/data/OFFLINE/*/ 2>/dev/null | head
```

The temp dir lives for hours after the failure — usually long
enough to complete the manual install.

### 2. Run `kontakt-install`

The script (`kontakt-install` in this dir, also in
`~/.local/bin/`) auto-detects the most recent `miaXXXX.tmp` extract,
copies the vst3, writes the registry key, writes the JSON marker,
and restarts NTKDaemon:

```sh
kontakt-install
```

The script is idempotent — running it twice is fine. It checks
file size on the vst3 before copying.

If NA's temp dir was already cleaned up, run `kontakt-install
--download` to be reminded of the manual install-attempt step.

### 3. Verify the activation in NA

Open NA. Kontakt 8 Player should now show as **Installed** in the
library. In `~/.wine-ableton/drive_c/users/Public/Documents/Native
Instruments/Logs/NTK/daemon.log` look for:

```
[ProductActivation.cpp:93] Successfully activated product '<UPID>' 'Kontakt 8 Player'
```

and a fresh `.jwt` at `ras3/<UPID>.jwt`.

### 4. (Optional) Rescan in your DAW

The next time Ableton (or your DAW) does a VST3 plugin rescan, it
will pick up `Kontakt 8.vst3`. No further config needed.

## Verification

| Check | Expected |
|---|---|
| `ls "$WINEPREFIX/drive_c/Program Files/Common Files/VST3/Kontakt 8.vst3"` | ~177 MB PE32+ DLL |
| `winedump -j export "$WINEPREFIX/drive_c/Program Files/Common Files/VST3/Kontakt 8.vst3"` | `GetPluginFactory` in export table |
| `wine reg query 'HKLM\SOFTWARE\Native Instruments\Kontakt 8'` | `InstallVST64Dir` + `Version` set |
| `ls "$WINEPREFIX/drive_c/users/Public/Documents/Native Instruments/Native Access/ras3/" \| grep 0e504595` | `0e504595-40d8-4982-978e-a242f036912d.jwt` |
| NA library | "Kontakt 8 Player — 8.11.1 — Installed" |

## What the wrapper extracted (for reference)

The relevant pieces inside the `miaXXXX.tmp/data/OFFLINE/` payload:

- `F4A5181D/<hash>/Kontakt 8.vst3` — the VST3 plugin (177 MB, self-contained PE32+ DLL, no external dependencies)
- `70201415/<hash>/Kontakt 8.exe` — the standalone (NOT installed by this fix)
- `17F0B091/<hash>/Kontakt 8.aaxplugin` — AAX (ProTools) format (NOT installed)
- `mFileBagIDE.dll/bag/vcredist_x64.exe` — the broken prereq (skipped)
