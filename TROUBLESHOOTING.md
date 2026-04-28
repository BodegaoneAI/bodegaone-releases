# Troubleshooting

Quick fixes for common install snags. For the full guide (providers, activation, agent loop, Cloud Boost) see [bodegaone.ai/help](https://bodegaone.ai/help).

---

## Windows

### "Windows protected your PC" / SmartScreen blue screen

The May 2026 beta ships unsigned. SmartScreen will warn — that's expected, not malware.

Click **More info → Run anyway**. Or use the PowerShell installer below, which downloads and unblocks in one shot:

```powershell
irm https://bodegaone.ai/install.ps1 | iex
```

A signed Windows build is coming post-beta. Until then, every fresh download trips SmartScreen the first time.

### `install.ps1` says "running scripts is disabled"

You ran the script after downloading it. Don't. Use the one-liner — `irm | iex` runs in-memory and doesn't trip ExecutionPolicy:

```powershell
irm https://bodegaone.ai/install.ps1 | iex
```

If you really need to run a saved copy:

```powershell
powershell -ExecutionPolicy Bypass -File .\install.ps1
```

### Antivirus quarantined the installer

Unsigned binaries trip heuristics. False positive. Three options:

1. Add an exclusion for `%LOCALAPPDATA%\Programs\BodegaOne` and re-run
2. Use the portable zip (`Bodega-One-latest-portable-x64.zip`) — extract anywhere, no install
3. Wait for the signed build (post-beta)

If your AV vendor is enterprise-managed, send them the SHA512 from `latest.yml`.

### Installer hangs on "Closing applications"

A previous Bodega One process is still running and the upgrade can't replace its files. Kill it:

```powershell
Get-Process | Where-Object { $_.ProcessName -like "*Bodega*" } | Stop-Process -Force
```

Then re-run the installer.

### Per-user vs. machine install

The NSIS installer installs **per-user** by default (`%LOCALAPPDATA%\Programs\BodegaOne`). No admin rights needed. If you need a machine-wide install, run the installer elevated and choose "All users."

---

## macOS

### "Bodega One is damaged and can't be opened"

Gatekeeper quarantine on an unsigned/un-notarized build. Strip the quarantine flag:

```bash
xattr -cr /Applications/Bodega\ One.app
```

Then open it normally. Apple notarization is in flight — once approved, this stops happening.

### Wrong DMG (Apple Silicon vs. Intel)

If the app crashes on launch or runs with bad performance, you grabbed the wrong arch.

- **Apple Silicon** (M1, M2, M3, M4): `Bodega-One-latest-arm64.dmg`
- **Intel** (pre-2020 Macs): `Bodega-One-latest-x64.dmg`

Check yours: Apple menu → About This Mac → Chip.

### "App can't be opened because Apple cannot check it for malicious software"

Same fix as the damaged-app message — `xattr -cr` strips quarantine. Or right-click the app → Open → Open (twice) to bypass once.

---

## Linux

### AppImage won't run

```bash
chmod +x Bodega-One-latest-x64.AppImage
./Bodega-One-latest-x64.AppImage
```

### "AppImages require FUSE"

Ubuntu 22.04+ removed FUSE2 by default. Install it:

```bash
sudo apt install libfuse2
```

Or extract and run without FUSE:

```bash
./Bodega-One-latest-x64.AppImage --appimage-extract
./squashfs-root/AppRun
```

### Sandbox / `chrome-sandbox` permission errors

Some distros require the sandbox helper to be SUID. Inside the extracted AppImage:

```bash
sudo chown root:root chrome-sandbox && sudo chmod 4755 chrome-sandbox
```

Or launch with `--no-sandbox` (less secure):

```bash
./Bodega-One-latest-x64.AppImage --no-sandbox
```

---

## Auto-update

### "Check for updates" says I'm up to date but a newer release is out

The check reads `latest.yml` from this repo. If the manifest hasn't been generated yet (release upload still in progress), the app sees the old version. Wait 5 minutes and check again.

To force a fresh check, restart the app — manifests are cached for the session.

### Updater downloads but won't install on Windows

NSIS upgrades require the old version to fully close. If you have the app open, the updater stages the new install and finishes on next launch. Just quit and relaunch.

---

## Beta expiry

May 2026 beta builds expire. After expiry, the app shows a terminal screen and stops working. Grab a newer build from the [latest release](https://github.com/BodegaoneAI/bodegaone-releases/releases/latest).

- Public binaries: expire **2026-06-01**
- Influencer/press binaries: expire **2026-07-07**

This is intentional — keeps stale beta builds from staying in circulation forever.

---

## Logs

When something breaks, the logs are usually the fastest path to diagnosis:

- **Windows**: `%APPDATA%\BodegaOne\logs\`
- **macOS**: `~/Library/Logs/BodegaOne/`
- **Linux**: `~/.config/BodegaOne/logs/`

The most recent file is `main.log`. For provider-specific issues, also check `backend.log`.

---

## Still stuck

Open an issue at [bodegaone.ai/feedback](https://bodegaone.ai/feedback) or email [hello@bodegaone.ai](mailto:hello@bodegaone.ai). Include:

- OS + version
- Bodega One version (Help → About, or check the installer filename)
- Last 50 lines of `main.log`
- What you ran, what you expected, what happened instead
