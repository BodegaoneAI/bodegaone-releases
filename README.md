# Bodega One

Binary distribution for [Bodega One](https://bodegaone.ai), the local-first AI IDE. The source lives in a separate private repo. This repo carries the built installers, the auto-update manifests, and the PowerShell install script.

## What is Bodega One?

A desktop AI development environment that runs on your machine. Two modes in one app:

- **Chat Mode**: streaming conversations, persistent memory, web search, file attachments, and tool use, against any model you point it at.
- **Code Mode**: a full IDE with the Monaco editor, a file tree, multiple terminals, a Git panel, and four AI panels (Agent, Research, Debug, Advisor).

The agent does not just generate code and hope. The Quality Enforcement Layer extracts a contract from your request, writes the code, compiles it, runs your tests, boots the result and probes it, then repairs failures with targeted instructions. It connects to 26 provider presets, local (Ollama, LM Studio, llama.cpp, vLLM, and more) or cloud (OpenAI, Anthropic, Google, Groq, and more), with one-click switching and no vendor lock-in.

Turn on air-gap mode and zero bytes leave your machine. 15 enforcement layers hold that line.

## What is new in beta.29

- **Routing rules**: an ordered, first-match-wins list that decides which model handles what, by mode, ask type, agent step, file path, message size, or daily spend. Every routed request says which rule decided it.
- **The agent sees type errors as it writes**: live diagnostics from a bundled language server (TypeScript, JavaScript, Python) land in the tool result, so the model fixes the error in the same pass. Fully local, works under air-gap.
- **Goals**: tell the agent what "done" means and it drives until the work verifies, with a second model sent in to attack the result before it can finish.
- **Verified Private Automation**: point Bodega at a task or a GitHub issue and it implements the change in an isolated worktree on your machine, runs full verification, and opens a pull request with the proof in the description. Nothing leaves your machine but the branch and the PR.

The full history is in [CHANGELOG.md](CHANGELOG.md).

## Install

### Windows (recommended)
```powershell
irm https://bodegaone.ai/install.ps1 | iex
```

Or grab the installer directly from the [latest release](https://github.com/BodegaoneAI/bodegaone-releases/releases/latest):
- `Bodega-One-Setup-latest.exe` (NSIS installer)
- `Bodega-One-latest-portable-x64.zip` (portable, no admin)

### macOS
- `Bodega-One-latest-arm64.dmg` (Apple Silicon: M1, M2, M3, M4)
- `Bodega-One-latest-x64.dmg` (Intel)

### Linux
- `Bodega-One-latest-x64.AppImage` and `Bodega-One-latest-arm64.AppImage`

## Auto-update

The desktop app uses [electron-updater](https://www.electron.build/auto-update). Each release ships its manifests: `latest.yml` (Windows), `latest-mac.yml`, and `latest-linux.yml`. When air-gap mode is on, update checks are blocked.

## Beta expiry

Beta builds carry a hard-coded expiry date baked into the binary. Current builds expire **2026-11-01** (both the public builds and the influencer or press builds). After expiry the app shows a terminal screen and stops working, so update to a newer release to continue.

## Troubleshooting

Common install snags (SmartScreen, quarantine, AppImage permissions, auto-update) are covered in [TROUBLESHOOTING.md](TROUBLESHOOTING.md). For the full guide (providers, activation, agent loop, Cloud Boost) see [bodegaone.ai/help](https://bodegaone.ai/help).

## Reporting bugs

The source repo is private. File issues at [bodegaone.ai/feedback](https://bodegaone.ai/feedback) or email [help@bodegaone.ai](mailto:help@bodegaone.ai).

## Verifying integrity

Each release ships SHA512 hashes inside `latest*.yml`. The PowerShell installer verifies automatically before extraction. To check by hand:

```powershell
(Get-FileHash -Algorithm SHA512 .\Bodega-One-Setup-latest.exe).Hash
```

Compare against the `sha512` field in `latest.yml`, decoded from base64.
