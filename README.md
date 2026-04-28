# Bodega One — public release mirror

Binary distribution for [Bodega One](https://bodegaone.ai), the local-first
AI IDE. Source code lives in a separate private repo;
this repo only carries built installers, auto-update manifests, and the
PowerShell installer script.

## Install

### Windows (recommended)
```powershell
irm https://bodegaone.ai/install.ps1 | iex
```

Or grab the installer directly from the [latest release](https://github.com/BodegaoneAI/bodegaone-releases/releases/latest):
- `Bodega-One-Setup-latest.exe` (NSIS installer)
- `Bodega-One-latest-portable-x64.zip` (portable, no admin)

### macOS
- `Bodega-One-latest-arm64.dmg` (Apple Silicon — M1/M2/M3/M4)
- `Bodega-One-latest-x64.dmg` (Intel)

### Linux
- `Bodega-One-latest-x64.AppImage` / `Bodega-One-latest-arm64.AppImage`

## Auto-update

The Bodega One desktop app uses [electron-updater](https://www.electron.build/auto-update). Manifests
in each release: `latest.yml` (Windows), `latest-mac.yml`, `latest-linux.yml`.

## Beta expiry

May 2026 beta builds carry a hard-coded expiry date baked into the binary:
- Public binaries: expire **2026-06-01**
- Influencer binaries (tagged `*-influencer` or `*-press`): expire **2026-07-07**

After expiry the app shows a terminal screen and stops working. Update to a
newer release to continue.

## Troubleshooting

Common install snags (SmartScreen, quarantine, AppImage perms, auto-update) are covered in [TROUBLESHOOTING.md](TROUBLESHOOTING.md). For the full guide (providers, activation, agent loop, Cloud Boost) see [bodegaone.ai/help](https://bodegaone.ai/help).

## Reporting bugs

The source repo is private. File issues on [bodegaone.ai/feedback](https://bodegaone.ai/feedback)
or email [hello@bodegaone.ai](mailto:hello@bodegaone.ai).

## Verifying integrity

Each release ships SHA512 hashes inside `latest*.yml`. The PowerShell
installer verifies automatically before extraction. For manual verification:

```powershell
(Get-FileHash -Algorithm SHA512 .\Bodega-One-Setup-latest.exe).Hash
```

Compare against the `sha512` field in `latest.yml` (decoded from base64).
