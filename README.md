<p align="center">
  <img src="assets/bodega-logo.png" alt="Bodega One" width="300" />
</p>

<h3 align="center">The local-first AI development environment</h3>

<p align="center">
  <a href="https://github.com/BodegaoneAI/bodegaone-releases/releases/latest">Download</a>
  ·
  <a href="CHANGELOG.md">Changelog</a>
  ·
  <a href="TROUBLESHOOTING.md">Troubleshooting</a>
  ·
  <a href="https://bodegaone.ai">bodegaone.ai</a>
</p>

---

Binary distribution for [Bodega One](https://bodegaone.ai). The app is developed in a private repository; this repo carries the built installers, the auto-update manifests, and the install script, so downloads and update checks live at stable public URLs.

## What it is

A desktop AI development environment that runs on your machine, against your models. Two modes share one app:

- **Chat mode** — streaming conversations with persistent memory, web search, file attachments, and tool use, against whatever model you point it at.
- **Code mode** — a full IDE (Monaco editor, file tree, terminals, a Git panel) plus AI panels for the agent, research, debugging, and advice.

The agent does not generate code and hope. The Quality Enforcement Layer reads a contract from your request, writes the code, compiles it, runs your tests, boots the result and probes it, then repairs what failed with targeted instructions. Point it at local models (Ollama, LM Studio, llama.cpp, vLLM) or cloud providers (OpenAI, Anthropic, Google, Groq, and 20+ more) and switch between them per request — no account with us, no vendor lock-in.

Turn on air-gap mode and nothing leaves your machine. That boundary is enforced in the engine, not promised in a policy.

The [changelog](CHANGELOG.md) records every release; the [latest release](https://github.com/BodegaoneAI/bodegaone-releases/releases/latest) is always current.

## Features

**Two modes, one app**
- **Chat mode** streaming conversations with persistent memory, web search, file and image attachments, and tool use.
- **Code mode** a full IDE (Monaco editor, file tree, integrated terminals, Git panel, code navigation) with AI panels for the agent, research, debugging, and advice.

**The agent, verified**
- **Quality Enforcement Layer (QEL)** the agent reads a contract from your request, writes the code, compiles it, runs your tests, boots the result and probes it, then repairs what failed. It proves the work is done instead of guessing.
- **Verification badges** every file the agent changes gets a pass, warning, or fail badge in the editor gutter, on the proposed diff, before you keep it.
- **Side conversations** open a read-only side chat on a second model without disturbing your main run, and hand a result back in.
- **Loops** schedule agentic tasks (cron or interval) that run in isolated worktrees and only apply when they pass QEL.
- **Fleet** run several agent sessions in parallel and compare or apply the winner.

**Your models, your machine**
- **Local-first** Ollama, LM Studio, llama.cpp, and vLLM, with a managed llama.cpp server, a self-updating model catalog, and GGUF downloads.
- **Cloud when you want it** OpenAI, Anthropic, Google, Groq, and 20+ more, switchable per request. No account with us, no lock-in.
- **Live GPU view** see the resident model, free VRAM, and whether the next model fits before you load it.
- **Spend caps** set daily and monthly limits on cloud usage; the app meters and stops at your cap.

**Private by design**
- **Air-gap mode** turn it on and nothing leaves your machine. The boundary is enforced in the engine at every outbound path, not promised in a policy.
- **Per-project air-gap** lock a single project offline regardless of the global setting.
- **Workspace trust** a cloned repository's committed config cannot change agent behavior until you approve the folder.
- **Zero telemetry** no analytics, no phone-home.

**Extensible**
- **MCP** connect Model Context Protocol servers for extra tools and context.
- **Custom agents, skills, prompts, and keybindings.**

## Install

### Windows (recommended)

```powershell
irm https://bodegaone.ai/install.ps1 | iex
```

Or take the installer straight from the [latest release](https://github.com/BodegaoneAI/bodegaone-releases/releases/latest):

- `Bodega-One-Setup-latest.exe` — NSIS installer
- `Bodega-One-latest-portable-x64.zip` — portable, no admin rights

### macOS

- `Bodega-One-latest-arm64.dmg` — Apple Silicon (M1–M4)
- `Bodega-One-latest-x64.dmg` — Intel

### Linux

- `Bodega-One-latest-x64.AppImage`
- `Bodega-One-latest-arm64.AppImage`

## Auto-update

The desktop app updates through [electron-updater](https://www.electron.build/auto-update). Each release ships its manifests — `latest.yml` (Windows), `latest-mac.yml`, `latest-linux.yml`. Air-gap mode blocks update checks along with everything else.

## Beta expiry

Beta builds carry an expiry date compiled into the binary. Current builds expire **2026-11-01** (public, influencer, and press builds alike). After that the app shows a notice and stops; install a newer release to continue.

## Troubleshooting

The common install snags — SmartScreen, macOS quarantine, AppImage permissions, auto-update — are covered in [TROUBLESHOOTING.md](TROUBLESHOOTING.md). The full guide (providers, activation, the agent loop, Cloud Boost) lives at [bodegaone.ai/help](https://bodegaone.ai/help).

## Reporting bugs

The source repo is private. File issues at [bodegaone.ai/feedback](https://bodegaone.ai/feedback) or email [help@bodegaone.ai](mailto:help@bodegaone.ai).

## Verifying a download

Each release ships SHA-512 hashes inside `latest*.yml`. The PowerShell installer checks them before it extracts anything. To verify by hand:

```powershell
(Get-FileHash -Algorithm SHA512 .\Bodega-One-Setup-latest.exe).Hash
```

Compare against the `sha512` field in `latest.yml` (base64-decoded).

## License

Bodega One is free to download and use, and is in public beta. Commercial licensing and full terms are at [bodegaone.ai](https://bodegaone.ai). Source licensing follows a Business Source License (BSL) after distribution; the Quality Enforcement Layer and agentic orchestration remain proprietary. This repository exists solely to host downloads, update manifests, and install scripts at public URLs.
