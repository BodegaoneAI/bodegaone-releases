# Changelog

All notable changes to Bodega One are documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
Versioning follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [Unreleased: v1.0.0-beta.29]

The routing release. Other IDEs decide which model handles your request and
call it a feature; Bodega hands you the routing table. Ordered rules you
write (by mode, ask type, agent step, file path, message size, or what
you've spent today) sit above the built-in heuristics and below your
explicit picks, and every routed request says exactly which rule decided it.

Under that headline the agentic loop got materially smarter. It sees type
errors as it writes them and can pull diagnostics on demand; it sends
read-only scouts to explore a codebase without burning the main context; it
holds a goal across the whole task and won't call it done until a second
model has tried and failed to break the result; and it switches to
test-driven repair when it gets stuck. Plus the OpenCode-gap work (verified
ACP, custom-model cost tracking, and self-hostable session export) and the
full per-model reasoning range.

And folded in late, but no smaller for it: Verified Private Automation. Point
Bodega at a task or a GitHub issue and it implements the change in an isolated
worktree on your machine, runs full QEL verification, and opens a pull request
with the proof in the description. Cursor's and Copilot's coding agents ship
your code to their cloud to do this. Ours runs on your hardware and proves its
work before it commits.

### Added
- **Routing rules.** Settings → AI Behavior → Routing: an ordered,
  first-match-wins rule list that decides which model handles what.
  Conditions: mode (chat/code), chat ask type, code step type
  (read/write/plan/verify), file path glob, message-size bounds, and
  cloud-spend-today. Actions: route to a model slot or an exact model,
  optionally stay-local. Backend-validated saves, drag priority, per-rule
  toggle, JSON import/export, and four one-click starter recipes (keep it
  free, cloud for reasoning only, private paths stay local, $10/day budget
  brake). Hard guards still clamp everything afterwards. A rule can never
  bypass air-gap, VRAM limits, or spend caps.
- **The built-ins became legible.** Smart Auto and the chat classifier now
  render as toggleable default rules in the same section, so the whole
  routing story reads top to bottom in evaluation order instead of hiding
  in Experimental.
- **OR/NOT rule conditions.** `anyOf` (match any of several condition
  groups) and `not` (exclude a group), authored in a per-rule JSON editor.
  "Auth code or env files, but not read steps" is one rule now.
- **Classifier patterns.** Teach the chat classifier your vocabulary: your
  own regex → tier entries run before the built-in tables ("anything
  mentioning kubernetes is a code ask"). Patterns that could hang a send
  (nested quantifiers, backreferences) are rejected at save with the reason.
- **Per-project routing rules.** A repo can ship rules in
  `.bodega/config.json` that evaluate above your own: "auth stays local"
  once, for the whole team. Always visible read-only in the Routing
  section, and a routed-by line from one always says "project rule": a
  repo can never route your requests invisibly.
- **Routing transparency everywhere.** The chat Auto pill previews which
  rule would catch the message as you type; the dry-run tester routes any
  hypothetical message through the real chain (including project rules)
  without sending; code-mode iterations log `routing_rule_applied` to the
  Debug panel; and unenforceable stay-local decisions say so instead of
  silently passing.
- **Verified Private Automation.** Point Bodega at a plain task or a GitHub
  issue and it works in an isolated git worktree on your machine, writes the
  code, runs full QEL verification (it boots your server and probes it, runs
  your tests, checks the contract), and opens a PR with the verification trace
  right in the description. A passing run gives you a ready PR; a failing one
  opens a draft, so an unverified change is never presented as merge-ready; a
  dry run verifies without opening anything. Connect a token in Settings →
  Integrations → GitHub, then click Automate in the Code activity bar (next to
  Source Control). Nothing leaves your machine but the branch and the PR. A
  fine-grained token needs both Contents and Pull requests write; if it's
  missing one, the run names the exact permission to grant instead of failing
  with a raw GitHub error, and one automation runs at a time so a pile of them
  can't take the backend down.
- **Run automation with your own agents.** The Automate dialog can run with any
  custom agent you have defined, so its system prompt, model, and tool
  allowlist drive the verified run. Three starter agents ship as one-click
  templates in Settings → Custom Agents: Code Reviewer, Test Runner, and Doc
  Checker.
- **Per-agent run history.** Every custom agent now has a History view. Each
  run shows its status, QEL score, files changed, and the full verification
  trace, opened from the agent's card.
- **GitHub fine-grained tokens are scanned.** The `github_pat_` token format
  joined the credential scanners, so a fine-grained PAT is redacted from agent
  output and repro bundles, the same as a classic `ghp_` token always was.
- **The agent sees type errors as it writes.** After every TypeScript or
  JavaScript edit, live diagnostics from a bundled language server land
  inside the tool result. The model fixes the type error in the same pass
  instead of discovering it at verification time. Works out of the box
  (the server ships with the app, nothing to install), works for
  modification tasks that previously had no in-loop compile feedback at
  all, works headless in Fleet and Loops runs, and works under air-gap
  (fully local, no downloads, and the server command can never be set by
  a repo or a setting). Errors-only and capped at 10 per file so a
  cascading failure can't flood a local model's context. Toggle:
  Settings → Agent → "Live type errors after agent edits" (on by default).
- **Cost tracking for your own models.** Set token prices for self-hosted,
  custom, or internal-gateway models in Settings → Spending → Custom model
  pricing, and their cost shows up in the dashboard like any cloud
  provider's, and overrides the built-in price when you set it. Stays on
  your machine (no registry lookup, works in air-gap). Closes the gap where
  a model the built-in table didn't know always read $0.
- **Share a session as a web page.** Right-click any chat → "Export as web
  page" writes a self-contained HTML file (conversation, model tags,
  tokens, timestamps, styled to match the app) that opens in any browser
  with no network. Send it to a colleague, attach it to a bug report. It's
  a local file you choose to share: nothing is uploaded, and it works in
  air-gap, unlike tools that share by posting your session to a public URL
  with no privacy policy.
- **Host your own session viewer.** "Export self-hostable viewer" (chat
  right-click) writes a single self-contained HTML file that is both a working
  copy of the session and a reusable read-only renderer: drop other exported
  bundles onto it, or serve it on your own intranet and point it at bundles
  with `?bundle=…`. Inline CSS and JS, zero external requests, works under
  air-gap. It is the enterprise share story without a share server. The
  renderer runs on your infrastructure and the data never touches ours. Every
  bundle is treated as untrusted: content renders as text, never markup.
- **The full reasoning range, per model.** The reasoning dial now exposes
  every tier a model actually has: GPT-5 gains **Minimal** (answer fast,
  barely think), and Claude 4.6+ adaptive models gain **Extra High** and
  **Max** above High. Each model's menu shows only the tiers it accepts
  (the per-message composer dial and the Settings per-model card both), and
  anything out of range is clamped to the model's nearest level before the
  request leaves, so flipping models with a tier set can never send an
  effort the API would reject.
- **Goals: tell the agent what done means, and it drives until it gets
  there.** Type `/goal API passes all auth tests with rate limiting` and the
  goal becomes a real object, not a line in the chat: it survives every
  message, its task list picks up exactly where the last run stopped (a run
  that hits its iteration cap reports what's left and resumes on your next
  message), an approved plan's steps become goal tasks automatically, and
  the task panel shows the goal with live progress. The agent can't just
  declare victory either. A goal only completes when the work actually
  verifies, and right before it does, a second model is sent in to attack
  the result ("find what breaks against this goal"). Anything it finds
  becomes new tasks and the agent keeps working. One attack per run, never
  blocks a finish on a reviewer hiccup, and you pick the reviewer model in
  settings. Two different models don't share blind spots.
- **Two new skills.** `/decompose` turns a big objective into a persistent
  goal with 3–7 verifiable tasks (each task has to name its own evidence:
  the test that passes, the file that exists) and works them in order;
  it's the trained companion to the goal feature. `/onboard` tours an
  unfamiliar repo (structure, entry points, one real flow traced end to
  end) and saves what it learned as project knowledge, so the same repo
  doesn't get re-explored from scratch every session. Read-only by design.
- **The agent can now ask for type errors, not just receive them.** The
  beta.29 LSP feeds errors to the agent after its own edits, but bug-fix
  tasks start from code that was already broken. The new `get_diagnostics`
  tool lets the agent pull the current errors for any TypeScript/JavaScript
  file before touching it (read-only, no approval prompt), instead of
  running the whole compiler through the shell. Same bundled language
  server, same air-gap guarantees, capped output so a cascading failure
  can't flood a small local model.
- **Send a scout instead of reading the repo yourself.** The new
  `dispatch_scout` tool hands an open-ended question ("how does X work?",
  "trace the Z flow") to a read-only sub-agent that explores on its own and
  reports back a short digest, so the greps and file-reads land in the
  scout's context, not the main conversation. The agent gets the answer, not
  the 40 files it took to find it, which is the single biggest thing eating
  context on a long task. The scout can't edit, run commands, or dispatch its
  own scout, and it runs against your primary local model. Exploration stays
  cheap and private.
- **Test-driven repair when the agent gets stuck.** When verification fails
  twice on a build-something task and the project has a test runner, a capable
  model now gets a stronger move than "try the same fix again": write one
  failing test that pins the missing behavior, prove it fails for the right
  reason, then fix until it's green. A passing test is concrete proof the
  behavior works. It's deliberately gated (weak local models that can't
  reliably call tools tend to spiral writing tests they can't satisfy, so they
  never get this) and it fires once, not in a loop. Toggle:
  `agent.test_first_repair` (on by default).
- **Stop a running Loop.** The Loops indicator gained a stop button (and a
  per-loop Stop in its hover tooltip) that cancels an automation mid-run.
  The agent halts at its next checkpoint and the run is marked cancelled with
  its partial work left in the isolated worktree for review, never
  auto-applied. A loop you started by mistake, or one heading the wrong way,
  stops cleanly instead of running to completion.
- **New models and two new providers.** A catalog refresh (v1.6.0) adds 14
  models (including the granite family and a glm-4.7-flash fix) and two new
  cloud providers, MiniMax and Z.ai. Cloud prices were corrected across the
  board so the spending dashboard and pre-send estimates match what you're
  actually billed.
- **Python type errors for the agent.** The in-loop language server that lets
  the agent see TypeScript errors as it writes now covers Python too (via
  pyright): an agent editing Python gets the same after-edit diagnostics and
  on-demand `get_diagnostics` lookups it already had for TS/JS, fully local.

### Changed
- **A visual pass over the whole app.** Buttons stopped being purple boxes
  with white text: an action now reads as its accent color, and the filled
  pills are gone except where a fill earns it (the send button, a confirm).
  Toggles are a new minimal design: a thin rail the knob rides, not a chunky
  filled track. The buttons across the agent panels are consistent now and
  tinted to their panel (purple Agent, blue Research, amber Debug, green
  Advisor), the model name included. The Help & Docs pages got scannable
  color back so a reference page isn't a wall of grey text, and the chat-
  history rows and sidebar tabs got borders so they stop melting into the
  background. The greeting's model picker sits inline with the chips, and the
  Chat / Code switch is bigger and clearer.
- **Claude Fable 5 is no longer available.** Anthropic withdrew the model
  from general availability, so Bodega no longer offers it as a cloud model:
  the catalog card and picker entry are gone, and it's filtered out of the
  live model list even if the API still returns it. If you had Fable 5
  selected, pick Claude Opus 4.8 (or any other model) in Settings.
- **Driving Bodega from other editors (ACP) is verified and documented.**
  `bodega --acp` has quietly worked since beta.26: Zed, JetBrains AI
  Assistant, or any ACP client can spawn Bodega as a headless agent over
  stdio, with every tool call still gated by the client's permission
  prompt. It now has a live acceptance test against the packaged binary
  layout (handshake, session creation, cwd confinement, clean JSON-RPC
  stdout) and proper setup docs on the Integrations page.
- **The editor language server graduated from Experimental. And now it
  actually works for everyone.** The old toggle quietly required
  `typescript-language-server` installed globally on your PATH, which meant
  red squiggles, go-to-definition, and refactoring silently never started
  for most people. The editor now uses the language server that ships with
  Bodega (the same one the agent uses to see type errors as it writes): no
  install, no PATH, nothing to configure. The setting moved to Settings →
  Editor → AI Features and is on by default; your previous choice carries
  over automatically.

### Fixed
- Routing rules now actually fire on live sends. The composer pre-resolved
  a concrete model before every send, which made rule evaluation
  unreachable in production (it worked in the dry-run tester only). Sends
  now distinguish an explicit pick (a pinned model, a fixed pill tier, a
  custom agent's model) from an auto-resolved default: picks are always
  honored as-is, defaults consult your rules. With no rules configured the
  send path is byte-for-byte the old one.
- A budget condition inside an OR/NOT group never fetched the day's spend
  total, which made "when NOT over budget" rules fire regardless of spend.
  Spend conditions now arm the fetch wherever they appear in a rule.
- The spacebar could stop working in the code editor (it would type once then
  die, and only register if you held another key down first). Monaco 0.55
  switched the editor's input to the browser's new EditContext API by default,
  and its composition handling drops the Space keystroke on Electron/Chromium.
  The editor now uses Monaco's long-proven hidden-textarea input instead, so
  typing is reliable again. Two related menu/completions crashes that surfaced
  while chasing this were also fixed: a menu's keyboard-nav handler that only
  recognized `<input>`/`<textarea>` as "the user is typing" (it now recognizes
  the editor too), and a missing `disposeInlineCompletions` hook the AI-
  completions provider needed for Monaco 0.55.
- Accessibility settings apply the moment you toggle them. Reduce Motion,
  Large Text, High Contrast, and Focus Style were save-gated. They did
  nothing until a separate Save button, unlike every other toggle in the app,
  which read as broken. They now take effect on change like the rest of
  Settings.
- Creating and renaming files from the file tree works anywhere, not just at
  the project root. Right-clicking a folder → New File, or renaming a nested
  file, set up the action but never showed the name input (it only rendered
  for root-level targets). Right-clicking the empty area of the tree now
  targets the project root too.
- Naming a new file in the editor no longer opens the OS file picker. A new
  Untitled buffer renames in-app (a small name bar) and saves into your
  project folder; existing files still use Save As.
- The text caret could sit a pixel or two off from the glyphs in the editor
  until something forced a relayout. Fonts are remeasured on load now, so the
  caret lines up from the first keystroke.
- Save As on a new file now re-derives the language from the chosen filename,
  so syntax highlighting and the language server attach immediately instead of
  the file staying plain text.

---

## [v1.0.0-beta.28] - 2026-06-11

The polish release. A 25-agent research sweep over everything already shipped became five waves of work: correctness fixes (cost tracking showed $0.00 for current flagship models, the air-gap cloud-provider warning never fired, two real memory/shutdown leaks), performance (llama.cpp loading no longer blocks boot, Map staleness checks parallelized, embedding index builds detach), a UX consistency sweep (one focus ring, shared icons, light-theme-safe status colors), and the small features users kept asking for: Run Tasks in the terminal, a unified Runs Inbox, one-click rules import from other editors, opt-in read-only auto-approve, pre-send cost estimates, delta-only re-review, loop countdowns, and visible progress on every model swap. No new direction, no new flags to learn: beta.27, tightened.

### Added
- Run Tasks: one-click dev-server launch from the terminal tab bar, and a labeled
  "Run tasks" chip in the agent panel's action row so it's not hidden. Auto-detects
  `dev`/`start`/`serve`/`watch` scripts from your project's package.json, supports
  custom named tasks via a `run_configs` key in `.bodega/config.json`, runs each
  task in its own terminal tab, and "Run all" launches everything at once.
  Stopping sends Ctrl+C so the process exits cleanly and the logs stay readable.
- The agent can open the Preview tab when you ask it to. "Open the preview at
  localhost:5173" now opens the tab, loads the URL, and brings it forward.
  Previously the agent could only navigate a preview you had already opened.
- **Runs Inbox.** One topbar surface for everything waiting on you (fleet
  sessions needing approval or ready to apply, loop runs parked for review or
  below their QEL bar) with click-through to the owning panel. Hidden when
  empty; the Fleet and Loops pills keep narrating live activity.
- **Coming from another editor? Bring your rules.** Opening a project with
  Cursor, Copilot, Cline, Windsurf, or Continue rules offers a one-click
  import into `.bodega-rules` (append-only, never overwrites yours, re-import
  skips what's already in). Manual re-run lives in Settings → Profile.
- **Auto-approve read-only tools in Ask mode** (off by default). Searching and
  reading (code search, grep, glob, symbol lookup, Map/memory/knowledge
  queries) can skip the approval prompt while every file write, web call,
  and shell command still asks. Shell can never auto-approve.
- **Pre-send cost estimate.** On cloud models the status bar shows roughly
  what your next message will cost ("~$0.012/msg") next to the Context meter,
  priced from the current context plus a typical reply. Local models show
  nothing. There's nothing to pay.
- **Review only what changed.** The source-control Review button now skips
  files that are unchanged since your last review and says so, with a
  "Review everything" escape hatch. Review → fix → review loops stop
  re-paying for the whole diff.
- **Loops tell you when they'll fire.** Interval and Map-staleness loops show
  a live "next in 14 min" countdown on their rows, and the topbar loops pill
  gained a per-loop Run-now button.
- **Run Tasks checks your dependencies.** Opening Run Tasks in a project with
  a package.json but no node_modules offers one-click `npm install` in its
  own terminal tab.
- **Every model swap shows progress.** Stage labels and elapsed time during
  llama.cpp model loads now appear for all swaps (not just vision-triggered
  ones) instead of a stage-less spinner.

### Changed
- **One focus ring.** Keyboard focus now draws the same accent ring on 35 controls
  across the app instead of a mix of browser defaults and one-off styles.
- **One grid, one shield.** The fleet four-square glyph and the safety shield are
  now single shared icons everywhere they appear. The Settings copy of the fleet
  icon had quietly drifted to rounded corners.
- **TopBar survives narrow windows.** The center section clips cleanly instead of
  overlapping the window controls, and the project name steps aside first. The
  Chat/Code pills always stay reachable.
- **Status colors read correctly on light themes** across fleet surfaces and
  toasts (semantic status tokens replace hardcoded dark-theme colors).
- **Faster where it drags.** Bodega Map staleness checks hash 16 files at a time
  instead of one; llama.cpp model loading no longer blocks the backend on boot
  (the dead startup spinner is gone); embedding index builds detach instead of
  hanging the request; project search, the spending dashboard, and the model
  picker do less work per render.

### Fixed
- Cost tracking showed $0.00 for Claude Opus 4.8 and Fable 5: the pricing table
  was missing both flagships and carried stale rates for Opus 4.6/4.7 and Haiku 4.5.
- The air-gap warning about configured cloud providers never fired for Anthropic
  (or any current provider): it was checking a deprecated setting.
- Long-lived editor connections via ACP grew memory forever; idle sessions are
  now evicted after 30 minutes (never mid-turn).
- Quitting Bodega mid-run no longer kills headless loops mid-write. Shutdown
  aborts them cleanly and waits for writes to land.
- Sending from the agent panel briefly shifted the whole app upward with dead
  space underneath. Message scrolling now stays inside the message list.
- "Dev servers detected" chips outlived their servers. Candidates are now
  liveness-probed every few seconds, so a killed server's chip clears itself.
- A failed send buried your prompt under an error banner with no way out. The
  banner now has a close button and restores your draft into the composer.
- When no model was selected, requests went out with a placeholder model name
  (`gpt-oss`) and failed confusingly. You now get a clear "No model selected"
  pointing at Settings → Models.
- Welcome-screen template chips did nothing when the agent panel was closed.
  They now open the panel with the template prefilled.
- Dragging a panel tab too far no longer detaches it into a stray floating
  mini-window (the source of the phantom-window report).
- The keyboard-shortcuts overlay (Ctrl+/) was missing eight shortcuts that
  actually work, including Open Agent panel (Ctrl+L) and Toggle Git panel
  (Ctrl+Shift+G). It now renders from the same table the audit checks.
- llama.cpp setup: if the extracted `llama-server` binary isn't where the
  installer expects (antivirus quarantine is the usual cause), setup now
  searches the extracted folder before failing. And the error finally says
  what to check instead of just "binary was not found".
- The Windows installer's finish page no longer renders its checkboxes
  illegibly in dark system themes.

---

## [v1.0.0-beta.27] - 2026-06-10

Run your agent on a schedule, on your machine, for free, with changes gated by QEL verification. That gate got a ground-up correctness and capability overhaul this release: QEL now *runs* your code instead of stopping at "it compiles," its accuracy is a measured number guarded in CI, and verification is roughly twice as fast. The terminal finishes what beta.26 started (pinned running-command header, a real Output panel, a real Debug Console), Bodega registers as a file handler so double-clicking a file just opens it, and a release-wide polish pass tightened buttons, sliders, settings density, and the source-control workflow (hunk-level staging).

### Added

- **QEL execution proofs: verified now means it runs.** For a server task, QEL boots the generated app in the same sandbox the compile gates use, forces an ephemeral port, and sends one request to the first route the task asked for. A response under 500 is the strongest pass evidence QEL has; a boot crash or a 5xx is a real failure; anything environmental (missing interpreter, no start command) stays neutral, never a false fail. Loopback only, secrets-free env, 12-second hard cap, and the process tree is always killed. Nothing outlives the proof. Settings → `qel.execution_proofs` (on by default). The QEL card in chat shows the probe (`GET /api/tasks → 200`) and how long verification took.
- **QEL speaks more languages.** JS/TS test files now run under vitest the way Python tests ran under pytest; Ruby and PHP deliverables get syntax gates (`ruby -c`, `php -l`); SQL and Dockerfile deliverables get built-in structural lints (no parser install, no execution). So a truncated `CREATE TABLE` or a prose "Dockerfile" fails instead of sailing through unverified.
- **The semantic judge works air-gapped. And can finally say no.** A localhost-served judge model (Ollama, llama.cpp, LM Studio) now runs under air-gap; only cloud judges stay blocked. And when the judge scores a *marginal* pass 0/3 ("this is semantically wrong"), it pulls the score just below the bar so the run parks for review instead of auto-applying. It can't touch confident scores and can't hard-fail anything. `qel.judge_can_veto` turns it off if you want bonus-only behavior.
- **QEL calibration harness.** 43 labeled known-good/known-bad scenarios swept through the real verifier at every model tier, with CI floors: broken work passing is a build failure. The first sweep immediately caught (and fixed) two scoring holes: trivially-empty code files riding a passing compile to a pass, and missing files silently dropping out of the completeness denominator. Scoring changes are evidence-driven from here.
- **Bodega Loops (Automations).** Define a named agent task that runs on a schedule. Each loop picks **what it runs with** (one of your custom agents, or a specific provider + model) so you can point a fast local model at small jobs and a cloud model at the heavy ones, the same way a heterogeneous Fleet does. Trigger it on a cron schedule, a fixed interval, or *Bodega Map staleness* (fire when N of your Map's module summaries go stale; no other tool can express that), with an apply policy. Every run executes headless in an isolated git worktree and lands as a reviewable diff with its QEL score and full trace. **Park for review** is the default; **auto-apply** only ever happens when the QEL score clears the bar you set; **dry run** never applies anything. The dashboard is a Code-mode sidebar panel (the loop icon in the activity bar, like Bodega Map) with live run status and expandable histories; loops are managed in Settings → AI Behavior → Loops; and a topbar pill shows running loops with a sticky red badge when a run fails. A loop can never fail silently. Scheduled runs refuse to execute without worktree isolation, respect your spend caps, inherit air-gap, and cap their own iterations.
- **Terminal sticky command header.** When a long-running command's output scrolls, a pinned header keeps showing *which* command it belongs to, with live status and a click-to-jump back to its start.
- **Output panel, for real.** The bottom-bar Output tab (a hidden stub since beta.26) now streams the backend's log channel with level filters, text search, follow-tail, and clear.
- **Debug Console, for real.** The Debug Console tab mirrors the agent loop's live diagnostics stream (iteration markers, tool executions, nudges) with expandable detail JSON. (Full breakpoint/DAP debugging remains on the roadmap; the panel says so honestly.)
- **Open files with Bodega One.** Bodega now registers as a file handler: double-click a `.md`/`.json`/`.ts`/`.py`/… or "Open with → Bodega One" and it opens in an editor tab, no project folder required (like opening a loose file in VS Code). Works from the shell too (`bodega path/to/file`). The native "Open File…" menu, which previously did nothing, now opens the file.
- **Searchable command history** in the terminal action bar: OSC-133-tracked commands, de-duped, insert-on-pick.
- **Hunk-level staging.** Each hunk in a source-control diff gets its own Stage/Unstage button. Commit exactly the lines you mean to. (A top community request in other AI IDEs; unshipped there.)
- **Conventional-commit type picker** next to the commit message: pick `feat`/`fix`/`chore`/… and it prefixes the message (replacing, never stacking). AI Generate fills the body; the picker sets the type.
- **Search history**: the clock icon in project search lists your last 10 queries, persisted across restarts.
- **Bodega Map: Mermaid export, edge explanations, and a Fast/Deep toggle.** Export the visible graph as a paste-anywhere Mermaid diagram (downloaded + copied); click any edge to get "why does A depend on B" answered; Ask-the-Map gains a Fast mode that answers from the already-generated module summaries near-instantly.
- **Custom agents: visible identity + shareable bundles.** Selecting a custom agent now shows in the status bar (`AgentName · model`, agent icon, accent color) and the panel header shows the agent's pinned model: no more guessing whether the default is running. Export/import all agents as a `bodega-agents.json` bundle to share them.
- **Fleet apply "Don't ask again."** Like the discard bypass: check it once and applies run immediately (merge strategy); reset both under Settings → General.
- **Fleet Parallel: cost + quality at a glance, and a re-race.** The compare view shows each session's QEL badge and the run's total cost ("Fleet cost: $0.04, 3 sessions"), and a **Re-race** button reruns the same task with the non-winning model combos in one click.
- **Loops: a pill that tells you more, and notifications that tell you enough.** Hovering the topbar loops pill shows each loop's last outcome, QEL score, and files changed; terminal runs fire a toast and optional OS notification with the same detail ("Nightly docs sync: parked for review, 3 files, QEL 82%"), sharing Fleet's notification and quiet-hours settings.
- **Crash Reporting opt-out** toggle in Privacy & Safety, and a refreshed Export Diagnostics bundle.
- **Clear history on the Performance tab.** The per-model QEL pass-rate stats can now be wiped: scores recorded before this release's verifier fixes poisoned the rolling averages, and there was no way to start fresh. Two-step confirm; tracking restarts with the next verified task.

### Changed

- **Verification runs once, not three times.** A creation turn used to run the proof-gate suite up to four times: the verifier's own gates, a separate compile check on the happy path, a full re-verification just to cache a bug pattern, and AutoVerify re-running the same `tsc` at the end. The result is now computed once and reused everywhere, and proof gates run in parallel. Same checks, same verdicts, roughly half the verification wall-clock.
- **Fleet and Loops cards came alive.** Status now reads at a glance: a color wash bleeding from the status edge (green for ready-to-apply, blue for running, amber for attention, red for error), a slow breathing glow on running cards, beacon status dots, and hover lift, on fleet cards, loop rows, and the Fleet Monitor's running rows. Idle stays calm so active work pops.
- **One chip shape across the app.** In-card labels (status badges, capability chips, model/tag/version chips, 41 of them across 19 components) are now rounded rectangles matching the buttons and cards they sit beside; capsule pills are reserved for actual controls (topbar toggles, composer pills) and counters.
- **Buttons and sliders got the send-button treatment.** One global slider style (recessed track, accent thumb with a glow on hover) replaces the bare browser-native controls; the app's most common button family gains hover-lift physics; Save buttons lift and glow; and 141 corner-radius inconsistencies were normalized across 77 components.
- **Settings density pass.** The content column is wider, and the Loops section fills its empty state with clickable quick-start recipe cards (nightly doc sync, Map self-maintenance, dead-code sweep, test backfill) that open the create dialog pre-filled, plus a quick-add row for recipes you don't have yet, and the project name on every loop row.
- **Fleet Monitor fits a normal sidebar.** The lower-value columns (Iter / Elapsed / Tokens) hide below 480px of panel width, so Branch / Status / Files / actions stay readable without dragging the sidebar wide.
- **Keybindings is now an honest reference.** The old rebind UI saved combinations that nothing ever applied (and promised they'd work "after restart"). It is now a searchable view-only list of the real bindings, until an actual keybinding engine ships. Reverting a file (`Ctrl+Shift+R`) now fires a "Ctrl+Z to undo" toast so the safety net is visible.
- The command palette tells the truth about shortcuts: `Ctrl+Shift+B` is Toggle Bodega Panel and `Ctrl+Shift+F` is Toggle Fleet Panel (new truthful entries); File Explorer and Search in Files keep their actions but lose the shortcut chips they never owned. "Show Keyboard Shortcuts" actually opens the overlay now.

### Fixed

- **QEL was failing correct TypeScript: the real cause of "0% pass" on local models.** A workspace without a `tsconfig.json` made bare `tsc` print its help banner and exit 1; QEL read that as a compile failure and hard-capped every correct TS creation at 50 on the product's most common stack. A tsc "failure" without a genuine `error TS` code is now inconclusive, and the mid-loop micro-check skips type-checking when there's no tsconfig instead of injecting phantom repair nudges.
- **Edits and bug fixes are verified again.** The entire modification-verification path existed since beta.25 but was dead code: the gate only admitted creation tasks, so every edit/fix got no verification, no QEL score, and (under Loops auto-apply) parked forever. Modification and fix tasks now route through their verifier and emit a real score and trace.
- **A code task QEL couldn't verify can no longer auto-pass.** When nothing compiled or tested successfully (no toolchain, no gates), structural credit alone could clear the bar: an auto-apply candidate with zero evidence. Such runs now park just below the threshold for human review. ("We couldn't verify this" is an honest verdict, not a pass.)
- **Model performance stats counted only failures.** The tracker logged QEL outcomes on the failure path but never on the common success path, manufacturing the "0% pass rate" reading. Successes are counted; the Performance tab now reflects reality.
- **A hung verification can't freeze the app anymore.** The post-write build check ran synchronously on the main process loop (every SSE stream stalled while one build ran, up to 30 s), and a stalled judge model could block verification forever. Both are async with hard timeouts now.
- **The hard cloud spend cap now covers every execution path, and counts it.** Background sessions, Fleet Parallel fan-outs, and Loops previously bypassed the cap entirely; worse, their BYOK cloud spend was never even *recorded*, so it stayed invisible to the running total. All headless on-ramps now both refuse over-cap runs ("spend cap reached", like chat) AND record their cost against the session that incurred it, so the cap actually accounts for background/fleet/loop spend, and each loop run shows its real per-run cost.
- Closing a dirty editor tab from the command palette now asks about unsaved changes, exactly like `Ctrl+W` always did (this was a real data-loss path).
- Streaming no longer re-renders the entire message history on every token (a memo-defeating inline callback + an unmemoized list row).
- The in-app docs hub no longer drags the whole markdown pipeline into the startup bundle (+423 KB regression since beta.26). It's back in its own lazy chunk.
- Custom agents: a pinned cloud model now routes to its own provider instead of erroring on the active local one; the model is picked from a provider-grouped dropdown; the profile chip shows the pinned model.
- **A failed search no longer pretends to be "No results found."** Invalid regex, a missing desktop bridge, or an internal search error now shows an error card saying what happened: the exact failure shape that made search feel broken.
- **Stale fleet sessions can be discarded again.** A session whose worktree was already gone (GC'd, crashed before creation, or cleaned externally) hit a "no worktree to discard" error forever. The card was permanently stuck. Discard is idempotent now. And a backend crash mid-run no longer leaves a phantom "still streaming" block: the guard checks for a *live* stream, not just the frozen status.
- **Discarded fleet cards no longer come back.** Discard (and apply) cleared the session's worktree but left its background flag set, so the next fleet refresh resurrected the card as "Unknown", forever. Both operations now fully retire the session from the fleet (the conversation stays in history), and a one-time boot migration heals cards already ghosted this way in existing databases.
- **A loop that changes nothing now says so.** A zero-change run used to park its empty worktree as a "Ready to apply" fleet card with +0/−0/0 files. It now finishes as **no changes**, the empty worktree is cleaned up, and the "finished: no changes" notification fires. (The check is untracked-file-aware, so in-flight work is never mistaken for empty.)
- **The About-page logo renders right in every theme.** Light themes no longer show jagged edges (the mark now sits on a dark brand tile, its native surface), and the default theme's logo no longer renders smaller than the others (the SVG carried ~56% empty margin in its viewBox).
- **An execution proof can't be satisfied by a server you already had running.** The boot-and-probe candidate ports included framework defaults like :3000 (where Bodega's own backend listens) so with the app open, a crashed generated app could "pass" off a response that wasn't its own. Ports that were already answering before the spawn are excluded now, and an app crashing on a taken port reads as environmental, not broken code.
- **The diff view's Apply button no longer plays dead.** It was disabled whenever the numstat file count read zero, which can happen while the diff text you're looking at shows real changes (the two come from separate git calls). Apply now enables whenever anything visible is applyable; only a genuinely empty diff disables it.
- **Running a loop no longer fires a "signal timed out" toast.** The run-now request held the response open for the whole run (minutes) while the frontend's default timeout was 15 seconds. Every real run tripped it, even though the run was fine. The request now opts out of the timeout, and the duplicate completion toast is gone too: the notification with the outcome, file count, and QEL score is the single announcement.
- **Fleet cards show the model that actually ran.** A custom-agent loop run could display a placeholder model name that was never used (the runner persisted the provider's echo instead of its own resolution). The card now reports the model the run was resolved to.
- Git panel: stage/unstage failures surface in the error strip instead of silently doing nothing; generate-commit-message failures surface too.
- Settings warning banners are readable on light themes: six banners across five sections hardcoded pale yellows; all use the theme-aware warning tokens now (dark amber on light, WCAG-safe).
- Prompts and Skills settings tabs no longer render a backend failure as an empty list.
- Git review: "nothing to review" is a calm message instead of an API error. Multi-write tool results stay adjacent to their assistant turn on Anthropic models. Refactor prompts no longer spin the commit-skill loop.
- The terminal agent greeting no longer double-posts.

### Security

- Loops hardening (pre-flag-flip, from the 2026-06-09 fresh-pass review fleet): scheduled runs require worktree isolation; `project_path` is bounded (absolute, non-root, non-OS-dir) before becoming an unattended agent's sandbox root; per-loop `max_iterations` is actually enforced; a custom QEL threshold is re-evaluated authoritatively by the apply gate; the concurrency cap is TOCTOU-free; orphaned `running` rows from a crash can no longer wedge the scheduler; run-now is gated behind `loops.enabled`.
- **The API server now honors air-gap.** With the local API server enabled and air-gap on, LLM and agentic endpoints were still served on the configured port: a hole in the no-network guarantee. The server now refuses to start under air-gap and rejects requests with 403 if toggled while running; the settings section shows an air-gap notice and warns when no API key is set (open local access).

---

## [v1.0.0-beta.26.1] - 2026-06-09

A model-only hotfix: adds Claude Fable 5 and corrects stale Anthropic pricing.

### Added

- **Claude Fable 5.** Anthropic's most capable widely-released model (generally available June 9, 2026) is now selectable: as **Claude Fable 5** and a **Claude Fable 5 (1M)** variant, matching the Opus layout. It runs at a 1M-token context window natively, with always-on adaptive reasoning, vision, and tools. Available in the model picker and the Discover catalog.

### Fixed

- **Stale Anthropic cost estimates.** The Opus tier was showing $15/$75 (the old Opus 4.0/4.1 price) but it's been $5/$25 since the 4.5 generation; Opus 4.8 showed no cost badge at all; and Haiku 4.5 read $0.80/$4 instead of $1/$5. All corrected against Anthropic's current pricing.
- **1M-context variants now show a cost badge.** The Opus 4.8 / 4.7 (1M) picks, and the new Fable (1M), matched no pricing entry, so their per-message cost badge was blank.

---

## [v1.0.0-beta.26] - 2026-06-08

Bring Bodega into your editor, define your own agents, and read the docs without leaving the app. Plus a sharper Bodega Map, a faster Claude path, local embeddings, spend caps, a terminal that links straight to your code, and a long list of fixes.

### Added

- **Bodega from Zed (ACP server).** Bodega can now run as an ACP *agent* an external editor drives over stdio. Launch it headless with `bodega --acp` and register it in Zed's `agent_servers`. The client-chosen working directory is confined to an allowed projects dir (`acp.allowed_projects_dir`, default your home), tool calls route through the same permission gate, and the JSON-RPC channel stays clean. This is the inverse of the existing "run Cursor/Claude Code/Gemini/Codex inside Bodega". Now Bodega goes the other way too. (See PROVIDERS → "Use Bodega from Zed".)
- **Custom agents.** Define your own agent profiles: system prompt, pinned model, tool allowlist, read-only flag, and a max-iteration cap. Manage them in Settings → Custom Agents and pick one from the Agent panel's profile dropdown. The tool allowlist is enforced at the execution gate as an intersection with the panel's own tools, so a custom agent can only ever *narrow* what runs, never widen it.
- **Local code review.** A Review button in the Git panel runs an AI review of your current local diff: the same flow as commit-message generation, no PR required.
- **In-app docs hub.** The Help panel is now a real docs hub: a left rail of sections, search across everything, and a guide for every feature (chat, code mode, the AI panels, models, vision, reasoning, fleet, the Map, memory & knowledge, QEL, tools, terminal, custom agents, integrations, privacy, hardware) plus a full keyboard + slash-command reference.
- **Bodega Map: staleness, overview, and Q&A as a tool.** The Map flags files whose AI summary is out of date (amber dots), generates a project-overview page that rolls up every module summary, and exposes "Ask the Map" to the agent as a `query_map` tool so it can ground answers in your code mid-task. Plus per-symbol line-jump from the node drawer and a cancellable Generate-Map run.
- **Claude Fast mode.** A **Fast** toggle in the message composer (next to the reasoning control, shown only for Claude models) that skips extended thinking on Claude for snappier replies; the per-message reasoning dial still overrides it.
- **Managed llama.cpp embeddings.** Point Codebase Embeddings at a Bodega-managed `llama-server --embedding` instead of Ollama, with an installed-GGUF picker so you choose from models you already have. Settings → Knowledge → Embeddings.
- **Spend caps.** Set a hard ceiling on total cloud spend, not just Cloud Boost. New requests stop when you hit it.
- **"Air-Gap Active" in the top bar.** When air-gap mode is on, the top bar carries a small indicator so the no-network guarantee is visible at a glance, not buried in Settings.
- **QEL score in the chat sidebar.** A completed session shows its last creation task's QEL score (and pass/fail) right in the session list, so verification results read at a glance without opening the trace card.
- **Terminal: clickable file paths.** Ctrl/Cmd-click a path in terminal output (`src/foo.ts`, `src/foo.ts:42`, `:42:10`, or the TypeScript `(42,10)` form) to open it in the editor at that line. Relative paths resolve against the terminal's working directory; web URLs still open in the browser.
- **Terminal: copy, paste, and clear.** Ctrl+Shift+C copies the selection (plain Ctrl+C still sends SIGINT), Ctrl+Shift+V pastes, and a right-click menu adds Copy / Paste / Select All / Clear.

### Changed

- **Settings, reorganized.** The whole Settings surface was restructured into seven labeled groups (Look & Feel, Models & Providers, AI Behavior, Workspace, Integrations, Privacy & Safety, and About & Support) with rewritten copy and a visual pass, so options sit where you'd expect them instead of in one long list. Fleet got its own entry, RAG settings moved under Knowledge, and Knowledge picked up its own icon (it used to share Memory's).
- The two embedding configs (Knowledge & Memory vs Codebase) now spell out what each one powers, in Settings and in the docs hub, so it's clear why there are two.
- **Per-hunk review is now one click from the Agent panel.** The change list in the Agent panel adds a per-file **Review hunks** button (accept or reject individual hunks), so granular review is reachable right where you land after an edit, not only from the expanded Changes panel. Typing `/help` now lists the available slash commands.

### Fixed

- **Backend no longer lingers on port 3000.** On Windows a child process isn't killed when its parent dies, so the backend could outlive the app and squat the port. A graceful quit tree-kills it (and waits); and the backend now independently watches the app process and self-exits if the app dies abruptly (Task Manager, a crash, a dev SIGINT), so the next launch never hits "port in use" either way.
- **Backend no longer crashes when its output pipe closes.** After the app window closed or reloaded, a routine background log write could hit a broken stdout pipe (EPIPE) and bring the backend down with an uncaught exception. The logger now swallows pipe errors and the periodic cleanup timer can't throw out of its callback, so the backend survives quit/reload races.
- **Security:** MCP server-command re-validation on update, a hardened subprocess env allowlist, and `dompurify` bumped, surfaced by a clean specialist security review (0 critical, 0 high) of the whole batch.
- The fleet list now refreshes the moment a background session finishes (it could lag up to ~15s behind the completion toast).
- The Bodega Map open path is more robust across the Dockview and legacy layouts.
- The queued-message indicator is a static dot instead of a spinner.
- **Claude Fast mode now actually wins.** It was being silently overridden by the legacy extended-thinking path and the "think hard" auto-intent, so turning it on didn't always skip thinking. Fast mode now takes precedence, and the per-message reasoning dial still overrides both.
- **Air-gap now covers the Git AI features.** Commit-message generation, PR descriptions, and local code review refuse to call a cloud model when air-gap is on (and they honor spend caps), so a working-tree diff never leaves the machine. Each also gets a timeout so a stalled model can't hang the request.
- **The Bodega Map renders Markdown.** Module explanations, the project overview, and "Ask the Map" answers now render formatted Markdown (headings, lists, code) instead of raw text.
- **IDE panels stopped looking unfinished.** Two placeholder bottom-panel tabs (Output, Debug Console) that did nothing are hidden until they ship for real, and the Problems, Outline, Debug, and status-bar panels got theme-token and legibility fixes.

---

## [v1.0.0-beta.25] - 2026-06-05

The biggest beta yet. Your codebase becomes something you can talk to, agents can race each other on the same task, and you can bring your own coding agents (Cursor, Claude Code, Gemini CLI, Codex) into Bodega's fleet. Plus a verification layer you can actually see, cost tracking, faster local inference, and a long list of fixes.

### Added

- **Bodega Map: your codebase, explained.** The dependency graph grew into a living wiki. Click any file for a plain-English "Explain this module" summary (cached by content hash, so it's instant after the first time), "Generate Map" to document every file at once, or "Ask the Map" to ask questions like *"where's auth handled?"* and get an answer grounded in your real code with clickable sources. Runs on whatever model you pick, fully local if you want. (Code mode → activity bar → Map.)
- **Fleet Parallel: race models on the same task.** Fan one prompt out to multiple sessions, each running a different model or strategy in its own isolated git worktree. When they finish, compare the diffs side by side and QEL recommends the highest-scoring output. A new Fleet Monitor panel shows every in-flight run live.
- **ACP (Agent Client Protocol): bring your own agents.** Run external coding agents as Fleet members inside Bodega: Cursor (subscription auth, no API key), Claude Code, Gemini CLI, and Codex. They get sandboxed file/terminal access through Bodega's own tools, so air-gap and sandbox rules still apply. Settings → ACP Agents.
- **QEL Trace: see the verification.** The Quality Enforcement Layer now surfaces a structured trace card after creation tasks: file / pattern / framework checks, proof-gate results, and a score breakdown with a PASS/FAIL badge. Framework-aware scoring covers Next, Nest, Svelte, Hono, Elysia, Prisma, Rails and more. An optional semantic LLM-judge (off by default, never in air-gap) adds a second opinion.
- **MCP servers.** Connect external Model Context Protocol tool servers: multiple at once, with encrypted credentials, namespaced tools, isolated child environments, and an air-gap guard. Settings → MCP Servers.
- **Spending dashboard.** Track your BYOK cloud spend and Cloud Boost cost over 7 / 30 / 90-day windows, with per-provider and per-model breakdowns. Settings → Spending.
- **Reasoning controls across providers.** A per-message reasoning pill (Off / Low / Medium / High) in the chat composer for any model that supports it: Anthropic adaptive thinking, OpenAI o-series + GPT-5, Gemini 2.5+, and local native-think models. Set a global default in Settings → Models.
- **Local Performance Mode.** Speculative decoding on llama.cpp: MTP self-draft plus KV-cache quantization for faster local inference, with a VRAM-aware draft-model picker and live warnings.
- **Background / Resumable Sessions.** Send a task, click "Run in Background," and walk away. The agent keeps going (optionally in an isolated worktree) and you get a toast plus a Fleet-badge notification when it's done.
- **Persistent Knowledge Base.** At the start of each session the agent silently recalls relevant saved knowledge cards into context: no manual retrieval. Pin findings from the Research panel or add cards directly.
- **1M-context Claude.** Selectable Opus 4.8 (1M context) and Opus 4.7 (1M context) variants in the model picker; the provider sends the context-1m beta header automatically.
- **llama.cpp embeddings.** Point Codebase Embeddings at your own `llama-server --embedding` instead of Ollama. Settings → Models → Codebase Embeddings.
- **Fresh model catalog.** New local models (Nemotron 3 Nano, Gemma 4 26B / 12B / E4B / E2B VLMs, Phi-vision, Qwen3-VL-8B) and refreshed cloud flagships (Claude Opus 4.8, GPT-5.5, Gemini 3.5, DeepSeek V4, Grok 4.3), with hardware-fit scoring and a quant-tier picker.
- **Hardware-tier UX.** Bodega classifies your hardware and tailors model suggestions plus a minimal-tier Cloud Boost banner.

### Changed

- **Reasoning unified.** "Extended thinking" (Anthropic-only) merged into a single cross-provider "reasoning effort" control with a consistent Off/Low/Medium/High scale; existing settings migrate automatically.
- **App-wide render performance.** Cut re-render churn in the app root and always-mounted components for noticeably less UI lag.
- Knowledge and memory are now scoped per user, closing a cross-user read.

### Fixed

- **Cloud models showed the wrong context size.** The Context Inspector applied your local GPU's VRAM context ceiling (e.g. 64K) to cloud models that actually have far more (e.g. 1M). The real window shows now. The agent's actual budget was always correct.
- **"yo" in code mode sent the agent on a loop.** Casual greetings were classified as commands and forced tool execution until a no-progress abort. Greetings are conversational now.
- **Anthropic deprecated `temperature` on Opus 4.7+.** Bodega omits it now, so the flagship cloud models work again.
- Clarification questions no longer over-fire on clear creation tasks (including HTML/CSS).
- The MCP client was dead at runtime (SDK subpath-export load). It connects now.
- Fleet Parallel launch and execution repaired (was DOA on an API-contract drift).
- Spending dashboard crash on open, plus zeroed totals in non-UTC timezones.
- **Security:** headless-session approval gate, MCP air-gap disconnect, patched CRITICAL/HIGH dependency CVEs (vitest, systeminformation, tmp), and wiki-endpoint hardening (path traversal, SSRF, input caps).
- Backend now logs LLM API errors at error level; numerous Map, vision, and agentic-loop robustness fixes.

### Notes

- Includes the beta.24.1 hotfix (Windows portable-install integrity, local-OOM message, model-picker name).

---

## [v1.0.0-beta.24.1] - 2026-06-02

Hotfix for three issues beta users hit on Windows and with local models.

### Fixed

- **The Windows portable install (`irm … | iex`) failed for everyone** with "Could not locate sha512 … in latest.yml". The portable `.zip` was never listed in electron-builder's update manifest, so the installer's integrity check could never pass. The installer now verifies the download against GitHub's per-asset SHA-256 digest (with `latest.yml` as a fallback), and the release pipeline writes the portable hash into `latest.yml` so the manifest is complete.
- **A local model that's too big for your memory showed a useless "Something went wrong."** When a model exceeds free VRAM or system RAM (e.g. a 14B on a 16 GB card with little free memory), Bodega now tells you that's the problem and suggests a smaller model, instead of pointing you at a diagnostics export.
- **The model picker truncated the model name** behind capability badges (e.g. `q…` beside code/tools/fim). The name stays readable now; badges are capped so they can't crowd it out.

---

## [v1.0.0-beta.24] - 2026-06-01

Local vision arrives. Run a text model and a vision model side by side on llama.cpp. Attach an image to any local chat model and Bodega routes it to your bound VLM, hot-swapping the model for the answer and swapping back automatically, entirely on your machine. Plus an in-app changelog, a refreshed guided tour, and an extended beta window.

### Added

- **Local vision (two-model VLM) on llama.cpp.** Download a vision model (e.g. Qwen2.5-VL) from Discover, bind it in Settings → Models → Vision, then attach images to any local text model. Bodega hot-swaps the `llama-server` process to the VLM for the answer and swaps back: no cloud, no data leaving your machine.
- **Inline vision-swap narration.** The swap shows as a quiet inline chat card ("Loading vision model… → Analyzing image…") with the model name and elapsed time, instead of a full-screen overlay.
- **Changelog in Settings → About.** Browse the full release history in-app. The "What's New" popup now shows only the current release.
- **Guided tour overhaul:** chapter-based and replayable any time from Settings → About.

### Changed

- **Beta period extended to November 1, 2026.**

### Fixed

- Local vision swaps no longer flash a generic full-screen "Switching models" modal, and the automatic swap-back is silent.
- Beta builds could not activate once the prior beta window lapsed, resolved by the extension above.

---

## [v1.0.0-beta.23] - 2026-05-25

Twelve-day §D cycle around the Dockview layout migration, two-model VLM orchestration, and a polish pass driven by Joe's live-smoke + four-agent review (Reviewer / Sentinel / QA / Doc Guardian). **73 new tests** across the dockview suite, **~50 commits since beta.21**. Tag headline: *panels you can move, vision that just works, dropdowns that no longer hide behind the editor.*

The beta.22 work was developed in-tree but never tagged. Beta.23 ships as the consolidated release.

### Added: UX polish + drag-over highlight (post-Day 12)

- **Direction B "Void Float" aesthetic:** panels float on a darker void with subtle card lift. `dockview-spaced` activated, `bg-base` parent floor, inset white top-edge highlight + outer drop shadow for dual-theme card-lift, editor canvas tinted via `.bodega-editor-canvas` class for distinct color, 2px tab radius (whisper-rounded).
- **Drag-over panel highlight:** new `useDockviewDragOverHighlight` hook + matching CSS. The panel under the cursor mid-drag gets a solid purple outline ("drop will land here"); during any dockview drag, all panels get a faint dashed outline ("any of these are valid targets"). Capture-phase listeners + `document.elementsFromPoint()` walk to find the underlying `.dv-groupview` past dockview's drop-target overlay.
- **`dndStrategy="html5"` forced** on DockviewReact: dockview's `auto` strategy disables HTML5 events on coarse-pointer-reporting devices (touchscreens, Wacom). Forcing HTML5 guarantees consistent dragstart/dragend across all desktop setups.
- **Settings becomes a real editor tab** (Option A migration): Settings now coexists with Preview/Editor as a tab in the editor area instead of overlaying as a secondary panel. New `services/settings/openSettingsService.ts` makes this mode-aware: code mode → editor tab; chat mode → secondaryPanel overlay (legacy).
- **3 composer dropdowns Portal'd** to escape Dockview's transform-ancestor stacking context: `ProviderModelPicker`, `PermissionModeDropdown`, `AttachDropdown`. z-index bumped to `[10000]` (above Dockview's drop-target at `9999`). Click-outside detection updated to respect Portal'd content (two-ref pattern).
- `aria-hidden` → `inert` on TopBar dropdown menus (proper focus management vs aria announcement-only).
- Legacy mode terminal Maximize button restored (was incorrectly nuked in early polish), terminal dock-side toggle gated to legacy mode.
- `panel_sizes` and `recent_projects` JSON.stringify 400-validation fixes.

### Added: §D Dockview migration (Days 7-12)

Replaces the CSS Grid + react-resizable-panels tree in Code mode with [Dockview v6.5.0](https://github.com/mathuo/dockview) gated behind `ui.layout_engine` (default `'dockview'`). Users keep the legacy tree as a one-flip Settings escape hatch through beta.23+.

`CodeLayout.tsx` becomes a 33-line shim picking between `DockviewLayout` and `LegacyCodeLayout` (the old body preserved verbatim, 269L). New `apps/desktop/desktop-app/src/components/ide/dockview/` folder ships nine focused files: pure ID/title/flag constants (`panelIds.ts`, 55L), the React-component map (`PanelRegistry.ts`, 46L), first-launch SerializedDockview JSON (`defaultLayout.ts`, 119L + 8 tests), debounced persistence with migration guards + `__proto__` strip + 512KB JSON cap (`layoutPersistence.ts`, 205L + 20 tests), the renderer host with `renderer:'always'` tagging of terminal/preview/panelSidebar/fleet (`DockviewLayout.tsx`, 200L + 8 tests), the first-launch dialog with bulk-write + error state (`MigrationDialog.tsx`, 120L + 9 tests), and a 41-line terminal wrapper that registers with `TerminalProvider` for 3-slot reparenting.

Critical contracts pinned by tests (already in dev from Day 6): `TerminalProvider.reparenting.test.tsx` (6 tests): xterm scrollback + PTY survive chat→code→dockview slot moves; `PreviewBridgeReceiver.dockview-lifecycle.test.ts` (6 tests): Electron `<webview>` ref + WebContents IPC survive `renderer:'always'` hide/show. The post-load + `api.onDidAddPanel` tagging loop covers both initial layouts and dynamic `api.addPanel` calls (e.g. `/preview` slash command, PortsPanel "Open in Preview").

Layout reset goes through `requestLayoutReset()` which atomically clears the persisted value AND dispatches a `bodega:layout-reset` window event that `DockviewLayout` re-hydrates from. Decouples the trigger (Settings, slash commands, menu items) from the renderer.

Theme: `dockview-theme-bodega.css` overrides 18 `--dv-*` variables onto Bodega's `--color-noizey` / `--color-surface` / `--color-text-*` tokens so the abyss palette doesn't bleed through to obsidian-wave-light + bodega-light themes.

`source-of-truth/beta.22-smoke-matrix.md` (NEW, 9 sections, ~50 ticks) is Joe's manual checklist for the tag: covers migration dialog flows on all 4 themes, default tree, persistence + corruption recovery, terminal 3-slot reparenting, preview webview survival, agent state, fleet, regression, crash/recovery, small-viewport (1280×720).

### Added: Two-model VLM orchestration (Days 3-5)

Text-model driver + vision sub-tool architecture lets any text model use vision automatically when a VLM is locally available. New `VisionQueryTool` (95L) exposes `vision_query(imageId, question)` to the agent. `BoundVisionService` auto-pairs the smallest installed VLM via parameter-size regex on a `<32x32 PNG live probe; air-gap mode + non-local Ollama URL = refuse to bind. `VisionRouter` (175L) routes `imageRegistry → boundVisionService → Ollama /api/generate` with IDOR-safe error messages (never echo the imageId). `SessionImageRegistry` (100L) is a nested `Map<sessionId, Map<imageId, dataUrl>>` with FIFO eviction at 10 per session and O(1) clearSession() called from session-delete paths. SSE event #21 `vision_unbound` surfaces an inline card when no VLM is bound and one isn't installable. 84 backend tests + 60 frontend tests.

### Added: SSE event 21

`vision_unbound` (chat-stream.ts): emitted when the agent attempts vision and no bound VLM exists. Frontend renders `VisionUnboundInlineCard` (status role, aria-live polite) with two copy variants: "no vision model is installed" or "a vision model is installed. Bind one explicitly". Sentinel-tested invariant: never echoes a model name or removal narrative.

### Changed

- `SettingsValidator` JSON case caps payloads at 512KB and accepts `null` for reset paths (Sentinel beta.22 HIGH). Circular-reference rejection added.
- `flushPendingLayoutSave` now FIRES the queued layout write synchronously on `beforeunload` / `visibilitychange:hidden` / React unmount. Previously discarded the last drag. Reviewer beta.22 M-2.
- `MigrationDialog` uses `apiClient.updateSettings` (bulk) instead of two sequential `updateSetting` calls so engine + shown flag persist atomically.
- 6 god-file splits in Day 2: `LLMService` 816→640L (extracted `llm/ByokMigrationRunner`, `PresetChangeCleanup`, `ProviderConstructor`, `RoleModelPruner`); `WorktreeManager` 913→669L; `server.ts` 605→414L; `session-background.ts` 520→274L; `useChat.ts` 597→507L; `ChatStage.tsx` 525→434L; `helpData.tsx` 402→9L.

### Fixed: Error classification

- **DeepSeek "Request failed" mystery**: HTTP 402 + "Insufficient Balance" + 6 sibling patterns now classify as `payment_required` `ErrorCategory` with provider-specific billing URLs (DeepSeek, OpenAI, Anthropic, OpenRouter, Mistral, Together, Groq, Fireworks). Replaces the previous generic "API error" with a clear "Add credit at <url>" surface. (`ProviderErrorClassifier.ts`)
- **VLM probe PNG CRC32 ship-blocker**: `BoundVisionService` `liveProbe` PNG had bad PLTE+IDAT CRC32 chunks; Ollama qwen2.5vl rejects the probe so auto-binding silently failed in production since Day 3. Regenerated via Node `zlib` + added regression test walking CRC32 of all chunks.

### Day 1 cleanup (5 bugs, all fixed)

- C1: "Take Screenshot" attachment button verified + toast wiring.
- C2: POST/SSE double-pull idempotency guard + preserve terminal status before activeDownloads.delete.
- C3: qwen2.5vl probe 400. Broaden llava pattern to catch llava-phi3 family.
- C4: clearSessionApprovals hookup + drain pending requests + E2E test for keep_history=false path.

### Security

- 512KB JSON cap on every `type:'json'` setting (SettingsValidator)
- `null` permitted for json settings (reset paths)
- Circular-reference rejection (JSON.stringify catch)
- `__proto__` chain stripping via JSON round-trip in `resolveDockviewLayout`
- `imageData` byte-length capped at 5MB before SessionImageRegistry stash
- Air-gap explicit guard added to `BoundVisionService` + `VisionRouter` (non-local Ollama URL + air-gap → refuse to bind)

### Tests

- Frontend: 1124 pass / 5 todo (+73 from beta.21's 1051): dockview suite contributes 60+ new tests, plus DragOver + Settings + helpDataFaq refresh
- Backend: 4962 pass / 54 skipped (+103 from beta.21's 4859): vision orchestration + SettingsValidator JSON cap + ProviderErrorClassifier payment_required cases
- SSE event count: 20 → 21
- No regressions

### Docs

- **In-app help refresh** for beta.23: Dockview FAQ added, Preview tab FAQ added, attach menu updated for screenshot/image, slash commands list reconciled to what's actually registered, Settings Layout section added, air-gap layer count corrected 10 → 14 (Sentinel-flagged accuracy drift). `helpDataFaq.tsx` 400L (at limit), `HELP_SECTION.md` minor edits. (Doc Guardian agent pass)
- `source-of-truth/deferred/beta24-followups.md` (NEW). 7 items documented for future revival: snap-to-close (root-cause Windows pointer-event mystery first), llama.cpp two-model VLM adapter (spec exists), chat-stream.ts split (HIGH god-file), WorktreeDryRun dedup (LOW), end-to-end vision UI smoke, Take Screenshot IPC smoke, cross-session awareness implementation.

### Snap-to-close: DEFERRED to beta.24+

`useDockviewSnapToClose.ts` works under computer-use mouse events but doesn't fire reliably on Joe's actual Windows + touchscreen setup despite 11 rounds of fixes (capture phase, `elementsFromPoint` walk, 80px zone, visible "Drop here to close" indicator, `dndStrategy='html5'` force, `lastDragoverY` fallback). The hook file is preserved on disk for revival once the pointer/HTML5 backend mismatch on coarse-pointer-reporting devices is properly diagnosed. Call site is commented out in `DockviewLayout.tsx`.

### Post-ship hardening: PR #451 (16 rounds of live-feedback fixes)

A bugbash driven by Joe's sanity test before binary build surfaced sixteen tightly-scoped issues across drag-drop, split-view, and project-switch flows. Each round landed as a focused commit on `fix/beta23-dockview-default-panel-sizes`. Highlights:

**5-zone drop overlay (rounds 1-2, 9-14).** The earlier drag-highlight was a single full-panel outline that didn't tell users where the drop would actually land. New overlay shows five labeled quadrants (`Split above` / `Split right` / `Split below` / `Split left` / `Add as tab`) with the active zone filled bright purple. Achieving 1:1 visual fidelity with Dockview's underlying drop math took:
- Anchoring the overlay on `.dv-content-container` (not `.dv-groupview`) so the tab bar isn't part of the "top" zone (Joe round 11).
- Matching Dockview's zone-priority order (`left → right → top → bottom → center`) for corner cursor positions (round 12).
- Re-anchoring on `document.body` with `position: fixed` + screen-space rect, so the overlay shows over `renderer:'always'` panels whose content lives in a sibling `.dv-render-overlay` element (round 13).
- Suppressing the overlay on the source group entirely. Dockview no-ops same-group drops, matching VS Code / JetBrains (round 11).
- Universal `pointer-events: none` on overlay + every descendant. CSS `pointer-events` doesn't cascade, so the inactive zones were silently eating dragover events without the `*` selector (round 10).
- `mouseup` cleanup as a safety net for the Electron dragend-swallow case (round 14).

**Drop-to-Agent-panel fix (round 15).** The Agent panel silently rejected every dropped tab while every other panel accepted them. Root cause was our own `useFileDrop` hook (wired to `AgentChatPanel` for OS file-attach): every handler called `e.preventDefault()` + `e.stopPropagation()` unconditionally (including for Dockview's internal tab drags) swallowing them before Dockview's drop forwarder could see them. Now every handler early-returns unless `dataTransfer.types.includes('Files')`, cleanly separating OS file drops from internal panel drags.

**Project-switch crash fix (round 16).** Switching projects via the hamburger menu crashed the entire code panel with `Cannot have two HTML5 backends at the same time`. react-arborist's `<Tree>` internally renders its own `<DndProvider backend={HTML5Backend}>`, and react-dnd's HTML5Backend throws if a second one ever calls `setup()` while the global `__isReactDndBackendSetUp` flag is still set. New `fileTreeDndManager.ts` creates exactly ONE `DragDropManager` at module load, caches it on `globalThis` (HMR-safe), and passes it to `<Tree>` via the `dndManager` escape-hatch prop. No more concurrent backends, no more crash.

**Split-view editor polish (rounds 3-8).** Ctrl+\ split now opens with the secondary pane mirroring the primary's full feature set: independent Monaco model per tab (URI prefix `split-secondary:`), file-type icons in the secondary tab bar (`getFileIcon`), breadcrumb row that tracks the secondary tab via a new `tabIdOverride` prop on `<Breadcrumbs>`, and click-to-switch tab affordance. Settings-in-split now suspends the split for non-file primaries (Settings/Browser/Preview) instead of rendering a blank Monaco. llama.cpp UI gated behind `state.localPreset === 'llamacpp'` in MyModelsTab to hide it when the user is on Ollama/cloud.

**Sidebar width preservation (rounds 5-7).** Closing one sidebar panel no longer collapses or balloons the other: `enforceSidebarWidths` runs inside `requestAnimationFrame` after `fromJSON` to lock pixel widths past Dockview's own layout pass, and re-runs via `onDidRemovePanel` whenever a sidebar group loses a panel.

**Test mock parity.** `DockviewLayout.test.tsx` mock api extended with `getPanel` (round 10) to match the new `enforceSidebarWidths` rAF callback. All 9 tests still pass.

### Reviewer + Sentinel + QA agent pass (4 agents, 0 blockers)

- **Reviewer**: APPROVE with 2 mediums (M-1 type cast in `useDockviewStateSync`, M-2 indicator DOM cleanup), **both fixed**.
- **Sentinel**: 0 critical/high, 1 medium (same M-2), 2 low/informational. All prior Day 7 findings (512KB cap, `__proto__` strip, null-permitted, circular guard) confirmed in place.
- **QA**: PASS, +73 tests vs baseline. Single P1 follow-up = add tests for `useDockviewDragOverHighlight` before next change (deferred to beta.24).
- **Doc Guardian**: in-app help refreshed (above).

### Known follow-ups (tracked for beta.24+ in `source-of-truth/deferred/beta24-followups.md`)

- Snap-to-close revival once Windows pointer-event diagnosis lands
- llama.cpp two-model VLM adapter (spec at `source-of-truth/specs/beta23-llamacpp-vision-adapter-spec-2026-05-24.md`)
- chat-stream.ts split (550L → ~370L), last HIGH-risk over-limit file
- WorktreeDryRun.ts dedupe (validation + runGit copies)
- `useDockviewDragOverHighlight` test coverage
- LegacyCodeLayout retirement after beta.23 confirms no Dockview regressions

---

## [v1.0.0-beta.21] - 2026-05-22

Three-day cycle driven by **Cachev's beta.20 feedback** + Joe's live smoke. Two big features, eight tracks of layout polish, twelve smoke-fix commits, and 108 new tests. **39 commits** since beta.20. Tag headline: *the agent can see the preview now, local-first, no cloud round trip.*

### Added: PreviewInteractionTool (agent ↔ webview bridge)

Single tool entry `preview_interaction` with action discriminator. Five actions: `screenshot` (PNG via `<webview>.capturePage()`), `getConsoleErrors` (200-entry ring buffer of sanitized console messages from the preview), `getDom` (outerHTML by CSS selector with 8 KB cap + credential scan), `navigate` (localhost-allowlist gated), `click` (per-URL-origin user approval gate). The screenshot flows through `MultimodalResultInjector` into a follow-up `role:user` message with `images: [base64]`. Ollama / OpenAI / Anthropic converters already handle the field, so the per-provider image-block serialization works for free.

Architecture in three layers: backend `PreviewInteractionTool.ts` validates inputs (tightened selector regex, localhost allowlist via the new shared `apps/desktop/shared/isLocalhostUrl.ts`); `PreviewActionDispatcher` (singleton) mirrors the existing `pendingApprovals` SSE relay pattern: emits a `preview_action_request` SSE frame keyed by `requestId`, blocks on a Promise; renderer's `PreviewBridgeReceiver` (module-level singleton) executes against the active `<webview>` via the new `main/ipcPreview.ts` IPC bridge, then POSTs the result back to `/api/chat/preview-action-result/:requestId` which resolves the dispatcher's pending Promise. SSE event type 19 → 20 (added `preview_action_request`).

Security: JSON.stringify selector interpolation is MANDATORY (Sentinel rev2 HIGH-01), selector regex tightened to reject quotes/parens (defense in depth), per-URL-origin click approval cache (not once-per-session), credential scan on DOM outerHTML + console messages (CREDENTIAL_PATTERNS mirror from ShellTool, backend + renderer copies kept in sync), ANSI + Unicode bidi-control stripping at capture time, static system prompt guardrail telling the model DOM + screenshot content is untrusted input not instructions, 12-second IPC timeout, graceful "preview_not_open" on closed tab. Two-layer localhost enforcement: tool layer + webview `will-navigate` blocker. Five bypass patterns explicitly tested: `127.0.0.1.evil.com`, `localhost@evil.com`, `localhost.evil.com`, `[::1]@evil.com`, `javascript://localhost`.

**108 new tests** across 7 files: tool unit (29), dispatcher (12), shared isLocalhostUrl (21), multimodal injector (7), credentialScan renderer (19), previewConsoleSlice (7), PreviewBridgeReceiver (13). Backend `buildLlamaServerArgs` extracted to a pure helper for testability (9 tests).

### Added: llama.cpp VLM support (paired mmproj download + `--mmproj` spawn)

The agent screenshot loop now works against llama.cpp vision models, not just Ollama. Catalog v1.2.0 adds `mmproj` block to the entry schema (HF repo + filename + sizeGB) and ships two vision entries: `llava-1.6-mistral-7b` (Mistral 7B vision, 32k ctx, with mmproj-model-f16.gguf) and `moondream2` (1.8B small/fast vision, 2k ctx). Picking a vision entry in Discover kicks off paired download: primary GGUF via the existing resumable path, then the mmproj sibling via a simpler magic-byte-validated fetch. `llamacpp_models` schema gained `mmproj_path TEXT` (with idempotent ALTER TABLE for existing installs). `LlamaServerManager.buildSpawnOptions` reads the path off the registry row; `LlamaServerProcess` appends `--mmproj <path>` to the argv when set; `ModelProfileData` picks up `llava-1.6-mistral-7b` and `moondream2` as keys so the agent loop's vision-capability check (`profile.capabilities?.includes('vision')`) returns true for the llama.cpp catalog IDs.

### Added: Layout Expansion (eight tracks, Cachev coexistence feedback)

Reverses C3 D2 (Preview lived in bottom-panel; Cachev's review showed code + preview + terminal *all visible* is the actual demand). Track A: Preview moves to editor-area tab via central `openPreviewService.openPreview(url, opts)`: one action, four triggers (auto-toast, agent, slash-command, ports-panel), Layer 1 localhost validation, single-tab pattern (opening a new URL replaces the existing Preview tab). Track B: auto-open toast + notification dot when `usePreviewAutodetect` sees a localhost dev server in terminal stdout: toast offers "Open Preview" (10s), notification dot on the Preview tab when one is open and a new URL is detected. Track C: global panel-size persistence via `ui.panel_sizes` settings key (one set across all projects, matches VS Code): fileTreeWidth, agentChatWidth, chatSidebarWidth, chatPreviewWidth, terminalSplitRatio, terminalDockSide, explorerSplitRatio. Track D: per-workspace preview URL persistence via SHA-256-first-16-chars hash of project_path. Track E: `/preview <url>` slash command (bare `/preview` opens the port-preset empty state). Track F: close-X visual separation between Bodega toggles and native title-bar close: 1 px divider on Windows/Linux, widened after live smoke. Track G: `ChatStage.tsx` split (extracted `ChatStageInputArea`). Track H: `QuickStartChips` removed from chat mode (always was code-only).

### Added: Provider-runtime quick-install in Providers tab

Users who onboarded with a cloud API key skipped the local-provider install during onboarding. Settings → Models → Providers now exposes a Quick-install card per local provider that hides itself if the runtime is already installed: Ollama gets `ollamaInstall.startInstall()` (same managed installer the onboarding flow uses), llama.cpp gets `llamacpp.startInstall('auto')` (managed binary install, SHA-256 verified). Status probe on mount hides the card when the runtime is present.

### Added: Vision capability chip + UX touchups across the model pickers

New amber `Vision` chip in `ProviderModelRow` + `ModelCard.ModelBadges`, threaded through `ModelProfile.supportsVision` from a new `modelProfileService.modelSupportsVision()` backend helper. Lights up for both cloud VLMs (Claude 4.x, GPT-4o, Gemini, qwen-vl-max) and local VLMs (qwen2.5vl, llava, moondream, minicpm-v, the new llama.cpp catalog entries). Chips priority: vision > thinking > code > tools > fim > fast. Cap of 3 chips per row.

Pickers retired the native `<datalist>` (flat alphabetical wall, scrolled the page, got cut off in non-fullscreen layouts) for a custom popover: provider-grouped (Anthropic / OpenAI / Google / Qwen / Meta / Mistral / DeepSeek / Other), groups collapsed by default with chevron toggle, typing auto-expands matching groups, max-h-72 + internal scroll. Compact mode (right-rail code-mode picker) used to strip every chip with `compact ? []`. Now preserves them at the 3-chip cap so width stays bounded and information density is the same as chat mode.

### Added: Sidebar Preview button + welcome-screen CTA + topbar polish

ActivityBar (left rail) gets a Preview action button below Fleet, separated from the panel-tab group by a 1 px divider so it doesn't read as a fifth tab. Highlights when the Preview editor tab is active. EditorWelcome (empty-editor state) adds "Open Preview" as a peer button to "New File" / "Open File…". `/preview` with no args opens the empty-state port picker (3000 / 5173 / 8080 / 4200) instead of the prior "Use /preview <url>" toast.

Topbar polish: removed the secondary-panel close-X (it sat next to the native title-bar close-X and confused users, Cachev + Joe both flagged). Secondary panels (Settings / Usage / Help / Agents / Network) now get their own floating close-X in the top-right corner of the panel region, in both ChatLayout and EditorArea. Native close-X separation widened to 5 px gap + full-opacity border so the visual break reads at normal viewing distance.

### Fixed: `Pull Any Model` silently lied "ready in My Models" for models Ollama rejected

Joe smoke 2026-05-22. `parseNdjsonStream` only broke the loop on `status === 'success'`; when Ollama emitted `{"status":"error","error":"pull model manifest: file does not exist"}` (e.g., a typo'd model name), the parser swallowed it, the stream closed cleanly, and the route synthesized a success event for the SSE consumer. User saw "ready in My Models" for a model that never downloaded. Parser now throws on error events with the Ollama-provided message; the outer try/catch promotes it through to the SSE `.catch` which emits `status:'error'` → frontend's onError fires the "Failed to pull" toast with the real reason. Plus the recommended Ollama VLM tag corrected: `qwen2.5vl:7b` (no dash) instead of the bogus `qwen2.5-vl:7b` that 404'd on Ollama's registry.

### Fixed: Pull progress lost on tab navigation + "already being downloaded" lockout

Mid-pull, navigating away from Discover unmounted the local state; coming back showed no progress and the next Pull click hit a 409 ("already being downloaded"). Two fixes: POST `/api/model-hub/pull` now returns 200 with `{ attached: true }` instead of 409 when the model is already in flight. The frontend opens SSE which attaches to the existing pull via the existing `isDownloading()` polling branch. `DiscoverTab` on mount probes `getActiveModelDownloads()` and re-attaches the SSE stream for any in-flight pull (with a new `attachToActivePull` helper). Mid-pull navigation now round-trips cleanly: progress bar reappears where you left off.

### Fixed: Preview tab "closed" when typing in chat input + pressing Enter

Joe smoke 2026-05-22. Focus shifts off the webview during chat input + submit fire `ERR_ABORTED (-3)` on the in-flight navigation. Electron logs `'GUEST_VIEW_MANAGER_CALL': ERR_ABORTED (-3) loading 'http://localhost:5173/'` twice (once for click, once for Enter). The aborted webview goes blank; users read "blank" as "the tab closed." Before: `did-fail-load` with -3 silently set `onLoadingChange(false)` and gave up. Now: on -3, throttle-reload(50 ms) on the same src once per second. -102 (CONNECTION_REFUSED) still shows the error card; -3 self-heals silently.

### Fixed: Anthropic vision converter dropped raw base64 silently (the actual reason Sonnet 4.6 "read HTML instead")

Joe smoke 2026-05-22 with Claude Sonnet 4.6: agent called `preview_interaction screenshot` + `navigate`, then fell back to `file_system.read('index.html')` instead of describing the page. Root cause was `AnthropicProvider.convertMessages`: it required a `data:image/...;base64,...` data URL and silently dropped images that arrived as raw base64. `PreviewBridgeReceiver.doScreenshot` emits raw base64 from `webview.capturePage().toPNG()`. Ollama strips the `data:` prefix if present, OpenAI wraps with `data:image/png;base64,` if missing. But Anthropic's regex `/^data:(image\/[^;]+);base64,(.+)$/` failed to match, the image block was never added to the user-message parts, and Claude saw only the text caption "Captured 1280×720 PNG of http://localhost:5173/". Without an image to describe, the model reasonably fell back to reading the HTML directly. Fix: Anthropic now defaults `media_type` to `image/png` when no `data:` prefix is present. All three provider converters share one contract. Four new vision unit tests cover the raw-base64, data-URL, empty-string, and mixed-format paths so this can't regress silently.

### Fixed: Late-day smoke pass: four stacked bugs that were keeping vision broken

Joe drove three live smoke runs after the initial tag prep; each one peeled back another layer. The full stack of fixes that landed before tag:

1. **Preview tab kicked out on every chat send.** `applyActiveCodeSessionId` in `codeSessionSlice` wiped `editorTabs:[]` whenever the active code session changed, including the first send that creates a fresh session. PreviewWebview unmounted mid-send, `setWebviewRef(null)` fired, the agent's screenshot landed against a null ref returning `preview_not_open`, and Claude fell back to reading source files. Fix: preserve `type:'preview'` tabs across session changes (they're URL-scoped to the project, not session-scoped). File tabs still get cleared as before.
2. **`display:none` on PreviewPanel suspended the webview's RenderFrame.** First attempt to keep PreviewPanel always-mounted used `display:none` for the inactive state. Electron 41 treats `display:none` on a `<webview>`'s container as a frame-suspension trigger; the next `capturePage()` crashed the renderer process, blanking the whole window and detaching DevTools. Fix: `visibility:hidden` + `pointer-events:none` + `zIndex:-1`. Keeps the WebContents alive without painting pixels.
3. **`<webview>.capturePage()` itself crashes Electron 41 on a persistent partition.** Even with PreviewPanel always-mounted and visibility-hidden correctly, the native call from the renderer reliably killed the renderer process 296 ms after invocation. Fix: moved capture to the main process via a new `preview:capture` IPC handler that calls `webContents.fromId(id).capturePage()` instead. Renderer hands over the guest WebContents ID; main captures and returns base64. `webContents.capturePage()` from the parent Chromium process is stable for this API.
4. **`truncateResult` cut the base64 image mid-string, JSON.parse failed, no image followup was built.** With main-process capture working, image bytes reached the tool pipeline. But the 16 KB tool-result cap was slicing a real 100 KB PNG base64 in half. `buildImageFollowupMessage` tried to JSON.parse the truncated string, failed, returned null, and Claude only saw the text "[partially truncated]", hallucinated the product as "Nexus" instead of "Pulse." Fix: new `buildImageFollowupFromRecord` operates on the raw `record.result` object before stringify+truncate, and `stripScreenshotImageData` replaces the bulky base64 with a length placeholder in the tool-result text so the image flows ONLY via the followup user message.

Live-verified end-to-end via computer-use against the Demo_Pulse landing page with Claude Sonnet 4.6: bit-perfect description of product name ("Pulse"), exact two-line headline, and every detail of the phone mockup (greeting, three habit names with times, 14-day streak indicator). One tool call, 5.8 s round-trip, zero crashes, zero timeouts.

### Fixed: Editor Welcome dead-end + B-track smoke bugs

Welcome screen Send button was decorative. Clicking the chip + arrow combo did nothing on first open (no project / no model). Now wires through `handleSubmit` so the user can land on the welcome screen, pick a chip, click send, and the new-session flow fires. Ctrl+Alt+N added as a global new-session shortcut (Ctrl+N reserved by Monaco). C/C++/Arduino syntax highlighting added to Monaco (qwen2.5-coder users with embedded projects were getting plaintext rendering of `.cpp` / `.h` / `.ino` files). NSIS installer finish-page text was black-on-black on Windows 11, re-themed. File Explorer search no longer crashes on deep trees. Sidebar search field no longer hides the close-X behind the input. `+` button added to the file tab bar for New File / Open File (Cachev observation: there was no obvious empty-state action). "Select Folder" button added for working-directory picker.

### Fixed: Demo_V2 dino-game polish (jump physics + game-over overlay)

Jump velocity / gravity retuned from `JUMP_VEL=-11, GRAVITY=0.85` (apex ~70 px, airtime ~28 frames, felt floaty) to `JUMP_VEL=-9, GRAVITY=0.9` (apex ~45 px, airtime ~20 frames, Chrome-dino feel). Game-over overlay alpha bumped 0.7 → 0.92 + heavier `backdrop-filter` blur so the frozen game frame reads as a backdrop, not "is the game still running?" Joe-flagged 2026-05-22.

### Added: Demo_Pulse (fictional landing page for vision-model demos)

`Desktop/Dev/Demo_Pulse`: single-page habit-tracker landing page (`Pulse`). Visually rich on purpose: sticky nav with logo + CTA; gradient hero with a CSS phone mockup showing morning routine + 14-day streak; 4-stat strip (active members, habits checked off, App Store rating, median first-streak); 6-card feature grid with colored icons (time-of-day windows, forgiveness mode, weekly review, one-tap check-off, smart pause, export everything); 3-tier pricing table with "Most chosen" badge; 3 testimonial cards with avatars; CTA form + footer. Static HTML + CSS, no build step. Pairs with the Preview tab as a demo target for the PreviewInteractionTool screenshot path.

### Test counts

Backend **4,859** passing / 54 skipped / 15 todo (+84 since beta.20 across the PreviewInteractionTool suite, llama.cpp VLM catalog/spawn coverage, and the GgufVisionFlow invariants). Frontend **1,051** passing / 5 todo (+60 across credentialScan renderer mirror, previewConsoleSlice ring buffer, and PreviewBridgeReceiver action dispatch). Backend tsc clean. Frontend tsc clean. Both ESLint clean at `--max-warnings 0`.

### Deferred to beta.22

- **Movable / re-arrangeable panels (Cursor-style)**: your D5 deferral from the Layout Expansion spec. Research done in task #23 (`react-mosaic` / `react-grid-layout` / `Dockview`); library pick + integration deferred.
- **Two-model VLM orchestration**: local 7B vision models can't reliably drive the agent loop (capability ceiling, not a bug; live-confirmed on qwen2.5vl:7b which made it through 10 iterations with 0 tool calls). Beta.22 design: text model orchestrates + calls a vision sub-tool that invokes the VLM purely for image Q&A. Unlocks the local screenshot loop on commodity hardware. Cloud vision (Claude 4.x, GPT-4o) handles the loop end-to-end today.
- **Take Screenshot attachment button live-verify** (B3, task #10).
- **POST/SSE double-pull race** in `model-hub.ts:146`: wasteful but Ollama is idempotent so user never sees it.
- **qwen2.5vl probe 400**: `ModelCapabilityProber` sends a request shape qwen2.5vl rejects; cosmetic warn-level log only.
- **Per-URL click approval session-boundary clear hookup**: `clearSessionApprovals()` is wired but `resetReadTracking()` only fires from tests today; cache is effectively process-lifetime instead of session-lifetime. Within-session gate works exactly per spec.

---

## [v1.0.0-beta.20] - 2026-05-19

Six days of work, **41 PRs (#389–#429)**, organized as three overlapping waves: **multi-provider routing recovery** (per-session provider stickiness so chat doesn't dredge a stale Ollama config when you switch to Kimi/Qwen mid-conversation), the **Fleet feature** (background agent sessions running in isolated git worktrees, with per-session worktree provisioning, apply/discard/merge UX, status indicators, oscillation guards, and a Send-to-Fleet button on the Agent panel), and a **two-day live-driver smoke pass** that uncovered a security-grade write-leak regression and four UX bugs which were all closed across PRs #428 + #429.

### Added: Fleet: background sessions in isolated git worktrees

Code-mode now has a Fleet sidebar (activity bar icon next to Files/Search/Git). Promote any session to the fleet via the "Send to Fleet" button on the Agent panel header (PR #426). Once promoted, the session runs in a per-session git worktree at `<userData>/bodega-worktrees/<project>/session-N/` on the branch `bodega/session-N`. The agent loop's `projectPath` is substituted to the worktree path so file_system writes, str_replace, shell, and git ops land in isolation. Apply with merge or squash strategy (auto-commit + auto-merge on success); discard removes the worktree + branch. Each FleetCard shows live activity ("Iter 5/13: read style.css"), a stale-amber timestamp, model badge, and an inline diff preview. Fleet cap of 4 concurrent background sessions. Sessions survive SSE disconnects (you can navigate away tabs, close panels, or switch modes. The loop keeps running and you re-attach via the live-stream endpoint when you come back).

Twelve PRs went into Fleet specifically: #401 (Phase D-E polish: notifications + keyboard + settings + onboarding tooltip), #402 (discard confirmation modal), #403 (AgentChatPanel session indicator), #404 (TopBar fleet status indicator), #405 (resource bounds + non-git degraded mode), #406 (worktree GC + disk pressure warnings), #407 (SessionsDrawer + context-menu fleet integration), #408 (diff polish: binary detection, pagination, conflict UX), #409 (FleetCard live activity + Active badge), #420 (sessions as first-class persistent agent runs), #423 (Apply auto-commits + multi-file QEL oscillation guard), #426 (Send-to-Fleet button on Agent panel).

### Added: Multi-provider routing recovery

Sessions are now sticky to the provider they were started with. PRs #389–#400 close the dredged-Ollama-fallback class of bugs that were biting beta testers (#338-era reports). Each session's `provider_context` is persisted to SQLite after every successful turn (preset id + actual model name as returned by the API). Subsequent turns read from there before falling back to the global `llm.preset`. Two-tier model picker drill-down (PR #397) replaces the flat dropdown: Cloud/Local groups, then provider, then model. Mid-session provider switches now warn ("session-1 was running on Kimi. Sure you want to send the next turn to DeepSeek?") but never block (PR #398). Picker pre-warms the provider catalog cache so cold-launch picker open isn't a 3-second freeze (PR #399). Settings → Models cleanup (PR #400) consolidated the 5-section sprawl into a coherent multi-provider layout.

### Fixed: Fleet write isolation (security-grade, beta.20 day-2 regression)

PRs #421 + #424 + #427 wired the worktree-path substitution into the chat-stream handler. PR #428 closed a regression that surfaced during the live driver smoke on 2026-05-19: when `BODEGA_USER_DATA_DIR` wasn't visible to the chat-stream process at request time, the resolver was falling through to `process.env.APPDATA` (a different directory on Windows), the synthesized base didn't match the stored `worktree_path`, validation correctly returned false, and the resolver classified this as "tampered" and **silently fell back to the request's projectPath: the live project tree**. Agent writes went to the live project; the worktree stayed clean. Reproduced live: `lib/constants.ts` showed `APP_AUTHOR = 'Joe'` uncommitted on master after a fleet session was supposed to write to its worktree.

The PR #427 unit tests passed but masked this code path because every test passed `userDataDirOverride`. Production never does. PR #428 drops the `APPDATA / XDG_DATA_HOME / HOME` fallback chain entirely (`backend-manager.ts:111` always sets `BODEGA_USER_DATA_DIR` in every Electron-spawned backend), changes the `tampered` and `lookup-failed` decisions to return `effectiveProjectPath: undefined` for fleet sessions (no silent fallback), adds a hard-abort in `chat-stream.ts` that emits an SSE error frame and ends the connection on those decisions, and adds a new integration test suite that exercises the production env-var path (no override), including the regression mode where `BODEGA_USER_DATA_DIR` mismatches between provision and request time.

### Fixed: FleetCard rendered `gpt-4o` for Kimi sessions (and other model-substitution providers)

PR #428 Bug 1. The `OpenAIStreamParser` was hardcoding the OUTGOING `model` string into every yielded chunk instead of reading `data.model` from the SSE response. Kimi (and other OpenAI-compat backends that route by model family) silently substitute the served model: sending `gpt-4o` returns `kimi-k2.5` in `data.model`. Without the fix, `provider_context.modelName` persisted the REQUESTED string and FleetCard rendered the wrong model. Now `chunk.model = data.model || requested_model`. Verified live: sessions show `kimi-k2.5`, `qwen-coder-plus`, `deepseek-v4-flash`, `claude-sonnet-4-6`.

### Fixed: Diff stats showed `+0 / -0 / 0 files` even when sessions had real changes

PR #428 Bug 2. `WorktreeManager.checkDiff` was running `git diff <baseRef> HEAD`: committed-only diff. PR #423's fire-and-forget auto-commit means `last_status = 'complete'` lands BEFORE the commit promise resolves, so the frontend fetches the diff during a race window where `HEAD == baseRef` and stats come back 0/0/0. Fix: compare against the working tree (`git diff <baseRef>` with one arg). Surfaces staged + unstaged AI writes regardless of auto-commit timing. Verified live: session-1 now shows `+19 / -0 / 1 files | ['lib/sizes.ts']` immediately after the agent completes.

### Fixed: Session model picker drifted to the global active model

PR #428 Bug 3. `useAgentPanelModel` was reading only from global settings keys; clicking back to a past Kimi session would show whatever was globally last-active (e.g. `qwen3:8b`). DB write path was correct; the frontend read path just never consulted `provider_context.modelName`. Now the hook reads `fleet[activeCodeSessionId].providerContext.modelName` and prepends it to the effective-model fallback chain. Foreground sessions aren't in `fleet[]` so the selector returns null and the old fallback runs unchanged.

### Fixed: Apply dialog's "Resolve in editor" opened a blank Monaco buffer

PR #428 Bug 4a. `FleetApplyDialog` was calling `openEditorTab` directly with `lastSavedContent: ''`. Monaco initialized from the empty string and never read from disk. Fix: read file content via `window.electronAPI.fsReadFileForEditor` first, populate the tab payload, then open. Mirrors `FileTreeNode.openFile`. Binary/oversized files surface a toast instead of opening blank.

### Changed: Apply dialog routes uncommitted changes to the Git panel (UX polish, PR #429)

The flow from "Apply blocked by uncommitted changes" was confusing. The previous "Resolve in editor" button opened the file as a tab, but the user still had to context-switch to the Git panel to actually commit or discard the changes. Now there's a single "Open Git panel" button at the bottom of the uncommitted-changes panel that navigates straight there (`setActiveLeftPanel('source-control')` + `setGitPanelOpen(true)` + close dialog). The explainer text is also no longer vague: "These are project edits that didn't come from this session. Commit or discard them in the Git panel, then retry Apply." Conflict path is unchanged. The dual-tab side-by-side comparison is still the right tool for that case.

### Fixed: Live-driver smoke close-out (10 bugs from 2026-05-18 marathon)

PR #413 covered 10 distinct fleet bugs surfaced during a 7-hour live driver pass: worktree disconnect handling (Bug #14), fleet sessions canceling on mode switch (Bug #15), tile-open action mapping (Bug #16), orphan worktree cleanup (Bug #17), fleet tool approvals surviving SSE disconnect (Bug #18), `BODEGA_USER_DATA_DIR` override propagation (Bug #19), FleetCard idle-while-iterating display (Bug #20), 2-second-stale activity timestamp (Bug #21), session-2 cancellation (Bug #22), mid-stream re-attach (Bug #23).

PRs #414–#419 piled on with `setActivePreset` atomicity (Bug #22 dredged Ollama-fallback), FleetCard live activity + correct panel counter (Bugs #23, #24), fleet history + auto-title + qwen picker (Bugs #32–#35), preventative orphan-state cleanup before worktree add (Bug #16 follow-on), mid-stream chat re-attach polish + local-provider visibility, empty model-cache trap + chat-panel background-activity banner.

### Fixed: Reviewer/Sentinel/QA gap cleanup (PRs #424, #425, #427)

Three post-merge consolidation PRs closed every minor edge surfaced by the auto-review agents: leading-dash strip on commit messages (Sentinel: prevents flag confusion), 200-char commit message cap (Reviewer: keeps git log readable), fallback to `'Apply fleet session'` when sanitization empties the string, squash conflict-marker detector (`git diff --cached` scan for `+<<<<<<<` / `+=======` / `+>>>>>>>` line prefixes after `git merge --squash`; if found, `git reset --merge` + typed error rather than committing markers as code), and Ollama "Use This Provider" persist bug (button only updated local form state. Clicking it appeared to succeed but the model picker still showed nothing because `preset` / `active_provider` / `providers_enabled` never got persisted; fixed via `apiClient.setPrimaryProvider(presetId)` + mirror writes to settings store).

### Changed: Settings → Models multi-provider cleanup (MP-C, PR #400)

C1–C5 sweep: each cloud BYOK provider gets its own labeled key field instead of one shared "API key" with hidden per-preset routing. Empty fields are visually distinct from "set but redacted" (sentinel display). The legacy `llm.openai_api_key` setting remains for back-compat reads but new key writes go to `llm.<presetId>_api_key`. The four-layer cloud-provider routing bug (PR #393) is closed by an explicit `providers_enabled` blob that the resolver consults before falling through to legacy globals.

### Changed: Auto-commit AI changes by default (PR #423)

`git.auto_commit_ai_changes` defaults to ON. Every agent file_system write completes, and on success Bodega stages + commits with author `Bodega AI <ai@bodega.one>` and a generated message describing the change. Trailers include `Auto-committed by Bodega One. Disable via Settings → Git.` This gives every session a clean audit trail of exactly what the agent wrote. Fire-and-forget (no awaiting the commit before completing the SSE) so chat completion latency is unaffected.

### Fixed: Onboarding stranded state detection (Joe smoke 2026-05-08 carry-forward)

[#418 carry-over and #427 hardening] If `general.has_completed_onboarding = true` but the user has no default model AND no API key for the active preset, FirstRunFlow re-mounts automatically. The check now reads per-provider keys (`llm.<presetId>_api_key`) instead of only `llm.openai_api_key`: the original Phase 2F bug that re-triggered FirstRunFlow on every launch even for properly-configured users with non-OpenAI presets.

### Test counts

Backend **4,775** passing / 54 skipped / 15 todo (+4 new chat-stream-worktree env-path tests). Frontend **991** passing / 1 skipped / 5 todo (FleetApplyDialog suite updated for the IPC-read pattern + Git panel routing assertion). Typecheck clean on both packages.

---

## [v1.0.0-beta.19] - 2026-05-13

Eight PRs covering three Cachev-driven fixes (beta tester Cachev surfaced four real issues on 2026-05-12; PR #345 handled one yesterday, three more land in this release), one structural-debt refactor (the biggest god-file in the repo), bundle-weight reduction, a settings-wiring audit + fix pass, two design specs that unblock the next session, and a review-fleet close-out PR.

### Fixed: Ollama 180s timeout fired before slow hardware could respond

[#347] `OllamaProvider.CONNECT_TIMEOUT` is now tier-aware. Minimal-tier (< 6 GB VRAM, includes CPU-only and integrated GPUs) gets 360s; GPU-accelerated machines stay at 180s. Surfaced by Cachev's report on an Intel UHD 770 (2 GB integrated): agent mode's ~3,500-token prompt prefill takes 3-4 min on CPU, but our 180s timeout fired before any token arrived. Yesterday's PR #345 made the error message accurate ("model is taking too long to load"); this PR makes the timeout match the hardware reality. Threshold tracks the budget-tier boundary in `HardwareTierClassifier`. Null-cache (pre-detect) defaults to 180s, optimistic, so misconfigured fast machines still fail fast.

### Fixed: New terminals opened in home directory instead of active project (Cachev #1)

[#350] Every IDE on the planet opens `terminal.cwd = project.path`; Bodega didn't. The infrastructure already supported it: `ipcPty.ts` accepts `cwd` in `pty-spawn`, `TerminalInstance` passes `tab.cwd` through, and the `TerminalTab` type has the optional field. The four `addTerminalTab` call sites (Ctrl+Shift+T, command palette "New Terminal Tab", auto-create-first-tab, and the + button on the tab bar) just weren't reading `codeProjectPath`. Each now reads it at tab-creation time. When no project is open, `cwd: undefined` and the PTY falls back to `os.homedir()` (unchanged for that case). Existing terminals keep their cwd if the user switches projects, matches shell semantics.

### Fixed: NSIS installer radio button labels rendered dark-on-dark (Cachev #4)

[#352] Install Mode page (the "Anyone who uses this computer" / "Only for me" radio buttons) was unreadable on our dark installer theme. The radio indicator circles honored `SetCtlColors` correctly, but the label TEXT fell through to Windows visual-theme defaults, grey on dark, basically invisible. Cause: on Win10/11, themed BUTTON-class controls partially ignore `SetCtlColors` for text rendering unless the visual theme is explicitly stripped. Fix: call `SetWindowTheme(hwnd, "", "")` on every enumerated control in `colorPageControls` before applying colors. Same uxtheme strip we already use for the InstFiles progress bar: generalizing it to the page-walker covers radio buttons and any other themed BUTTON children at any depth.

### Changed: Bundle size reduced by ~2.15 MB

[#348] Three independent wins from yesterday's Performance Profiler bug-hunt:

- Deleted unused `Bodega_Default_Full.png` (1.4 MB). `themes.ts` imports the `*_transparent` variant; this file had zero references.
- Dropped Inter font fallback (~444 KB across 4 weights). CSS stack was `'Geist', 'Inter', -apple-system, ...` but Geist is bundled locally and always loads, so Inter never painted. Removed the 4 `@font-face` blocks and pulled `'Inter'` from the 3 font-family chains.
- Stopped copying `.ico` into renderer dist (353 KB). `main.ts` resolves the icon from `process.resourcesPath` (packaged) or `../assets` (dev), never from the renderer dist. electron-builder still copies it via package.json for the installer.

Renderer output now ships 6 woff2 files instead of 10. 595 frontend tests still pass.

### Changed: SettingsService 915L god-file split into focused modules

[#349] `SettingsService.ts` was the largest single-responsibility violation in the codebase, untracked on the CLAUDE.md watchlist, drifting upward with every new setting. Split into an orchestrator + 5 sub-modules:

| File | Lines | Role |
|---|---|---|
| `services/SettingsService.ts` | **335** (was 915) | Singleton, cache, get/set/setMany/getAllRedacted |
| `services/settings/SettingDefaults.ts` | 349 | 137 setting definitions + `SettingDefinition` interface |
| `services/settings/SettingsSecrets.ts` | 41 | `isSecretSettingKey` + redaction sentinel |
| `services/settings/SettingsMigrations.ts` | 133 | `SettingsMigrationRunner`: startup-time data fixups |
| `services/settings/AirGapMarker.ts` | 41 | Disk-side echo of `general.air_gap` |
| `services/settings/ProjectConfigLoader.ts` | 65 | `.bodega/config.json` reader + whitelist |

Public API is unchanged. Every external caller (30+ files) keeps importing `SettingsService`, `settingsService`, `isSecretSettingKey`, `REDACTED_SECRET_SENTINEL`, `PROJECT_CONFIG_ALLOWED_KEYS`, and `SettingDefinition` from `'./SettingsService'`. The migrations module takes `(db, cache, setFn)` as constructor args so each migration is independently testable. All 4175 backend tests still pass.

### Added: Settings wiring audit + fixes

[#353] Crawled all 140 settings in `SETTING_DEFAULTS` against backend reads and UI controls. Audit + report at `source-of-truth/specs/settings-wiring-audit-2026-05-13.md`. Two real gaps found and fixed:

- **`editor.font_family`** was missing from `SETTING_DEFAULTS`. Monaco's `editorOptions.ts:17` reads it but only worked because of a hard-coded `||` fallback. Added the entry with default `'"Geist Mono", Consolas, "Courier New", monospace'` so `getAll()` enumerates it and the value is validated like any other setting.
- **`git.auto_commit_ai_changes`** had backend wiring at `AiCommitService.ts:212` (default: ON, core "rogue AI defense" auto-committing every agent file write) but no UI control. Added a toggle in SafetySection under a new "AI Audit Trail" section, with a toast warning when disabled.

Net audit result: 45 fully wired, 92 intentionally backend-only (model tier routing, system state flags, power-user JSON keys), 2 orphans (placeholder/deferred), 1 missing-default (fixed), 1 missing-UI (fixed). 0 actual dead writes.

### Fixed: Settings rollback skipped secret decryption (Reviewer H1)

[#354] `SettingsService._setManyImpl`'s rollback path reloaded the cache from DB rows with bare `JSON.parse`, missing the `SecretCipher.decrypt` pass that `initialize()` does. Any encrypted `*_api_key` row would end up in cache as `enc::` ciphertext after a failed bulk save, and the next provider read would hand cipher-text to the LLM client (silent 401). Pre-existing from beta.18.3; surfaced by the beta.19 review fleet. Fix: extracted the cache-load logic from `initialize()` into a shared private `reloadCacheFromDb({ applyEnvOverrides })` method used by both init (with env overrides) and rollback (without, env overrides are first-boot-only). Also covers the secondary bug where rollback didn't merge defaults for missing keys.

### Fixed: Audit-trail toast type + dependency CVE

[#354] Two small fixes from the close-out review pass:

- Disabling the AI Audit Trail toggle previously fired a green `'success'` toast. Switched to `'warning'`. Disabling a safety feature should visually signal significance.
- `npm audit fix` cleared the `fast-uri` HIGH-severity advisory in both backend + desktop-app lockfiles. The remaining moderate `dompurify`-via-`monaco-editor` advisory needs a breaking Monaco downgrade and is deferred to beta.20.

### Added: Cachev follow-up design specs

[#351] Two specs in `source-of-truth/specs/`:

- **`installer-dark-theme-spec-2026-05-13.md`**: maps existing NSIS coverage per MUI2 page, ranks five hypotheses for which control could still render dark-on-dark, lists three verification paths. Hypothesis D was the one that landed as #352; hypotheses A/B/C/E remain on the shelf if other dark-on-dark surfaces emerge.
- **`multi-code-sessions-spec-2026-05-13.md`**: design doc for Cursor-style multi-session tabs. Current model is single-active-session with destructive switching; target model is `activeCodeSessionId` + `openCodeSessionIds[]` + per-session scratch state map. 14-file touch list, 5-phase delivery plan, ~2-3 days focused work. Spec self-flags the unverified backend claims that need tracing before implementation.

### Cachev follow-up: what's left

Of Cachev's four reports from 2026-05-12: #1 (terminal in project) shipped in this release, #4 (NSIS dark fonts) shipped. #2 (folder preview expandable) still needs his clarification on which surface he hit, likely the `@mention` menu or the project picker, not the FileTree (already expandable). #3 (multi-code-sessions) is specced and ready to implement.

---

## [v1.0.0-beta.18.3] - 2026-05-11

Patch release. Twelve fixes covering provider-switch state hygiene, three classifier regressions Joe caught during live cloud-provider testing, and the long-running IDE↔Chat session leak that's been hitting users since `useChat` was extended to power both the chat panel and the IDE Agent panel. Three PRs merged: #336 (provider switch), #337 (classifier round 2), #338 (QA findings).

### Fixed: IDE Agent panel writes leaked into chat session (THE LEAK)

`useChatSend.handleSend` had `const currentSessionId = sessionId ?? useStore.getState().activeSessionId`. For `ChatStage`'s `useChat` instance that's fine: `sessionId` IS `activeSessionId`, the `??` never fires. For `AgentChatPanel`'s `useChat` instance with `codeSessionId=null` (any time before the user explicitly creates a code session), `sessionId` is null → the fallback returns `activeSessionId`, the chat session's id. Result: sending in code mode wrote the user message to the chat session in the DB, the backend broadcasted `session_message{session_id: chatId}` via WebSocket, the chat panel's filter (`sid === activeSessionId`) passed, and the message appeared in chat. Meanwhile the optimistic `addMessage` on code mode's local state rendered the same message in the code panel. Identical conversation visible in both modes, exactly what Joe screenshotted on every test instance for weeks.

The bug had been there since `useChat` was generalized to handle both modes (pre-beta.18 split). Console-trace instrumentation isolated it: `setLocalMessages` fired for code mode with chat-content from `executeSend.ts`'s reconcile path, while `setCodeSessionId` never fired at all. Fix: gate the fallback to `sessionType === 'chat'`. For `sessionType === 'code'`, null `sessionId` means "no session yet" and `autoCreateSession` kicks in below to spin up a fresh code session.

Defense in depth in `services/websocket.ts`: the `session_message` handler now triple-checks before adding to chat's global slice: `sid === activeSessionId` AND `sid` is in `state.sessions` (server-filtered chat-only list) AND `sid` is NOT in `state.codeSessions`. Even if any future code path sets `activeSessionId` to a code-session id (e.g. `AgentPanel.handleOpenSession` doesn't filter by type), the type check now blocks the leak structurally.

### Fixed: Legacy `llm.openai_*` keys bled across presets

Five backend sites read `llm.openai_base_url` + `llm.openai_api_key` unconditionally for every non-Ollama preset. When a user previously used OpenAI then switched to Featherless/Anthropic/DeepSeek/etc., the stale `llm.openai_base_url` still pointed at `https://api.openai.com/v1` and these endpoints either hammered OpenAI with no key (401 every 3s, CPU spike) or, worse, paired the saved OpenAI key with another provider's URL.

Migrated to the preset-aware `lookupBaseUrlForPreset` / `lookupApiKeyForPreset` helpers (same path `LLMService.reconfigureFromSettings` uses):
- `routes/llm.ts`: `/llm/running-models` 30-second poll
- `routes/chat-stream-helpers.ts`: deferred reconfigure path
- `routes/llm-test-connection.ts`: Test-Connection sentinel-substitution fallback
- `services/EmbeddingService.ts`: OpenAI-compat embedding path
- `services/STTService.ts`: OpenAI Whisper-API transcription path

### Fixed: Stale role models survived preset switches

`ProviderCard.handleSetActive` cleared `llm.default_model`, `llm.chat_model`, `llm.code_model`, `fim.model` on switch. Extended to all 11 role keys (`fast_model`, `smart_model`, `research_model`, `debug_model`, `advisor_model`, `fim_model`, `image_generation_model`) so the iteration router + side panels also drop stale model names. Also pre-clears `availableModels` to `[]` and fires `/llm/health` immediately after the preset flip so the role combobox repopulates within ~200ms instead of waiting for the next 30-second health tick.

### Fixed: `useModelsPanel.populateFrom` zeroed all role-model locals on hydrate

BUG-DM-15 (perf optimization) reduced the `settings` object passed downstream to a 3-key subset via per-key Zustand selectors. `useModelsPanel`'s `populateFrom(settings)` then resolved every `llm.*_model` key to `''` (absent in the subset), and the next `handleSaveLLM` wrote those empty strings back, wiping the user's saved default. Now reads the full settings via `useStore.getState()` inside the effect: no extra re-renders because it's a one-shot imperative read, and the dep array still gates on `llm.preset` so re-hydration only fires on actual preset switches.

### Fixed: Featherless WarmingBanner leaked to every cloud preset

`useModelWarmup` checked `CLOUD_PRESETS.has(preset)` against 15 providers. Switching Qwen → DeepSeek flashed "Warming up qwen..." then "Warming up deepseek..." in the chat header: pure noise because every other cloud OpenAI-compat provider responds in ~200ms with no serverless cold-start. Gated to `preset === 'featherless'` only; the dedicated SSE coordinator + state machine still owns Featherless's actual cold-start UX.

### Fixed: `STOP READING FILES` nudge fired on legitimate folder questions

`RuntimeLayer.isExplorationIntent` missed `"What are the contents of source-of-truth folder?"` because:
- `VERBS` list had `'what is'` / `"what's"` / `'what does'` but **not `'what are'`**
- `TARGETS` list had `'files'` but **not `'folder'`** / **`'contents'`**

After 3 file reads, `NudgeOrchestrator` emitted "STOP READING FILES. This is a knowledge question..." into DeepSeek-v4-pro's stream. DeepSeek looped on the contradiction for 440 seconds before producing anything. Enriched both lists; added 5 regression tests in `RuntimeLayer.test.ts` + 2 in `NudgeOrchestrator.test.ts` to lock the Joe-hit prompts.

### Fixed: DeepSeek-v4-flash chat-mode raw `<function_calls>` XML in user response

In the chat-mode advisory lane (`toolCount: 0`, `shouldIncludeTools: false`), DeepSeek-v4-flash emitted literal `function_calls / /function_calls` text. The model has been trained on Anthropic's plural-form XML and tries to use it even when no tools are provided. `ResponseGrounder.stripToolMarkup` only knew about the singular `<function_call>`. Extended to strip the plural angle-bracketed form, the bare-word `function_calls\n.../function_calls` form, and the empty-block variant. Widened the BUG-09 XML_ONLY guard in `AdvisoryLaneExecutor` to recognize the new shapes for empty-bubble fallback.

### Fixed: Featherless DeepSeek-V3 emits Python-style tool calls as text

Featherless-served `deepseek-ai/DeepSeek-V3-0324` in code mode emits tool calls as Python code fences instead of OpenAI native tool_calls or XML:

```
Let me search the knowledge base:

```python
query_knowledge(query="Claude project rules", limit=5)
```
```

The model thinks it's calling a tool, but the parser doesn't recognize the format. The agentic loop ends, no tool was actually invoked, and the user sees raw Python text. The default code-fence protection in `stripToolMarkup` was preserving these as "legit user code". Pre-pass now strips ```python|py|js|javascript|ts|typescript``` fences when the body starts with a registered tool name + paren. Whitelist anchor keeps false positives near zero. Doesn't make the tool actually execute (that needs a deeper model-profile rework tracked separately) but at least the user no longer sees broken Python output.

### Fixed: Qwen `/think` directive leaked as first line of every response

qwen-coder-plus via DashScope's OpenAI-compat endpoint echoes its `/think` thinking-mode directive at the start of every response, leaving `/think\n\n` as the literal first line of user-visible markdown. Confirmed in DB during QA pass. Every qwen-coder-plus message had `/think` as the first 6 characters of content. Extended `ThinkTokenStripper` with a new `stream_start` state that silently eats leading `/think` or `/no_think` followed by any whitespace before transitioning to normal content mode. Mid-stream `/think` is still treated as content (so prose like "the /think directive" survives). 9 new test cases covering chunked delivery, reset, flush-on-bare-directive, and the non-directive slash path (e.g. `/some/path/to/file`).

### Fixed: Chat input prefilled text concatenated with user typing

The composer's "What can you help me with?" appears to be a placeholder but is actually pre-filled text via `setPrefillMessage`. Clicking mid-text and typing inserted at the cursor position, producing concatenated garbage like `"What can you help!Hi! What model are you?... me with?"` in the actual sent message (confirmed in DB during QA pass). Fix: after the prefill effect sets the input value, also focus + select-all so the user's first keystroke replaces the whole prefill, the natural behavior when prefill is meant as a starter that can be edited.

### Fixed: Retry button leaked React `SyntheticEvent` to the model param

`ErrorBanner.tsx` and `ChatErrorBanner.tsx` had `<button onClick={onRetry}>` where `onRetry = handleRetry` had an optional `resolvedModel?: string` parameter. React passes a `SyntheticBaseEvent` as arg 0 to `onClick` handlers. That flowed into `handleRetry`'s `resolvedModel` slot, through `executeSend → streamChatToLLM`, and surfaced as `model: undefined` after the `coerceString` drop, producing the user-facing "Request failed. Something went wrong" on Retry click. Wrapped as `onClick={() => onRetry()}` and added a defensive `typeof === 'string'` guard inside `handleRetry` itself.

### Fixed: API key field looked unfilled when configured

`ProviderCard`'s API-key input rendered empty when a key was stored (backend sends `__BODEGA_REDACTED__` and `getSecretFieldState` strips it to `''` for display). The default `'sk-...'` placeholder then made the field look unfilled, which Joe read as "key didn't persist" during cross-provider testing. Now shows `"••••••••• (saved. Paste to replace)"` placeholder when `apiKeyConfigured`, plus a green `"✓ Key saved for this provider"` hint below the input.

### Fixed: 23 pre-existing SettingsService tests failing on `SecretCipher.initialize()`

Settings tests were failing with `SecretCipher.initialize() must be called before get()` because `SettingsService.initialize(db)` calls `SecretCipher.get()` to decrypt stored secret values, but the test setup never seeded the cipher singleton. Pre-existing failure (not caused by today's work) surfaced when running the full backend suite to verify no regressions. Each `beforeEach` now calls `SecretCipher.resetForTests()` then `SecretCipher.initialize(tmpDir)`, with matching cleanup in `afterEach`. With this fix the backend suite reports 4610 passed / 54 skipped, 0 failures.

### Test coverage delta

- Backend Vitest: 4059 → **4610** tests (+551: pre-existing SettingsService tests now runnable + ~30 new regression tests for today's fixes).
- Frontend Vitest: 593 → **595** tests (+2).
- Both `tsc --noEmit` clean.
- Webpack dev build compiles in 8 seconds.

### Deferred to next round (tomorrow's work)

- **Qwen-cloud DashScope tool calls**: `qwen-coder-plus` via DashScope never invokes file_system / grep / glob tools, claims folders "don't exist" instead of reading them. Backend log shows `format:"xml"` `nativeTools:true` and `tools` is sent in the request, but DashScope's OpenAI-compat endpoint may be silently dropping the array. Needs live request-payload inspection.
- **Featherless DeepSeek-V3 tool execution**: the current stripper prevents broken Python text from leaking to the user, but the tool call still doesn't actually run. Either parse Python-style calls + execute them, or force Featherless models to use XML/native format via prompt template tweak.
- **File-read pagination**: `file_system.read` truncates tool result at 16KB (via `ToolCallProcessor.truncateResult`); large files (e.g. 85KB CHANGELOG.md) can't be fully read. Add `offset` + `length` params to the tool schema.

---

## [v1.0.0-beta.18.2] - 2026-05-11

Hotfix for a critical install-time failure on the packaged Windows portable build: a fresh install ran into "Setup hit a snag, API error: Bad Request" on both the llama.cpp and Ollama setup paths. Surfaced via Discord feedback within 48 hours of beta.18.1 ship.

The bug never showed in local fresh-install testing because the test harness ran in `npm run dev` mode, where the path resolution worked. The bug only manifested in packaged Electron builds, exactly the artifacts users download.

### Fixed: Install fails on packaged Windows portable with "API error: Bad Request"

- **Root cause.** Three install/catalog services (`LlamaBinaryInstaller`, `OllamaInstaller`, `GgufCatalogService`) computed their config-file paths as `path.resolve(__dirname, '../../../../configs/X.json')`. In `npm run dev` mode `__dirname` is under the source tree and this lands on `apps/desktop/configs/`. In a packaged Electron build `__dirname` is under `<resources>/backend/dist/backend/src/services/...`, and four parents up lands on `<resources>/backend/dist/configs/`, which doesn't exist. `electron-builder`'s `extraResources` block copies the JSON files to `<resources>/configs/` instead. The three services threw on `fs.readFile`, which propagated as a generic HTTP 400 "Bad Request" to the renderer.
- **Migration to the existing utility.** All three services now use `resolveConfigPath()` from `apps/desktop/backend/src/utils/configPath.ts`, a utility built specifically to prevent this regression after a prior incident. Its first candidate is `process.resourcesPath/configs/<filename>`, which is the canonical packaged-Electron layout. Resolution is lazy (called on first config read, not at module load) so `process.resourcesPath` is populated by Electron before the lookup runs. Other backend services (`ModelCatalogService`, `ModelProfileService`, the agent-config / capabilities loaders) were already using this utility correctly. Only the post-beta.16 install paths missed adoption.
- **Utility candidate expansion.** `configPath.ts` previously handled config loaders one level deep under `services/` (3-up and 5-up `__dirname` candidates). The new install services live two levels deep (`services/llama/`, `services/ollama/`), so 4-up and 6-up candidates have been added. Production was already handled by the `process.resourcesPath` first candidate; the extension closes the gap for dev mode and compiled-test mode.
- **Backend error surfacing in the API client.** `baseClient.ts` previously discarded the `body.error` field returned by backend routes on 4xx responses and threw a generic `API error: <statusText>` (e.g. "API error: Bad Request"). Backend routes return `{ error: <human reason> }` consistently on 4xx. The client now uses that reason in the thrown Error message, falling back to `statusText` only when the body has no `error` field or isn't JSON. Future install failures (and any other 4xx) now show the actual reason, not a generic status string.

### Fixed: Ollama manifest validator over-rejected cross-platform paths

End-to-end verification of the previous fix on a Windows machine uncovered a latent bug the config-path fix unmasked: `OllamaInstaller.validateManifestPaths` iterated **every** platform entry in `ollama-installer-versions.json` and validated each against the **current OS's** allowed-prefix list. On Windows that meant the `macos-universal` entry's `/Applications/Ollama.app` failed validation because `/Applications` is only on the darwin allowlist. The whole config load threw, re-breaking Ollama install on Windows portable just with a different error message than the original bug.

Pre-beta.18.2, the JSON file was never located in packaged builds at all, so the validator never ran in production. Once the loader was fixed, the validator was the next thing to fail.

- **Validation scope narrowed** to only the entry for the current OS (`config.platforms[detectPlatform()]`). Cross-OS entries are inert on this machine (their paths are never used in any code path) so over-validating them was always wrong.
- **Security invariant preserved.** The original Sentinel HIGH-2 concern was defense-in-depth against a future user-importable or remote-loaded manifest containing a malicious path. Any path that could actually be used on this OS is still validated against this OS's allowlist; the change only stops checking paths that are inert on the current platform.
- **Two regression tests added** in `OllamaInstaller.sandbox.test.ts`: one confirming a multi-platform manifest now loads successfully on the current OS, one confirming a malicious path on the current-OS entry is still rejected.

### Improved: Cloud-model warmup debounce on activate-model changes

- **`useModelWarmup` 500ms debounce.** During cloud onboarding the active model can churn 3+ times in a few hundred milliseconds as settings persist and the store rehydrates. Pre-fix this fired the cloud warmup endpoint three times per onboarding completion. The hook now debounces by 500ms so the warmup fires once after the model has stabilized. Same dedup guard inside the timeout (60s same-model window) means rapid back-and-forth between two models still doesn't waste warmup quota.

### Added: Profile entries for current top-tier local coding models

- **`qwen3-coder-next`** (80B MoE, 3B active, 70.6% SWE-Bench Verified, top-ranked local coding model) and **`qwen3:30b-a3b`** (30B MoE, 3B active, best price/speed/quality triangle on local) added to `WELL_KNOWN_MODELS` in `ModelProfileData.ts`. Both were already in the onboarding catalog (`configs/model-catalog.json`) but missing from the profile lookup, so agent routing fell through to sizeClass defaults instead of resolving the correct MoE/coding-tier profile. Capabilities (tools, thinking, FIM where applicable), MoE flag, parameter counts, and family set per the model cards.

### Test coverage delta

- Backend Vitest: 4057 → **4059** tests (+2 from the OllamaInstaller validator regression suite).
- Frontend Vitest: 593 tests (no change: the warmup debounce is exercised manually; the existing 60s dedup tests still pass).
- New regression coverage for two-level-deep service config paths in `configPath.test.ts`.
- New regression coverage for the body.error surfacing path in `baseClient.test.ts`.

### Verified end-to-end

A standalone Node script simulated the packaged-Electron environment (mocked `process.resourcesPath`, fresh `BODEGA_USER_DATA_DIR`, configs copied to a temp `<resources>/configs/`) and exercised the compiled installers directly. All four loaders now succeed where they would previously have thrown:

- `LlamaBinaryInstaller.getPinnedVersion` → `b9045`
- `LlamaBinaryInstaller.resolveFlavor("auto")` → `win-x64-cuda13` on an RTX 5090
- `OllamaInstaller.getPinnedVersion` → `v0.23.1`
- `GgufCatalogService.getWithFit` → 17 GGUF entries loaded with hardware-tier scoring applied

---

## [v1.0.0-beta.18.1] - 2026-05-09

Hotfix for two regressions surfaced in Discord the morning after beta.18 shipped, plus the cleanup batch originally slated for beta.19, folded into one release so users get every fix in one update. Two more bugs surfaced during live verification of the hotfix and were fixed before tagging.

The headline shape:

1. **Self-hosted providers lost their custom Base URL** on menu reload. Typed-and-saved values reverted to localhost the moment the user navigated away.
2. **No native preset for Featherless AI**, even though their setup page is explicitly branded for Bodega users.
3. **Adding a HuggingFace-aggregator provider could freeze the app.** Featherless's `/v1/models` returns 6,700+ entries, enough to drain the Windows socket pool, flood logs with metadata-resolver lines, and have the model picker auto-select an alphabetically-first random fine-tune. Caught and fixed live.
4. **Diagnostics-export broke when there was no active session**, exactly the moment users needed it to file a bug report. Fixed.

### Fixed: Custom Base URL persistence

- **`lookupBaseUrlForPreset(presetId)`** added in `LLMService.ts`: generalizes the qwen/kimi region-override pattern so ANY OpenAI-compat preset can override its base URL via `llm.<presetId>_base_url`. Special cases preserved: `ollama` keeps using `llm.ollama_url` (legacy name), `azure` continues to be assembled from `llm.azure_resource`. The plain `openai` and `custom` presets keep their `llm.openai_base_url` legacy fallback so pre-Phase-2 setups don't break.
- **Backend URL resolution refactored** in `LLMService.reconfigureFromSettings()` to call the helper for all openai-compat presets. Net effect: a typed Base URL on the llama.cpp card now actually flows through to the backend's `OpenAIProvider` baseUrl, so reconfigure connects to the user's server instead of localhost.
- **Frontend mirror `baseUrlSettingForPreset(presetId)`** added in `modelUtils.ts`. ProviderCard reads its initial URL value from `settings[baseUrlSettingForPreset(preset.id)]` first (falling back to detected → preset.defaultBaseUrl), syncs from the store when settings change without clobbering in-progress typing, and persists the URL on both `Test Connection` success and `Set as Active`. The empty-string vs preset-default distinction is honored. Clearing the URL field back to the default removes the override rather than storing it as an explicit string.

### Added: Featherless AI preset

- **New `featherless` preset** in `ProviderPresets.ts`: fully OpenAI-compatible, Bearer auth, `https://api.featherless.ai/v1` base URL, model names in HuggingFace `org/model` format. Standard endpoints: `/v1/models`, `/v1/chat/completions`, `/v1/completions`, `/v1/tokenize`. The `setupTip` points to `featherless.ai/account` and notes the `rc_` key prefix; the `modelTip` shows three real model name examples (`deepseek-ai/DeepSeek-V4-Pro`, `Qwen/Qwen3.6-35B-A3B-Instruct`, `meta-llama/Llama-3.1-70B-Instruct`); the `limitations` field warns about subscription-tier-gated models (`is_gated=true` on `/v1/models` returns 403 on lower plans) and that slashes are part of the model name (not a path).
- **`llm.featherless_api_key`** added to `lookupApiKeyForPreset` so the key has its own dedicated storage instead of falling back to the legacy shared `llm.openai_api_key`.
- **Wired into `cloud-key-validate`**: uses the existing `validateOpenAiCompat` helper since Featherless follows standard Bearer + `/models` semantics. Honors a custom `baseUrl` when provided (e.g. for users routing through an internal corporate proxy).
- **Featherless added to the V2 cloud onboarding picker** (`useFirstRunMachine.CLOUD_ONBOARDING_PRESET_IDS`), the Settings → Cloud API Keys section with `rc_…` placeholder, the Settings search index, and the **Cloud Boost provider picker** so users can wire Featherless as their boost provider too.

### Fixed: HuggingFace-aggregator model storm

- **`OpenAIProvider.capListedModels()`** caps any provider's `/v1/models` response at **500 entries** at the boundary. When the upstream response exceeds the cap, models from a curated allowlist of foundation-model orgs (`meta-llama`, `deepseek-ai`, `Qwen`, `mistralai`, `google`, `microsoft`, `NousResearch`, `HuggingFaceH4`, `CohereForAI`, `allenai`, `tiiuae`, `01-ai`, `WizardLMTeam`, `moonshotai`, `NexaAIDev`) are always retained; the rest fill remaining slots alphabetically. Models outside the cap stay reachable via `/api/models/:name/info` for power users.
- **`/llm/health` defense-in-depth cap** at 500 with a warning log if upstream slips past `capListedModels`. Stops a single 30-second poll from shipping a multi-megabyte profile/recommended-settings JSON to the renderer.
- **`pickDefaultModelForPreset` curated-model preference.** The picker now accepts a curated-model list and prefers any entry that's also in `availableModels` over the alphabetical fallback. `CURATED_CLOUD_MODELS.featherless` seeds 10 foundation models (Llama 3.3-70B, DeepSeek-V3/R1, Qwen 2.5-72B/Coder, Mistral Large, Mixtral 8x22B, Gemma 2-27B) so first-run users land on a real model instead of `000ADI/Qwen2.5-…-Gensyn-Swarm-…` (the alphabetically-first fine-tune Featherless happened to host).

### Fixed: Diagnostics-export IPC accepts undefined session id

- **`ShapeType` learns optional fields** via the `?` suffix (`'number?'`, `'string?'`, etc.) in `ipcValidation.ts`. The optional variants accept `undefined` and `null` in addition to the base type; required variants behave unchanged.
- **`ipcDiagnostics.ts`** declares every renderer-context field as optional. Diagnostics export now succeeds in every app state: no active session, no logged UI errors, fresh install, all fine. The bundle is the user's escape hatch and must work when nothing else does.

### Added: Model-search filter applies to role assignment

- **MyModelsTab role dropdowns** (Default, Chat, Fast, Smart, Agent, Research, Debug, Advisor) now respect the existing top-level "Search models..." input. Type `llama` and every dropdown narrows to llama-* matches. The currently-selected value is always preserved in its dropdown so a search filter can never strand a user mid-edit.

### Beta.19 cleanup batch (folded into 18.1)

The seven items originally slated for beta.19 ride into 18.1 so users get the whole quality bar in one update:

1. **`StreamingStatusBar` reassurance thresholds** extracted to module-level `REASSURANCE_THRESHOLDS` and `REASSURANCE_COPY` constants.
2. **ARCHITECTURE.md §9** documents three previously-undocumented SSE events: `stream_reset`, `queue_processed`, `reconfigure_failed`. Header count corrected to 18.
3. **`'kimi'` is now a first-class `ModelFamily` union member.** Seven kimi/moonshot profiles updated (`family: 'generic'` → `family: 'kimi'`), family-defaults Record entry added, family detector now recognizes kimi/moonshot before falling through to qwen.
4. **`AgenticLoopSetup.isSimpleTask`** now consults `RuntimeLayer.isExplorationIntent(lastUserMessage)` so the shadow check mirrors `RuntimeLayer.classify` exactly. Closes a latent dual-source-of-truth bug where exploration intents could slip into the simple-task fast path.
5. **`emitByokCost` reads preset at request start**, not at SSE flush time. Captured once in `chat-stream.ts` and threaded into `chat-stream-postprocess.ts` via an optional parameter. Preset switches mid-stream now bill against the preset that was active when the request was made.
6. **Defense-in-depth comment** on the `\bexplain\b` regex remnant in `NudgeOrchestrator.detectOverEagerToolUse` explaining why it's NOT redundant after the `isExplorationIntent` guard.
7. **`CloudCostTracker` async cost queries scoped by `user_id`** (mirrors the `memory_store` pattern). Idempotent migration adds `user_id TEXT DEFAULT 'default'` to `usage_logs`; `recordUsage` writes it; `getDailyCostAsync` / `getMonthlyCostAsync` filter by it. Single-user installs see no behavior change.

### Fixed: Renderer freeze + 60fps re-render storm after Featherless reconfigure

Live testing surfaced a separate cascade: even with the model-storm cap shipped, adding Featherless triggered a 200+ second renderer freeze after onboarding, then a burst of /llm/health calls at render rate (~60fps). Backend pid would burn 24% of one core sustained, renderer the same. Root cause: the `useLLMHealthCheck` lazy-fetch added in an earlier round called `setModelProfiles({...spread})` on every health response, creating a new object reference each time. Every Zustand subscriber re-rendered. At least one of those subscribers re-fired /llm/health, closing the loop.

- **Removed the lazy-fetch entirely.** Components that need the active-model profile can call `apiClient.getModelInfo` from their own `useEffect` with a `loaded` ref guard. For now, model cards render without family/sizeClass badges when no profile is cached, small UX cost in exchange for an unfrozen app.
- **`Object.keys(...).length > 0` guards** added to every `setModelProfiles` / `setRecommendedSettings` call site (`useLLMHealthCheck`, `useSettingsState`, `useModelSettings`, `DiscoverTab`, `DiscoverModelCard`, `useFirstRunMachine` ×2). Backend ships `{}` when `profilesIncluded=false` and even an empty object is a new reference that triggers re-renders.
- **`useLLMHealthCheck` shallow-compare for `availableModels`** before calling `setAvailableModels`. Was unconditionally shipping a fresh array reference every 30s poll even when the model list was identical. Every component subscribed to `availableModels` re-rendered for nothing.
- **`useLLMHealthCheck` skip-`setSettings`-if-pick-unchanged** in the auto-model-select branch. Was creating a new settings reference every poll where no model was configured, even when `pick` was the same string as the existing default.
- **Idle CPU dropped from sustained ~49% of one core to 1.3% total** across all 11 Bodega processes over a 30s window. ~38x reduction.

### Fixed: idle CPU spikes (renderer perf audit)

- **`GreetingStarburst` rAF pause-on-hidden.** The component's `IntersectionObserver` had been setting `visibleRef` but the rAF tick rescheduled itself unconditionally. Only the DOM mutations were gated. Result: continuous 60fps wake-up of the renderer process even when the welcome screen was scrolled off, covered, or the window minimized. Now the tick early-returns and settles to 0 when off-screen; the observer re-kicks on visible.
- **`useLLMHealthCheck` + `NetworkMonitor` `visibilitychange` listeners.** Pause polling when window is minimized/backgrounded. Resume on focus return with an immediate first check so returning users see fresh state without waiting up to 30s for the next tick.

### Fixed: `useLLMHealthCheck` interval leak race (round 8, code review catch)

The post-perf-audit visibility handler had a subtle race: if the window flipped hidden→visible during the initial 2-second startup setTimeout, the visibility branch created `interval`, then the setTimeout fired and overwrote it with a SECOND `setInterval`, leaving the first orphaned and running forever. That doubled the polling load and could re-introduce the very CPU spike the round was meant to prevent.

Fix: extracted idempotent `startPolling()` / `stopPolling()` helpers that guard against double-arm via early return on `if (interval) return`. Mirrors `NetworkMonitor`'s pattern. Caught by Reviewer agent during pre-tag code review.

### Fixed: Providers tab "Selected · offline" badge stuck

After cloud onboarding completed, `ProviderCard`'s yellow `Selected · offline` badge persisted even though the key was saved and reconfigure had succeeded. Root cause: `apiKeyConfigured` was `useState(storedField.isConfigured)`, frozen at mount-time. The cloud-onboarding callback saved the key via `apiClient.updateSetting` AND seeded the local store, but by the time the user navigated to Settings → Models → Providers, the ProviderCard re-mounted and sampled `storedField` once during render, then never updated.

Fix: derive `apiKeyConfigured` from `storedField` on every render. No `useState`, no sync effect needed. The Zustand subscription already re-renders when settings change; the derived value updates with it.

### Fixed: Providers tab `(0/0)` stuck after onboarding

The single-shot `useEffect([])` in `useModelsPanel` for `getProviderPresets()` would silently fail when issued during the mount-time IPC fanout burst (Settings panel mounts → 5+ components each fire their own getLLMHealth/getProviderPresets in parallel). Failure was permanent. Providers tab showed (0/0) for both Local and Cloud forever even once `/llm/health` flipped to "Connected: N models".

Fix: retry-with-backoff on initial fetch (4 attempts at 0/1/3/8s) + tab-focus refetch when the user opens Providers and the list is still empty. Recovers from any transient failure with one click.

### Added: Together AI in cloud onboarding picker

Preset, key storage, and `cloud-key-validate` were already in place. The picker entry was the last missing piece. Standard Bearer auth + `/v1/models` probe shared by all OpenAI-compat cloud providers, zero-risk one-line addition.

### Added: Featherless gated-model filter

`OpenAIProvider.listModels` now respects per-model availability metadata when the provider ships it. Featherless's `/v1/models` response includes `is_gated` (subscription-tier-locked) and `available_on_current_plan`. Live test on a free trial: 16,274 total models, 57 gated, all 16,274 available. The filter drops entries with `available_on_current_plan === false` outright, and entries with `is_gated === true` when no availability field is present (defensive: avoid 403-bound models). Generic: any provider that ships these fields gets the same treatment.

### Added: Code Mode role pickers gate embedding-only models

`MyModelsTab` role-assignment combobox datalist now excludes embedding-only models. Backend `pickAutoDefaultModel` already filters embedders; this closes the matching frontend UI gap so users can't manually pick `nomic-embed-text` for Chat / Fast / Smart / Agent / Research / Debug / Advisor roles either. Embeddings have their own dedicated picker in EmbeddingsSettings.

### Fixed: First-send silent fail (the ship-blocker)

After 20 rounds of patches throughout the day, root-cause traced to an architecture gap: the `ChatErrorBanner` only rendered in the active-chat layout, never in the empty-state `ChatGreeting`. When the first send timed out (backend cold-start blocking the event loop while parsing the Featherless model list), the error fired but had nowhere to render. Users saw "composer clears, nothing happens", the silent fail.

- **`ChatErrorBanner` now renders in BOTH empty-state and active-chat layouts.** Errors during the empty-to-active transition are now visible.
- **Code-mode `ErrorBanner` now uses `formatErrorMessage`**: friendly text ("Request timed out. The model may still be loading. Try again in a moment.") instead of raw "signal timed out". Was diverged from chat-mode's `ChatErrorBanner` which already formatted.
- **Express `keepAliveTimeout = 65s`, `headersTimeout = 70s`, `requestTimeout = 0`**: Node 19+ defaults `keepAliveTimeout` to 5s; combined with Chromium's keep-alive socket pool in Electron this caused follow-up POSTs after the 5s mark to silently hang on stale sockets. Defense-in-depth.
- **`handleEmptySend` clears stale errors on entry** to mirror round-20's autoCreateSession-path fix.
- **`executeSend` retry toast**: first retry now surfaces "Connection slow, reconnecting..." so the previously-silent 15s window has visible feedback.
- **Iteration-cap warning gated on tool usage**: pure-text conversational answers in code mode no longer get the misleading "may be incomplete" footer when the model genuinely finished.

### Added: Pre-warm Featherless on cloud onboarding (Layer 1)

Featherless's serverless inference cold-starts at 30-60s. Without help, the first user message lands on a cold GPU and times out. Layer 1 pre-warms the chosen model during the welcome screen seconds:

- **`POST /api/llm/warmup` endpoint**: fires a 1-token completion via the active provider. Fire-and-forget: returns 202 immediately and runs the warmup in the background.
- **`useFirstRunMachine.handlePickCloudProvider`** triggers warmup right after the cloud preset reconfigure committed.
- **`useModelWarmup` hook** in `App.tsx` watches `activeRoutedModel` and re-fires warmup when the user picks a different model from the dropdown or pins a new role-model in Settings. 60s same-model dedup, cloud-preset gated, air-gap respected.
- **Live result:** first send after fresh Featherless onboard now works on the first try (no Retry needed).

### Added: Backend `/llm/health` cache + renderer health-poll pause (Layer 2)

- **5s TTL cache + in-flight dedup on `/llm/health`** in `routes/llm.ts`. Coalesces the burst of health calls during onboarding (Providers tab + FIM + Embeddings + main poll all fire on mount) and the 30s steady-state poll. Cache key includes preset so reconfigure invalidates implicitly.
- **`useLLMHealthCheck` skips the poll while `isAgentStreaming || isChatStreaming`** is true. Eliminates the "Cannot reach Featherless" yellow banner flickering mid-chat that previously fired every 30s when the backend's event loop was busy awaiting Featherless's response. New `isChatStreaming` slot in uiSlice mirrors the existing `isAgentStreaming` for code mode.

### Fixed: BUG-DM-16/17/18 (deferred from earlier audit)

- **BUG-DM-16: prefix-match boundary in `pickDefaultModelForPreset`**: bare `m.startsWith(curated)` false-positived on differently-purposed forks. Curated `Qwen/Qwen2.5-7B` would match the multimodal `…-Vision-Instruct` fork before the quantization `…-Instruct-FP8`. Now requires a structural separator (`-`, `:`, `.`, `/`) immediately after the prefix, rejects suffixes containing purpose-shift tokens (vision/coder/math/embed/rerank/guard/reward/moderation), and prefers the shortest valid remainder.
- **BUG-DM-17: `/api/models/:name/info` input length cap** at 200 chars. Caps the user-controlled path param before it reaches the regex scan + profile lookup.
- **BUG-DM-18: SSRF guard on `/cloud-key/validate` baseUrl`**: rejects private/loopback addresses (localhost, 127.x, 10.x, 192.168.x, 172.16-31.x, 169.254.x cloud metadata, IPv6 loopback `::1`/`::`, IPv6-mapped IPv4 `::ffff:7f00:1`, IPv6 ULA `fc00::/7`, IPv6 link-local `fe80::/10`, trailing-dot FQDN form `localhost.`, non-http protocols). Self-hosted local providers still go through `/llm/test-connection` which is intentionally permissive about local hosts.

### Polished: Models tab UI (Joe E2E sweep)

- **Redundant search box removed from My Models + Providers tabs.** The 8 inline role pickers in My Models are themselves search inputs (with shared autocomplete); the global filter was duplicate effort. Search input now only renders on Discover where catalog browsing actually benefits from a global filter. Stale "Type to search · X of Y models" hint replaced with a clean "X models available" count.
- **Dark-theme white-box bug fixed.** Chromium applies a yellow/white `:-webkit-autofill` background when an `<input list="...">` gets a value via the native datalist autocomplete. Live-repro'd in the Default role picker: pick a model → input box turned white-on-dark. CSS override in `index.css` uses the inset box-shadow trick (autofill ignores `background-color` directly but respects inset shadow) plus `text-fill-color` and `caret-color` locks to keep the typed value in the theme's text color.

### Test counts

- Backend: 120 test files passing (5 skipped): adds 30 new netGuards unit tests, 9 new cloud-key-validate integration tests
- Frontend: 40 test files passing: adds 4 BUG-DM-16 tests in modelUtils, mocks updated for `setChatStreaming` in 3 useChat test files
- E2E mock: 92 test files passing
- All tsc clean, all ESLint clean, webpack build successful, doc-validation clean
- macOS Smoke: build + launch probe passing on PR #312

---

## [v1.0.0-beta.18] - 2026-05-08

Headline: **V2 Phase 2 lands and the agentic loop gets honest.** Cloud providers get the same painless-install treatment llama.cpp and Ollama got. Per-message cost tracking lands for BYOK so you stop wondering what that 8-iteration repo-tour just charged your card. And two ship-blocker bugs were uncovered live during pre-tag verification: `/think` was being prepended to DeepSeek requests it didn't understand, and the over-eager-tool-use nudge was firing on legitimate exploration prompts: telling models to "STOP READING FILES" right after the user asked them to. Both fixed, both have regression tests, and the panel will now surface model reasoning between iterations so the next class of bug self-reports visually instead of vanishing into a step counter.

### Added: V2 Onboarding Phase 2: Cloud API key flows

- **Per-provider BYOK keys.** Every cloud provider now has its own settings field (`llm.openai_api_key`, `llm.anthropic_api_key`, `llm.gemini_api_key`, `llm.openrouter_api_key`, `llm.azure_api_key`, `llm.mistral_api_key`, `llm.cohere_api_key`, `llm.deepseek_api_key`, `llm.fireworks_api_key`, `llm.groq_api_key`, `llm.together_api_key`, `llm.qwen_api_key`, `llm.kimi_api_key`). Switching presets keeps each provider's key intact: no more re-pasting when you flip from DeepSeek to Mistral.
- **Cloud API key onboarding flow.** New "Cloud API" pitch in FirstRunFlow leads to an 11-provider grid → per-provider key entry → validation → first chat. Mirrors the llama.cpp / Ollama install patterns. Region toggles for Qwen and Kimi (international vs. China endpoints), resource-hostname input for Azure.
- **Cloud key validation route.** `POST /api/cloud-key/validate` does provider-specific lightweight calls (Anthropic uses `x-api-key` + `anthropic-version: 2023-06-01`, Gemini uses `?key=` query param, Fireworks uses a `max_tokens: 1` chat probe since they have no public `/models` endpoint, everyone else uses standard `Authorization: Bearer` + `GET /models`). 10s timeout. Air-gap returns 403.
- **Settings → Cloud API Keys section.** Single section that surfaces all 13 cloud keys at once. Show/hide toggles, save/clear buttons, optional per-provider extras (Azure resource hostname, Qwen/Kimi region toggles).
- **Optional API key field for local providers** (Dave's request). LocalAI behind a reverse proxy, llama-server with `--api-key`, LM Studio with auth: every `providerType: 'openai'` preset now exposes the API key field, labeled "optional" for local providers and "required" for cloud.

### Added: BYOK cost tracking

- **Per-message cost badge.** Every cloud BYOK response now ships with a `$0.0091`-style badge tucked next to the message metadata. Click to see input/output token split. Cloud Boost users get the same badge wired through the existing Boost cost path.
- **Settings → Cloud API Keys → Spend summary.** Session / today / this-month columns plus a per-provider breakdown so you can see whether DeepSeek or OpenAI is your line item this week.
- **Pricing tables** for 13 cloud providers verified 2026-05-08 (DeepSeek v4 pro $0.27/$1.10, flash $0.07/$0.28; Anthropic Opus $15/$75, Sonnet $3/$15, Haiku $0.80/$4; OpenAI gpt-4o $2.50/$10; Kimi K2 $2.40/$9.60; Qwen DashScope max $1.60/$6.40, plus $0.40/$1.20; Mistral, Gemini, Cohere, Groq, Together, OpenRouter, Fireworks, Azure). Prefix fallback for dated variants (e.g. `deepseek-v4-flash-2025-08-01` matches `deepseek-v4-flash`).
- **Storage:** `usage_logs.byok_cost_usd` + `byok_provider` columns. New `cost` SSE event emitted from `chat-stream-postprocess.ts` when active preset is a BYOK cloud provider and `prompt_tokens > 0`. New `CloudCostTracker.ts` singleton service mirrors the existing `BoostCostTracker.ts` shape but writes to a separate column so Cloud Boost spending stays isolated.

### Added: Agentic loop reasoning persistence (Phase 2J complete)

- **`reasoning_content` survives mid-conversation.** Cloud thinking-mode models (DeepSeek R1, Qwen3, Kimi K2 thinking) now have their reasoning blocks persisted to the database AND threaded back through the agentic loop's in-memory message array, closing both halves of the [Phase 2J reasoning_content] gap. Pre-fix, the second tool-call iteration would 400 with `"The reasoning_content in the thinking mode must be passed back to the API."`
- **Storage:** `messages.reasoning_content` column added via migration. Persisted on assistant message INSERT in `MessageService`. Loaded back into context in `RequestNormalizer`.
- **Intra-loop fix:** the streaming path in `LLMCallExecutor.ts` now attaches `reasoning_content` to the message object the loop pushes onto its message stack; the non-streaming path in `OpenAIProvider.ts` does the same. Previously thinking tokens lived in a separate `response.thinking` field that got dropped on `messages.push(response.message)`.

### Added: Smarter iteration cap

- **`RuntimeLayer.isExplorationIntent`**: new static classifier on top of the existing `isComplexCommand` / `isCreationTask` heuristics. Looks for an exploration verb (explain / describe / walk through / understand / analyze / trace / how does / where does / what does / show me / tell me / overview of / summary of) combined with a codebase target (repo / codebase / project / system / architecture / structure / module / component / package / service / flow / data flow / control flow / function / class / file / works). Both must be present.
- **Net effect:** "Explain this repo structure" → routed to full lane, gets up to 25 iterations. "Pick a number 1-9" → still the simple-task fast path. The previous heuristic flagged any "explain X" message as a simple question with an 8-iteration cap, which was wrong for codebase-scoped exploration.
- **Bumped guided-lane simple-task cap from 8 to 12.** With exploration intent now bypassing `isSimpleTask`, the strict 8-cap was over-strict for the remaining "simple" cases that still legitimately need a few file reads.

### Added: Agent panel stream visibility

- **Mid-loop reasoning disclosure.** `ThinkingDisclosure` was nested inside the streaming-response block in `MessageList.tsx` so it only rendered AFTER `streamingContent` arrived. During the agentic loop's tool-calling iterations (where reasoning streams between tool decisions but no final-response tokens are sent yet) the panel went dark. New render path surfaces reasoning during tool-calling phases. **The same change would have caught today's two ship-blocker bugs visually within seconds instead of timing out at the iteration cap 7+ minutes later.**
- **Path context in tool rows.** `Listed agentic` → `Listed …/services/agentic`. Tool-call rows now show the last three path segments with a leading ellipsis when deeper. Mixed Windows/Unix separators normalized. Full path still on hover via the tooltip.
- **Time-aware reassurance copy** in `StreamingStatusBar`. At 15s the static phase label swaps to "Thinking. This is taking a moment". At 60s, "Still thinking. This is a longer one". At 120s, "Still working. Large context or rate limit may slow this down". Only fires in the thinking phase; tool-call and writing phases keep their dynamic labels. The animated dots and elapsed timer are unchanged.

### Fixed: `/think` prefix sent to non-Qwen models (ship-blocker, found live)

- **`LoopStateMachine.computeQwenThinkingPrefix`** had been extended to the DeepSeek family on the false premise that DeepSeek R1+ understands Qwen3's `/think` slash-prefix toggle. It does not. DeepSeek's thinking mode is automatic via inline `<think>...</think>` blocks; no input control prefix is recognized.
- **Joe-hit symptom on deepseek-v4-flash, "Explain this repo structure":** every iteration was prepended with `/think\n\n`, the model parsed it as an unknown user slash command, and refused: *"`/think` is not a registered command in this environment. I have no handler or skill for it. I've stopped reading files. I'm not calling tools."* Loop hit the 20-iteration cap with $0.034 burned and no answer.
- **Fix reverts the family check to qwen-only.** Twelve regression tests pin the contract for non-qwen families across every iteration count + signal permutation.

### Fixed: Nudge classifier mismatch (ship-blocker, found live)

- **`NudgeOrchestrator.detectOverEagerToolUse`** and **`detectExplorationLoop`** now route through `RuntimeLayer.isExplorationIntent` first. Pre-fix, two classifiers disagreed: `RuntimeLayer` correctly routed exploration prompts to the full lane (PR #308), but `NudgeOrchestrator` kept its own simpler heuristic that flagged "explain X" as a simple question regardless of whether X was a codebase target.
- **Joe-hit symptom on qwen3-coder-plus, same prompt:** after 3 file reads the loop emitted *"STOP READING FILES. This is a knowledge question that does not require file access. Answer from your training knowledge directly. Do not call any more tools."* The model produced a 5-point rebuttal quoting the nudge text back to the user as a contradiction, then capped out at iteration 25.
- **Fix:** exploration intent now skips both nudges. Existing nudge cases keep working: "explain closures in JavaScript", "what is 2+2?", "how does JavaScript work?" all have no codebase target, all still trigger the nudges correctly. Nine regression tests cover the Joe-hit prompt, four other exploration phrasings, and three negative cases.

### Fixed: Cross-mode state leaks (3 fixes)

- **Tool approval cards** now scope by `sessionId`, so a pending approval from a code-mode session can no longer surface in a chat-mode session and vice versa.
- **Clarification cards** scope the same way.
- **Plan approvals** scope the same way.
- Backwards-compat: untagged legacy approvals (no `sessionId`) still match the active session so users mid-flight don't lose work.

### Fixed: Qwen iteration thinking-only loop

- Qwen models occasionally enter a state where they emit `reasoning_content` for an entire iteration without producing any final content or tool calls. The agentic loop interpreted the empty content as "model failed" and retried. New `thinkingCaptured` thread through `AgenticLoopCoordinator` recognizes the pattern as legitimate progress and lets the loop continue.

### Fixed: FIM model routing (2 fixes)

- **FIM is now opt-in.** Default `fim.mode` is `'off'`. When `'auto'`, the system tries to pick a FIM-capable model based on the active provider. Pre-fix default was `'auto'` and an over-permissive compat filter was routing `devstral-small-2` (Mistral family) into DeepSeek FIM requests, returning 400.
- **Strict per-preset model allowlist.** `ProviderPresets.STRICT_MODEL_PREFIXES` maps each cloud preset to a list of family-prefix strings (`deepseek-`, `kimi-`, `qwen-`, etc.). `isModelCompatibleWithProvider(model, providerType, presetId)` enforces the allowlist when `presetId` is given. Local llamacpp / Ollama users still have free choice.

### Fixed: DeepSeek model profiles + Kimi temperature lock

- **DeepSeek v4 profiles added.** `deepseek-v4-pro` (131k context, 16k output, ~671B MoE) and `deepseek-v4-flash` (131k / 16k / ~50B MoE) replace the deprecated `deepseek-chat` / `deepseek-reasoner` aliases that hard-error on 2026-07-24.
- **Kimi K2 temperature locked to 1.0.** Per Moonshot's docs, Kimi K2 is fine-tuned at temperature=1; lower values produce degraded output. The model card now carries `temperatureLocked: true`. The temperature slider in Settings disables for locked models. Per-message override paths skip the override on locked models.

### Fixed: Window leak guard

- The streaming code in `chatApi-streaming.ts` referenced `window.electronAPI` without guarding for SSR / non-Electron contexts. Defensive coercion (`coerceString`, `coerceNumber`, `coerceBoolean`) added before `JSON.stringify` of any user-provided fields.

### Fixed: GuidedTour re-fire

- Tour was re-firing on session rename or model change because the trigger filter was matching unrelated events. New `tourDismissedRef` plus stricter `s.type === undefined || s.type === 'chat'` filter in `App.tsx` prevents the re-fire. `markTourCompleted` now syncs to Zustand explicitly.

### Architecture

- **BYOK cloud providers and Cloud Boost are now architecturally separated** at the storage layer (per-provider `llm.*_api_key` vs. `boost.*` settings) and the routing layer (`LLMService` for primary, `ProviderRegistry.getProviderForRequest` for boost). The two features can coexist on the same user: e.g., DeepSeek as primary + OpenRouter as boost. Both honor air-gap. Settings copy in both sections now points at the other for users who confuse the two.
- **`RuntimeLayer.isExplorationIntent` is now the single source of truth** for exploration-intent detection. Used in three places: `RuntimeLayer.classify` (lane assignment), `NudgeOrchestrator.detectOverEagerToolUse` (skip over-eager nudge), `NudgeOrchestrator.detectExplorationLoop` (skip exploration-loop nudge).

### Migration

- One-shot BYOK migration runs on first launch after upgrade. Copies `llm.openai_api_key` → active preset's per-provider key (only when active preset is one of the new cloud providers and the destination is empty). Copies `boost.api_key` → `llm.anthropic_api_key` only when active preset is `anthropic`. Source keys are never deleted. Cloud Boost retains its key. Gated by `general.byok_migration_v2_complete` so it runs once per user. Existing setups don't lose access.
- DB migration for `messages.reasoning_content`, `usage_logs.byok_cost_usd`, `usage_logs.byok_provider` runs idempotently on first launch.

### Live verification

- **DeepSeek-v4-flash on "Explain this repo structure to me, and its purpose"** post-fix: 2,556 output tokens at 103.2 tok/s for $0.0091, accurate repo description with real numbers from CLAUDE.md (228 components, 14 Zustand slices, 10 API modules, 25 tools, 21 providers), correctly identified the QEL pipeline (Contract → Tool Use → Verification → Proof Gates → Repair). Vs pre-fix: 20-iteration refusal loop costing $0.034 with no answer. **73% cheaper, 25 seconds vs 7+ minutes.**

### Known issues

- **100k session token cap fires post-completion as a warn-level log** for long exploration sessions. Non-blocking (the response completes) but worth raising the cap (or making it per-provider configurable) in beta.19 if users on cheaper-per-token models hit it routinely.
- **Pre-existing file-size violations:** `LLMService.ts` 736L (over 700), `useChat.ts` 429L (over 400). Both grew with BYOK lookup helpers and cloud-cost wiring. Splits planned for beta.19 (`LLMServiceByok.ts` extraction).
- **Anthropic extended-thinking blocks roundtrip** uses a different protocol than OpenAI-compat reasoning_content (signed content blocks). `AnthropicProvider.parseResponse` still drops thinking blocks. Latent bug if a user enables extended thinking with Claude, separate fix path from PR #307. Tracked for beta.19.
- **Cross-family session handoff** matrix only covers OpenAI-compat ↔ OpenAI-compat. Anthropic ↔ OpenAI-compat hot-swap needs a format-conversion layer. Tracked for beta.19.

### Test counts

- Backend: 208 files, 3,918 tests (+146 over beta.17) + 54 skipped (live-Ollama-gated)
- Frontend: 39 files, 525 tests (+64 over beta.17)
- All tsc clean, all ESLint clean, webpack build successful

---

## [v1.0.0-beta.17] - 2026-05-07

Headline: **Ollama joins the painless-install club.** Set up Ollama from inside Bodega (no terminal, no detour to ollama.com) matching the llama.cpp managed install we shipped in beta.16. Plus a stack of ~12 onboarding polish bugs caught during a real demo recording session.

### Added: V2 Onboarding Phase 1: Ollama managed install

- **One-click Ollama install.** FirstRunFlow's "I'll install Ollama" path now downloads the official Ollama installer (pinned to v0.23.1, SHA256-verified against Ollama's `sha256sum.txt`), runs it under UAC on Windows / extracts to `/Applications/` on macOS / extracts to `~/.local/bin/` on Linux, polls `localhost:11434` until the service is live, then surfaces a curated 6-model picker. Old path bounced users to ollama.com in an external browser; new path keeps everything inside Bodega.
- **Smart "already installed" detection.** Probes `%LOCALAPPDATA%\Programs\Ollama\ollama.exe` (Win), `/Applications/Ollama.app` (Mac), `~/.local/bin/ollama` (Linux) before downloading. If Ollama is installed but the service is stopped, skips the 2 GB download and just starts the service via `Start-Service Ollama` / `open -a Ollama` / `ollama serve &`.
- **Hardware-tier-aware model picker.** 6 curated starter models (Qwen 3 1.7B / 4B / 8B / 14B + Qwen 3 Coder 7B / 30B). Recommended badge highlights the tier-default for your VRAM but you can pick anything from the list.
- **Truly silent installer on Windows.** `/VERYSILENT` so Ollama's installer doesn't pop a competing UI over Bodega. Wait fix: PowerShell `Start-Process -PassThru + WaitForExit()` instead of `-Wait` (the latter races on UAC-elevated children and used to leave Bodega stuck on "Installing…").

### Added: Demo recording infrastructure

- **`demo-fresh-install.ps1`** at the worktree root: pre-flight script that wipes a sandboxed `BODEGA_USER_DATA_DIR`, optionally stops Ollama (`-KeepOllama` switch), and launches Bodega in dev mode. Used for marketing-video captures that need a clean first-run state.
- **`10-onboarding.demo.ts`** Playwright spec with three variants (`onboarding-local-ready`, `onboarding-llamacpp-full`, `onboarding-llamacpp-offer`) that drive the FirstRunFlow deterministically while OpenScreen captures pixels.
- **`source-of-truth/V2-ONBOARDING-SPEC.md`**: full spec for the V2 initiative: three flows (Local AI, Cloud API, Bring your own), four install patterns, per-provider effort estimates. Phase 1 (Ollama, this release) shipped; Phases 2 (Cloud APIs) + 3 (LM Studio) scoped for next sprint.

### Fixed: Onboarding polish (12 bugs from the demo session)

- **Critical: code mode broken on llama.cpp.** `handleLlamacppComplete` used to write `llm.preset` before `llm.default_model`; reconfigure-on-preset-change had no model name to spawn llama-server with, OpenAIProvider fell back to `https://api.openai.com/v1`, connection failed, preset auto-reverted to `ollama`. User landed in chat with `preset=ollama` and a llamacpp record-id as the model name. Order fixed.
- **Modal stays up during model warm-up.** Polls `/api/llm/health` after install until the model is actually serving (was dismissing in 100 ms, leaving the user staring at "No AI Provider Detected" for 5-30 s).
- **Rate limiter loopback bypass.** `/api` route's 1000-req/15-min limiter was throttling the desktop renderer's localhost calls, causing "Catalog failed: Too Many Requests" and "Failed to save theme" cascades during heavy onboarding polling. Loopback IPs (`127.0.0.1`, `::1`) now bypass; network-exposed traffic still throttled.
- **High-VRAM users get a small-model option in the catalog.** Per-category recommendations on a 24+ GB GPU were all 10+ GB models; "More options" showed siblings of the same heavyweights. The `recommend()` algorithm now appends a small/fast model (e.g. SmolLM 1.7B at 1.06 GB) when no recommendation is under 3 GB.
- **`managed-1778…` GGUF record ids no longer leak to the UI.** New `resolveModelDisplayName(raw, available)` util substitutes the loaded model's friendly filename in StatusBar, ModelSelector chip + dropdown, MessageBubble badge, PanelSidebar header, PanelModelBadge per-panel pills.
- **Catalog downloads survive navigation.** Lifted the SSE subscription out of the React component into a `llamacppDownloadCoordinator` module-level singleton + `hubLlamacppDownloads` Zustand slice. Inline progress bar with stage / pct / bytes / rate. Cancel button. Switching tabs no longer drops the in-flight download.
- **GuidedTour rewrite.** Per-step `requiresMode` / `requiresAgentPanel` / `requiresTerminal` / `requiresLeftPanel` flags so the tour auto-switches the app into the right state instead of silently auto-skipping. ResizeObserver-tracked spotlight stays glued through animation. Auto-flip tooltip position when the requested side has no room (no more tooltip-on-top-of-target). Dynamic tooltip-height measurement so longer descriptions land in the right spot. Backdrop holds during step transitions, no UI flash.
- **`data-tour="file-tree"` added to the empty-state branch** of FileTreePanel, was only on the project-loaded branch, so the file-tree tour step silently auto-skipped on fresh installs.
- **Permission Modes tour copy mentions Plan**, was Ask + Act only.
- **OllamaOnboarding: "Pulling X" label shows the actually-pulling model**, not the recommended one. Three-state separation: `recommendedModel` (system suggestion) / `pickedModelId` (current UI selection) / `pullingModelId` (what `startModelPull` actually got).
- **OnboardingChecklist removed.** The floating bottom-right pill was redundant once GuidedTour shipped; two parallel onboarding systems competed for attention.
- **Routing tier picker stays visible in chat mode for single-active providers.** Hiding it after llamacpp loaded felt like the mode selector vanished; picker now stays, routing just maps to the same model.

### Fixed: Security

- **Shell injection hardening in OllamaInstaller.** PowerShell + exec sites used single-quote string interpolation for installer/extract paths; a Windows username with a `'` (e.g. "O'Brien") would break out of the literal. Replaced with `JSON.stringify` PS escaping + array-args `spawn()` everywhere else (no shell layer at all).

### Polish

- Agent panel default width 300 → 380 px so tab labels + status pill fit without dragging mid-recording.
- `BodegaPanel.tsx` deleted: superseded by `PanelSidebar.tsx`, dead since beta.15. Removing prevents future "wired the wrong file" confusion.
- `streamSse` exported from `llamacppApi.ts` so V2 phases 2 + 3 can reuse the same SSE plumbing.

### Known issues

- **macOS + Linux Ollama install paths are scaffolded but untested on real hardware.** Logic mirrors the llama.cpp Mac/Linux installs that have been in production since beta.16 + we used `ditto` / `tar --zstd` (the standard tools), so there's high confidence, but they haven't been smoke-tested on a real Mac or Ubuntu/Fedora box. File a bug if you hit anything weird.
- **Linux `tar --zstd` requires GNU tar 1.31+.** Ubuntu 20.04 LTS ships 1.30. Older systems get a clear error message ("install the `zstd` package or update GNU tar to 1.31+ for --zstd support") instead of failing silently.
- **Pre-existing file-size violations:** `FirstRunFlow.tsx` (569 lines) and `GuidedTourOverlay.tsx` (467 lines) are over the 400 React component cap. Acknowledged tech debt: splits planned for the next sprint before V2 Phase 2 work begins.

---

## [v1.0.0-beta.16] - 2026-05-06

Headline: **first-class llama.cpp support.** Bodega now installs and manages a llama-server binary, downloads GGUFs from a curated 17-model catalog, hot-swaps between them with crash recovery, and stays consistent with cloud providers when you switch back. Plus a cascade of correctness + security fixes from a multi-agent review pass.

### Added: llama.cpp UX overhaul

- **Managed llama-server install.** Settings → Models → llama.cpp → "Install latest". Bodega downloads, SHA256-verifies, and extracts the platform-appropriate llama-server binary (CUDA 13 for NVIDIA, Metal for Apple Silicon, CPU fallback). Pinned to b9045. Air-gap-aware (refuses with clear message + sandbox-checked install path).
- **GGUF catalog + hardware-tier recommendations.** 17 curated models across families (Qwen2.5-Coder ×5, Qwen3 ×4, Llama 3.1/3.3, Mistral / Devstral, Phi-4, Gemma-3, DeepSeek-R1-Distill, SmolLM). The recommend endpoint reads detected VRAM and surfaces models that fit, sorted by headroom. Each entry has size + license + tools-capability + FIM-capability metadata.
- **Hot-swap between GGUFs.** `POST /api/llamacpp/swap` with a record id; the manager terminates the current llama-server, waits for the file lock to release (Windows), then spawns the new model. Mutex-coalesced under spam: 10 rapid swap requests collapse to one start + one queued-next.
- **Sideload existing GGUFs.** `POST /api/llamacpp/sideload` with an absolute path registers the file with full metadata (family, supportsTools, supportsFIM, contextLength). Available even in air-gap mode.
- **Crash recovery.** Event-driven exit detection (LlamaServerProcess emits `exit` events keyed by spawn time so stale signals are ignored). When llama-server dies unexpectedly, the manager auto-restarts up to 3 times in a 60s window. Verified live: `taskkill /F` → recovered with new PID in ~11s.

### Added: Onboarding

- **Multi-model picker on local-ready screen.** When you have multiple chat-capable models from a single provider, the FirstRunFlow now shows radio cards instead of silently picking the first one. Embedding-only models (nomic-embed, mxbai-embed, bge-*, etc.) are filtered out at every selection site.
- **llama.cpp managed-install offered when no provider is detected.** If detection finds no Ollama / LM Studio / cloud key, FirstRunFlow routes to a "Set up llama.cpp" screen with the catalog + recommended-for-your-hardware picks, before falling through to the legacy no-provider screen.

### Fixed: Code mode tools

- **Code mode now actually calls tools on llama.cpp.** Pre-fix: agentic loop shipped `effectiveTools=[]` because the model name resolution returned a record id (`managed-...`) that didn't match any profile, so `caps.nativeFunctionCalling` defaulted to `false`. Now the OpenAIProvider gets the served filename in-memory while settings keep the record id. Verified live: `loop_setup` reports `toolCount=21, family=qwen, nativeFunctionCalling=true`.
- **Cross-family caps alignment.** Phi-4 + Gemma-3 catalog entries previously claimed `supportsTools: true` but their profiles pinned `nativeFunctionCalling: false` (Joe-tested earlier: Gemma "degenerates into thinking-only output after 3-4 successful tool calls"). Catalog now matches profile reality so the `--jinja` flag and the agentic tool-passing decision stay consistent.
- **DeepSeek-R1-Distill-Qwen no longer mis-classified as qwen.** `detectFamily` checked qwen before deepseek, so distills inherited the wrong family caps. Re-ordered + new `deepseek-r1-distill` profile entry pins R1 distills to `nativeFunctionCalling: false` (the thinking/tool interleaving breaks down in the distilled weights).
- **Persistence on restart now works.** Pre-fix: every Bodega launch with `preset=llamacpp` configured silently fell back to default Ollama provider because `GgufRegistry.setDb()` ran AFTER `LLMService.reconfigureFromSettings()`, throwing "DB not initialized" inside the auto-load path. The error logged at `level=warn` was easy to miss; users had to manually toggle preset to recover. Init order corrected. Auto-load verified live within ~30s of restart.

### Fixed: Context window cap (Ollama vs llama.cpp architectural difference)

- **Agent context budget no longer over-promises what llama.cpp can serve.** Ollama negotiates `num_ctx` per request; llama-server's `-c` is fixed at spawn. Bodega's resolver was using the hardware ceiling for both, but the catalog deliberately spawns Qwen3-30B-A3B with `-c 32768` for VRAM safety. Pre-fix: agent thought it had 65K context, llama-server only 32K → mid-stream truncation/errors at long contexts. Now: `LlamaServerManager.getActiveContextWindow()` exposes the spawn `-c`; resolver caps to it for the active llamacpp preset. Crucially the cap is gated on `settings.llm.preset === 'llamacpp'`. Without the gate, switching preset to Anthropic in the same session would silently truncate every cloud chat to 32K.

### Security

- **Removed `PENDING_VERIFY` SHA256 bypass in binary install** (CRIT). The sentinel let entries log a warning and proceed with an unverified binary. SHA256 is the only integrity gate on `llama-server.exe`. A future config entry shipping with that sentinel would have meant arbitrary code execution on a CDN compromise / MITM. Now hard-fails when the hash is missing or unverified.
- **Sandboxed `binary_dir` and `models_dir` settings to user-data dir.** Pre-fix: a settings DB write or crafted settings-import file could redirect the verified binary extraction to an OS-privileged directory (`/etc`, `C:\Windows\System32`). Now both settings assert `path.resolve(override)` falls within `BODEGA_USER_DATA_DIR`.
- **`GgufDownloader.startDownload` now gates on `general.air_gap` at the service level**, not just at the route. Defense-in-depth: any future caller that bypasses the route still hits the gate.
- **Fetch monkey-patch redacts query strings + fragments.** The network log used to capture full URLs including Gemini-style `?key=` API tokens. Now path-only logging; status / duration / size / method preserved. Bonus: `/infill` and `/v1/completions` added to the LLM-request filter so FIM routing is finally observable in the diagnostics bundle.

### Fixed: FIM routing

- **`/infill` is correctly routed for llama.cpp.** Beta.14 introduced the `/infill` endpoint but a `servingType` propagation gap meant FIM completions silently fell through to `/v1/completions`. Now `LLMService.reconfigure` derives `servingType` from preset id (`llamacpp` → `'llamacpp'`) so the `/infill` path actually fires.
- **FIM is gated on the loaded model's `supportsFIM`.** Pre-fix: sending FIM requests to non-FIM-trained models like Qwen3 returned conversational reasoning instead of code completion (200 OK + garbage). Now: when llama.cpp is loaded with a non-FIM model, FIMService skips `/infill` and `/completions` entirely and uses prompt-injection (more robust on non-FIM-trained models).

### Fixed: Provider switch UX

- **Preset switch reverts the setting when reconfigure fails.** Pre-fix: switching preset to Ollama with Ollama down left settings in inconsistent state (`preset='ollama', provider='openai'`, confused user about which one was actually serving). Now reverts the setting and surfaces a clear warning naming both attempted and reverted values.
- **Recovery race fixed.** If a user clicked reconfigure during the 2s post-crash cooldown, the recovery would terminate the user's spawn and reject. Now: state guard before re-spawn. Recovery skips if state has already advanced past `crashed`.

### Fixed: Onboarding (continued from beta.15)

- **Embedding-filter holes.** beta.15 fixed 3 of 5 silent-complete sites; the reactive fast-path and the multi-model fast path still picked `availableModels[0]` blindly. Filled both. Multi-model picker now shows a real chooser (radio cards) for users with multiple chat-capable models.

### Internal

- Removed orphaned `llamacpp.last_loaded_model` setting (was write-only: the answer comes from `llm.default_model` plus `ggufRegistry.markLoaded`'s `lastLoadedAt` timestamp).
- Removed dead `/settings/model/:modelName` routes + API client surface + `model_settings` SQL table type. Per-model overrides go through `llm.model_overrides` JSON (one storage mechanism, not two).
- Phase 6 validation gate fixed (was sending malformed request body, all 6 questions silently failing). Now 4/6 PASS against Qwen3-30B-A3B; remaining 2 are real model-behavior findings (Q1 Qwen3 conversational tool-call rate, Q4 long-context handling) tracked for v1.0.2.
- Dev-mode `DB_PATH` now defaults to `<BODEGA_USER_DATA_DIR>/bodega.db` when set, falling back to literal `bodega.db` only when neither env var is present. Production unchanged (Electron sets DB_PATH explicitly via backend-manager).

### Tests

Backend 3,728 → **3,772** (+44 covering crash recovery, alreadyLoaded liveness, stale-spawn ignore, llamacpp context-cap branch, cross-preset cap leak regression). Frontend 461 unchanged. tsc + ESLint + webpack all clean.

---

## [v1.0.0-beta.15] - 2026-05-05

Cross-platform fix bundle from Joe's beta.14 install-test on Windows + Martin's on Intel Mac. Same-day follow-up to beta.14: 8 fixes plus a major expansion of the diagnostics-bundle export so future "user got an error and is gone" reports are debuggable from a single dropped file.

### Fixed: First-launch UX

- **First-run flow no longer picks an embedding model as the default.** Martin's macOS Intel install ended up with `llm.default_model: "nomic-embed-text"` because Ollama's tag list returned the embedder before any chat-capable model. Two failed 0-token completions before he manually switched to `qwen3:4b`. New `isEmbeddingModelName()` filter in `modelUtils.ts` covers `nomic-embed`, `mxbai-embed`, `bge-`, `all-minilm`, `jina-embed`, `gte-`, `e5-`, `instructor-`, `voyage-`, and the `-embedding`/`-embed` suffix shapes, applied at every selection site in `FirstRunFlow.tsx` plus the `resolveAutoCodeModel` / `pickFastModel` / `pickSmartModel` auto-resolvers. Embedders are still selectable in the embedding settings; they're just no longer eligible defaults for chat/code.

### Fixed: Agentic loop

- **`gemma4:26b` no longer wedges in the agent panel after a few tool calls.** Joe-hit on Windows: 3 successful file reads then 5 thinking-only iterations stripped to empty by the filter, hitting iter cap with no answer. Root cause: `ModelProfileService` blindly trusted Ollama's GGUF `tools` capability flag for every model family. New `NATIVE_TOOLS_DENYLIST_FAMILIES` (currently `['gemma']`) keeps the family default (text-injection format) when Ollama claims native tool support but the family is known to degenerate. Explicit `gemma4` entry in `model-profiles.json` pins this for the model card too.

### Fixed: macOS

- **Traffic lights no longer overlap the top-bar controls when not fullscreen.** Martin-reported on Intel Mac. `BrowserWindow.trafficLightPosition: { x: 12, y: 9 }` shifts the lights down + right; `.platform-darwin .titlebar-left-pad { padding-left: 76px }` insets the hamburger / gear / search row to clear them. `platform-darwin` class is set in `App.tsx` based on `navigator.platform`.

### Fixed: Code-mode chrome

- **Switching sessions mid-stream now tells the user it cancelled.** Joe-asked: "not sure if this is a bug or intended." Stopgap fix: a toast informs the user when the switch tore down an in-flight response. Full background-streams rework (Cursor-style: keep the stream alive in the original session, lift `AbortController` to a per-session map in Zustand) is tracked as a v1.0.2 task.

### Fixed: Model Hub

- **Pull-by-name search bar surfaces validation errors instead of silently failing.** Frontend was swallowing the 400 with `.catch(swallowLog)`, then the SSE progress stream errored seconds later with a generic "Download failed", useless. Now awaits the `startModelPull` request so the actionable error from the backend ("Use only letters, numbers, dots, colons, …") reaches the user. Backend trims whitespace before validating and returns a message that names the offending input + lists allowed characters.

### Added: Diagnostics bundle expansion

The "Export Diagnostics Bundle" button in Settings → About now writes a much richer report. Bundle contents:

- **App + Process versions** (existing)
- **System**: platform, arch, OS release, CPUs, RAM (existing)
- **UI Context** (NEW): active app mode, panel, model, session ID
- **Hardware** (NEW): VRAM tier classification, total/free VRAM, max model param, smart/code tier capability
- **LLM Health** (NEW): provider status, available models
- **Settings (redacted)** (existing)
- **Recent Sessions** (NEW): last 5 with title, msg count, timestamps
- **Recent Errors** (NEW): UI ring buffer of last 50 classified errors with model + session ID at time-of-error; `classified.raw` truncated to 200 chars to avoid prompt leakage
- **Network Activity** (NEW): last 20 outbound LLM calls (method, URL, status, latency, redacted error)
- **Recent Telemetry** (existing)
- **Backend Logs** (NEW): last 500 lines of pino JSONL, redacted

The backend logger now writes to `<userData>/logs/backend-YYYY-MM-DD.jsonl` in addition to stdout (rotating daily) so the bundle has something to tail. The renderer ships its in-memory error ring buffer + active session/mode/model via the IPC payload. Disclaimer was softened from "safe to share publicly" to "review before sharing widely". Automatic redaction covers credentials, but user-typed content (chat messages, file paths, error excerpts) can still appear and may want a manual review.

### Pre-merge code review fixes

Two-reviewer pass before merge caught and fixed:

- **BLOCKER:** `activeSessionId: null` (first-launch state) would crash the diagnostics export. The `assertShapeOptional` validator typed the field as `'number'` and rejected `null`. Stripped to `undefined` at the renderer boundary.
- **HIGH:** unhandled `'error'` event on the new pino file stream would crash the Node process on disk-full / Windows-AV-locked log paths. Added an `.on('error')` handler that writes to stderr and lets `pino.multistream` continue on stdout.
- **HIGH:** `classified.raw` and network `ev.error` could echo prompt content from provider errors, both now truncated and redacted before serializing.
- **MEDIUM:** `isEmbeddingModelName` regex catch-all anchor was too loose (would false-match `embedding-coder`-style fine-tunes). Tightened.

---

## [v1.0.0-beta.14] - 2026-05-05

Layout overhaul, agentic loop hardening, llama.cpp provider expansion, and a long tail of Joe-tested polish. The IDE layout no longer fights the user's drag, the agent doesn't wedge at iter cap, and the Code-mode chrome (model badge, context ring, composer) now reflects what's actually selected.

### Added: llama.cpp provider expansion

- **`/infill` endpoint** in `OpenAIProvider`: llama-server's mid-context completion path. Bodega's Code mode now drives the same FIM (fill-in-the-middle) flow the upstream llama.cpp completions tool uses, so `Qwen2.5-Coder-*-Instruct` GGUFs route through `/infill` instead of getting forced into chat-completions for inline suggestions. Test GGUF + invocation guide in `apps/desktop/backend/scripts/test-llamacpp.md`.
- **`configs/` bundled in extraResources** so packaged builds can resolve `model-profiles.json` (E3). Fixes "Failed to load model catalog" on first launch of an installed `.dmg` / `.exe`, same ENOENT signature Martin hit on beta.12.2.
- **Terminal side-dock.** Terminal can dock right OR bottom in Code mode (`terminal.dock_side` setting). Right-dock is the layout most useful next to a wide editor; bottom-dock keeps the VS Code default for muscle memory.

### Fixed: IDE layout (E1)

- **CSS Grid migration in CodeLayout + ChatLayout.** `grid-template-columns: 40px ${left}px minmax(200px, 1fr) ${right}px` replaced a flex+coordinator approach that fought five rounds of patches. Grid handles overflow correctly: when total > viewport, the `1fr` track shrinks to its minmax minimum (200 px); explicit pixel tracks then compress proportionally. Right sidebar can never be pushed off-screen because the grid container constrains all tracks. The JS coordinator that tried to shrink one sidebar when the other grew is deleted.
- **Hard-cap on sidebar width.** Wide markdown content was bursting the grid track when streaming hit a long fenced block. The panel was honoring `agentChatWidth` from the store even after the user had implicitly dragged it past the container minimum. Now clamped at the container level.
- **Panel inner divs need explicit `w-full + min-w-0`.** `react-resizable-panels`' `Panel` is `display: flex` row-direction by default; without an explicit cross-axis width on the inner div it sizes to intrinsic content (Monaco at ~800 px, xterm at ~640 px) and pushes the right sidebar off-screen the moment the terminal opens. Diagnosed and fixed at every Panel inner-div in CodeLayout.

### Fixed: Dropdowns

- **AttachDropdown freeze on fullscreen** + vertical clip on tall content. Dropdown now subscribes to `ResizeObserver` + `fullscreenchange` and recomputes `maxHeight` dynamically so it doesn't re-render with a stale clip. Same pattern propagated to `RoutingPills`, `PermissionModeDropdown`, `ModelSelector`, `TopBarSearch`, and `Breadcrumbs`.

### Fixed: Agentic loop

- **`ITERATION_CAP` transition was dead code.** `LoopStateMachine` could enter `LLM_CALL` → `TOOL_PROCESSING` → `LLM_CALL` indefinitely without any state ever observing the cap. Added the explicit transition + a forced-synthesis branch with observability: when the loop hits `maxIter` without producing a textual response, the orchestrator runs one final no-tools LLM call to summarize what was done, instead of leaving the user with a silent abort.
- **Abort vs iter-cap race.** A click-Stop during the synthesis branch was being attributed to "aborted by user" even when the cap had already fired. Only the LSM's `LLM_CALL` phase counts as aborted; tool processing during forced-synthesis stays attributable to the cap.
- **Question-intent short-circuit must NOT fire on the agent panel.** The classifier was bouncing "what does X do?" past the agent loop with a chat-style response, even when the user was clearly in Code mode interrogating their codebase. Now scoped to chat panel only.

### Fixed: Tools / contracts

- **`SkillContractBridge`: workflow steps were misinterpreted as file deliverables.** A workflow with a `generate_file` step ID was creating a file literally named `generate_file` because the step → deliverable inference treated the step ID as a basename. Step → deliverable inference removed; deliverables now require an explicit `path`.
- **`Ctrl+Shift+D` triple-handler collision.** EditorArea had its own keyboard handler bound to the same shortcut as the global composer toggle and a third path inside Monaco. Reconciled to a single handler that toggles `secondaryPanel='composer'`.

### Fixed: Onboarding / first-run

- **Multi-model fast-path no longer silent-completes.** The fast-path was completing onboarding with `general.has_completed_onboarding=true` on any provider with ≥1 model. But if Joe had three, it picked one without him seeing the choice. Single-model only silently completes; multi-model routes back to the local-ready picker.

### Fixed: Persistence / multi-user

- **`SessionService` + `ProjectService` now scope queries by `user_id`.** Cross-user memory leak path was a `WHERE branch = ?` query without a `user_id` filter, flagged in CLAUDE.md as a security boundary. New `memoryScope` util provides `getMemoryUserId()` and is now the only path that constructs the user-scoping clause.

### Fixed: UX polish

- **Compact LLM 45 s timeout.** Compaction was hanging indefinitely on slow/loaded models; now bounds the request and surfaces an error toast on timeout.
- **Escape works with focus inside the chat input.** Was being swallowed by the textarea handler before the global shortcut could close the active overlay.
- **Toast `ErrorBoundary` with `fallback={null}`.** A faulty toast no longer takes down the entire toast system; the bad toast disappears silently rather than rendering "Error" placeholder chrome.
- **Build-index toast lifecycle fixed.** 5-minute duration matches the operation's worst case + explicit `removeToast` in the `finally` so success/error toasts replace it cleanly.
- **Terminal blank after chat→code switch.** `registerSlot` now eagerly moves the portal container when a matching slot registers, instead of waiting for the next render to bind. Was a lazy-mount race that made xterm appear blank for 100–300 ms after every mode switch.

### Fixed: Code-mode chrome (display sync)

- **Model dropdown override pin.** Agent panel was missing the user-pick guard that `ChatStage` already had. Every settings tick was overwriting the user's dropdown choice with the configured default. `useAgentPanelModel` now mirrors ChatStage's `userPickedModelRef` + `lastEffectiveDefaultRef` pattern.
- **Model badge sync.** Once the dropdown override held, two display readouts still showed the default model (PanelSidebar header + bottom StatusBar). `useAgentPanelModel` now writes the agent panel's `selectedModel` to `activeRoutedModel` on user pick and on resolved-default change; PanelSidebar prefers `activeRoutedModel` over `settings['llm.code_model']` for the agent badge.
- **Context ring routing in code mode.** The chatbar's `ContextBudgetBar` was firing `toggleContextInspector` (a global flag), but the only `<ContextInspector>` mount lived inside the chat-mode div in App.tsx. Now mode-aware: code mode dispatches `bodega:toggle-context-inspector` (matching the existing `bodega:toggle-sessions-drawer` pattern) and PanelSidebar listens to toggle its local slide-up inspector.
- **Composer reopen path.** Closing the Changes panel left no discoverable way back. ReviewBar's expand button is hidden inside the agent panel and only renders on pending changes. Added a persistent "N pending / N reviewed" button in StatusBar's left section whenever `sessionChangedFiles` has entries.

---

## [v1.0.0-beta.13.1] - 2026-05-04

Hotfix bundle for v1.0.0-beta.13. **beta.13 was drafted (un-published) within the hour of release** because the macOS x64 build shipped an arm64 sqlite3 binding (broke Intel Macs) and the Linux arm64 build shipped an x86_64 sqlite3 binding (broke arm64 Linux). Martin caught the dlopen error on his Intel Mac and three additional UX regressions during smoke testing before any external user saw the binary.

### Fixed: Cross-arch native modules

- **`npmRebuild` flipped from `false` to `true`.** electron-builder now runs `@electron/rebuild` for each TARGET arch during packaging, instead of relying on whatever arch `electron-builder install-app-deps` happened to install during the `postinstall` step. Each platform package now contains the correct native binding for its architecture.
- **Linux build matrix split into per-arch runners.** `ubuntu-latest` (x86_64) builds the x64 `.deb` + `.AppImage` only; the new `ubuntu-24.04-arm` runner builds the arm64 variants. Previously a single `ubuntu-latest` job claimed to build both archs but couldn't compile aarch64 ELF without a cross-compiler. macOS keeps a single runner (Apple's universal toolchain handles both archs natively).

### Fixed: UX regressions caught during Martin's smoke

- **OnboardingChecklist "Send your first message" now ticks immediately.** `message_count` was being read from the sessions list (which mirrors the backend's stored count and isn't refetched per-send), so the first-message item never completed until the user navigated away and back. `addSessionMessage` now bumps the matching session's `message_count` locally as messages arrive, so the checklist reflects state in real time.
- **Quick Message buttons under the logo now populate the input.** SuggestionChips were calling `chat.setInput()` directly; some state-flow path between the chat-prop and the controlled textarea wasn't picking up the value reliably (Martin's repro: clicking a sub-prompt did nothing). Switched the chip click to route through `setPrefillMessage` (the same store path FirstRunFlow uses), and changed the consumer effect in `ChatStageInput` to depend on `prefillMessage` so it actually fires on post-mount changes (was `[]` deps, fired on mount only).
- **First-run flow auto-recovers users stranded by the C-3 brick.** beta.12.2 had a bug where clicking "Skip setup" on the detection spinner set `general.has_completed_onboarding=true` with no provider configured, leaving the user in a no-model Chat mode permanently. The C-3 fix in beta.13 prevented future bricking but didn't recover existing victims (Martin was one). `useAppSettings` now detects the stranded state on launch (flag is set, but no `default_model`, no `chat_model`, no `openai_api_key`), clears the flag and re-mounts FirstRunFlow.

### Verified before publish

Every published artifact's `node_sqlite3.node` binding inspected with `file`:

| Artifact | Expected | Got |
|---|---|---|
| Mac x64 `.dmg` / `.zip` | Mach-O x86_64 | (verify pending CI) |
| Mac arm64 `.dmg` / `.zip` | Mach-O arm64 | (verify pending CI) |
| Linux x64 `.deb` / `.AppImage` | ELF x86-64 | (verify pending CI) |
| Linux arm64 `.deb` / `.AppImage` | ELF aarch64 | (verify pending CI) |
| Windows `.exe` / portable `.zip` | PE32+ x86-64 | (verify pending CI) |

Build verification pass blocks publication. If any binding mismatches, the release stays drafted.

### Verified before publish

Every published artifact's `node_sqlite3.node` binding inspected with `file`:

| Artifact | Expected | Got |
|---|---|---|
| Mac x64 `.dmg` / `.zip` | Mach-O x86_64 | (verify pending CI) |
| Mac arm64 `.dmg` / `.zip` | Mach-O arm64 | (verify pending CI) |
| Linux x64 `.deb` / `.AppImage` | ELF x86-64 | (verify pending CI) |
| Linux arm64 `.deb` / `.AppImage` | ELF aarch64 | (verify pending CI) |
| Windows `.exe` / portable `.zip` | PE32+ x86-64 | (verify pending CI) |

Build verification pass blocks publication. If any binding mismatches, the release stays drafted.

### Shipped in v1.0.0-beta.13 (carried forward; never reached production)


Everything in beta.13 is in beta.13.1. See the beta.13 entry below for the full feature/fix list (notarize restored, taskbar AppUserModelId fix, update-check semver fix, AUTOUPDATE-RESILIENCE marker-file fallback, UX onboarding criticals incl. C-3 first-run brick, ProUpgradePrompt default).

---

## [v1.0.0-beta.13] - 2026-05-04 (DRAFTED, replaced by beta.13.1)

> ⚠️ This release was drafted (un-published) within an hour of going live due to cross-arch native module bug. See beta.13.1 for the actual shipped version. The feature changes below all carry forward.

Notarization restored, parked taskbar/update-check fixes shipped, plus a UX audit's worth of first-run polish before the public beta announcement.

### Fixed: macOS

- **Notarization re-enabled.** `afterSign` hook + `mac.notarize: true` restored after Apple's notary queue stall ([electron/notarize#179](https://github.com/electron/notarize/issues/179)) cleared. Gatekeeper opens the app on first launch. No more "Open Anyway" detour.

### Fixed: Windows

- **Taskbar icon for the running app.** `app.setAppUserModelId('com.bodegaone.app')` was registering the wrong identifier; electron-builder packages under `com.bodegaone.ai`. Updated to match. Desktop and Start Menu icons were already correct from beta.12.2.

### Fixed: Auto-update

- **"Check for Updates" only fires when there's actually a newer version.** Replaced the buggy `!!result?.updateInfo?.version` check (true any time any release existed) with `semver.gt(latest, current)`. No more false "Update available!" prompts on the latest build.
- **AUTOUPDATE-RESILIENCE marker-file fallback** (promoted from v1.0.1). The main-process air-gap probe falls back to `<userData>/air-gap.lock` when the backend HTTP probe fails. The backend writes/removes that file whenever `general.air_gap` changes (`SettingsService.syncAirGapMarker`). Air-gap users with a broken backend stay protected; everyone else continues receiving updates.

### Fixed: First-run / onboarding (UX audit findings)

- **Skip during provider detection no longer bricks the next launch.** Previously, clicking "Skip setup" on the detection spinner marked onboarding complete; on re-launch the user landed in Chat mode with no provider configured. Skip on the spinner now routes to the no-provider screen without marking onboarding done. The user gets to actually pick a path.
- **Skip button reveal aligned with detection wall-clock.** `DETECTING_SKIP_DELAY_MS` raised from 6 s to 8 s so the option appears at the same moment detection would have given up anyway, instead of disappearing into nothing.
- **`EmailCaptureScreen` has proper modal semantics** (`role="dialog"`, `aria-modal`, `aria-labelledby`). Validation errors announce via `role="alert"`.
- **Detection skip button is keyboard-discoverable** (`focus-visible:ring`, `aria-label`).
- **Install-link buttons fall back to `window.open`** when the Electron preload's `openExternal` is unavailable.

### Fixed: Other

- **`ProUpgradePrompt` says "Beta plan" to beta users hitting a pro gate.** Default selector tier flipped from `'personal'` to `'beta'` to match the LicenseService default flip in beta.12.

### Tested

- 461/461 frontend tests pass.
- 3,724/3,724 backend tests pass (52 skipped: live-Ollama gated).
- macOS notarization confirmed healthy via `v1.0.0-beta.13-canary-notary` before the real tag was cut.

---

## [v1.0.0-beta.12.2] - 2026-05-04

Stability hotfixes for the beta.11 ship cycle. Auto-update from beta.11/beta.12 → beta.12.2 works on Windows and Linux. macOS users on beta.11 should download the new .dmg manually; auto-update returns once notarization is restored in beta.13.

### Fixed

- **License activation works on first launch**: `licenseService.initialize(db)` was missing from the backend boot sequence, so every email-capture submit since beta.1 was silently returning HTTP 422. Fresh installs no longer get stuck at the activation screen ([#231](https://github.com/BodegaoneAI/Bodegaone/pull/231)).
- **Windows taskbar and Start Menu icons render at the correct size**: the .ico file's index header advertised every embedded frame as 256×256, so Windows down-scaled a 256-pixel image into the 32-pixel taskbar slot. The icon now embeds 7 correctly-sized frames ([#231](https://github.com/BodegaoneAI/Bodegaone/pull/231)).
- **Database, telemetry, and settings persist on Mac and Linux**: relative `bodega.db` path was resolving inside the read-only app bundle on macOS and AppImage. Files now write to the standard per-user data directory via `DB_PATH` and `BODEGA_USER_DATA_DIR` ([#228](https://github.com/BodegaoneAI/Bodegaone/pull/228)).
- **Backend no longer requires a system Node install**: packaged backend now spawns the Electron binary in Node mode (`ELECTRON_RUN_AS_NODE=1`) instead of shelling out to `node` ([#225](https://github.com/BodegaoneAI/Bodegaone/pull/225)).
- **Auto-updater and Git IPC fail-open when the backend is unreachable**: previous fail-closed behavior trapped users in broken-backend states. Air-gap users still get full protection via the primary HTTP path; v1.0.1 will add a marker-file disk fallback ([#228](https://github.com/BodegaoneAI/Bodegaone/pull/228)).

### Changed

- macOS smoke workflow triggers tightened to mac-relevant code paths only, was 10× billing driver.
- E2E timeouts bumped (smoke 20→35 min, critical 30→45 min). Chronic cancels were masking real regressions.

### Known issues

- macOS builds are signed but **not notarized**. Apple's notary queue is in a multi-day stall ([electron/notarize#179](https://github.com/electron/notarize/issues/179)). Notarization returns in beta.13 once the queue recovers.
- "Check for Updates" menu item shows a false "Update available!" prompt, cosmetic only, no broken update fires. Fix parked in PR #233 for beta.13.
- Windows taskbar icon uses the generic Electron icon (AppUserModelId mismatch). Desktop and Start Menu icons are correct. Fix parked in PR #233 for beta.13.

---

## [v1.0.0-beta.11] - 2026-05-04

First Mac-signed beta build. Sets up the public release mirror at `BodegaoneAI/bodegaone-releases`, certificates configured, three-OS matrix shipping artifacts.

### Added

- **macOS code signing**: Developer ID Application certificate (Joseph Pelaez, team `82AMG3HD28`) configured in CI. Mac `.dmg` and `.zip` artifacts for x64 and arm64 are now signed.
- **install.ps1 Windows installer script**: `bodegaone.ai/install.ps1` redirects to the latest published script.
- **Public release mirror published**: all three-OS artifacts now land on the public `BodegaoneAI/bodegaone-releases` GitHub repo. Auto-update reads from this mirror.
- **Apple Silicon + Intel Mac downloads on the website**: split download buttons live on `bodegaone.ai/download`.

### Known issues at this version (resolved in beta.12+)

- Backend silently fails to start on first launch (system Node required), fixed in beta.12.
- License activation POST returns HTTP 422, fixed in beta.12.2.
- DB path resolves inside read-only bundle on Mac/Linux, fixed in beta.12.1.
- macOS notarization unavailable due to Apple notary queue stall, deferred to beta.13.

### Notes

Six release iterations across `.11 → .12 → .12.1 → .12.2`, with `.11` and `.12.1` deleted from the public mirror. Full timeline and root causes in [POSTMORTEM-2026-05-04-BETA-11-12.md](source-of-truth/POSTMORTEM-2026-05-04-BETA-11-12.md).

---

## [v1.0.0-beta.10] - 2026-04-28

> First public-facing beta. Everything since v1.0.0-beta.8 (2026-03-30): four
> weeks of work covering the licensing/beta-gate flow, signed release pipeline,
> public release mirror, encryption at rest, demo recording pipeline, provider
> matrix E2E, FIM/embeddings hardening, and ~95 PRs of bug-hunt + UX polish.
>
> **Versioning note:** v1.0.0-beta.9 was skipped. Internal RC cut on
> 2026-04-20 (PR #126), never tagged publicly. v1.0.0-beta.10 is the next
> public tag. ~422 commits / ~95 PRs since v1.0.0-beta.8.

### Added: Licensing & beta gate

- **Email-capture beta activation** (#184, #188): first-launch screen captures
  an email, fires fire-and-forget POST to Loops with cohort source
  (`beta-may-2026` or `beta-may-2026-influencer`). Local activation succeeds
  even when Loops is unreachable. Air-gap mode skips the Loops call entirely.
- **LemonSqueezy licensing infrastructure**: feature gates, lock screen UI,
  online verification, grace period, dev-mode bypass. Wired for the post-beta
  paid flow; sits dormant during the May beta which uses email capture only.
- **Beta expiry hard-baked into the binary** (#197): `BETA_EXPIRY_ISO` injected
  at build time. Public binaries expire 2026-06-01; influencer/press binaries
  expire 2026-07-07. Tag-suffix (`-influencer`, `-press`) drives cohort
  selection during the `electron-builder` run.
- **Dedicated `BetaExpiredScreen`**: terminal screen when the build's hard
  expiry passes. Surfaces the user's machine ID with copy-to-clipboard,
  support email, and a download CTA pointed at the actual download page.
  ARIA `dialog` + labelled heading.

### Added: Release pipeline

- **Signed release pipeline** (#191): SignPath wiring for Windows EV cert
  (gated on secrets, currently no-op until cert lands), Apple Developer ID
  Application + Installer wiring for macOS notarization. `release.yml` matrix
  consumes both.
- **Portable Windows zip + PowerShell installer** (#192): `irm bodegaone.ai/install.ps1 | iex`
  one-liner. SHA512 verification against `latest.yml`. `Unblock-File`
  post-extract bypasses SmartScreen on unsigned beta builds.
- **Public release mirror** (#199, `BodegaoneAI/bodegaone-releases`): binary
  distribution and auto-update manifests live in a separate public repo so
  the source repo stays private. `release.yml` publishes via a cross-repo
  PAT (`RELEASES_REPO_TOKEN`). `electron-updater` reads from the public repo
  automatically.
- **`*-latest.*` artifact naming** (#199): stable filenames across releases
  so Vercel rewrites on `bodegaone.ai/download/*` resolve without per-tag
  config churn. Version info flows through `latest.yml` for the auto-updater.
- **macOS smoke build** (#181): CI job that produces a notarized DMG on
  every dev push to catch packaging regressions before tag.

### Added: Encryption at rest (HIGH-03)

- **`SecretCipher` service**: AES-256-GCM with a random 32-byte key file at
  `<dbDir>/.bodega-cipher-key` (mode 0600 on POSIX). License keys, license
  instance IDs, and `*.api_key` settings are now encrypted before they hit
  SQLite. One-shot migration on first boot post-fix re-encrypts any legacy
  plaintext rows. Threat model + safeStorage trade-off documented in the
  source.

### Added: Phase 14 game plan (#147)

- **User-Definable Agents** spec: multi-agent orchestration roadmap drafted
  for post-GA. Implementation deferred to v1.1.

### Added: Demo recording pipeline (#130, #132–#146)

- **Scriptable end-to-end demo capture**: Playwright drives the Electron
  app while ffmpeg (or Playwright's video recorder) captures the window.
  Auto-build, real backend probes, free-port handling, isolated user-data
  dir, force-hide the no-provider card and What's New modal, fixed
  1920×1080 window, EPIPE guards. Used to produce launch demo video.

### Added: Other

- **Per-project config** (#62): `.bodega/config.json` for project-specific
  settings (whitelisted keys only).
- **`@codebase` and `@memory` context mentions** (#61): inline references
  in chat compose.
- **Context window indicator** (#60): token usage display after each response.
- **Hunk-level staging**: stage individual hunks in the Git panel.
- **Skills + MCP scaffolding** (#119, #123): SkillRegistry singleton,
  SkillMatcher tests, MCP coverage tests.

### Fixed: Security (this release's swarm + ultrareview cycles)

- **CRIT-01 settings redaction round-trip**: PR #194 redacted `*.api_key`
  values in `GET /api/settings`, but the renderer's "save all" round-trip
  could overwrite stored keys with the redacted placeholder. Hardened with a
  centralized `REDACTED_SECRET_SENTINEL` + PUT-side guard that detects the
  sentinel and skips the write. Test-connection endpoints (`/llm/test-connection`,
  `/boost/test-connection`, `/code/fim-test-connection`) substitute the
  stored key when probing so users can hit "Test" without re-pasting. Pattern
  coverage broadened from `*.api_key` only to also include `*.api_token`,
  `*.access_token`, `*.password`, `*.secret`.
- **HIGH-01 license API bypassed air-gap**: `LicenseService.activate / deactivate / verifyOnline`
  called LemonSqueezy without a `general.air_gap` check, an 11th outbound
  network path past the documented 10 air-gap layers. Gated `lsRequest` at
  the chokepoint with a typed `AirGapBlockedError`. `verifyOnline` catches
  it (background check, grace period absorbs); user-initiated `activate` /
  `deactivate` propagate it to a 403 response.
- **HIGH-03 license + API keys plaintext at rest**: see SecretCipher above.
- **Critical security fixes from Sentinel + Architect review**: multiple
  vulnerability classes addressed before beta cut: SSRF tightening,
  ShellTool credential scrubbing, FileSystemTool sandbox hardening
  (symlink resolution, `..`/`~` pre-check, blocked-extension write guard).
- **Quality audit pass** (#59): 15 security/quality fixes documented across
  6 audit reports.

### Fixed: UX & onboarding

- **EmailCaptureScreen + LicenseLockScreen race**: both screens silently
  no-op'd when `machineId` hadn't hydrated by the time the user pressed
  Enter. Now distinguish three button states (Activating / Preparing /
  Ready) and surface explicit messages instead of silent disable.
- **AI-slop copy**: replaced "Something went wrong" / "encountered an
  unexpected error" in `ErrorBoundary` and `ChatErrorBanner` with concrete,
  actionable copy. Added air-gap-aware error mapping in `formatErrorMessage`.
- **ARIA + clickable links on `LicenseLockScreen`**: added `role="dialog"`,
  `aria-modal`, labelled heading, autofocus + autocomplete-off + spellcheck-off
  input, real `<a>` for "bodegaone.ai" instead of plain text.
- **Air-gap onboarding banner**: when air-gap is on during first-launch,
  the model picker now shows a banner above the grid explaining why
  Download is disabled and pointing at Settings. Was a silent hover-only
  title before.
- **Fresh-install empty states** (#165): ModelSelector and EditorWelcome
  no longer show stale data on first launch.
- **Ctrl-shortcuts firing from inputs + About section gating** (#170):
  shortcuts now work when an input is focused; About section no longer
  hidden behind advanced.
- **Two shipping bugs from Joe's live session logs** (#109).
- **What's New modal fetch bug** (#110).

### Fixed: FIM, embeddings, prompt

- **FIM auto-heal on incompatible model** (#193): `FIMService.resolveModel`
  drops stale `llm.fim_model` values that don't match the current provider
  family, falling through to `llm.code_model` → `llm.default_model`. Stops
  the "FIM silently disabled after switching providers" failure mode.
- **Embeddings integrity findings** (#185): 7 ultrareview findings around
  concurrent index writes, mixed dimensions, and realpath resolution.
- **Embeddings UX wiring** (#190): 4 ultrareview UX findings.
- **Prompt correctness** (#187): 3 ultrareview findings on family hint
  emission and tool-format detection.
- **3 follow-up ultrareview findings** (#194): install.ps1 SHA512
  verification, PUT-side guard rounding, additional polish.

### Fixed: Editor + git

- **Editor tab + rename selectors match actual DOM** (#163, #169, #172):
  multi-tab tests now use dblclick to bypass preview tabs.
- **Commit-message selector matches input** (#174): product uses `<input>`,
  not the assumed selector.
- **Non-git workspace detection** (#173): return `isRepo: false` on
  `rev-parse` error instead of crashing the git panel.

### Fixed: Pre-beta hardening sweep (#149)

- **155 new tests + live cloud/local verification**. Closed three shipping
  bugs uncovered while writing the tests.

### Fixed: Pipeline & install

- **NSIS installer build**: uninstaller hook bug (`customUnWelcomePage` /
  `customUnInstFilesPage` aren't actually exposed by `electron-builder`,
  defined functions tripped a "not referenced" warning that NSIS escalates
  to error). Removed dead code; `bodegaGuiInit` class brush still keeps the
  uninstaller frame dark.
- **`package.json:homepage` leaked old personal handle**: pointed at
  `https://github.com/Mayimbe07/Bodegaone`, baked into uninstaller "About /
  Help / Updates" metadata. Flipped to `https://bodegaone.ai` and
  `repository.url` to the public mirror.
- **Portable zip filename mismatch**: `portable` artifactName was set but
  `portable` wasn't in `win.target`; the active `zip` target produced a
  spaces-in-name version-suffixed file that 404'd against
  `bodegaone.ai/download/portable`. Added a top-level `zip` block override.

### Fixed: Earlier rolling fixes (since beta.8)

- **Research panel crash**: removed file tools from research panel; added
  45s timeout; capped iterations at 3.
- **Abort race condition**: per-session stream lock in `chat-stream.ts`.
- **Model switch context loss**: explicit `modelName` param in
  `ContextAssemblyService` (dynamic resolution was using the wrong model).
- **BoostToggle never wired**: added to both Chat mode (ChatStageInput) and
  Code mode (ChatInput) input bars.
- **Smooth scroll jitter on fast GPUs**: 300ms throttle (5090 at 500+ tok/s
  was causing flicker).
- **Session archive API**: accepts both `archived` and `is_archived` field
  names.
- **Boost health check fallback** via `/v1/messages` + diagnostic logging.
- **Portal rendering**: search dropdown and settings tooltips escape
  `overflow-hidden` containers.
- **Electron sandbox crash**: `--no-sandbox` for Linux CI.
- **Windows taskbar icon**: shows Bodega logo instead of default Electron
  icon.
- **19 Playwright selector mismatches** corrected (#70).
- **Settings cleanup** (#66): removed duplicates, fixed Anthropic boost,
  wired LSP toggle.
- **Bug hunt round** (#67): boost health recovery, agent init, webpack
  warnings.
- **Stale help section data** (#68): tool count 23→25, preset count 15→16,
  added Anthropic to the cloud list.
- **Reconfigure picks API key per provider type**: Anthropic / Azure /
  Gemini / OpenAI now select the correct stored key based on `llm.preset`.

### Changed

- **Model lineup** (#189): refreshed for May beta, dropped legacy entries.
  Most-recent generation only (Qwen3, Gemma 4, Llama 3.3+, Claude 4.7).
- **Renderer perf: PrismLight syntax-highlighter** (#182): switched from
  full PrismJS to PrismLight with explicitly-registered languages. Reduces
  bundle size on chats with code blocks.
- **CI noise** (#180, #195, #196): disabled scheduled + auto-on-push runs
  burning Actions minutes; removed `workflow_dispatch` from `release.yml`;
  renamed `provider-matrix.live` test gates so local runs stop emitting
  4-failure noise.
- **E2E benchmark split into fast and slow lanes** (#159, #160): `@slow`
  tag drives lane selection. Fast lane runs without LLM-heavy specs.
- **Per-test timeout** (#158): bumped from 180s to 360s for LLM tests.
- **Crash reporting: Sentry Electron SDK integration** (Phase 4.14).

### Tests: Provider matrix E2E (#171, #175, #176, #177, #178, #166, #167)

- **Provider conformance matrix**: fixture parametrization + 40 starter
  tests per spec covering Ollama, llama.cpp, LM Studio, vLLM, OpenAI,
  Anthropic, Gemini, OpenRouter, Together, Groq, Azure, Mistral, Cohere,
  DeepSeek, Fireworks. Overnight-safe: cloud lanes split by provider so
  one provider's outage doesn't fail the rest. Local runner targets a
  user's actual Ollama install. Auto-fires on dev push. Incremental JSON
  reporter so a partial run is still readable.

### Tests: Days-1-9 launch sprint (#110–#125)

- Day 1: prompt-audit blockers + What's New fetch bug + UX harness (#110).
- Day 2: editor-widgets UX spec: bottom panel + inline command bar +
  inline fix (#113); prompt audit STRONG carryovers + editor-tab UX spec
  (#112); 18-test contract for `PromptTemplateService` + stale count fix
  (#111).
- Day 3: chat input UX spec (#115), chat context surface spec (#117),
  data-testid hooks for chat UX (#116).
- Day 4: agentic-panel empty-state UX spec (#118).
- Days 5–9: provider matrix + embeddings live + settings nav + MCP
  coverage (#119), Day 6 UI + SkillMatcher + queue (#123), remaining
  skipped items: approval / resize / attachments / live (#125).

### Tests: Stabilization & infrastructure

- **Onboarding fixture hardening** (#161): graceful close + tree-kill on
  Windows.
- **Ollama unload between tests** (#157): stops RAM creep across long runs.
- **Watchdog script ASCII-only** (#156): em-dashes were breaking PowerShell
  5.1 parser.
- **Tree-kill on Windows teardown** (#152, #154): only when graceful close
  times out.
- **Benchmark hardening against resource exhaustion** (#155).
- **3s timeout on Ollama unload** (#162): fixture teardown no longer hangs.
- **`__ZUSTAND_STORE__` always exposed for tests** (#168): webpack inlines
  `NODE_ENV`, so the dev-only condition was being optimized out.

### Tests: This-release smoke coverage (HIGH-03 follow-on)

- **`SecretCipher.test.ts`** (19 tests): round-trip, fresh-IV uniqueness,
  sentinel detection, tampered-ciphertext rejection, legacy plaintext
  passthrough, cross-key isolation, key file persistence.
- **`model-hub.test.ts`** (12 tests): air-gap 403, validation
  (missing/wrong-type/shell-injection/oversize/accepted charset),
  concurrency (409 dup, 200 fresh), cancel (200/404/injection-safe), GET
  catalog shape.
- **`autoUpdater.test.ts`** (5 tests): handler registration, dev-mode
  short-circuit before any network call, fail-closed when backend
  unreachable.

### Documentation

- **Comprehensive user-facing troubleshooting guide**
  (`source-of-truth/WEBSITE-TROUBLESHOOTING-2026-04-28.md`): install /
  first-launch / providers / models / Cloud Boost / BYOK / chat + code
  mode / FIM / air-gap / sandbox / auto-update / settings + logs +
  factory reset / beta expiry / telemetry. ~50 distinct failure modes
  documented.
- **Public-repo `TROUBLESHOOTING.md`** in `BodegaoneAI/bodegaone-releases`:
  short, technical, dev-friendly. SmartScreen / install.ps1 / AV
  quarantine / Mac quarantine / AppImage chmod / FUSE2 / sandbox helper /
  auto-update.
- **Beta-readiness specs** (#183): model lineup spec + email-capture
  activation spec + provider-matrix testing spec.
- **Provider matrix spec** (#167). Joe approved with 4 decisions.
- **RELEASE-SIGNING.md**: full SignPath + Apple Dev cert wiring guide.
- **Phase 14 game plan**: User-Definable Agents architecture.
- **SOT cleanup**: archived 5 stale docs (`JoeTracker-SessionLog-3`,
  `SESSION-2026-04-20`, `RC-NOTES-1.0.0-beta.10-2026-04-20`,
  `PRETAG-TEST-RESULTS-2026-04-27`, `PLAYWRIGHT-RESULTS-2026-04-21`,
  `MARTIN-LAUNCH-HANDOFF-2026-04-27`); refreshed CLAUDE.md numbers via
  Doc Guardian agent.

### Tests: totals

- Backend: **168 files, 3,290 tests** (was 134/2,673 at beta.8, +34 files /
  +617 tests, +23%)
- Frontend: **29 files, 416 tests** (was 21/328 at beta.8, +8 files /
  +88 tests, +27%)
- E2E: 62 mocked integration suites + 686 benchmark tests (live Ollama)
- QEL: 180-test quality gate suite
- Chat stress: 125-test routing/behavioral suite

---

## [v1.0.0-beta.8] - 2026-03-30

> Agentic loop reliability, CLAUDE.md optimization, E2E Round 2 bug sweep,
> lifecycle hooks, and the Runtime Layer architecture.

### Fixed

**E2E Round 2 bug fixes (BUG-R1 through R9)**

- **BUG-R1** Plan approval card: added `min-height` so plan text is always visible; fallback text when plan is empty
- **BUG-R2** "Reached iteration limit" message now only appears on `ITERATION_CAP` phase, not on clean task completion (`COMPLETE`) or circuit-breaker abort (`FAILED`)
- **BUG-R4** `InlineFixWidget` portal now hides when ANY secondary panel is open (settings, usage, etc.), not just settings
- **BUG-R5** New files created by the agent auto-open in the editor as preview tabs via `FileChangeTracker.confirmFileChange`
- **BUG-R6** `ContextInspector` accepts `modelOverride` prop; code-mode panels (BodegaPanel, PanelSidebar) pass `llm.code_model` so the inspector shows the actual code model instead of chat's routed model
- **BUG-R7** Panel session IDs clear when the project path changes: prevents git/debug panels from inheriting context from the previous project
- **BUG-R8** Research progress display stays visible after completion when `sourcesFound === 0` so users know why there's no web context
- **BUG-R9** `todo_write` now available to small models (≤30B): was missing from `SMALL_MODEL_TOOLS` filter, causing the Todo panel to never appear during E2E testing with qwen3.5:9b; also added `todo_write` to tool selection guides
- **Test-11** Repeat-write guard: after 3 writes to the same file path in one agentic loop, injects a `[DELIVERABLE SATISFIED]` system message and sets `pendingNudge` to break the rewrite cycle
- **Approval cards** `PlanApprovalWrapper` moved inside `ChatMessageArea`'s scroll container so auto-scroll carries it to the bottom as streaming content arrives

### Added

**Docs**

- `source-of-truth/E2E-TESTING-GUIDE.md`: 35-test manual testing protocol with exact prompts, pass/fail criteria, and regression markers for all 9 fixed bugs
- `source-of-truth/marketing/discord-posts.md`: building-in-public post archive for Discord announcements
- `CLAUDE.md §14 Token & Cost Optimization`: model selection, effort level, prompt cache hygiene, peak-hour throttling guidance for the agent team

**Runtime Layer: Chat→Runtime→Loop→QEL architecture (Phases 1-3)**

- `RuntimeLayer.ts` (369L): new execution authority between the chat route and the agentic loop
- `LoopPolicy` typed return from `classify()` replaces ~150 lines of inline pre-loop classification in `AgenticChatService`
- `ExecutionLane` enum: `full | restricted | guided | advisory`, expressively describes what the loop is allowed to do
- Advisory lane bypass: single LLM call with zero loop overhead for conversational requests
- Planning gate gated on `policy.allowPlanning`. Guided/restricted lanes skip it
- `CapabilityProfile` type: semantic tiers (`strong/medium/weak/none`) over raw `ModelCapabilities` booleans
- `deriveCapabilityProfile()`: maps model flags to expressive tier
- `createPolicy()` for worker agent dispatch (halved budget, downgraded lane)
- Session-scoped failure tracker: 3 consecutive tool errors → automatic guided lane downgrade
- `trackToolFailure / trackToolSuccess / getSessionLaneOverride / clearSession` APIs
- No-progress abort replaces temperature bumping for non-full lanes

**Anthropic direct provider**: `anthropic` added as a Cloud Boost preset (`api.anthropic.com/v1`). Users with an Anthropic API key can now configure Claude directly as the boost provider without going through OpenRouter.

### Fixed

**Agent loop root-cause fixes**

- **Compaction crash**. Conversation compaction failed when the context window was already at capacity, killing the agent mid-run. Compaction now detects this state and prunes older turns before calling the LLM.
- **Iteration bloat**. Tool-use iterations were not being counted toward the max-iterations limit in certain code paths, allowing runaway loops on stubborn tasks.
- **Mode leak**. Permission mode (ask/plan/act) was occasionally bleeding across sessions when a session was reloaded. The mode is now reset to the persisted setting on session load.
- **Advisor write**. The Advisor panel was attempting file writes using the agent tool set instead of the read-only tool set, bypassing the intended restriction.
- **Web search returning 0 results**. DuckDuckGo redirect URL filter was stripping the search result URLs along with ad redirects. Filter now preserves result links.

**E2E bug fixes (March 29 testing session)**

- **Compact Now button**. The Context Inspector "Compact Now" button dispatched an `input` event but never dispatched the `Enter` keydown, so the chat input submit handler never fired. Now correctly dispatches both events.
- **Plan approval scroll**. Plan approval cards appearing mid-session did not scroll into view in the agent panel. Added auto-scroll effect when a plan approval becomes pending.
- **Agent panel empty state**. Advisor/debug/research panel empty state showed generic "Type a message to start coding" text regardless of panel type. Now shows panel-appropriate guidance.
- **InlineFixWidget z-index**. The inline fix widget (portal at z-200) remained visible on top of the settings overlay. Widget now hides when the settings panel is open.
- **False iteration limit message**. The "Reached the iteration limit" footer was appended to responses even when the loop exited via a normal COMPLETE transition (e.g., no-progress abort after a text-only response). Now suppressed when `loopSM.getPhase() === COMPLETE`.
- **Misleading error recovery text**. The non-serializable message history error told users to "start a new session" which would lose all context. Changed to "Reload the page and try again."
- **Web search iteration starvation**. Simple tasks were capped at 4 guided-lane iterations, which is insufficient for web search (requires search → read results → refine → synthesize → respond). Cap raised to 8 when `effectiveWebSearch` is true.
- **Temperature bump on panel text responses**. Non-full lanes (advisor/debug/research) called `recordNoToolIteration()` in the no-tool branch, which bumped temperature for stuck-loop recovery despite the comment saying it shouldn't. Added `recordNoToolIterationCount()` to `LoopStateMachine` for count-only tracking; non-full lanes now use it.
- **VRAM warning noise**. `ingestOllamaMetadata()` unconditionally invalidated the recommended settings cache on every 30-second health poll, triggering VRAM recomputation for all 17 models even when nothing changed. Cache now only invalidated when `contextLength`, `parameterCount`, `quantizationLevel`, or `capabilities` actually differ.

**UX fixes**

- **Loop recovery**. The "Retry" button in the error banner did not restart the agent loop; it only re-sent the last user message without agent context. Now correctly resumes from the interrupted state.
- **TodoPanel visibility**. The Todo panel in Code mode was hidden when the file tree was closed, making it inaccessible without opening the sidebar first.
- **Ctrl+I shortcut**. The Context tab keyboard shortcut (Ctrl+I) was opening the Bodega panel but not switching to the Context tab. Now correctly routes to Context.
- **Bulk delete**. Selecting all sessions in the Chat sidebar and pressing Delete closed the dialog instead of confirming deletion. Confirm button now has correct focus on open.
- **Terminal focus tracking**: `terminal.textarea` is now used for xterm focus tracking instead of the deprecated `.terminal` selector, fixing focus lost after window blur/focus on some systems.
- **ESLint cleanup**. 8 ESLint warnings that were blocking CI lint job resolved.

**Settings and polish**

- **Settings/review bar conflict**. Opening Settings while a diff review was active in the editor left the DiffReviewPanel visible instead of showing Settings. Opening any secondary panel (Settings, Usage, Agents, etc.) now atomically exits review mode.
- **Orphaned settings removed**: `verification.debate_mode` (VerificationDebate is not wired) and `llm.image_generation_model` (no image generation feature) removed from settings registry. The `agents.*` namespace fallbacks and stale keyword references cleaned up.

### Tests

- 2,127 backend tests (96 files), 326 frontend tests (20 files)

---

## [1.0.0-beta.7] - 2026-03-24

### Added

**Follow-up Message Queue**: send messages while the agent is working

You can now type and queue follow-up messages while the agent is actively running. Queued messages inject between tool execution cycles, so you can course-correct mid-run without restarting from scratch.

- Input stays open during agent runs, type whenever
- A badge shows how many messages are pending ("2 queued")
- Queued messages appear below the active stream with a spinner and a per-message cancel button
- Stop button cancels both the current stream and the full queue
- Classification: `stop/cancel/abort/nevermind` → abort immediately; `actually/instead/correction` → inject as correction; everything else → append as context

---

### Fixed

- **Ghost text in panels**. The inline completion in Agent, Research, Debug, and Advisor panels was returning the model's previous conversational reply as ghost text instead of a real code suggestion.

- **Context overflow in long conversations**. Sessions with 380+ messages could fill the entire context window with history, leaving no room for the model's response. The trimmer now reserves 20% of the window for output and retains the last 5 user turns (was 2). A warning appears in the UI when a session exceeds 100 messages.

- **qwen3:14b capped at 10K context**. A double-counting bug in the VRAM formula was computing available context from free VRAM (post-load) rather than total VRAM. A 12 GB GPU with a 7.7 GB model loaded appeared to have only 4 GB free, capping context at ~10K instead of 32K+. Fixed to use total VRAM as the baseline.

- **Research "how to" queries never generating guide variants**. Guide-type variant detection ("what is X", "how to X") was checking extracted keywords instead of the original question. Those prefixes had already been stripped, so guide variants never triggered.

---

### Improved

- **Tool calls grouped in panels**. Repeated calls of the same type show as one row ("Web Search: 3 queries") instead of three separate rows. Calls pending approval remain individual.
- **Streaming status bar**. Unified phase indicator shows what the agent is doing: thinking, web search, file operations, shell commands.
- **Message pagination**. Large sessions load incrementally (50 messages per page, max 200). Faster initial load for long threads.
- Editor file size limit raised to 20 MB (was 10 MB), with a soft warning at 5 MB.
- Model switch failures surface a toast: "Model switch failed. Still using previous model."
- Archive and bulk delete in Code mode now match Chat mode behavior.

---

### Tests

- 35 memory compression tests (HeuristicExtractor, MemoryConsolidator, standalone spike, not yet wired)
- 5 missing SSE event type tests: think_token, token_metrics, clarification_needed, clarification_resolved, boost_cost
- 5 regression tests for this session's visual fixes

**Final counts: 1,884 backend tests (84 files) / 326 frontend tests (20 files)**

---

### Internal

Code health work this session (not user-visible):

- AgentChatPanel split: `useAgentPanelModel.ts` + `useAgentPanelPermissions.ts` extracted
- `db/initialize.ts` compressed 522 → 59 lines: `schema.ts` + `migrations.ts` + `indexes.ts` extracted
- `ToolCallDisplay.tsx` split 459 → 72 lines: `IndividualCallRow.tsx` + `ToolGroupRow.tsx` extracted
- 6 more coordinator splits: MessageList, ChatInput, SearchPanel, InlineFixWidget, GitPanel, CodeSidebar
- MessageQueueService: atomic position INSERT replaces SELECT MAX + INSERT (concurrent enqueue safety)
- `source-of-truth/` unified at repo root. `master-agent/source-of-truth/` removed
- Design tokens: `--color-status-info` family added; `stream-border-pulse` animation added
- `window as any` → typed `Window & { electronAPI?: unknown }` interface
- `@noble/hashes` package-lock conflict resolved (was breaking `npm ci` in CI)
- SMART-AUTO-SCOPE.md: scope doc for per-iteration model routing in code mode (next feature)

---

## [1.0.0-beta.6] - 2026-03-23

Bug sweep and repo cleanup following beta.5.

### Fixed

**30 bugs fixed across the app:**

**Code Editor + FIM (9 fixes):** git blame annotations broken on annotated tags; memory leak in editor mount/unmount cycle; null ref crash on post-unmount editor access; FIM skipped when cursor at EOF; LSP crash recovery missing restart delay; wrong abort timer cleared on FIM cancel; Windows path separator mismatch in file open; stale merge conflict debounce firing after resolution; telemetry queue not flushing on shutdown. 5 Monaco TypeScript errors fixed.

**Terminal + Problems Panel (7 fixes):** PowerShell `-e` flag blocking grep-based search; terminal AI assist missing `\r` causing collapsed output lines; resize observer firing without debounce (layout thrash); Problems panel path duplication on project reload; sort instability in diagnostics list; dead code in terminal color parser; overly broad npm warning regex suppressing real errors.

**Chat + Agent Loop (6 fixes):** session delete missing transaction (partial deletes); attachment query using wrong column reference; illegal message ordering in agent loop (assistant before user); circuit breaker dead code branch; read cache returning stale content after writes; orphaned pending approvals not cleaned up on session close.

**Settings + UI (6 fixes):** orphaned project memories not deleted when project removed; wrong memory deletion API called; double-quoted values in context key injection; sync file writes blocking the main thread; no keybinding conflict detection on custom bind save; negative entry count shown when memories pruned.

**Agent Chat Isolation (1 critical fix):** agent sub-chat sessions were overwriting the IDE's main session ID, causing agent responses to leak into the user's primary chat history.

**Other:**
- Model profile fuzzy-match returning wrong family for hyphenated names
- Shell tool auto-read flag ignored when model profiles path contained spaces (Windows)
- Model profiles path resolution broken on non-default install locations

---

### Changed

- Roadmap deferred/speculative items relabeled as "Research Pending" or "Deferred"
- All source-of-truth docs updated to reflect post-fix state

---

### Tests

Backend: 1,824 tests passing (up from 1,818), 80 test files.

---

## [1.0.0-beta.5] - 2026-03-22

334 commits since beta.4. Major themes: onboarding overhaul, inline code completions, QEL compile gates, the 4-panel AI system, a full design token sweep, and security hardening.

### Added

**Onboarding**
- Provider-agnostic first-run wizard targeting a 60-second install-to-first-message experience
- Guided tour overlay with per-element spotlights
- Onboarding checklist that auto-checks items as you complete them

**Inline Completions (FIM)**
- Auto-detection: scans installed models at startup and activates FIM completions with zero config when a capable model is found
- Cache invalidation on model switch
- Thinking-model FIM tuning

**QEL: Quality Enforcement Layer**
- Compile-based proof gates for Go, Rust, Java, and C#: generated code must compile before the task is marked complete
- Per-model pass rate tracking across sessions
- Cloud Boost suggestion after 3 consecutive QEL failures
- Dual-agent verification cross-check for ambiguous outcomes
- Multi-patch sampling when `str_replace` edits fail

**IDE & Code Mode**
- Code-heavy operations automatically route to the code model
- Free VRAM detection (available vs. total, not just total)
- Git blame annotations: `Alt+B` toggles per-line author, date, summary
- Inline character-level diff highlighting within changed lines
- Language mode picker in the status bar
- Elapsed timer during streaming + regenerate button on responses
- `Ctrl+G` go-to-line overlay
- `Ctrl+F` settings search: filters all settings tabs by keyword

**AI Panels**
- 4-tab panel system in Code mode: Agent, Research, Debug, Advisor
- Per-panel session isolation, model selection, and persona injection
- Context handoff between panels mid-session
- GPU concurrency management across panels (priority queue)
- Structured decision audit trail in Advisor panel
- ANSI bright color support in terminal panels
- Stack trace parsing and error detection in Debug panel

**Chat & UI**
- Context compact button at 80%+ context usage
- LaTeX math rendering (KaTeX, lazy-loaded)
- Mermaid diagram rendering in artifact preview
- Persistent web search badge in chat input
- Step counter in the thinking indicator
- Recent projects list on Code mode welcome screen
- Quick-start action suggestions on welcome screen

---

### Changed

**Architecture**
- 9 more components/services split to stay within file size limits (45 total since project start)
- 163 raw SQL queries moved from route handlers to the service layer
- 24 `as any` casts eliminated
- Circular dependency between PersistentMemoryService and SettingsService broken

**Model Catalog**
- Updated with March 2026 models: Qwen3.5, GLM-4.7, Devstral 2
- SWE-bench data added for catalog entries
- VRAM detection unified to report `max` not `sum` across multi-GPU setups
- `qwen3-coder` FIM flag corrected (does not have native FIM tokens)

**QEL & Verification**
- Structural verifier body scan limit raised 500 → 2,000 lines
- Generic stub detection added
- `isIncompleteResponse` false positive rate reduced
- Budget repair tracks score trajectory: converging runs get a bonus attempt
- Proof gate `tsc` adds `--skipLibCheck`

**Settings**
- Repo map budget increased 25% → 30% of project context
- 5-minute wall-clock timeout added to the agentic loop
- GA date updated to July 6, 2026

---

### Fixed

**Critical**
- 19 frontend TypeScript interface gaps (beta.5 CI blocker)
- 6 route handlers missing async error wrapping (crash risk)
- `OnboardingChecklist` false-completion on fresh install

**Onboarding**
- 5 bugs: checklist storage keys, tour trigger, Ollama install path detection, dead wizard state, panel tour entry point

**Inline Completions**
- FIM fence stripping: trailing fence tokens no longer appended to completions
- `qwen3-coder` excluded from FIM auto-detect (false positive)

**Model & Provider**
- Model download stuck on "Verifying": Ollama pull stream completion now handled correctly
- DeepSeek-v3 description corrected

**Security**
- IPv6 SSRF gap patched: non-abbreviated loopback and ULA `fc00::/7` range now blocked
- `rm -rf ./` relative-path form blocked in shell tool
- Git blame command injection via `execSync` → safe `runGit` wrapper
- Air-gap Layer 10 missing fail-closed fallback added
- Memory `user_id` filter missing in two queries
- `feImage` paired-tag bypass in SVG sanitizer patched

**UI**
- 27 critical/high UX audit findings resolved
- Hardcoded colors replaced with design tokens across 15+ components
- Context Inspector blank in Code mode
- Tool approval card auto-scroll on appearance
- Rate limit 429 handling + chat reset on tab switch
- Markdown tables wrap with horizontal scroll on narrow viewports

**Agent Loop**
- Circuit breaker tool-scoped reset + memory failure counter reset
- `FAILED` state correctly emitted when circuit breaker breaks
- 3 critical QEL gaps: `str_replace` verification, tool timeout, path deduplication

---

### Security

- Shell security tests rewritten to cover production code rather than mocks (140 tests)
- WebFetch SSRF hardening: IPv6 loopback and ULA ranges added to blocklist
- StrReplaceTool sandbox escape test suite (23 tests)
- Air-gap enforcement automatically tested for 4 of 10 layers
- Memory cross-user isolation test suite added
- `sqlite3` upgraded to v6.0.1; `copy-webpack-plugin` upgraded to v14; `flatted` prototype pollution resolved; `url-regex` ReDoS patched
- 0 critical/high vulnerabilities in dependency audit

---

## [1.0.0-beta.4] - 2026-03-15

Previous release baseline. Introduced the multi-agent swarm runtime, LSP integration, unified Model Hub, research mode, and the V2 overhaul master spec. Full details in git history: `git log --oneline v1.0.0-beta.3..v1.0.0-beta.4`

---

*Full commit history: `git log --oneline v1.0.0-beta.4..v1.0.0-beta.5`*
