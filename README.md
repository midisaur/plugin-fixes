# plugin-fixes

Per-product Wine workarounds for plugin installers that don't work out of the box.
Currently focused on Native Instruments (Native Access 2) and the subset of NI
products whose InstallAware bundles interact badly with Wine 11, plus a
generic recipe for the other vendors (Neural DSP, etc.) that just need a
known-good Wine tree and prefix.

Target environment: Linux + Hyprland, `wine-d2d1-nspa-11.11` (a custom Wine
11 tree with the d2d1 + nspa patches, hosted in `~/.local/opt/`), and the
shared `~/.wine-ableton` prefix.

## Scope: install-time patches, not Wine shims

Every fix in this repo is an **install-time patch applied to a Windows
installer bundle** (MSI table edits with `msibuild`, LDIF registry
imports, `wine sc create` service registrations, extraction of files from
`InstallShield SFX` / `Inno Setup` / `InstallAware` wrappers), combined
with a launcher script that sets the right `WINEPREFIX`, `WINEDEBUG`,
and Wine tree.

**We do not ship Wine DLL shims, loader overrides (`DllOverrides`
registry entries that redirect builtin → native binary), or any
modification to the on-disk Wine tree.** Those techniques fix narrow
symptoms at the cost of breaking unrelated apps in the same prefix, and
they don't compose across products. Install-time patches + a good
launcher cover the same problems without the foot-guns.

## Prerequisites

This repo assumes the Wine tree and prefix layout from
[**shibco's `ableton-linux`**](https://github.com/shibco/ableton-linux).
That project builds the d2d1 + nspa Wine 11 patches and stages them at
`~/.local/opt/wine-d2d1-nspa-11.11/`, and provides a launcher (`ableton-linux`)
that drives the shared `~/.wine-ableton` prefix with the right
environment for Ableton Live and friends.

Before using anything in this repo, install `ableton-linux` per its
README and verify the launcher works for your DAW. Then this repo
layers per-plugin-install workarounds on top of that base.

**Why this matters:** the Wine tree and prefix layout from `ableton-linux`
exist to give Ableton Live 12 a stable, graphics-patched Wine environment
under Wayland. Most plugin installers in this repo were designed
to run in that same environment (so the Wine version, the prefix's
existing `vcredist`/`dotnet`/`NDI` packages, and the shared
wineserver state are all consistent). If you use a different Wine
build or a different prefix, the workarounds here may not apply
unchanged — but the per-product docs call out the exact Wine-version
sensitivity where it matters.

If you don't need Ableton and just want a Wine environment for plugins,
shibco's `ableton-linux` is still a good starting point — the Wine tree
and prefix work fine for any Windows VST3 / standalone app. You don't
have to actually install Ableton.

## Repo layout

```
plugin-fixes/
├── README.md                       # this file
├── NI/                             # Native Instruments / Native Access
│   ├── Native-Access-Wine-Install.md
│   ├── Kontakt-8-Wine-Install.md
│   ├── native-access               # launcher
│   └── kontakt-install
└── <vendor>/                       # one dir per vendor
    └── <product>/                  # one dir per product that needs a fix
        ├── <product>-install       # the workaround script
        └── <product>-Wine-Install.md
```

Each subdir is self-contained: a doc explaining what's wrong, a script (or
set of scripts) that implements the fix, and a verification table you can
run after the fix to confirm the install is actually working.

If you only need the workaround for a single product, you can copy that
one subdir out — nothing in here is cross-coupled except the Wine tree /
prefix paths, which you can override with environment variables.

## Quick reference: launching an installer

For a vendor installer that **doesn't** need any workaround (most plugin
manufacturers, including Neural DSP's Archetype series, STL Tones, etc.),
launch it directly into the shared ableton-wine prefix:

```sh
WINEPREFIX=~/.wine-ableton ~/.local/opt/wine-d2d1-nspa-11.11/bin/wine \
    /path/to/installer.exe
```

The two non-default bits are:

- **`WINEPREFIX=~/.wine-ableton`** — the shared prefix. Keeps plugin installs
  in one place, alongside your DAW (Ableton, etc.) so the plugin scanner
  sees them automatically.
- **`~/.local/opt/wine-d2d1-nspa-11.11/bin/wine`** — the custom Wine tree.
  Pinned directly (not via the system `wine` on PATH) because:
  - The system `wine` may resolve to `/opt/wine-staging/bin/wine` and pick
    up a different wineserver, which would corrupt the prefix's running state.
  - The d2d1 + nspa patches are required by some plugins and by Ableton
    for stable graphics.

If the installer is in a non-ASCII path or has spaces, quote the full path.
Run from a terminal (not via `xdg-open`/double-click) so you can see
errors and Ctrl-C if it hangs.

For installers that are wrapped in a portal app (Native Access, iLok
License Manager, plugin managers that need to stay running, etc.), see
the per-vendor doc — those usually need an additional step like starting
a service before launching the app.

## Quick reference: launching a product after install

Most plugins install a `.vst3` (or `.dll` for VST2) into the standard
`drive_c/Program Files/Common Files/VST3/` (or
`drive_c/Program Files/<Vendor>/VSTPlugins 64 bit/`) inside the prefix.
Your DAW will discover them on the next plugin rescan; you don't need to
do anything else.

Plugins that ship their own portal (Native Access, iLok, etc.) have a
launcher script in their subdir that handles service-start + wine-env +
`exec wine ...app.exe`. The launchers are idempotent and safe to call
multiple times.

## When a product needs a per-vendor workaround

If `wine installer.exe` fails with anything that doesn't look like a
generic "DLL not found" or "OS version too old" (e.g. exit code 1 with
no useful error, the install gets partway then hangs forever, the portal
app installs but then can't reach a service, the plugin installs but
isn't picked up by your DAW), it's a Wine-vs-this-installer
incompatibility. The pattern is:

1. Run the installer once with the standard command above to get a clean
   failure with the right error code.
2. Inspect the installer's temp / OFFLINE payload for what's left behind
   after the failure (most NI InstallAware + InstallScript bundles extract
   the actual files to a temp dir before they fail on a prereq check).
3. Identify the missing piece (a service registration, a vcredist bundle
   that conflicts, a launch condition referencing `VersionNT64`, files
   that need extracting from a wrapped installer bundle, etc.).
4. Document the issue and the fix in `<vendor>/<product>/<product>-Wine-Install.md`,
   and package the fix as a `<product>-install` script. Typical fixes
   are install-time patches (MSI table edits with `msibuild`, LDIF
   registry imports, `wine sc create` service registrations, extraction
   of files from `InstallShield SFX`/`Inno Setup`/`InstallAware` bundles)
   combined with a known-good Wine tree/prefix combo from
   `ableton-linux`. **No Wine DLL shims, no loader overrides, no
   on-disk Wine-tree modifications** — those fix narrow symptoms and
   break other apps, and we steer clear of them.
5. Verify: re-run the install, check the product is detected by your
   portal app (if any), and check the plugin is found by your DAW.

## Current contents

- **`NI/`** — Native Instruments / Native Access 2
  - **`Native-Access-Wine-Install.md`** — install NA 2 in the ableton-wine
    prefix. The NSIS installer runs through to completion under
    `wine-d2d1-nspa-11.11`; the only real obstacle is NA's first-run
    dep screen, which is fixed by registering `NTKDaemonService` via
    the bundled Inno Setup installer and starting the service before
    launching NA.
  - **`Kontakt-8-Wine-Install.md` + `kontakt-install`** — manual fix for
    Kontakt 8 Player (Kontakt 8.11.1 bundles `vcredist_x64.exe` 14.32
    which conflicts with the vcredist 14.44 already installed, so its
    InstallScript wrapper fails before copying the VST3). Extracts the
    vst3 from the wrapper's OFFLINE payload, writes the registry key,
    and restarts NTKDaemon to trigger activation. Most other NI
    products (Battery 4, Massive X, Komplete Kontrol, FM8, Reaktor) do
    NOT need this and install cleanly through NA.
  - **`native-access`** — NA launcher used by the `.desktop` file.
    Starts the NTKDaemon service before exec'ing the app.

- **`ilok/`** — PACE License Support / iLok License Manager 6.0.0
  - **`iLok-License-Manager-Wine-Install.md` + `ilok-install`** —
    LicenseSupport.exe is an InstallShield SFX around a WiX Burn bundle
    that fails on Wine because the inner MSIs use a Windows-10 launch
    condition and `ChangeServiceConfig2(SERVICE_FAILURE_ACTIONS)` (which
    Wine doesn't implement). The fix extracts the inner MSI, patches the
    failing tables with `msitools`, and installs the patched MSI directly
    with `msiexec`. Adapted from sirsipe's iLok-pace-license-support-wine11-scripts.
  - **`ilok`** — iLok License Manager launcher.
  - **`ilok-license-manager.desktop`** — menu entry.

## Adding a new product or vendor

1. Create `<vendor>/<product>/`.
2. Write `<product>-Wine-Install.md` following the same shape as the
   existing docs: short "what's wrong" section, step-by-step fix, a
   verification table. Link it from this README.
3. If the fix is automatable, write `<product>-install` as a self-contained
   bash script. Use the same env-var overrides
   (`WINE_TREE`, `WINEPREFIX`, `TMPDIR`) so the script is portable.
4. Add an entry under "Current contents" above.
5. If the fix involves a `.desktop` file or a launcher script for the
   product's portal app, add them to the subdir too.

## Environment variables

All install/launcher scripts in this repo accept these overrides:

| Variable | Default | Purpose |
|---|---|---|
| `WINE_TREE` | `~/.local/opt/wine-d2d1-nspa-11.11` | Path to the custom Wine build. |
| `WINEPREFIX` | `~/.wine-ableton` | Wine prefix to install into / launch from. |
| `TMPDIR` | `/tmp` | Scratch space for extraction. Some scripts (e.g. `ilok-install`) need ~280 MB free. |

## Why a shared prefix

Keeping plugins and the DAW in the same prefix (`~/.wine-ableton`)
means your DAW's VST3 scanner finds them automatically and the
wineserver stays single-process for the whole setup. Mixing
multiple prefixes works but adds wineserver overhead and forces
manual VST path config in the DAW.

If you need to keep plugins isolated from your DAW install, the
`WINEPREFIX` env var is the switch — every script here respects it.

## Tested on

- PikaOS 4 (Debian-based) on Ryzen 9 5900XT / 31 GB RAM
- Hyprland 0.55.4 (Wayland)
- Wine 11 (custom `wine-d2d1-nspa-11.11` build with d2d1 + nspa patches
  from [shibco's `ableton-linux`](https://github.com/shibco/ableton-linux))
- Ableton Live 12 (the resident DAW in the shared prefix)

Other Wayland compositors, other Wine builds, and other DAWs should
work but haven't been verified. The `ableton-linux` project's own docs
cover the rest of the environment setup; this repo is downstream.
